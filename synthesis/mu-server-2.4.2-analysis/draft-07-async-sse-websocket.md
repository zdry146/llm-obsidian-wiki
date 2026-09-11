---
title: "mu-server 2.4.2 异步 / SSE / WebSocket"
category: synthesis
tags: [java, mu-server, async, sse, websocket, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "AsyncHandle 异步模式, SsePublisher/AsyncSsePublisher 长连接, WebSocket support with BaseWebSocket / MuWebSocketSessionImpl"
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.82
lifecycle: draft
lifecycle_changed: 2026-09-11
created: 2026-09-11
updated: 2026-09-11
---

# Draft 07 — Async, SSE, WebSocket, Rate Limiting, Stats

> Scope: the higher-level patterns built on top of the protocol + dispatch
> layers.

## 1. Three modes of request handling

mu-server supports three mutually-exclusive modes per request:

| Mode | Trigger | Returns | Notes |
|---|---|---|---|
| **Synchronous** | Default | `void` / `boolean` | Handler runs on `muhandler` thread pool |
| **Asynchronous (`AsyncHandle`)** | `request.handleAsync()` | `true` once async; call `handle.complete()` later | Used by SSE, custom WebSocket handlers |
| **Async streaming (listener)** | `handleAsync().setReadListener(...)` | Callback-based body read | Servlet-3-style |

The state machine in `NettyRequestAdapter`:
- Initial: `state = HEADERS_RECEIVED`
- After `claimingBodyRead(...)`: `state = RECEIVING_BODY`
- After final body byte: `state = COMPLETE`
- On error/cancel: `state = ERRORED`

`isAsync()` is true iff the user has called `handleAsync()`. The
dispatcher (`NettyHandlerAdapter.onHeaders`) enforces: if a handler
calls `handleAsync()` and returns `false`, the next handler must not be
called (line 43 throws `IllegalStateException`).

## 2. `AsyncHandle` (59 lines) — six methods

```java
public interface AsyncHandle {
    void setReadListener(RequestBodyListener readListener); // body chunks
    void complete();                                         // finish response
    void complete(Throwable throwable);                      // finish w/ error
    void write(ByteBuffer data, DoneCallback callback);      // async write
    Future<Void> write(ByteBuffer data);                     // async write w/ future
    void addResponseCompleteHandler(ResponseCompleteListener l);
}
```

The implementation `NettyRequestAdapter.AsyncHandleImpl`:

- `setReadListener` → creates a `RequestBodyReader.ListenerAdapter` (which
  converts each `ByteBuf` chunk to a `ByteBuffer` via `nioBuffer()`).
- `complete` → schedules on event loop if off-loop, else calls
  `httpExchange.complete()` directly.
- `complete(Throwable)` → either `complete()` or, if there's a throwable,
  `NettyHandlerAdapter.useCustomExceptionHandlerOrFireIt(...)`.
- `write` (both overloads) → delegates to
  `NettyResponseAdaptor.writeAndFlush(ByteBuffer)`.

## 3. `DoneCallback` — the single-method callback

```java
public interface DoneCallback {
    void onComplete(Throwable error);
}
```

Used everywhere a future/listener is needed: body read completion,
write completion, exception firing. The implementation often wraps a
Netty `ChannelFuture.addListener` or `CompletableFuture.whenComplete`.

## 4. SSE — two flavours, blocking vs callback

### 4.1 `SsePublisher` (sync, 200 lines)

Frame format:

```
id: <id>\n                              (optional)
event: <event>\n                          (optional)
data: <line1>\n                           (one line per source line)
data: <line2>\n
\n                                       (event terminator)
```

Comments use `:` prefix; reconnect time uses `retry:` prefix. Newline
characters in `event` / `id` / `comment` cause `IllegalArgumentException`
(deliberately — SSE protocol violation).

Each `send(...)` call blocks on `asyncHandle.write(buf).get()`. If the
write fails (client disconnected), `close()` is called and an
`IOException` is thrown.

**Use case:** few subscribers, slow publishers (database queries per
event), where blocking is acceptable.

### 4.2 `AsyncSsePublisher` (193 lines)

Same protocol, but `send(...)` returns `CompletionStage<?>` instead of
blocking. Also installs an auto-close via
`addResponseCompleteHandler(info -> ssePublisher.close())` so the SSE
stream ends cleanly on response completion.

**Use case:** many subscribers, fast publishers (broadcasters).

### 4.3 JAX-RS SSE (`rest/` package)

For JAX-RS endpoints annotated with `@Produces(SERVER_SENT_EVENTS)`,
the REST layer implements `SseEventSink` and `SseBroadcaster`:

- `JaxSseImpl` — `@Context Sse` injection.
- `JaxSseEventSinkImpl` — per-connection sink (one `AsyncHandle`).
- `SseBroadcasterImpl` — fan-out from one event to many sinks (a
  `CopyOnWriteArrayList<SseEventSink>`).

The JAX-RS implementation reuses `AsyncSsePublisher` internally.

## 5. WebSocket internals

### 5.1 `WebSocketHandler.handle` flow

```java
if (request.method() != Method.GET) return false;
if (Mutils.hasValue(path) && !path.equals(request.relativePath())) return false;
if (!isWebSocketUpgrade(request)) return false;     // checks Upgrade: websocket
HttpHeaders nettyHeaders = new DefaultHttpHeaders();
Http1Headers responseHeaders = new Http1Headers(nettyHeaders);
MuWebSocket muWebSocket = factory.create(request, responseHeaders);
if (muWebSocket == null) return false;              // factory declined
NettyRequestAdapter reqImpl = (NettyRequestAdapter) request;
try {
    return reqImpl.websocketUpgrade(muWebSocket, nettyHeaders, idle, ping, maxFrame);
} catch (UnsupportedOperationException e) {        // version mismatch
    response.status(426);
    response.headers().set(HeaderNames.SEC_WEBSOCKET_VERSION, "13");
    return true;
}
```

Note: `MuWebSocketFactory.create` can return `null` — the request
doesn't have to be a WebSocket. This is how the factory can inspect
cookies/auth before deciding.

### 5.2 `NettyRequestAdapter.websocketUpgrade`

1. Build `WebSocketServerHandshakerFactory(maxFramePayloadLength)`.
2. Get a `WebSocketServerHandshaker` (Netty's WS protocol impl).
3. Replace the pipeline's `idle` handler with a new `IdleStateHandler`
   using the WebSocket idle/ping config (replacing the HTTP idle config).
4. Create `MuWebSocketSessionImpl(ctx, muWebSocket, connection)`.
5. `handshaker.handshake(...)` and on success fire
   `ExchangeUpgradeEvent(session)`.

### 5.3 `MuWebSocketSessionImpl` (313 lines)

Wraps the `ChannelHandlerContext` + `MuWebSocket`. Implements
`MuWebSocketSession` and `Exchange`. Has:

- `WebsocketSessionState`: NOT_STARTED, OPEN, SERVER_CLOSING, SERVER_CLOSED, CLIENT_CLOSING, CLIENT_CLOSED, TIMED_OUT, ERRORED.
- `ContinuationState`: NONE, TEXT, BINARY — for fragmented messages.
- `sendText/sendBinary/sendPing/sendPong/close` — all async.
- Frame-by-frame handling for inbound: `TextWebSocketFrame`,
  `BinaryWebSocketFrame`, `ContinuationWebSocketFrame`, `CloseWebSocketFrame`,
  `PingWebSocketFrame`, `PongWebSocketFrame`.

The implementation uses Netty's WebSocket protocol codecs (the handshaker
installs them). Ping frames are emitted after `pingAfterWriteMillis` of
inactivity (via the new `IdleStateHandler`).

## 6. Rate limiting

### 6.1 The flow

`MuServerBuilder.withRateLimiter(RateLimitSelector)` registers one
limiter. Internally:

1. Create `HashedWheelTimer` ("mu-limit-timer" thread).
2. For each limiter, wrap selector + timer in `RateLimiterImpl`.
3. Store in `ServerSettings.rateLimiters`.

On each request, `ServerSettings.block(MuRequest)` (line 44-52) walks
limiters and returns true if any returns `false`.

### 6.2 `RateLimiterImpl.record(MuRequest)` (71 lines)

```java
RateLimit rateLimit = selector.select(request);
if (rateLimit == null || rateLimit.bucket == null) return true;
String name = rateLimit.bucket;
AtomicLong counter = map.computeIfAbsent(name, s -> new AtomicLong(0));
if (counter.get() >= rateLimit.allowed) {
    if (rateLimit.action == RateLimitRejectionAction.SEND_429) return false;
    // otherwise: silent drop
} else {
    counter.incrementAndGet();
    timer.newTimeout(t -> {
        long newVal = counter.decrementAndGet();
        if (newVal <= 0) map.remove(name);
    }, rateLimit.per, rateLimit.perUnit);
}
return true;
```

This is a classic **token bucket** with sliding window approximation:
the `HashedWheelTimer` decrements after `rateLimit.per` time units,
giving a roughly constant throughput.

### 6.3 `RateLimitBuilder` (87 lines)

```java
RateLimitBuilder.withRate(100)         // 100 per window
   .withWindow(1, TimeUnit.SECONDS)
   .withBucket(request.remoteAddress()) // key
   .withRejectionAction(SEND_429)
   .build()
```

`RateLimitRejectionAction` enum: `SEND_429` (default) or
`READ_REQUEST_BUT_DONT_REPLY` (silent drop — useful for DDoS mitigation).

### 6.4 Multi-limiter

You can register multiple limiters (`withRateLimiter` × N). All must
allow the request — `block()` ANDs them. So `per-IP` + `per-API-key`
+ `per-route-cost` composes naturally.

## 7. Stats

### 7.1 `MuStats` interface (56 lines)

Nine counters:

- `completedConnections` — TCP-level
- `activeConnections`
- `completedRequests` — HTTP-level
- `invalidHttpRequests`
- `bytesSent` / `bytesRead` — via Netty `TrafficCounter`
- `rejectedDueToOverload` — when the handler executor threw `RejectedExecutionException`
- `failedToConnect` — exceptions in the connection-accepted pipeline
- `activeRequests` — set of `MuRequest`s

### 7.2 `MuStatsImpl` (114 lines)

Plain `AtomicLong` counters + a `ConcurrentHashMap.newKeySet()` of
in-flight `MuRequest`s. `onRequestStarted` / `onRequestEnded` are
called by the protocol layer (`HttpExchange.create`,
`Http2Connection.onHeadersRead`).

### 7.3 Per-connection stats

`Http1Connection.connectionStats` and `Http2Connection.connectionStats`
are separate `MuStatsImpl(null)` instances (no traffic counter). They
track per-connection `completedRequests`, `invalidHttpRequests`,
`rejectedDueToOverload`, `activeRequests`. Exposed via
`HttpConnection.completedRequests()` etc.

## 8. The interplay: SSE + Rate limiting + Stats

A typical SSE broadcast scenario:
- `RateLimiter` accepts the request (or 429s).
- `ServerSettings.block` passes.
- `MuStatsImpl.onRequestStarted` increments.
- Handler returns `SsePublisher.start(request, response)`.
- `handleAsync()` is called → `isAsync() = true`.
- Dispatcher doesn't call `response.complete()`.
- Publisher loops sending events from another thread.
- On close, publisher calls `asyncHandle.complete()`.
- `HttpExchange.onEnded` fires; `MuStatsImpl.onRequestEnded` runs;
  the request is removed from `activeRequests`.

If the connection closes mid-stream, `userEventTriggered` fires
`channelInactive` → `onConnectionEnded` → `onCancelled(CLIENT_DISCONNECTED)`
→ `SsePublisherImpl.send` next iteration throws `IOException`.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
