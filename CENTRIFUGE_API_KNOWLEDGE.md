# Centrifuge Java 0.5.0 API Knowledge Base

**Last Updated**: March 2026  
**Scope**: 44 public/internal classes across `io.github.centrifugal.centrifuge`  
**Focus**: LLM-optimized dense documentation for exact behavior and edge cases

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Client Lifecycle](#client-lifecycle)
3. [Subscription Lifecycle](#subscription-lifecycle)
4. [Core Classes](#core-classes)
5. [Event System](#event-system)
6. [Token & Auth Flow](#token--auth-flow)
7. [Error Taxonomy](#error-taxonomy)
8. [Threading Model](#threading-model)
9. [Known Pitfalls](#known-pitfalls)
10. [Android Considerations](#android-considerations)
11. [Recommended Patterns](#recommended-patterns)

---

## Architecture Overview

### High-Level Structure

Centrifuge Java is a **WebSocket-based real-time protocol client** built on **OkHttp3** for transport and **Protocol Buffers** for serialization.

**Core Components:**
- **`Client`** (57KB): Manages WebSocket connection, reconnects, command queue, and server-side subscriptions. Single `ExecutorService` runs all internal logic sequentially.
- **`Subscription`** (19KB): Manages channel subscription state, resubscription, token refresh, and delta decoding. Instances hold their own backoff and resubscribe timers.
- **`ServerSubscription`**: Lightweight holder for channel state during automatic server-side subscriptions (e.g., channels sent in Connect).
- **`Options` / `SubscriptionOptions`**: Configuration builders (fluent API). Immutable after `Client`/`Subscription` instantiation.
- **`EventListener` / `SubscriptionEventListener`**: User-provided callback interfaces. Called from internal executor thread, not UI thread.

### Class Relationship Map

```
Client
├─ manages → Map<String, Subscription>  (clientSubs: local subscriptions)
├─ manages → Map<String, ServerSubscription>  (serverSubs: server-side subscriptions)
├─ owns → ExecutorService executor  (single-threaded, sequential)
├─ owns → ScheduledExecutorService scheduler  (ping, reconnect, refresh timers)
├─ owns → WebSocket ws  (OkHttp3, protobuf binary frame)
├─ uses → Options
├─ calls → EventListener (user callbacks)
└─ uses → Backoff (reconnect delay calculation)

Subscription
├─ holds → Client reference (to send commands, schedule tasks)
├─ uses → SubscriptionOptions
├─ calls → SubscriptionEventListener (user callbacks)
├─ holds → Backoff (resubscription delay)
└─ holds → StreamPosition (offset + epoch for recovery)
```

### Protocol & Serialization

- **Transport**: WebSocket binary frames (OkHttp3 `WebSocketListener`)
- **Format**: Protocol Buffers (centrifugal/protocol)
- **Frame Structure**: Delimited protobuf messages (`Message.writeDelimitedTo()` / `Message.parseDelimitedFrom()`)
- **Max Frame Size**: Limited by OkHttp (default ~1MB); exceeding triggers WebSocket code 1009 → `DISCONNECTED_MESSAGE_SIZE_LIMIT`

---

## Client Lifecycle

### State Machine

```
┌─────────────────────────────────────────────────────────┐
│  ClientState { DISCONNECTED, CONNECTING, CONNECTED }    │
└─────────────────────────────────────────────────────────┘

DISCONNECTED (initial)
    ↓ .connect() called
CONNECTING
    ├─ fire ConnectingEvent(CONNECTING_CONNECT_CALLED)
    ├─ call handleConnectionOpen()
    │  ├─ if token needed → call tokenGetter.getConnectionToken()
    │  │  ├─ if UnauthorizedException → fail with DISCONNECTED_UNAUTHORIZED
    │  │  └─ on token received → sendConnect()
    │  └─ else → sendConnect()
    │
    ├─ WebSocket.onOpen() → handleConnectionOpen()
    ├─ WebSocket.onMessage(protobuf) → processReply()
    │  ├─ match reply.getConnect() → handleConnectReply()
    │  │  ├─ on error → fire ErrorEvent + close WS
    │  │  │  ├─ if code==109 (token expired) → refreshRequired=true, close WS
    │  │  │  └─ if temporary → close WS (reconnect)
    │  │  │  └─ if permanent → processDisconnect(false) → state=DISCONNECTED
    │  │  ├─ on success → state=CONNECTED, fire ConnectedEvent
    │  │  ├─ resubscribe all Subscriptions
    │  │  ├─ handle server-side subscriptions
    │  │  ├─ send queued commands (connectCommands, connectAsyncCommands)
    │  │  ├─ schedule ping timeout
    │  │  └─ if expires → schedule token refresh
    │  │
    │  └─ if reply.hasPush() → handlePush() → route to subscription/server events
    │
    ├─ WebSocket.onClosed(code, reason) → processDisconnect(shouldReconnect)
    │  ├─ parse WS close code → shouldReconnect decision
    │  │  ├─ code < 3000 → usually reconnect
    │  │  ├─ code 3000-3500 → no reconnect (explicit server close)
    │  │  ├─ code 3500-5000 → no reconnect
    │  │  ├─ code 4000-4500 → reconnect
    │  │  └─ code >= 5000 → reconnect
    │  ├─ if shouldReconnect → state=CONNECTING, fire ConnectingEvent
    │  └─ else → state=DISCONNECTED, fire DisconnectedEvent
    │
    └─ WebSocket.onFailure(Throwable, Response?) → processDisconnect(true) + scheduleReconnect()

CONNECTED
    ├─ .disconnect() called
    │  └─ processDisconnect(false) → state=DISCONNECTED
    ├─ .send(data, callback) → sendSynchronized()
    ├─ .rpc(method, data, callback) → rpcSynchronized()
    ├─ .publish(channel, data, callback) → publishSynchronized()
    ├─ .history(channel, opts, callback)
    ├─ .presence(channel, callback)
    ├─ .presenceStats(channel, callback)
    ├─ ping from server → handlePing() → reset ping timeout
    ├─ token expires → sendRefresh() scheduled
    │  ├─ call tokenGetter.getConnectionToken()
    │  ├─ on success → send RefreshRequest
    │  ├─ on error → schedule retry with backoff
    │  └─ on temporary error → schedule retry, on permanent → disconnect
    │
    └─ connection lost → scheduleReconnect() with exponential backoff
        └─ backoff.duration(reconnectAttempts, minDelay, maxDelay)
```

### Key State Codes

**Disconnect Codes** (passed to `DisconnectedEvent`):
- `0` (`DISCONNECTED_DISCONNECT_CALLED`): User called `.disconnect()`
- `1` (`DISCONNECTED_UNAUTHORIZED`): Auth failed (token rejected)
- `2` (`DISCONNECTED_BAD_PROTOCOL`): Malformed protobuf or SDK bug
- `3` (`DISCONNECTED_MESSAGE_SIZE_LIMIT`): Frame exceeded max size
- WS codes 3000-3999: Server errors (permanent, no reconnect)
- WS codes 1009: Message size limit exceeded

**Connecting Codes** (passed to `ConnectingEvent`):
- `0` (`CONNECTING_CONNECT_CALLED`): User called `.connect()`
- `1` (`CONNECTING_TRANSPORT_CLOSED`): WS closed unexpectedly
- `2` (`CONNECTING_NO_PING`): No ping received for `pingInterval + maxServerPingDelay`
- `3` (`CONNECTING_SUBSCRIBE_TIMEOUT`): Subscribe command timeout
- `4` (`CONNECTING_UNSUBSCRIBE_ERROR`): Unsubscribe command error

### Reconnection Logic

```java
private void scheduleReconnect() {
    if (getState() != CONNECTING) return;
    long delayMs = backoff.duration(
        reconnectAttempts,
        opts.getMinReconnectDelay(),
        opts.getMaxReconnectDelay()
    );
    reconnectTask = scheduler.schedule(
        this::startReconnecting,
        delayMs,
        TimeUnit.MILLISECONDS
    );
    reconnectAttempts++;  // incremented on schedule, not on attempt
}
```

**Backoff Calculation**: Uses exponential backoff with jitter. Reset to 0 when successfully connected (`reconnectAttempts = 0` in `handleConnectReply`).

### Processing Disconnects

```java
void processDisconnect(int code, String reason, Boolean shouldReconnect) {
    // Cancels ALL scheduled tasks
    pingTask.cancel(true);
    refreshTask.cancel(true);
    reconnectTask.cancel(true);
    
    // Move all subscriptions to SUBSCRIBING if reconnecting
    if (shouldReconnect) {
        for (Subscription sub : subs.values()) {
            if (sub.getState() != UNSUBSCRIBED) {
                sub.moveToSubscribing(SUBSCRIBING_TRANSPORT_CLOSED, "transport closed");
            }
        }
    }
    
    // Fail all pending futures (sends/rpcs/etc)
    for (CompletableFuture<Reply> f : futures.values()) {
        f.completeExceptionally(new IOException());
    }
    
    // Notify server-side subscriptions
    if (previousState == CONNECTED && serverSubs.size() > 0) {
        for (String channel : serverSubs.keySet()) {
            listener.onSubscribing(this, new ServerSubscribingEvent(channel));
        }
    }
    
    setState(shouldReconnect ? CONNECTING : DISCONNECTED);
    ws.close(NORMAL_CLOSURE_STATUS, null);
}
```

---

## Subscription Lifecycle

### State Machine

```
┌────────────────────────────────────────────────────────────────┐
│  SubscriptionState { UNSUBSCRIBED, SUBSCRIBING, SUBSCRIBED }   │
└────────────────────────────────────────────────────────────────┘

UNSUBSCRIBED (initial)
    ↓ .subscribe() called
SUBSCRIBING
    ├─ fire SubscribingEvent(SUBSCRIBING_SUBSCRIBE_CALLED)
    ├─ if token needed → call tokenGetter.getSubscriptionToken(channel)
    │  ├─ on UnauthorizedException → failUnauthorized(true) → UNSUBSCRIBED
    │  └─ on success → sendSubscribe() with token
    │
    ├─ send SubscribeRequest over WebSocket
    │  ├─ recovery params included (recover=true, offset, epoch, positioned)
    │  ├─ delta negotiation enabled if opts.getDelta()=true
    │  └─ join/leave tracking enabled if opts.isJoinLeave()=true
    │
    ├─ wait for SubscribeReply (with timeout = opts.getTimeout())
    │  ├─ on error (temporary) → scheduleResubscribe() with backoff
    │  │  └─ resubscribeAttempts++
    │  ├─ on error code 109 (token expired) → token="", scheduleResubscribe()
    │  ├─ on error (permanent) → failUnauthorized() → UNSUBSCRIBED
    │  └─ on success → moveToSubscribed()
    │
    └─ [TRANSPORT_CLOSED during SUBSCRIBING]
       └─ moveToSubscribing(SUBSCRIBING_TRANSPORT_CLOSED) → stay SUBSCRIBING
          └─ resubscribeIfNecessary() when client reconnects

SUBSCRIBED
    ├─ fire SubscribedEvent(wasRecovering, recovered, positioned, recoverab, streamPos, data)
    ├─ receive publications via onPublication()
    ├─ receive join/leave events (if enabled)
    ├─ if token expires → sendRefresh() scheduled
    │  ├─ call tokenGetter.getSubscriptionToken(channel)
    │  ├─ on error → schedule retry with backoff
    │  └─ on success → send SubRefreshRequest
    │
    ├─ .publish(data, callback) → queued until SUBSCRIBED
    ├─ .history(opts, callback) → queued until SUBSCRIBED
    ├─ .presence(callback) → queued until SUBSCRIBED
    ├─ .presenceStats(callback) → queued until SUBSCRIBED
    │
    ├─ .unsubscribe() called
    │  └─ _unsubscribe(true, UNSUBSCRIBED_UNSUBSCRIBE_CALLED) → UNSUBSCRIBED
    │     ├─ sends UnsubscribeRequest
    │     ├─ clears all pending futures
    │     └─ fire UnsubscribedEvent
    │
    └─ server-initiated unsubscribe (Unsubscribe push)
       ├─ if code < 2500 → moveToUnsubscribed(false, code, reason)
       └─ if code >= 2500 → moveToSubscribing(code, reason) → resubscribe

UNSUBSCRIBED (final)
    └─ .subscribe() allowed (restart)
```

### Recovery Mechanism

**When**: If `opts.isRecoverable()=true`, subscription includes recovery params in SubscribeRequest.

```java
if (isRecover) {
    builder.setRecover(true)
           .setEpoch(streamPosition.getEpoch())
           .setOffset(streamPosition.getOffset());
}
```

**Server Response** (in `SubscribeResult`):
- `wasRecovering`: Was this subscription recovering?
- `recovered`: Did recovery succeed (all publications available)?
- `positioned`: Are offset/epoch valid (positioned subscription)?
- `recoverab`: Can future recoveries happen from this epoch?
- `publications`: Recovered publications (if `recovered=true`)

**Behavior**:
- If recovery succeeds → receive `recovered=true`, publications delivered
- If recovery fails → receive `recovered=false`, older publications lost (gap)
- If positioned → can use new offset/epoch for next recovery

### Delta Compression

**When**: If server sends `Publication.delta=true` and client negotiated delta (`SubscribeResult.delta=true`).

```java
if (deltaNegotiated) {
    byte[] prevData = getPrevData();
    if (prevData != null && pub.getDelta()) {
        pubData = Fossil.applyDelta(prevData, pubData);
    }
    setPrevData(pubData);  // ALWAYS updated, even if delta failed
}
```

**Critical**: `setPrevData()` is called unconditionally after decoding. If delta application fails, the publication data is lost (Fossil library throws).

---

## Core Classes

### `Client`

**Purpose**: Main entry point. Manages WebSocket connection, command dispatch, subscription registry.

**Key Fields**:
```java
private WebSocket ws;                               // OkHttp3 WebSocket
private volatile ClientState state;                 // DISCONNECTED, CONNECTING, CONNECTED
private final Map<String, Subscription> subs;      // Local subscriptions
private final Map<String, ServerSubscription> serverSubs;  // Server-initiated subs
private final Map<Integer, CompletableFuture<Protocol.Reply>> futures;  // Pending commands
private final ExecutorService executor;            // Single-threaded, all logic serialized here
private final ScheduledExecutorService scheduler;  // Timers: ping, reconnect, refresh
private final Backoff backoff;                     // Reconnect delay calculation
private int reconnectAttempts;                     // Incremented on schedule (not attempt)
private boolean refreshRequired;                   // Token refresh needed after disconnect
```

**Constructor**:
```java
public Client(String endpoint, Options opts, EventListener listener)
```

**Public Methods**:
- `void connect()` - Submit to executor, then transition to CONNECTING
- `void disconnect()` - Submit to executor, then transition to DISCONNECTED (no reconnect)
- `void setToken(String token)` - Submit to executor
- `boolean close(long awaitMilliseconds)` - Disconnect + shutdown executors. Returns false if timeout. **Not reusable after this.**
- `ClientState getState()` - **Volatile, use for UI hints only, not control flow**
- `Subscription newSubscription(channel, options, listener)` throws `DuplicateSubscriptionException`
- `Subscription getSubscription(channel)` - Returns null if not exists
- `void removeSubscription(Subscription sub)` - Unsubscribes + removes from registry
- `void send(byte[] data, CompletionCallback cb)` - Async message (no reply expected)
- `void rpc(String method, byte[] data, ResultCallback<RPCResult> cb)` - Async RPC
- `void publish(String channel, byte[] data, ResultCallback<PublishResult> cb)` - Publish without subscription
- `void history(String channel, HistoryOptions opts, ResultCallback<HistoryResult> cb)`
- `void presence(String channel, ResultCallback<PresenceResult> cb)`
- `void presenceStats(String channel, ResultCallback<PresenceStatsResult> cb)`

**Limitations**:
- **Single executor**: All internal logic runs sequentially on `executor`. Callbacks are NOT run on UI thread.
- **Command queueing**: Commands sent while CONNECTING are queued in `connectCommands` (sync) or `connectAsyncCommands` (async). Sent once connected.
- **No command cancellation**: Once queued, command cannot be cancelled. Timeouts managed by futures.
- **WebSocket replaced**: Calling `_connect()` cancels previous WebSocket and creates new one.

### `Subscription`

**Purpose**: Manages subscription to a single channel. Handles recovery, token refresh, resubscription.

**Key Fields**:
```java
private volatile SubscriptionState state;          // UNSUBSCRIBED, SUBSCRIBING, SUBSCRIBED
private final Client client;                       // Reference to parent client
private final String channel;                      // Channel name
private final SubscriptionEventListener listener;  // User callbacks
private final SubscriptionOptions opts;            // Configuration
private String token;                              // Subscription token (if any)
private boolean recover;                           // Attempt recovery on resubscribe
private long offset;                               // Stream position offset
private String epoch;                              // Stream position epoch
private final Backoff backoff;                     // Resubscribe delay calculation
private boolean deltaNegotiated;                   // Server accepted delta compression
private byte[] prevData;                           // Previous publication data (for delta)
private int resubscribeAttempts;                   // Incremented on schedule
```

**Public Methods**:
- `void subscribe()` - Transition to SUBSCRIBING, start subscription flow
- `void unsubscribe()` - Transition to UNSUBSCRIBED, send UnsubscribeRequest
- `SubscriptionState getState()` - **Volatile**
- `String getChannel()` - Channel name
- `void publish(byte[] data, ResultCallback<PublishResult> cb)` - Publish to this channel
- `void history(HistoryOptions opts, ResultCallback<HistoryResult> cb)` - Get history
- `void presence(ResultCallback<PresenceResult> cb)` - Get current presence
- `void presenceStats(ResultCallback<PresenceStatsResult> cb)` - Get presence stats

**Internal Methods** (package-private, called by Client):
- `void moveToSubscribing(code, reason)` - Transition with event
- `void moveToUnsubscribed(sendUnsubscribe, code, reason)` - Transition with event
- `void moveToSubscribed(Protocol.SubscribeResult result)` - Transition, fire event, schedule token refresh
- `void handlePublication(Protocol.Publication pub)` - Decode delta, fire event
- `void subscribeError(ReplyError err)` - Handle subscribe error
- `void resubscribeIfNecessary()` - Called by client when reconnected
- `void sendRefresh()` - Token refresh timer callback

**Limitations**:
- **Queued operations**: `.publish()`, `.history()`, `.presence()`, `.presenceStats()` wait for SUBSCRIBED state. Queued via UUID-keyed futures.
- **Delta application fails silently?**: If `Fossil.applyDelta()` throws, publication is lost (no exception propagated). **Ambiguity: need to verify error handling.**
- **Resubscription automatic**: If transport closes, automatically moves to SUBSCRIBING and waits for reconnect.

### `Options`

**Fluent builder for Client configuration**:
```java
Options opts = new Options()
    .setToken("jwt-token")
    .setName("client-name")
    .setVersion("1.0")
    .setData(new byte[]{})
    .setHeaders(Map.of("Authorization", "Bearer token"))
    .setTokenGetter(connectionTokenEvent, (err, token) -> {...})
    .setOkHttpClient(okHttpClient)
    .setDns(dns)
    .setSSLSocketFactory(sslSocketFactory)
    .setTrustManager(trustManager)
    .setProxy(proxy)
    .setProxyLogin(login)
    .setProxyPassword(password)
    .setMinReconnectDelay(1000)           // ms
    .setMaxReconnectDelay(20000)          // ms
    .setMaxServerPingDelay(8000)          // ms (total ping timeout = pingInterval + this)
    .setTimeout(5000);                    // ms (command timeout)

new Client(endpoint, opts, listener);
```

**Key Behavior**:
- Immutable after Client instantiation (internally)
- `tokenGetter` called on connect/reconnect if token initially empty or expires
- `headers` added to every WebSocket upgrade request
- OkHttp customization allowed (proxy, SSL, DNS)

### `SubscriptionOptions`

**Fluent builder for Subscription configuration**:
```java
SubscriptionOptions opts = new SubscriptionOptions()
    .setToken("sub-token")
    .setData(new byte[]{})
    .setTokenGetter(subTokenEvent, (err, token) -> {...})
    .setSince(new StreamPosition(offset, epoch))  // Recovery position
    .setRecoverable(true)                         // Enable recovery
    .setPositioned(true)                          // Request positioned subscription
    .setJoinLeave(true)                           // Receive join/leave events
    .setDelta(true)                               // Negotiate delta compression
    .setMinResubscribeDelay(1000)
    .setMaxResubscribeDelay(20000);

client.newSubscription(channel, opts, listener);
```

**Key Behavior**:
- If `tokenGetter` set and token initially empty → call tokenGetter before subscribe
- If `since` set → recovery enabled automatically (recover=true in SubscribeRequest)
- If `delta=true` → client negotiates delta compression; publications may be delta-encoded

### `ServerSubscription`

**Lightweight holder for server-initiated subscriptions**:
```java
public class ServerSubscription {
    private boolean recoverable;   // Can future recoveries happen?
    private long lastOffset;       // Last known offset
    private String lastEpoch;      // Last known epoch
    // ...
}
```

**Purpose**: Centrifugal servers can initiate subscriptions (e.g., channels returned in Connect). This object tracks state.

**Lifecycle**:
1. Created during `handleConnectReply()` from `result.getSubsMap()`
2. Receives server-initiated publications
3. Removed on disconnect or when server removes it

---

## Event System

### Client Events (EventListener)

```java
public interface EventListener {
    void onConnecting(Client client, ConnectingEvent event);
    void onConnected(Client client, ConnectedEvent event);
    void onDisconnected(Client client, DisconnectedEvent event);
    void onError(Client client, ErrorEvent event);
    void onMessage(Client client, MessageEvent event);
    void onSubscribing(Client client, ServerSubscribingEvent event);  // server-initiated
    void onSubscribed(Client client, ServerSubscribedEvent event);    // server-initiated
    void onUnsubscribed(Client client, ServerUnsubscribedEvent event);
    void onPublication(Client client, ServerPublicationEvent event);  // any channel
    void onJoin(Client client, ServerJoinEvent event);
    void onLeave(Client client, ServerLeaveEvent event);
}
```

**Event Firing Order**:

1. **Connect Flow**:
   - `ConnectingEvent(CONNECTING_CONNECT_CALLED)` → `onConnecting`
   - WS opens → `handleConnectionOpen()`
   - Token needed? → call tokenGetter
   - Send `ConnectRequest`
   - Receive reply → `handleConnectReply()`
   - `ConnectedEvent` → `onConnected`
   - For each subscription → `ServerSubscribingEvent` → `onSubscribing`
   - For each server subscription → `ServerSubscribedEvent` → `onSubscribed`
   - (followed by publications if any)

2. **Disconnect Flow**:
   - If reconnecting → `ConnectingEvent` → `onConnecting`
   - If disconnected → `DisconnectedEvent` → `onDisconnected`
   - If already connected and disconnecting → for each server sub → `ServerSubscribingEvent` → `onSubscribing`

3. **Publication Flow**:
   - `ServerPublicationEvent` → `onPublication`
   - Contains: channel, data, offset, tags, ClientInfo

4. **Join/Leave Flow**:
   - `ServerJoinEvent` → `onJoin` or `ServerLeaveEvent` → `onLeave`
   - Contains: channel, ClientInfo

### Subscription Events (SubscriptionEventListener)

```java
public interface SubscriptionEventListener {
    void onSubscribing(Subscription sub, SubscribingEvent event);
    void onSubscribed(Subscription sub, SubscribedEvent event);
    void onUnsubscribed(Subscription sub, UnsubscribedEvent event);
    void onPublication(Subscription sub, PublicationEvent event);
    void onJoin(Subscription sub, JoinEvent event);
    void onLeave(Subscription sub, LeaveEvent event);
    void onError(Subscription sub, SubscriptionErrorEvent event);
}
```

**Event Firing Order**:

1. **Subscribe Flow**:
   - `SubscribingEvent(SUBSCRIBING_SUBSCRIBE_CALLED)` → `onSubscribing`
   - Token needed? → call tokenGetter
   - Send `SubscribeRequest`
   - Receive reply → `handleSubscribeReply()`
   - `SubscribedEvent(wasRecovering, recovered, positioned, recoverable, streamPos, data)` → `onSubscribed`
   - (followed by recovered publications)

2. **Publication Flow**:
   - `PublicationEvent` → `onPublication`
   - Contains: data (delta-decoded if applicable), offset, tags, ClientInfo

3. **Unsubscribe Flow**:
   - `UnsubscribedEvent(code, reason)` → `onUnsubscribed`

4. **Error Flow**:
   - `SubscriptionErrorEvent(Throwable)` → `onError`
   - Wraps: `SubscriptionTokenError`, `SubscriptionRefreshError`, `SubscriptionSubscribeError`

5. **Transport Close**:
   - `SubscribingEvent(SUBSCRIBING_TRANSPORT_CLOSED, "transport closed")` → `onSubscribing`
   - No unsubscribe event; state moves back to SUBSCRIBING, waiting for reconnect

### Event Timing

**All events fired from client's `executor` thread (single-threaded), NOT UI thread.**

- **Not safe to**: Block, sleep, or perform I/O in callback
- **Safe to call**: Client/Subscription methods (submitted to same executor)
- **UI updates**: Use Handler, post to main thread, or other async mechanism

---

## Token & Auth Flow

### ConnectionTokenGetter

```java
public interface ConnectionTokenGetter {
    void getConnectionToken(
        ConnectionTokenEvent event,
        TokenCallback callback  // (Throwable err, String token) -> void
    );
}
```

**When Called**:
1. On `connect()` if `token` is empty and `tokenGetter` set
2. On reconnect if `refreshRequired=true` (token expired, error code 109)
3. On `sendRefresh()` if token has TTL

**Callback Semantics**:
- **`callback.onToken(null, "jwt-token-string")`**: Success, use token
- **`callback.onToken(new UnauthorizedException(), null)`**: Auth failed, disconnect permanently (DISCONNECTED_UNAUTHORIZED)
- **`callback.onToken(new Exception("network"), null)`**: Temporary error, retry with backoff
- **`callback.onToken(new UnauthorizedException(), "stale-token")`**: Ambiguous (which to use?). **Behavior unclear; appears to use token if provided.**

**Critical**: Callback is NOT submitted to executor; it's called immediately from user code. If user code blocks, network thread blocks.

### SubscriptionTokenGetter

```java
public interface SubscriptionTokenGetter {
    void getSubscriptionToken(
        SubscriptionTokenEvent event,  // channel: sub.getChannel()
        TokenCallback callback
    );
}
```

**When Called**:
1. On `subscribe()` if `token` is empty and `tokenGetter` set
2. On resubscribe (after error code 109 or temporary error)
3. On `sendRefresh()` if subscription token has TTL

**Same callback semantics as ConnectionTokenGetter**.

**Critical**: If multiple subscriptions have the same `tokenGetter`, they all call it concurrently. Implementation must be thread-safe or synchronized externally.

### Token Refresh Flow

**Connection Token Refresh** (if Connect response has `expires=true`):
```
Schedule refresh after TTL seconds
  ↓ sendRefresh()
  ├─ call tokenGetter.getConnectionToken()
  ├─ on error → call listener.onError(RefreshError)
  │  ├─ if temporary → schedule retry with backoff
  │  └─ if permanent → disconnect
  └─ on success → send RefreshRequest + reschedule if still expires
```

**Subscription Token Refresh** (if SubscribeResult has `expires=true`):
```
Schedule refresh after TTL seconds
  ↓ sendRefresh()
  ├─ call tokenGetter.getSubscriptionToken(channel)
  ├─ on error → call listener.onError(SubscriptionRefreshError)
  │  ├─ if temporary → schedule retry with backoff
  │  └─ if permanent → unsubscribe
  └─ on success → send SubRefreshRequest + reschedule if still expires
```

### Failure Handling

**What happens if tokenGetter never calls callback?**

- **Connection**: Hangs in CONNECTING indefinitely (no timeout, callback-based)
- **Subscription**: Hangs in SUBSCRIBING indefinitely (no timeout, callback-based)

**Workaround**: Implement timeout in application layer; call callback from separate thread with timeout.

**What happens if tokenGetter throws exception?**

- **Caught by**: SDK does NOT catch exceptions from user callbacks
- **Result**: Uncaught exception propagates to WebSocket listener, SDK state undefined
- **Mitigation**: Wrap tokenGetter implementation in try-catch

---

## Error Taxonomy

### Exception Hierarchy

```
Throwable
├─ Exception
│  ├─ IOException              (Sent/RPC/Publish/etc failed)
│  ├─ UnauthorizedException    (Token rejected by server)
│  ├─ ConfigurationError       (Missing required option, e.g., tokenGetter)
│  ├─ UnclassifiedError        (Wraps internal exception)
│  ├─ DuplicateSubscriptionException  (newSubscription with existing channel)
│  └─ (Error classes extending Exception)
│     ├─ ReplyError            (Server error in command reply)
│     ├─ TokenError            (Error during token fetch)
│     ├─ RefreshError          (Error during connection token refresh)
│     ├─ SubscriptionTokenError
│     ├─ SubscriptionRefreshError
│     ├─ SubscriptionSubscribeError
│     └─ SubscriptionStateError
│
└─ CompletionException          (Future timeout or execution exception)
```

### Error Classes

**`ReplyError`** (server error response):
```java
public class ReplyError extends Exception {
    private int code;           // Server error code (e.g., 109 token expired)
    private String message;     // Server error message
    private boolean temporary;  // Is this a transient error?
    
    public int getCode();
    public String getMessage();
    public boolean isTemporary();  // Should retry?
}
```

**Semantics**:
- `code < 0`: Permanent (no retry)
- `code >= 0`: Check `temporary` flag
- `code == 109`: Token expired (refresh required)
- `code == 3000+`: Permanent server error (disconnect)

**`UnauthorizedException`** (token rejected):
```java
public class UnauthorizedException extends Exception { }
```

**Thrown by**: User's `tokenGetter` callback to signal auth failure.

**Result**: Client disconnects with `DISCONNECTED_UNAUTHORIZED`.

**`TokenError`** (tokenGetter failed):
```java
public class TokenError extends Exception {
    private final Throwable cause;  // Original exception from tokenGetter
}
```

**Fired in**: `ErrorEvent` via `listener.onError(this, new ErrorEvent(new TokenError(err)))`

**`RefreshError`** (connection token refresh failed):
```java
public class RefreshError extends Exception {
    private final Throwable cause;  // ReplyError or other
}
```

**Fired in**: `ErrorEvent` during refresh retry.

**`SubscriptionTokenError`**, **`SubscriptionRefreshError`**, etc.: Similar pattern for subscription-level errors.

**`ConfigurationError`** (SDK misconfiguration):
```java
public class ConfigurationError extends Exception {
    // Typically: "tokenGetter function should be provided..."
}
```

**`DuplicateSubscriptionException`** (subscription already exists):
```java
public class DuplicateSubscriptionException extends Exception { }
```

**Thrown by**: `client.newSubscription(channel, ...)` if channel already subscribed.

### Error Event Wrapping

**`ErrorEvent`**:
```java
public class ErrorEvent {
    private final Throwable exception;  // Can be any Throwable
    private final Integer statusCode;   // HTTP status code (if from HTTP error)
}
```

**How errors reach user**:
1. **Connection errors**: `listener.onError(client, new ErrorEvent(exception))`
2. **Subscription errors**: `listener.onError(sub, new SubscriptionErrorEvent(exception))`
3. **Callback errors**: `ResultCallback.onDone(error, null)` (if error not null)

### Silent Failures

**No error fired if**:
- Delta decoding fails (Fossil.applyDelta throws) → publication data lost
- User callback throws exception → SDK state may be inconsistent
- tokenGetter never calls callback → connection/subscription hangs indefinitely

---

## Threading Model

### Executor Strategy

**Client has TWO executors**:

```java
private final ExecutorService executor = Executors.newSingleThreadExecutor();
private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
```

**`executor` (single-threaded)**:
- ALL internal state mutations serialized here
- All user callbacks (`EventListener`, `SubscriptionEventListener`) run here
- All command processing (`processReply`, `handlePush`, etc.) runs here
- All subscription state transitions run here

**`scheduler` (scheduled pool)**:
- Ping timeout scheduling
- Reconnect delay scheduling
- Token refresh scheduling
- Resubscribe delay scheduling

### Thread Safety

**Safe from ANY thread**:
- `client.connect()`
- `client.disconnect()`
- `client.setToken(token)`
- `client.send(data, callback)`
- `client.rpc(method, data, callback)`
- `client.publish(channel, data, callback)`
- `client.history(channel, opts, callback)`
- `client.presence(channel, callback)`
- `client.newSubscription(channel, opts, listener)` (uses `synchronized (subs)`)
- `client.removeSubscription(sub)` (uses `synchronized (subs)`)
- `client.getSubscription(channel)` (uses `synchronized (subs)`)
- `subscription.subscribe()`
- `subscription.unsubscribe()`
- `subscription.publish(data, callback)`
- `subscription.history(opts, callback)`
- `subscription.presence(callback)`
- `subscription.presenceStats(callback)`

**NOT Safe from ANY thread**:
- Directly accessing `client.state` or `subscription.state` (use `.getState()` which is volatile)
- Accessing `client.subs` or `client.serverSubs` without synchronization
- Modifying `Options` or `SubscriptionOptions` after Client/Subscription creation

### Callback Thread

**All user callbacks run on `executor` thread**:

```java
listener.onConnected(client, event);          // executor thread
listener.onSubscribed(client, event);         // executor thread
resultCallback.onDone(null, result);          // executor thread
subscriptionEventListener.onPublication(...);  // executor thread
```

**Implications**:
- **Blocking in callback blocks all internal processing** (connect, subscriptions, pings, etc.)
- **Safe to call client/subscription methods** (submitted to same executor, queued)
- **NOT safe to update UI** (Android main thread); post to Handler/Looper
- **Network operations in callback will deadlock** if waiting for reply (executor busy)

### Shutdown

```java
public boolean close(long awaitMilliseconds) throws InterruptedException {
    disconnect();                     // Async submission
    executor.shutdown();              // No new tasks accepted
    scheduler.shutdownNow();          // Cancel all pending timers
    
    if (awaitMilliseconds > 0) {
        return executor.awaitTermination(awaitMilliseconds, MILLISECONDS);
    }
    return false;
}
```

**Behavior**:
- `shutdown()` prevents new task submissions; existing tasks complete
- `shutdownNow()` cancels pending timers
- `awaitTermination()` waits for executor to finish processing
- **Not reusable after close**: Create new Client instance if needed

---

## Known Pitfalls

### 1. Blocking in Callbacks

**Problem**: Callback runs on executor thread; blocking stalls connection.

```kotlin
// ❌ WRONG
listener.onConnected = { client, event ->
    Thread.sleep(5000)  // Blocks all processing
}

// ✅ CORRECT
listener.onConnected = { client, event ->
    handler.post {  // Post to UI thread
        updateUI()
    }
}
```

### 2. Modifying Options After Creation

**Problem**: Options are internally frozen; changes ignored.

```kotlin
// ❌ WRONG
val opts = Options().setToken("token1")
client = Client(endpoint, opts, listener)
opts.setToken("token2")  // Ignored

// ✅ CORRECT
val opts = Options().setToken("token2")
client = Client(endpoint, opts, listener)
```

### 3. Duplicate Subscriptions

**Problem**: Subscribing to same channel twice throws `DuplicateSubscriptionException`.

```kotlin
// ❌ WRONG
val sub1 = client.newSubscription("channel", listener1)
val sub2 = client.newSubscription("channel", listener2)  // Throws!

// ✅ CORRECT
val sub = client.newSubscription("channel", listener1)
// Or unsubscribe first:
client.removeSubscription(sub1)
val sub2 = client.newSubscription("channel", listener2)
```

### 4. Calling Methods Before Connected

**Problem**: `.send()`, `.rpc()`, `.publish()`, `.history()`, `.presence()` queued if called while CONNECTING.

```kotlin
// May seem wrong but is actually fine (queued)
client.connect()
client.rpc("method", data, callback)  // Queued, sent once connected

// But timeout still applies to queued commands
// If connect takes > opts.getTimeout(), command fails
```

### 5. Subscription Operations While Not Subscribed

**Problem**: `.publish()`, `.history()`, `.presence()` queued until SUBSCRIBED.

```kotlin
val sub = client.newSubscription("channel", listener)
subscription.subscribe()
sub.publish(data, callback)  // Queued until SUBSCRIBED

// If subscribe fails, callback gets SubscriptionStateError
```

### 6. TokenGetter Never Calls Callback

**Problem**: If tokenGetter doesn't call callback, connection hangs.

```kotlin
// ❌ WRONG (hangs forever)
tokenGetter = { event, callback ->
    // Forgot to call callback
}

// ✅ CORRECT
tokenGetter = { event, callback ->
    fetchTokenAsync { token, err ->
        callback(err, token)  // Always call
    }
}
```

### 7. Exception Thrown in TokenGetter

**Problem**: Exception not caught by SDK; undefined state.

```kotlin
// ❌ WRONG
tokenGetter = { event, callback ->
    throw RuntimeException("oops")  // SDK doesn't catch this
}

// ✅ CORRECT
tokenGetter = { event, callback ->
    try {
        val token = getToken()
        callback(null, token)
    } catch (e: Exception) {
        callback(e, null)
    }
}
```

### 8. Delta Decoding Failure

**Problem**: If `Fossil.applyDelta()` throws, publication is silently lost.

```kotlin
// ❌ No recovery mechanism
listener.onPublication = { sub, event ->
    // If delta applied and failed, data is null/garbage
}

// ✅ Defensive coding
listener.onPublication = { sub, event ->
    if (event.data == null) {
        // Likely delta failure; request full sync
        sub.unsubscribe()
        sub.subscribe()
    }
}
```

### 9. Not Waiting for Connected Before Operations

**Problem**: State is volatile; checking it is racy.

```kotlin
// ❌ WRONG (race condition)
if (client.getState() == CONNECTED) {
    client.rpc(...)  // May not be connected anymore
}

// ✅ CORRECT
client.rpc(...)  // Always use callbacks, not state checks
```

### 10. Reusing Client After close()

**Problem**: Executors shut down; client not reusable.

```kotlin
// ❌ WRONG
client.close(5000)
client.connect()  // RejectedExecutionException or hangs

// ✅ CORRECT
client.close(5000)
client = Client(endpoint, opts, listener)  // Create new
```

### 11. Connection Token Expires But Not Refreshed

**Problem**: If Connect response doesn't have `expires=true`, token never refreshed.

```proto
message ConnectResult {
    bool expires = X;  // If false, no refresh scheduled
    int32 ttl = Y;
}
```

**Workaround**: Implement out-of-band token refresh if server doesn't provide expires.

### 12. Server Sends Unsubscribe with Recoverable Code

**Problem**: Code >= 2500 triggers resubscribe, not unsubscribe.

```
server sends: Unsubscribe(code: 2500)
SDK response: moveToSubscribing(2500, "...") → resubscribe
```

**Behavior**: Server can force resubscribe by sending recovery code. Not typical; verify server behavior.

---

## Android Considerations

### Lifecycle Management

**Problem**: Activity/Fragment destroyed; Client callbacks cause crashes.

```kotlin
// ❌ WRONG (Activity destroyed, callback causes crash)
class MyActivity : AppCompatActivity() {
    private lateinit var client: Client
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        client = Client(endpoint, opts, object : EventListener {
            override fun onConnected(client: Client, event: ConnectedEvent) {
                // If activity destroyed, this crashes
                updateUI()
            }
        })
    }
}

// ✅ CORRECT (lifecycle-aware)
class MyActivity : AppCompatActivity() {
    private lateinit var client: Client
    private var isAlive = true
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        isAlive = true
        client = Client(endpoint, opts, object : EventListener {
            override fun onConnected(client: Client, event: ConnectedEvent) {
                if (isAlive) updateUI()
            }
        })
    }
    
    override fun onDestroy() {
        super.onDestroy()
        isAlive = false
        client.close(5000)
    }
}
```

### Background/Foreground

**Problem**: App goes to background; WebSocket may be killed by OS.

```kotlin
// ✅ RECOMMENDED
class MyApp : Application() {
    private lateinit var client: Client
    private var lifecycleCallbacks = object : ComponentCallbacks2 {
        override fun onTrimMemory(level: Int) {
            if (level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN) {
                // App in background
                client.disconnect()  // Don't reconnect
            }
        }
        
        override fun onConfigurationChanged(config: Configuration) {}
    }
    
    override fun onCreate() {
        super.onCreate()
        registerComponentCallbacks(lifecycleCallbacks)
    }
}
```

### OkHttp Integration

**Problem**: Network interceptors may interfere with WebSocket.

```kotlin
// ✅ CORRECT (custom OkHttp for Centrifuge)
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(LoggingInterceptor())
    .connectionPool(ConnectionPool(8, 5, TimeUnit.MINUTES))
    .build()

val opts = Options()
    .setOkHttpClient(okHttpClient)

client = Client(endpoint, opts, listener)
```

### ProGuard / R8

**Problem**: Protocol Buffers classes stripped by obfuscation.

```proguard
# Keep Protocol Buffer classes
-keep class io.github.centrifugal.centrifuge.internal.protocol.** { *; }
-keep class io.github.centrifugal.centrifuge.** { *; }

# Keep OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
```

### Permissions

**Required in AndroidManifest.xml**:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

---

## Recommended Patterns

### Pattern 1: Application-Scoped Client

```kotlin
class CentrifugeManager(private val context: Context) {
    private lateinit var client: Client
    private val listeners = mutableSetOf<EventListener>()
    
    fun initialize(endpoint: String, token: String) {
        val opts = Options()
            .setToken(token)
            .setTokenGetter(::refreshToken)
            .setMinReconnectDelay(1000)
            .setMaxReconnectDelay(30000)
        
        client = Client(endpoint, opts, object : EventListener {
            override fun onConnected(client: Client, event: ConnectedEvent) {
                listeners.forEach { it.onConnected(client, event) }
            }
            override fun onError(client: Client, event: ErrorEvent) {
                listeners.forEach { it.onError(client, event) }
            }
            // ... other methods
        })
        
        client.connect()
    }
    
    fun addListener(listener: EventListener) = listeners.add(listener)
    fun removeListener(listener: EventListener) = listeners.remove(listener)
    
    fun subscribeToChannel(channel: String, onPublication: (Publication) -> Unit) {
        try {
            client.newSubscription(channel, object : SubscriptionEventListener {
                override fun onSubscribed(sub: Subscription, event: SubscribedEvent) {}
                override fun onPublication(sub: Subscription, event: PublicationEvent) {
                    onPublication(event)
                }
                // ... other methods
            })
        } catch (e: DuplicateSubscriptionException) {
            // Handle
        }
    }
    
    private fun refreshToken(event: ConnectionTokenEvent, callback: TokenCallback) {
        // Fetch from server, call callback
    }
    
    fun destroy() {
        client.close(5000)
    }
}
```

### Pattern 2: ViewModel with LiveData (Android)

```kotlin
class ChatViewModel : ViewModel() {
    private val pubsLiveData = MutableLiveData<Publication>()
    val publications: LiveData<Publication> = pubsLiveData
    
    private lateinit var client: Client
    private var sub: Subscription? = null
    
    fun connect(endpoint: String, channel: String) {
        val opts = Options().setToken(getToken())
        
        client = Client(endpoint, opts, object : EventListener {
            override fun onConnected(client: Client, event: ConnectedEvent) {
                subscribeToChannel(channel)
            }
            // ... other methods
        })
        
        client.connect()
    }
    
    private fun subscribeToChannel(channel: String) {
        try {
            sub = client.newSubscription(channel, object : SubscriptionEventListener {
                override fun onPublication(sub: Subscription, event: PublicationEvent) {
                    pubsLiveData.postValue(Publication(event.data))
                }
                // ... other methods
            })
            sub?.subscribe()
        } catch (e: DuplicateSubscriptionException) {
            // Handle
        }
    }
    
    override fun onCleared() {
        sub?.unsubscribe()
        client.close(5000)
    }
}
```

### Pattern 3: Reactive with RxJava

```kotlin
class CentrifugeRepository(private val client: Client) {
    fun subscribeToChannel(channel: String): Observable<Publication> {
        return Observable.create { emitter ->
            try {
                val sub = client.newSubscription(
                    channel,
                    object : SubscriptionEventListener {
                        override fun onSubscribed(sub: Subscription, event: SubscribedEvent) {
                            emitter.onNext(Publication(isSubscribed = true))
                        }
                        override fun onPublication(sub: Subscription, event: PublicationEvent) {
                            emitter.onNext(Publication(data = event.data))
                        }
                        override fun onError(sub: Subscription, event: SubscriptionErrorEvent) {
                            emitter.onError(event.exception)
                        }
                        override fun onUnsubscribed(sub: Subscription, event: UnsubscribedEvent) {
                            emitter.onComplete()
                        }
                        // ... other methods
                    }
                )
                sub.subscribe()
                emitter.setCancellable { sub.unsubscribe() }
            } catch (e: Exception) {
                emitter.onError(e)
            }
        }
    }
}
```

### Pattern 4: Error Handling with Retry

```kotlin
fun <T> executeWithRetry(
    operation: () -> Unit,
    maxRetries: Int = 3,
    backoffMs: Long = 1000
) {
    var attempts = 0
    
    fun retry() {
        if (attempts >= maxRetries) {
            // Give up
            return
        }
        attempts++
        Thread.sleep(backoffMs * attempts)  // Simple backoff
        operation()
    }
    
    client.rpc("method", data) { error, result ->
        if (error != null && error is ReplyError && error.isTemporary) {
            retry()
        } else if (error != null) {
            // Permanent error
            handleError(error)
        } else {
            // Success
            handleSuccess(result)
        }
    }
}
```

### Pattern 5: Channel Multiplexing

```kotlin
class MultiChannelListener(private val onData: (String, ByteArray) -> Unit) : SubscriptionEventListener {
    override fun onPublication(sub: Subscription, event: PublicationEvent) {
        onData(sub.getChannel(), event.data)
    }
    // ... other methods
}

fun subscribeToChannels(vararg channels: String) {
    val listener = MultiChannelListener { channel, data ->
        // Handle data from any channel
    }
    
    channels.forEach { channel ->
        try {
            val sub = client.newSubscription(channel, listener)
            sub.subscribe()
        } catch (e: DuplicateSubscriptionException) {
            // Already subscribed
        }
    }
}
```

---

## Ambiguities & Gaps in Source

### 1. Delta Decoding Failure

**Source**: `Subscription.handlePublication()` calls `Fossil.applyDelta()` but no try-catch.

**Behavior on exception**: **NOT DOCUMENTED**. Appears to propagate uncaught, potentially killing callback.

**Recommendation**: Verify `Fossil` source code; add defensive null checks in callback.

### 2. TokenGetter Callback Never Called

**Timeout**: **NONE**. If tokenGetter submits callback to async queue and that queue is full, callback may never execute.

**Behavior**: Connection/subscription hangs indefinitely.

**Recommendation**: Implement application-level timeout; call callback with error after timeout.

### 3. TokenCallback With Both Error and Token

**Source**: TokenCallback interface allows both `error != null` and `token != null`.

**Behavior**: **AMBIGUOUS**. Code appears to prioritize error, but not explicitly handled.

**Recommendation**: Never call callback with both set; use one or the other.

### 4. Unsubscribe Code >= 2500

**Behavior**: If server sends `Unsubscribe` push with code >= 2500, SDK treats it as recoverable and resubscribes.

**Why**: Codes < 2500 are permanent; >= 2500 are transient.

**Concern**: Server could force infinite resubscription loops if not careful.

### 5. Connection Token Refresh Schedule

**Source**: If `ConnectResult.expires=false`, no refresh is scheduled.

**Behavior**: Token never refreshed, even if it expires later.

**Concern**: Out-of-band token expiration not handled.

### 6. Concurrent TokenGetters

**Behavior**: If multiple subscriptions share same `tokenGetter`, they all call it concurrently.

**Thread safety**: Application's `tokenGetter` must be thread-safe or synchronized.

**Recommendation**: Use single thread or concurrent queue for token refresh requests.

---

## Summary of Key Takeaways

1. **Single-Threaded Executor**: All logic runs sequentially on `executor`. Callbacks not on UI thread.
2. **Command Queueing**: Commands sent while connecting are queued and sent once connected.
3. **No Explicit State Checks**: Use callbacks instead of polling `getState()`.
4. **Token Refresh Callback-Driven**: No timeout; if callback never called, hangs indefinitely.
5. **Delta Decoding Silent Failure**: No exception handling visible; publication lost if delta fails.
6. **Reconnection Automatic**: With exponential backoff, resets on successful connect.
7. **Server-Initiated Subscriptions**: Channels in Connect response auto-managed via `ServerSubscription`.
8. **Recovery Selective**: Only subscriptions with `recoverable=true` attempt to recover positions.
9. **Ping Timeout**: If no ping received for `pingInterval + maxServerPingDelay`, disconnects.
10. **Close Non-Reusable**: After `close()`, create new Client; executors shut down permanently.

---

**End of Knowledge Base**
