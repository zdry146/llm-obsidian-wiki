---
title: "mu-server 2.4.2 分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)"
category: synthesis
tags: [java, mu-server, dispatch, builder-pattern, routing, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "核心调度器 NettyHandlerAdapter 在独立 ExecutorService 跑 user handler, MuServerBuilder fluent API (40KB+), 路由系统 PathMatch/Matcher, URL 模板"
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

# Draft 03 — Dispatcher Layer & Routing

> Scope: `MuServer`, `MuServerBuilder`, `MuServerImpl`, `NettyHandlerAdapter`,
> `Routes`, `RouteHandler`, `MuHandler`, `ContextHandler` /
> `ContextHandlerBuilder`, `WebSocketHandler` / `WebSocketHandlerBuilder`,
> `MuWebSocketSessionImpl`, `MuWebSocketFactory`, server lifecycle, graceful
> shutdown.

## 1. The public contract — `MuServer`

`MuServer.java` (168 lines) is the runtime handle. Notable points:

- **`artifactVersion()` static** reads `/META-INF/maven/io.muserver/mu-server/pom.properties` (the Maven build injects this) and returns "0.x" if missing.
- **`stop(duration, unit)`** is a `boolean` returning false if there were in-flight requests not completed in the graceful period.
- **`changeHttpsConfig(...)`** is the only mutation exposed on a live server — see `MuServerImpl.changeHttpsConfig` (lines 124-133) which rebuilds a Netty `SslContext`, swaps it into `SslContextProvider`, and updates the HTTPS URI on `SSLInfoImpl`.
- **`activeConnections()`** returns the live `ConcurrentHashMap.newKeySet()` — safe to iterate without locking but no consistent snapshot.

## 2. The dispatcher — `NettyHandlerAdapter`

`NettyHandlerAdapter.java` (98 lines) is the **only** class that runs user
handlers. Construction in `MuServerBuilder.start()`:

```java
NettyHandlerAdapter nettyHandlerAdapter = new NettyHandlerAdapter(
    handlerExecutor, handlers, responseCompleteListeners, requestRejectListeners);
```

`handlerExecutor` defaults to:

```java
new ThreadPoolExecutor(8, 400, 60, TimeUnit.SECONDS,
    new SynchronousQueue<>(),
    new DefaultThreadFactory("muhandler"));
```

Min 8 / max 400 threads, no queue, 60s idle TTL — designed so that under
saturation the executor throws `RejectedExecutionException`, which is
caught and converted into a 503 response (see `HttpExchange.create`
lines 299-303 and the equivalent in `Http2Connection.onHeadersRead`).

### `onHeaders(HttpExchange)` — the hot path

```java
void onHeaders(HttpExchange muCtx) {
    executor.execute(() -> {                       // ← off the event loop
        if (muCtx.state().endState()) return;
        NettyRequestAdapter request = muCtx.request;
        NettyResponseAdaptor response = muCtx.response;
        try {
            boolean handled = false;
            for (MuHandler muHandler : muHandlers) {
                handled = muHandler.handle(request, response);
                if (handled) break;
                if (request.isAsync()) {
                    throw new IllegalStateException(
                        muHandler.getClass() + " returned false however " +
                        "this is not allowed after starting to handle a request asynchronously.");
                }
            }
            if (!handled) {
                throw new NotFoundException();      // → 404 via onException
            }
            if (!request.isAsync() && !response.outputState().endState()) {
                response.flushAndCloseOutputStream();
                muCtx.block(muCtx::complete);       // block on event loop
            }
        } catch (Throwable ex) {
            useCustomExceptionHandlerOrFireIt(muCtx, ex);
        }
    });
}
```

The async-after-false rule is a defensive check: if you called
`handleAsync()` you must eventually return `true` from some handler.

`useCustomExceptionHandlerOrFireIt` (lines 59-71) is the exception router:

```java
if (server.unhandledExceptionHandler != null
    && !(ex instanceof RedirectionException)
    && server.unhandledExceptionHandler.handle(exchange.request, exchange.response, ex)) {
    exchange.response.flushAndCloseOutputStream();
    exchange.block(exchange::complete);
} else {
    exchange.fireException(ex);
}
```

If the user handler returns `true` (i.e. it consumed the exception), we
flush & complete; otherwise we let `HttpExchange.fireException` do the
default 500 page (with the UUID error ID).

### `onResponseComplete` and `onRequestRejected`

Both are pure-listener dispatch — no state mutation, just call all
listeners and log failures.

## 3. Routing — the tiny DSL

`MuHandler` is just `boolean handle(MuRequest, MuResponse) throws Exception`
— 26 lines.

`Routes.route(method, uriTemplate, muHandler)` (52 lines) wraps a
`UriPattern` (JAX-RS spec §3.7.3) around the user handler. The wrapped
handler matches `method` and `uriPattern.matcher(request.relativePath()).fullyMatches()`.

`UriPattern.uriTemplateToRegex` (`UriPattern.java:102-182`) implements
exactly the algorithm from JAX-RS 2.0 §3.7.3:

1. Split template into literal + named-group parts.
2. Escape literals via `Pattern.quote(Mutils.urlEncode(literal))` (note
   `leniantUrlDecode` for `%` handling).
3. Replace each `{name}` with `(?<name>DEFAULT_REGEX|MATRIX_PARAMETERS)` —
   default regex is `[^/]+?`, matrix params are
   `(;[A-Za-z0-9-._~@!$&'()*+,=%]*)*`.
4. Repeat-name groups use back-reference `\k<name>` (line 164).
5. Strip trailing `/`, then append `(/.*)?` to allow trailing-path URIs.

The named groups are URL-decoded and split into `MuPathSegment`s via
`MuUriInfo.pathStringToSegments`.

## 4. ContextHandler (73 lines) — `addContext`

`ContextHandler` is itself an `MuHandler` that:

- Trims & URL-encodes its context (e.g. `"some context"` →
  `"some%20context"`).
- On request: if `relativePath == "/" + context`, redirect to
  `"/" + context + "/"`.
- If `relativePath` starts with `"/" + context + "/"` (or context is empty):
  - Mutates the request via `NettyRequestAdapter.addContext(context)` —
    this **adds** to `contextPath` and **trims** from `relativePath`.
  - Runs child handlers.
  - Restores the original `contextPath`/`relativePath` via
    `NettyRequestAdapter.setPaths(originalContextPath, originalRelativePath)`
    so subsequent handlers outside the context see the original path.

This push/pop discipline is unique to mu-server — it lets context handlers
nest without leaking path state.

`ContextHandlerBuilder.context("api").addHandler(...)` is the builder
pattern; it shares the `addHandler(Method, uriTemplate, RouteHandler)`
ergonomics with `MuServerBuilder`.

## 5. WebSocket upgrade flow

The WebSocket path crosses multiple files. Sequence:

```
Http1Connection.channelRead0(msg=HttpRequest)
    ↓
HttpExchange.create(...)
    ↓
NettyHandlerAdapter.onHeaders(exchange)
    ↓ (executor)
[handler loop]
    ↓
WebSocketHandler.handle(request, response)
    ├ if not GET / no Upgrade → return false (pass through)
    ├ if not WebSocket upgrade request → return false
    ├ factory.create(...) → MuWebSocket (or null → false)
    └ reqImpl.websocketUpgrade(muWebSocket, responseHeaders, ...)
            ├ WebSocketServerHandshakerFactory.newHandshaker(...)
            ├ pipeline.replace("idle", "idle", new IdleStateHandler(...))
            ├ MuWebSocketSessionImpl session = new ...
            ├ handshaker.handshake(...)
            │   └─ on success: pipeline.fireUserEventTriggered(ExchangeUpgradeEvent(session))
            └ return true
    ↓
Http1Connection.userEventTriggered(ExchangeUpgradeEvent)
    ├ success → currentExchange = session; session.onUpgradeComplete(ctx); ctx.read()
    └ failure → ctx.close()
```

Note: the WebSocket path requires HTTP/1; for HTTP/2 the
`WebSocketHandler.handle` method runs but the handshaker returns `null`
because `Http2To1RequestAdapter` doesn't carry enough upgrade info — this
is a known limitation, and the test suite exercises WebSocket only over
HTTP/1.

`MuWebSocketFactory` is the user-provided factory:

```java
public interface MuWebSocketFactory {
    MuWebSocket create(MuRequest request, Headers responseHeaders);
}
```

Returning `null` means "don't handle this request as a WebSocket".

## 6. Server startup — `MuServerBuilder.start()`

The flow is in `MuServerBuilder.java:642-733`:

1. Validate that at least one of httpPort/httpsPort is set.
2. Build `ServerSettings` (max headers, max URL, max body, gzip config, rate limiters).
3. Build the handler executor (default: cached-style thread pool).
4. Build `NettyHandlerAdapter`.
5. Create `NioEventLoopGroup` (boss=1, worker=`nioThreads`).
6. Create `GlobalTrafficShapingHandler(workerGroup, 0, 0, 1000)` — read/write
   rates of 0 mean "no shaping", but the handler is present so the
   `TrafficCounter` is exposed for `MuStats.bytesSent()` / `bytesRead()`.
7. Build the shutdown `Function<Duration, Boolean>`:
   ```java
   wheelTimer.stop();
   for (Channel ch : channels) ch.close().sync();
   bossGroup.shutdownGracefully(0, 0, MILLISECONDS).sync();
   boolean stillInFlight = gracefulWait(gracefulDuration, stats);
   workerGroup.shutdownGracefully(0, 0, MILLISECONDS).sync();
   finalHandlerExecutor.shutdown();
   return stillInFlight;
   ```
8. Build the HTTP channel (no SSL, no H2) and/or the HTTPS channel via
   `createChannel(...)`.
9. Extract the actual bound port via `getUriFromChannel(...)`.
10. Call `server.onStarted(httpUri, httpsUri, shutdown, address, sslContextProvider)`.
11. Optionally add a JVM shutdown hook.

### `gracefulWait(Duration, MuStatsImpl)` (lines 735-741)

```java
private boolean gracefulWait(Duration gracefulDuration, MuStatsImpl stats) throws InterruptedException {
    long endTime = System.currentTimeMillis() + gracefulDuration.toMillis();
    while (!stats.activeRequests().isEmpty() && System.currentTimeMillis() < endTime) {
        Thread.sleep(100);
    }
    return !stats.activeRequests().isEmpty();
}
```

100 ms polling. `stats.activeRequests()` returns the
`ConcurrentHashMap.newKeySet()` of in-flight `MuRequest`s.

## 7. Stats — `MuStatsImpl`

`MuStatsImpl` (114 lines) holds:

- 6 `AtomicLong` counters: activeConnections, totalConnections, completedRequests, invalidHttpRequests, rejectedDueToOverload, failedToConnect.
- A `Set<MuRequest> activeRequests = ConcurrentHashMap.newKeySet()`.
- A Netty `TrafficCounter` reference.

Connection stats are separate: every `Http1Connection` and `Http2Connection`
has its own `MuStatsImpl connectionStats` (initialised with
`new MuStatsImpl(null)`) to track per-connection counters that aren't
visible through `MuStats`.

## 8. Netty server bootstrap — pipeline construction

In `MuServerBuilder.createChannel()` (lines 749-789), the `ChannelInitializer`
for each accepted `SocketChannel` adds:

```
idle                           ← IdleStateHandler (all=idleTimeoutMills)
traffic-shaping                ← GlobalTrafficShapingHandler
HAProxyMessageDecoder?         ← if withHAProxyProtocolEnabled
HAProxyMessageHandler?         ← if withHAProxyProtocolEnabled
sni?                           ← if SSL, MuSniHandler with DomainWildcardMappingBuilder
BackPressureHandler?           ← if HTTP/2 enabled, before ALPN
alpn?                          ← if HTTP/2 enabled, AlpnHandler
conerror                       ← catch-all increments stats.onFailedToConnect()
[then either HTTP/2 builder or setupHttp1Pipeline()]
```

The HTTP/1 pipeline added by `setupHttp1Pipeline()` (lines 791-807):

```
decoder        ← HttpRequestDecoder(maxUrl + 17, maxHeaders, 8192)
encoder        ← HttpResponseEncoder (overridden isContentAlwaysEmpty)
compressor?    ← SelectiveHttpContentCompressor (if gzip enabled)
keepalive      ← HttpServerKeepAliveHandler
flowControl    ← MuFlowControlHandler
pressure       ← BackPressureHandler
preread        ← PreReader
muhandler      ← Http1Connection
```

`PreReader` is a small handler that reads the first byte of the request to
help `HttpRequestDecoder` (Netty best-practice for backpressure on huge
requests).

## 9. Handler ordering — async vs sync

The doc on `MuServerBuilder.addHandler(MuHandler)` says:

> "Note that handlers are executed in the order added to the builder, but
> all async handlers are executed before synchronous handlers."

But looking at `NettyHandlerAdapter.onHeaders` — it's a plain `for` loop.
The "async first" promise is **not** enforced in this file. It appears to be
aspirational documentation (or a planned optimisation that wasn't done).
This is a minor inconsistency between the API docs and the actual
implementation.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
