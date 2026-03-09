# Centrifuge-Java API Knowledge Base

**Library**: `centrifuge-java` — WebSocket client for [Centrifugo](https://github.com/centrifugal/centrifugo) server and [Centrifuge](https://github.com/centrifugal/centrifuge) library  
**Latest version at time of writing**: 0.5.0  
**Compatibility**: Centrifugo v6, v5, v4; Centrifuge ≥ 0.25.0. For Centrifugo v2/v3 or Centrifuge < 0.25.0 use v0.1.0.  
**Maven**: `io.github.centrifugal:centrifuge-java`  
**Protocol**: Protobuf over WebSocket (negotiated via `Sec-WebSocket-Protocol: centrifuge-protobuf`)  
**Transport**: OkHttp3  
**Source inventory**: 57 `.java` files (56 public-facing + 1 internal `Backoff.java`)  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Client Lifecycle & State Machine](#2-client-lifecycle--state-machine)
3. [Subscription Lifecycle & State Machine](#3-subscription-lifecycle--state-machine)
4. [Configuration — Options](#4-configuration--options)
5. [Configuration — SubscriptionOptions](#5-configuration--subscriptionoptions)
6. [Event System — Client Events](#6-event-system--client-events)
7. [Event System — Subscription Events](#7-event-system--subscription-events)
8. [Token & Authentication Flow](#8-token--authentication-flow)
9. [Error Taxonomy](#9-error-taxonomy)
10. [Data & Result Types](#10-data--result-types)
11. [Recommended Patterns (from demo/ and example/)](#11-recommended-patterns)
12. [Pitfalls & Breaking-Change History](#12-pitfalls--breaking-change-history)
13. [Threading Model](#13-threading-model)
14. [Android Considerations](#14-android-considerations)
15. [Internal Utilities](#15-internal-utilities)
16. [Complete File Inventory](#16-complete-file-inventory)

---

## 1. Architecture Overview

### High-Level Structure

Centrifuge-Java is a **WebSocket-based real-time protocol client** built on **OkHttp3** for transport and **Protocol Buffers (Lite)** for serialization.

**Core components:**

| Component | File(s) | Role |
|---|---|---|
| `Client` | `Client.java` (~56 KB) | Manages WebSocket connection lifecycle, reconnects, command queue, server-side subscriptions, and top-level publish/history/presence/RPC operations |
| `Subscription` | `Subscription.java` (~19 KB) | Manages channel subscription state, resubscription with backoff, subscription token refresh, delta decoding (Fossil) |
| `ServerSubscription` | `ServerSubscription.java` | Internal holder for server-side subscription state (offset, epoch, recoverable flag) |
| `Options` | `Options.java` (~6 KB) | Client-level configuration: token, tokenGetter, name, version, timeouts, reconnect delays, proxy, DNS, SSL, OkHttpClient |
| `SubscriptionOptions` | `SubscriptionOptions.java` (~2.6 KB) | Subscription-level configuration: token, tokenGetter, positioned, recoverable, joinLeave, delta, since, resubscribe delays |
| `EventListener` | `EventListener.java` | Abstract class with no-op defaults for all client-level events |
| `SubscriptionEventListener` | `SubscriptionEventListener.java` | Abstract class with no-op defaults for all subscription-level events |
| `Fossil` | `Fossil.java` (~6 KB) | Package-private delta compression (Fossil delta algorithm) for applying binary diffs |
| `Backoff` | `internal/backoff/Backoff.java` | Full-jitter exponential backoff for reconnect/resubscribe delays |

### Class Relationship Map

```
Client
├─ manages → Map<String, Subscription>       (subs: client-side subscriptions)
├─ manages → Map<String, ServerSubscription>  (serverSubs: server-side subscriptions)
├─ owns    → ExecutorService executor         (single-threaded, sequential)
├─ owns    → ScheduledExecutorService scheduler (ping, reconnect, refresh timers)
├─ owns    → WebSocket ws                     (OkHttp3, protobuf binary frames)
├─ uses    → Options
├─ calls   → EventListener                   (user callbacks)
└─ uses    → Backoff                          (reconnect delay calculation)

Subscription
├─ ref     → Client                           (parent — for sending commands)
├─ uses    → SubscriptionOptions
├─ calls   → SubscriptionEventListener        (user callbacks)
├─ uses    → Backoff                          (resubscribe delay calculation)
└─ uses    → Fossil                           (delta decompression when negotiated)
```

### Dependency on `streamsupport-minifuture`

The library depends on `streamsupport-minifuture` (backport of `CompletableFuture`). If your project already uses `streamsupport-cfuture`, you can safely exclude minifuture:

```groovy
implementation('io.github.centrifugal:centrifuge-java:{version}') {
    exclude group: 'net.sourceforge.streamsupport', module: 'streamsupport-minifuture'
}
```

---

## 2. Client Lifecycle & State Machine

### States (enum `ClientState`)

```
DISCONNECTED ──connect()──→ CONNECTING ──(connect reply ok)──→ CONNECTED
     ↑                           │                                  │
     │                      (transport                         (disconnect()
     │                       failure)                          or permanent
     │                           │                              error)
     │                           ↓                                  │
     │                    CONNECTING                                 │
     │                  (auto-reconnect                              │
     │                   with backoff)                               │
     │                                                              │
     └──────────────────────────────────────────────────────────────┘
                            CLOSED (after close())
```

| State | Enum value | Meaning |
|---|---|---|
| `DISCONNECTED` | `ClientState.DISCONNECTED` | Not connected. No reconnect scheduled. Initial state. |
| `CONNECTING` | `ClientState.CONNECTING` | Attempting to (re)connect. WebSocket may be opening or waiting for backoff delay. |
| `CONNECTED` | `ClientState.CONNECTED` | WebSocket open, connect handshake complete, ping/pong active. |
| `CLOSED` | `ClientState.CLOSED` | After `close()` — executor and scheduler shut down. Client is no longer usable. |

### Disconnect Codes (internal constants in `Client`)

| Constant | Value | Meaning |
|---|---|---|
| `DISCONNECTED_DISCONNECT_CALLED` | 0 | User called `disconnect()` |
| `DISCONNECTED_UNAUTHORIZED` | 1 | Unauthorized (empty token from getter, or `UnauthorizedException`) |
| `DISCONNECTED_BAD_PROTOCOL` | 2 | Protocol error (corrupted data, unexpected null token) |
| `DISCONNECTED_MESSAGE_SIZE_LIMIT` | 3 | WebSocket code 1009 |

### Connecting Codes

| Constant | Value | Meaning |
|---|---|---|
| `CONNECTING_CONNECT_CALLED` | 0 | User called `connect()` |
| `CONNECTING_TRANSPORT_CLOSED` | 1 | Transport closed, will reconnect |
| `CONNECTING_NO_PING` | 2 | Server ping not received in time |
| `CONNECTING_SUBSCRIBE_TIMEOUT` | 3 | Subscribe command timed out |
| `CONNECTING_UNSUBSCRIBE_ERROR` | 4 | Unsubscribe command failed |

### Key Methods

| Method | Signature | Behavior |
|---|---|---|
| `Client(endpoint, opts, listener)` | Constructor | Creates client. Does NOT connect. Stores token from `opts.getToken()`. |
| `connect()` | `void` | Submits connect task to executor. No-op if already CONNECTED/CONNECTING. Sets state → CONNECTING, fires `onConnecting`. |
| `disconnect()` | `void` | Submits disconnect task. Cancels all timers. Moves subs to SUBSCRIBING (transport closed). Fires `onDisconnected`. No reconnect. |
| `setToken(token)` | `void` | Updates connection token on executor thread. Useful for resetting token on logout (set to empty string, v0.3.0+). |
| `close(awaitMilliseconds)` | `boolean` | Calls `disconnect()`, shuts down executor and scheduler. Returns whether termination completed within timeout. **Client is not usable after this.** |
| `newSubscription(channel, opts, listener)` | `Subscription` | Creates and registers subscription. Throws `DuplicateSubscriptionException` if channel already registered. |
| `newSubscription(channel, listener)` | `Subscription` | Overload with default `SubscriptionOptions`. |
| `getSubscription(channel)` | `Subscription` (nullable) | Retrieves subscription from internal registry. |
| `removeSubscription(sub)` | `void` | Unsubscribes and removes from registry. After removal, `newSubscription` can be called again for that channel. |
| `publish(channel, data, cb)` | `void` | Publishes to channel without being subscribed. Requires publish permission on server. |
| `rpc(method, data, cb)` | `void` | Sends RPC request to server. |
| `send(data, cb)` | `void` | Sends async message (no reply expected from server). `CompletionCallback` fires when data is written to connection. |
| `history(channel, opts, cb)` | `void` | Gets publication history for a channel (useful for server-side subscriptions). |
| `presence(channel, cb)` | `void` | Gets presence information for a channel. |
| `presenceStats(channel, cb)` | `void` | Gets presence stats (numClients, numUsers) for a channel. |

### Reconnection Logic

- On transport close: if WebSocket close code indicates reconnect is appropriate (`code < 3500 || code >= 5000 || (code >= 4000 && code < 4500)`), state goes to CONNECTING and reconnect is scheduled.
- Reconnect uses **full-jitter exponential backoff** via `Backoff.duration(attempts, minDelay, maxDelay)`.
- Defaults: `minReconnectDelay = 500ms`, `maxReconnectDelay = 20000ms`.
- On successful connect, `reconnectAttempts` resets to 0.

### Server Ping Handling

- After connect, server sends `ping` interval and whether client should respond with `pong`.
- Client schedules a timer for `pingInterval + maxServerPingDelay` (default 10s extra). If no ping arrives, reconnects with code `CONNECTING_NO_PING`.
- If `sendPong` is true, client sends an empty Command as pong.

---

## 3. Subscription Lifecycle & State Machine

### States (enum `SubscriptionState`)

```
UNSUBSCRIBED ──subscribe()──→ SUBSCRIBING ──(subscribe reply ok)──→ SUBSCRIBED
      ↑                            │                                     │
      │                       (token err /                          (unsubscribe()
      │                        temp error)                          or permanent
      │                            │                                 error)
      │                            ↓                                     │
      │                      SUBSCRIBING                                 │
      │                    (auto-resubscribe                             │
      │                     with backoff)                                │
      └──────────────────────────────────────────────────────────────────┘
```

| State | Enum value | Meaning |
|---|---|---|
| `UNSUBSCRIBED` | `SubscriptionState.UNSUBSCRIBED` | Not subscribed. Initial state. |
| `SUBSCRIBING` | `SubscriptionState.SUBSCRIBING` | Subscribe command pending or waiting for resubscribe backoff. |
| `SUBSCRIBED` | `SubscriptionState.SUBSCRIBED` | Active subscription. Publications are delivered. |

### Subscribing Codes

| Constant | Value | Meaning |
|---|---|---|
| `SUBSCRIBING_SUBSCRIBE_CALLED` | 0 | User called `subscribe()` |
| `SUBSCRIBING_TRANSPORT_CLOSED` | 1 | Transport closed, will resubscribe on reconnect |

### Unsubscribed Codes

| Constant | Value | Meaning |
|---|---|---|
| `UNSUBSCRIBED_UNSUBSCRIBE_CALLED` | 0 | User called `unsubscribe()` |
| `UNSUBSCRIBED_UNAUTHORIZED` | 1 | Unauthorized (empty token from getter, or `UnauthorizedException`) |
| `UNSUBSCRIBED_CLIENT_CLOSED` | 2 | Client was closed |

### Key Methods

| Method | Signature | Behavior |
|---|---|---|
| `subscribe()` | `void` | No-op if already SUBSCRIBED/SUBSCRIBING. Sets state → SUBSCRIBING, fires `onSubscribing`, sends subscribe command. |
| `unsubscribe()` | `void` | Sends unsubscribe to server, moves to UNSUBSCRIBED, fires `onUnsubscribed`. Completes pending futures with `SubscriptionStateError`. |
| `publish(data, cb)` | `void` | Waits for SUBSCRIBED state before publishing. Uses `CompletableFuture` with timeout. |
| `history(opts, cb)` | `void` | Waits for SUBSCRIBED state before requesting history. |
| `presence(cb)` | `void` | Waits for SUBSCRIBED state before requesting presence. |
| `presenceStats(cb)` | `void` | Waits for SUBSCRIBED state before requesting presence stats. |
| `getState()` | `SubscriptionState` | Returns current state (volatile). |
| `getChannel()` | `String` | Returns the channel name. |

### "Wait for Subscribe" Pattern

`publish`, `history`, `presence`, and `presenceStats` on `Subscription` use a `CompletableFuture` pattern:
1. A future is created and stored in `this.futures` map.
2. If state is already SUBSCRIBED, the future completes immediately.
3. When `moveToSubscribed` is called, all pending futures are completed.
4. If the subscription moves to UNSUBSCRIBED, pending futures complete with `SubscriptionStateError`.
5. All futures have a timeout from `client.getOpts().getTimeout()` (default 5000ms).

### Delta Compression (Fossil)

- Enabled via `SubscriptionOptions.setDelta("fossil")`. Must also be enabled server-side.
- When delta is negotiated in subscribe result (`result.getDelta()` is true), subsequent publications may carry `delta=true` flag.
- `Fossil.applyDelta(prevData, deltaData)` reconstructs the full payload from the previous data and the delta.
- The subscription maintains `prevData` internally. On first publication after subscribe, `prevData` is `null`, so the first publication is always a full payload.

### Recovery

- If `SubscriptionOptions.setSince(StreamPosition)` is set or if the server indicates `recoverable=true`, the subscription tracks offset/epoch.
- On resubscribe, `recover=true` is sent with the last known offset and epoch.
- The `SubscribedEvent` reports `wasRecovering` and `recovered` booleans.

---

## 4. Configuration — Options

`Options` configures a `Client` instance. Set properties before passing to the `Client` constructor.

| Property | Type | Default | Description |
|---|---|---|---|
| `token` | `String` | `""` | Connection JWT. Obtain from your backend. |
| `tokenGetter` | `ConnectionTokenGetter` | `null` | Callback to fetch/refresh connection tokens. Required for token refresh. |
| `name` | `String` | `"java"` | Client name (not unique per client — identifies the app). |
| `version` | `String` | `""` | Application version. Used for server-side observability. |
| `data` | `byte[]` | `null` | Custom data sent to server in Connect command. Useful with Centrifugo connect proxy. |
| `headers` | `Map<String,String>` | `null` | Custom headers for WebSocket Upgrade request. |
| `timeout` | `int` | `5000` | Timeout for requests in milliseconds. |
| `minReconnectDelay` | `int` | `500` | Minimum time before reconnect attempt (ms). |
| `maxReconnectDelay` | `int` | `20000` | Maximum time between reconnect attempts (ms). |
| `maxServerPingDelay` | `int` | `10000` | Extra time to wait beyond server ping interval before considering connection dead (ms). |
| `proxy` | `Proxy` | `null` | Proxy for default OkHttpClient. Ignored if `setOkHttpClient` used. |
| `proxyLogin` / `proxyPassword` | `String` | `null` | Proxy credentials. Ignored if `setOkHttpClient` used. |
| `dns` | `Dns` | `null` | Custom DNS resolver. Ignored if `setOkHttpClient` used. |
| `sslSocketFactory` | `SSLSocketFactory` | `null` | Custom SSL socket factory. Ignored if `setOkHttpClient` used. |
| `trustManager` | `X509TrustManager` | `null` | Custom trust manager. Used with `sslSocketFactory`. |
| `okHttpClient` | `OkHttpClient` | `null` | Pre-configured OkHttpClient. When set, proxy/dns/ssl options are ignored. |

---

## 5. Configuration — SubscriptionOptions

`SubscriptionOptions` configures a `Subscription` instance. Passed to `client.newSubscription()`.

| Property | Type | Default | Description |
|---|---|---|---|
| `token` | `String` | `""` | Subscription JWT. |
| `tokenGetter` | `SubscriptionTokenGetter` | `null` | Callback to fetch/refresh subscription tokens. |
| `data` | `byte[]` | `null` | Custom data sent in Subscribe command. |
| `minResubscribeDelay` | `int` | `500` | Minimum time before resubscribe attempt (ms). |
| `maxResubscribeDelay` | `int` | `20000` | Maximum time between resubscribe attempts (ms). |
| `positioned` | `boolean` | `false` | Request positioned subscription (server tracks offset). |
| `recoverable` | `boolean` | `false` | Request recoverable subscription (server recovery on reconnect). |
| `joinLeave` | `boolean` | `false` | Request join/leave events for the channel. |
| `delta` | `String` | `""` | Delta compression type. Only `"fossil"` is currently supported. Must also be enabled server-side. |
| `since` | `StreamPosition` | `null` | Resume from a specific stream position (offset + epoch). Sets `recover=true`. |

---

## 6. Event System — Client Events

`EventListener` is an abstract class with no-op defaults. Override only the methods you need.

### Connection Lifecycle Events

| Method | Event Class | Key Fields | When Fired |
|---|---|---|---|
| `onConnecting(Client, ConnectingEvent)` | `ConnectingEvent` | `code: int`, `reason: String` | State transitions to CONNECTING (connect called, transport closed, no ping, etc.) |
| `onConnected(Client, ConnectedEvent)` | `ConnectedEvent` | `client: String` (server-assigned client ID), `data: byte[]` | Connect reply received successfully |
| `onDisconnected(Client, DisconnectedEvent)` | `DisconnectedEvent` | `code: int`, `reason: String` | State transitions to DISCONNECTED (disconnect called, unauthorized, bad protocol) |

### Error Events

| Method | Event Class | Key Fields | When Fired |
|---|---|---|---|
| `onError(Client, ErrorEvent)` | `ErrorEvent` | `error: Throwable`, `httpResponseCode: Integer` (nullable) | Transport failure, protocol error, token error, refresh error, configuration error |

### Server Message Event

| Method | Event Class | Key Fields | When Fired |
|---|---|---|---|
| `onMessage(Client, MessageEvent)` | `MessageEvent` | `data: byte[]` | Server push message received (not channel-specific) |

### Server-Side Subscription Events

These fire for subscriptions managed by the server (channels returned in connect result or via server push):

| Method | Event Class | Key Fields | When Fired |
|---|---|---|---|
| `onSubscribed(Client, ServerSubscribedEvent)` | `ServerSubscribedEvent` | `channel`, `wasRecovering`, `recovered`, `positioned`, `recoverable`, `streamPosition: StreamPosition`, `data: byte[]` | Server-side subscription established |
| `onSubscribing(Client, ServerSubscribingEvent)` | `ServerSubscribingEvent` | `channel` | Server-side subscription re-subscribing (transport closed) |
| `onUnsubscribed(Client, ServerUnsubscribedEvent)` | `ServerUnsubscribedEvent` | `channel` | Server-side subscription removed |
| `onPublication(Client, ServerPublicationEvent)` | `ServerPublicationEvent` | `channel`, `data: byte[]`, `info: ClientInfo`, `offset: long`, `tags: Map<String,String>` | Publication on server-side subscription channel |
| `onJoin(Client, ServerJoinEvent)` | `ServerJoinEvent` | `channel`, `info: ClientInfo` | Client joined server-side subscription channel |
| `onLeave(Client, ServerLeaveEvent)` | `ServerLeaveEvent` | `channel`, `info: ClientInfo` | Client left server-side subscription channel |

---

## 7. Event System — Subscription Events

`SubscriptionEventListener` is an abstract class with no-op defaults. Override only the methods you need.

| Method | Event Class | Key Fields | When Fired |
|---|---|---|---|
| `onSubscribing(Subscription, SubscribingEvent)` | `SubscribingEvent` | `code: int`, `reason: String` | State transitions to SUBSCRIBING |
| `onSubscribed(Subscription, SubscribedEvent)` | `SubscribedEvent` | `wasRecovering: Boolean`, `recovered: Boolean`, `positioned: Boolean`, `recoverable: Boolean`, `streamPosition: StreamPosition`, `data: byte[]` | Subscribe reply received successfully |
| `onUnsubscribed(Subscription, UnsubscribedEvent)` | `UnsubscribedEvent` | `code: int`, `reason: String` | State transitions to UNSUBSCRIBED |
| `onPublication(Subscription, PublicationEvent)` | `PublicationEvent` | `data: byte[]`, `info: ClientInfo`, `offset: long`, `tags: Map<String,String>` | Publication received on channel |
| `onJoin(Subscription, JoinEvent)` | `JoinEvent` | `info: ClientInfo` | Client joined channel |
| `onLeave(Subscription, LeaveEvent)` | `LeaveEvent` | `info: ClientInfo` | Client left channel |
| `onError(Subscription, SubscriptionErrorEvent)` | `SubscriptionErrorEvent` | `error: Throwable` | Subscription-level error (token, refresh, subscribe) |

---

## 8. Token & Authentication Flow

### Connection Token

1. **Static token**: Set via `Options.setToken(token)` before connecting.
2. **Dynamic token** (recommended for expiring tokens): Set `Options.setTokenGetter(connectionTokenGetter)`.
3. **Token refresh flow**:
   - On connect, if token is empty and `tokenGetter` is set, `getConnectionToken(ConnectionTokenEvent, TokenCallback)` is called.
   - If token expires (server returns error code 109), `refreshRequired` is set to true, and token is re-fetched on next connect attempt.
   - After a successful connect, if the connect result has `expires=true`, a refresh is scheduled after `ttl` seconds.
   - During refresh, `tokenGetter.getConnectionToken()` is called, then a `RefreshRequest` is sent to the server.

### Subscription Token

1. **Static token**: Set via `SubscriptionOptions.setToken(token)`.
2. **Dynamic token**: Set `SubscriptionOptions.setTokenGetter(subscriptionTokenGetter)`.
3. **Token refresh flow**:
   - On subscribe, if token is empty and `tokenGetter` is set, `getSubscriptionToken(SubscriptionTokenEvent, TokenCallback)` is called.
   - `SubscriptionTokenEvent` contains `channel: String` so the getter knows which channel the token is for.
   - After a successful subscribe, if the result has `expires=true`, a refresh is scheduled after `ttl` seconds.

### TokenCallback Interface

```java
public interface TokenCallback {
    void Done(Throwable e, String token);
}
```

- Call with `(null, token)` on success.
- Call with `(error, null)` on failure.
- Call with `(new UnauthorizedException(), null)` to signal the user is unauthorized → client disconnects (connection) or subscription unsubscribes.
- **v0.3.0 change**: Returning `(null, "")` (empty string) from `ConnectionTokenGetter` no longer disconnects the client. It's a valid scenario (e.g., anonymous access). To explicitly disconnect, throw `UnauthorizedException`.
- For `SubscriptionTokenGetter`: returning `(null, "")` or `(null, null)` still triggers unauthorized unsubscribe for the subscription.

### `setToken(token)` Method (v0.3.0+)

Allows resetting the connection token at any time (useful for logout → set to empty string).

---

## 9. Error Taxonomy

### Error Class Hierarchy

All error classes extend `Throwable` (not `Exception`), except where noted. They wrap an inner error accessible via `getError()`.

| Class | Extends | Purpose | Wraps |
|---|---|---|---|
| `ReplyError` | `Throwable` | Server-side error in command reply | `code: int`, `message: String`, `temporary: boolean` |
| `TokenError` | `Throwable` | Error while fetching connection token | Inner `Throwable` |
| `RefreshError` | `Throwable` | Error during connection refresh | Inner `Throwable` |
| `ConfigurationError` | `Throwable` | SDK misconfiguration (e.g., missing tokenGetter) | Inner `Throwable` |
| `UnclassifiedError` | `Throwable` | Unexpected internal error | Inner `Throwable` |
| `SubscriptionTokenError` | `Throwable` | Error while fetching subscription token | Inner `Throwable` |
| `SubscriptionRefreshError` | `Throwable` | Error during subscription refresh | Inner `Throwable` |
| `SubscriptionSubscribeError` | `Throwable` | Server returned error on subscribe | Inner `Throwable` |
| `SubscriptionStateError` | `Exception` | Operation attempted in wrong subscription state | `state: SubscriptionState` |
| `SubscriptionErrorEvent` | `Throwable` | Wrapper event for subscription errors delivered via `onError` | Inner `Throwable` |
| `UnauthorizedException` | `Exception` | Signals unauthorized access in token callbacks | Message: `"unauthorized"` |
| `DuplicateSubscriptionException` | `Exception` | Thrown when creating subscription for already-registered channel | — |

### ErrorEvent

```java
public class ErrorEvent {
    Throwable error;           // The actual error
    Integer httpResponseCode;  // Non-null only for HTTP/transport failures (from OkHttp onFailure)
}
```

Delivered to `EventListener.onError()`. The `error` field will be one of:
- `UnclassifiedError` — unexpected exception during processing
- `TokenError` — connection token fetch failed
- `RefreshError` — connection refresh failed
- `ConfigurationError` — SDK misconfiguration
- `ReplyError` — server returned an error in connect reply
- Raw `Throwable` — OkHttp transport failure

### ReplyError Details

```java
public class ReplyError extends Throwable {
    int code;          // Server error code
    String message;    // Human-readable error message
    boolean temporary; // If true, operation may be retried
}
```

Special code: **109** = token expired. Triggers automatic token refresh.

---

## 10. Data & Result Types

### ClientInfo

```java
public class ClientInfo {
    String user;         // User ID
    String client;       // Client ID (unique connection identifier)
    byte[] connInfo;     // Connection-level custom info
    byte[] chanInfo;     // Channel-level custom info
}
```

Parsed from protocol via static `ClientInfo.fromProtocolClientInfo()`. Can return `null` if the protocol object is null.

### StreamPosition

```java
public class StreamPosition {
    long offset;    // Publication offset in stream
    String epoch;   // Stream epoch (changes when stream is recreated)
}
```

Used in `SubscriptionOptions.setSince()`, `HistoryOptions.Builder.withSince()`, and returned in `SubscribedEvent`/`ServerSubscribedEvent`.

### HistoryOptions (Builder pattern)

```java
HistoryOptions opts = new HistoryOptions.Builder()
    .withLimit(-1)           // -1 = all publications (default: 0)
    .withSince(streamPos)    // Resume from position (optional)
    .withReverse(true)       // Reverse order (default: false)
    .build();
```

### Result Types

| Type | Fields | Used By |
|---|---|---|
| `PublishResult` | (empty class) | `client.publish()`, `sub.publish()` |
| `HistoryResult` | `publications: List<Publication>`, `offset: long`, `epoch: String` | `client.history()`, `sub.history()` |
| `PresenceResult` | `clients: Map<String, ClientInfo>` | `client.presence()`, `sub.presence()` |
| `PresenceStatsResult` | `numClients: Integer`, `numUsers: Integer` | `client.presenceStats()`, `sub.presenceStats()` |
| `RPCResult` | `data: byte[]`, `error: ReplyError` | `client.rpc()` |

### Publication

```java
public class Publication {
    byte[] data;        // Publication payload
    long offset;        // Stream offset
    ClientInfo info;    // Publisher info (may be null)
}
```

### Callback Interfaces

```java
// For operations with a result
public interface ResultCallback<T> {
    void onDone(Throwable e, T result);  // Either e or result is non-null
}

// For fire-and-forget operations
public interface CompletionCallback {
    void onDone(Throwable e);  // e is null on success
}
```

### Dns Interface

```java
public interface Dns {
    List<InetAddress> resolve(String host) throws UnknownHostException;
}
```

Set via `Options.setDns()` for custom DNS resolution. Only used when building the default OkHttpClient (ignored if `setOkHttpClient` is used).

---

## 11. Recommended Patterns

> All patterns below are **verified from the authoritative `demo/` and `example/` directories** in the repository.

### Basic Client + Subscription Setup (from `example/Main.java`)

```java
EventListener listener = new EventListener() {
    @Override
    public void onConnected(Client client, ConnectedEvent event) {
        System.out.printf("connected with client id %s%n", event.getClient());
    }
    @Override
    public void onConnecting(Client client, ConnectingEvent event) {
        System.out.printf("connecting: %s%n", event.getReason());
    }
    @Override
    public void onDisconnected(Client client, DisconnectedEvent event) {
        System.out.printf("disconnected %d %s%n", event.getCode(), event.getReason());
    }
    @Override
    public void onError(Client client, ErrorEvent event) {
        System.out.printf("connection error: %s%n", event.getError().toString());
    }
    // Override onMessage, onSubscribed, onPublication, etc. as needed
};

Options opts = new Options();
// opts.setToken("your-jwt-token");  // Set if auth required

Client client = new Client(
    "ws://localhost:8000/connection/websocket",
    opts,
    listener
);
client.connect();
```

### Creating a Subscription (from `example/Main.java`)

```java
SubscriptionEventListener subListener = new SubscriptionEventListener() {
    @Override
    public void onSubscribed(Subscription sub, SubscribedEvent event) {
        System.out.println("subscribed to " + sub.getChannel() + ", recovered " + event.getRecovered());
    }
    @Override
    public void onPublication(Subscription sub, PublicationEvent event) {
        String data = new String(event.getData(), UTF_8);
        System.out.println("message from " + sub.getChannel() + " " + data);
    }
    @Override
    public void onError(Subscription sub, SubscriptionErrorEvent event) {
        System.out.println("subscription error " + sub.getChannel() + " " + event.getError().toString());
    }
};

SubscriptionOptions subOpts = new SubscriptionOptions();
try {
    Subscription sub = client.newSubscription("chat:index", subOpts, subListener);
} catch (DuplicateSubscriptionException e) {
    e.printStackTrace();
    return;
}
sub.subscribe();
```

### Publishing Data (from `example/Main.java`)

```java
// Via client (does NOT wait for subscription):
String data = "{\"input\": \"hi from Java\"}";
client.publish("chat:index", data.getBytes(), (err, res) -> {
    if (err != null) {
        System.out.println("error publish: " + err);
        return;
    }
    System.out.println("successfully published");
});

// Via subscription (WAITS for subscribe success):
sub.publish(data.getBytes(), (err, res) -> {
    if (err != null) {
        System.out.println("error publish: " + err);
        return;
    }
    System.out.println("successfully published");
});
```

### Presence Stats (from `example/Main.java`)

```java
sub.presenceStats((err, res) -> {
    if (err != null) {
        System.out.println("error presence stats: " + err);
        return;
    }
    System.out.println("Num clients connected: " + res.getNumClients());
});
```

### History (from `example/Main.java`)

```java
sub.history(new HistoryOptions.Builder().withLimit(-1).build(), (err, res) -> {
    if (err != null) {
        System.out.println("error history: " + err);
        return;
    }
    System.out.println("Num history publication: " + res.getPublications().size());
    System.out.println("Top stream offset: " + res.getOffset());
});
```

### Cleanup (from `example/Main.java`)

```java
sub.unsubscribe();
client.removeSubscription(sub);  // Removes from internal registry
client.disconnect();

boolean ok = client.close(5000);  // Wait up to 5s for graceful shutdown
if (!ok) {
    System.out.println("client was not gracefully closed");
}
```

### Android Activity Lifecycle (from `demo/MainActivity.java`)

```java
@Override
protected void onPause() {
    super.onPause();
    activityPaused = true;
    client.disconnect();  // Disconnect when going to background
}

@Override
protected void onResume() {
    super.onResume();
    if (activityPaused) {
        client.connect();  // Reconnect when coming to foreground
        activityPaused = false;
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();
    try {
        client.close(5000);  // Clean up resources
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

### Token Getter Pattern (from `example/Main.java`, commented)

```java
opts.setTokenGetter(new ConnectionTokenGetter() {
    @Override
    public void getConnectionToken(ConnectionTokenEvent event, TokenCallback cb) {
        // Request token from your backend here
        cb.Done(null, "your-new-jwt-token");
    }
});
```

### Server-Side Subscription Events (from `example/Main.java`)

```java
// These are handled on the EventListener (client level), not SubscriptionEventListener:
@Override
public void onSubscribed(Client client, ServerSubscribedEvent event) {
    System.out.println("server side subscribed: " + event.getChannel());
}
@Override
public void onPublication(Client client, ServerPublicationEvent event) {
    String data = new String(event.getData(), UTF_8);
    System.out.println("server side publication: " + event.getChannel() + ": " + data);
}
@Override
public void onJoin(Client client, ServerJoinEvent event) {
    System.out.println("server side join: " + event.getChannel() + " from client " + event.getInfo().getClient());
}
```

---

## 12. Pitfalls & Breaking-Change History

### Critical Pitfalls

1. **Client is NOT usable after `close()`**: The executor and scheduler are shut down. Create a new `Client` instance if you need to reconnect.

2. **`client.publish()` does NOT wait for subscription**: It publishes immediately. If the client isn't connected yet or the channel subscription isn't established, the message may be lost. Use `sub.publish()` to wait for subscription.

3. **Callbacks execute on the internal executor thread, NOT the UI thread**: On Android, wrap UI updates with `runOnUiThread()` (as shown in `demo/MainActivity.java`).

4. **`DuplicateSubscriptionException` on re-subscribe**: You must call `client.removeSubscription(sub)` before creating a new subscription to the same channel.

5. **Token empty string semantics changed in v0.3.0**: Previously, returning `""` from `ConnectionTokenGetter` caused disconnect. Now it's valid. Use `UnauthorizedException` to explicitly signal unauthorized.

6. **ProGuard/R8 obfuscation**: Protobuf Lite uses reflection. The library includes consumer ProGuard rules in the JAR (since v0.2.5), but if you encounter connection errors with obfuscation enabled, check the [ProGuard rules](centrifuge/src/main/resources/META-INF/proguard).

7. **Background connection on mobile**: OS may silently close WebSocket connections when the app goes to background. Always disconnect in `onPause()` and reconnect in `onResume()`.

8. **`HistoryOptions.withLimit(-1)`**: Needed to get all publications. Default limit is 0 which returns no publications (since Centrifuge v0.18.0 / Centrifugo v3+).

9. **Error code 109 = token expired**: Both `Client` and `Subscription` handle this automatically by re-fetching tokens. No manual intervention needed, but ensure your `tokenGetter` is properly configured.

10. **`newSubscription` with `SubscriptionOptions`**: Pass options at creation time (v0.2.6+). Options cannot be changed after subscription is created.

### Breaking Change History

| Version | Change |
|---|---|
| **0.5.0** | Custom `OkHttpClient` support added via `Options.setOkHttpClient()`. OkHttp dependency bumped to **4.12.0**. When `okHttpClient` is set, individual proxy/dns/SSL options are ignored. |
| **0.4.3** | Custom `SSLSocketFactory` support added via `Options.setSSLSocketFactory()` / `Options.setTrustManager()`. |
| **0.4.2** | Delta compression (Fossil) support added: `SubscriptionOptions.setDelta("fossil")`. New `since` option (`SubscriptionOptions.setSince(StreamPosition)`) to set starting stream position for recovery. Protobuf-javalite version bumped. |
| **0.4.1** | `Publication.getInfo()` / `PublicationEvent.getInfo()` now accessible (was missing previously). |
| **0.4.0** | Dependency changed: `streamsupport-cfuture` replaced with `streamsupport-minifuture`. Users who manually excluded `cfuture` must update their exclusion. `ErrorEvent` gains a new `httpResponseCode: Integer` field (non-null only on HTTP/transport failures). |
| **0.3.0** | Token semantics changed: empty string from `ConnectionTokenGetter` no longer disconnects. Use `UnauthorizedException` for explicit disconnect. `setToken` method restored. |
| **0.2.6** | `newSubscription` overload with `SubscriptionOptions` added. Unsubscribe API fixed (was using legacy command format). |
| **0.2.2** | **Major rewrite**: New protocol iteration for Centrifugo v4. Complete API redesign to match [SDK API spec](https://centrifugal.dev/docs/transports/client_api). v0.2.0 and v0.2.1 were broken. |
| **0.1.0** | History API changed: default no longer returns all publications. Use `HistoryOptions.Builder.withLimit(-1)` to get all. Updated for Centrifuge ≥ 0.18.0 / Centrifugo v3. |
| **0.0.2** | Subscription API changed: create first, then manage lifecycle (subscribe/unsubscribe), then remove with `removeSubscription()`. |

---

## 13. Threading Model

### Single-Threaded Executor

- All internal state mutations run on a **single-threaded `ExecutorService`** (`Executors.newSingleThreadExecutor()`).
- All public methods (`connect`, `disconnect`, `subscribe`, `publish`, etc.) submit work to this executor.
- User callbacks (`EventListener`, `SubscriptionEventListener`) are invoked **on this executor thread**.

### Scheduler

- A **`ScheduledExecutorService`** with pool size 1 handles timed tasks:
  - Reconnect backoff delays
  - Resubscribe backoff delays
  - Server ping timeout
  - Token refresh scheduling

### Thread Safety

- `ClientState` and `SubscriptionState` fields are `volatile`.
- `subs` map operations are synchronized via `synchronized (this.subs)`.
- `futures`, `connectCommands`, `connectAsyncCommands`, `serverSubs` use `ConcurrentHashMap`.
- OkHttp `WebSocket` callbacks (`onOpen`, `onMessage`, `onClosed`, `onFailure`) submit work to the executor, so processing is single-threaded.

### Implications for Users

- **Do NOT perform blocking operations in callbacks** — they block all client processing.
- **Android**: Callbacks are NOT on the main/UI thread. Use `runOnUiThread()` for UI updates.
- **`close(awaitMilliseconds)`**: Shuts down both executor and scheduler. The `awaitTermination` only waits for the executor.

---

## 14. Android Considerations

### Required Permission

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### Background Handling

When a mobile app goes to background, the OS may silently close persistent connections. The recommended pattern (from `demo/MainActivity.java`):

```java
// Disconnect in onPause:
client.disconnect();

// Reconnect in onResume:
client.connect();

// Full cleanup in onDestroy:
client.close(5000);
```

### UI Thread Safety

All callbacks run on the internal executor thread. Use `Activity.runOnUiThread()`:

```java
@Override
public void onConnected(Client client, ConnectedEvent event) {
    MainActivity.this.runOnUiThread(() ->
        tv.setText(String.format("Connected with client ID %s", event.getClient()))
    );
}
```

### ProGuard / R8

Since v0.2.5, consumer ProGuard rules are included in the JAR automatically. If you still encounter issues with obfuscated builds, see [ProGuard rules](centrifuge/src/main/resources/META-INF/proguard).

---

## 15. Internal Utilities

### Backoff (`internal/backoff/Backoff.java`)

Implements **full-jitter exponential backoff** per [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/).

```java
public long duration(int step, int minDelay, int maxDelay) {
    // Full jitter technique.
    // https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
    double currentStep = step;
    if (currentStep > 31) { currentStep = 31; }
    double min = Math.min(maxDelay, minDelay * Math.pow(2, currentStep));
    int val = (int)(Math.random() * (min + 1));
    return Math.min(maxDelay, minDelay + val);
}
```

- `step` = number of attempts so far (capped at 31 to avoid overflow)
- Returns a random value between `minDelay` and `min(maxDelay, minDelay * 2^step)`

### Fossil (`Fossil.java`)

Package-private delta compression utility:
- `Fossil.applyDelta(byte[] source, byte[] delta)` — applies a Fossil delta to reconstruct the target
- `Fossil.checksum(byte[] arr)` — computes a checksum for delta verification
- Uses a custom Reader/Writer for parsing the delta format
- Throws `Exception` on any delta corruption (bad checksum, size mismatch, unknown operator)

### Protocol (generated)

`internal/protocol/Protocol.java` is auto-generated from `.proto` definitions during Gradle build. Not included in source inventory as it's a build artifact.

---

## 16. Complete File Inventory

### Main Package: `io.github.centrifugal.centrifuge`

| # | File | Size | Category |
|---|---|---|---|
| 1 | `Client.java` | ~56 KB | Core — connection management |
| 2 | `Subscription.java` | ~19 KB | Core — channel subscription |
| 3 | `Options.java` | ~6 KB | Configuration — client |
| 4 | `SubscriptionOptions.java` | ~2.6 KB | Configuration — subscription |
| 5 | `Fossil.java` | ~6.1 KB | Core — delta compression |
| 6 | `EventListener.java` | ~942 B | Events — client listener (abstract) |
| 7 | `SubscriptionEventListener.java` | ~650 B | Events — subscription listener (abstract) |
| 8 | `ConnectedEvent.java` | ~386 B | Events — client connected |
| 9 | `ConnectingEvent.java` | ~371 B | Events — client connecting |
| 10 | `DisconnectedEvent.java` | ~375 B | Events — client disconnected |
| 11 | `ErrorEvent.java` | ~572 B | Events — client error |
| 12 | `MessageEvent.java` | ~221 B | Events — server message |
| 13 | `SubscribedEvent.java` | ~1.2 KB | Events — subscription subscribed |
| 14 | `SubscribingEvent.java` | ~373 B | Events — subscription subscribing |
| 15 | `UnsubscribedEvent.java` | ~375 B | Events — subscription unsubscribed |
| 16 | `PublicationEvent.java` | ~752 B | Events — publication on subscription |
| 17 | `JoinEvent.java` | ~230 B | Events — join on subscription |
| 18 | `LeaveEvent.java` | ~231 B | Events — leave on subscription |
| 19 | `SubscriptionErrorEvent.java` | ~290 B | Events — subscription error |
| 20 | `ServerPublicationEvent.java` | ~928 B | Events — server-side publication |
| 21 | `ServerSubscribedEvent.java` | ~1.3 KB | Events — server-side subscribed |
| 22 | `ServerSubscribingEvent.java` | ~265 B | Events — server-side subscribing |
| 23 | `ServerUnsubscribedEvent.java` | ~267 B | Events — server-side unsubscribed |
| 24 | `ServerJoinEvent.java` | ~392 B | Events — server-side join |
| 25 | `ServerLeaveEvent.java` | ~394 B | Events — server-side leave |
| 26 | `ConnectionTokenEvent.java` | ~81 B | Events — connection token request |
| 27 | `SubscriptionTokenEvent.java` | ~272 B | Events — subscription token request |
| 28 | `ConnectionTokenGetter.java` | ~208 B | Auth — connection token getter (abstract) |
| 29 | `SubscriptionTokenGetter.java` | ~214 B | Auth — subscription token getter (abstract) |
| 30 | `TokenCallback.java` | ~249 B | Auth — token callback interface |
| 31 | `TokenError.java` | ~278 B | Errors — token fetch failed |
| 32 | `RefreshError.java` | ~282 B | Errors — refresh failed |
| 33 | `ReplyError.java` | ~775 B | Errors — server reply error |
| 34 | `UnauthorizedException.java` | ~174 B | Errors — unauthorized signal |
| 35 | `UnclassifiedError.java` | ~292 B | Errors — unclassified |
| 36 | `ConfigurationError.java` | ~294 B | Errors — SDK misconfiguration |
| 37 | `SubscriptionTokenError.java` | ~302 B | Errors — subscription token |
| 38 | `SubscriptionRefreshError.java` | ~306 B | Errors — subscription refresh |
| 39 | `SubscriptionSubscribeError.java` | ~310 B | Errors — subscribe command |
| 40 | `SubscriptionStateError.java` | ~304 B | Errors — wrong subscription state |
| 41 | `DuplicateSubscriptionException.java` | ~278 B | Errors — duplicate subscription |
| 42 | `ClientInfo.java` | ~1.7 KB | Data — client/user info |
| 43 | `ClientState.java` | ~120 B | Data — client state enum |
| 44 | `SubscriptionState.java` | ~119 B | Data — subscription state enum |
| 45 | `Publication.java` | ~538 B | Data — publication |
| 46 | `StreamPosition.java` | ~747 B | Data — stream position |
| 47 | `HistoryOptions.java` | ~1.3 KB | Data — history request options |
| 48 | `HistoryResult.java` | ~654 B | Data — history result |
| 49 | `PresenceResult.java` | ~330 B | Data — presence result |
| 50 | `PresenceStatsResult.java` | ~453 B | Data — presence stats |
| 51 | `PublishResult.java` | ~74 B | Data — publish result |
| 52 | `RPCResult.java` | ~400 B | Data — RPC result |
| 53 | `CompletionCallback.java` | ~229 B | Callbacks — completion |
| 54 | `ResultCallback.java` | ~241 B | Callbacks — result |
| 55 | `ServerSubscription.java` | ~764 B | Internal — server sub state |
| 56 | `Dns.java` | ~303 B | Config — DNS resolver interface |

### Internal Package: `io.github.centrifugal.centrifuge.internal.backoff`

| # | File | Size | Category |
|---|---|---|---|
| 57 | `Backoff.java` | ~568 B | Internal — exponential backoff |

### Generated (not in source):

- `internal/protocol/Protocol.java` — generated from `centrifuge/main/proto/*.proto` by `protobuf` Gradle plugin

---

*All 57 source files verified against the actual repository source code.*
