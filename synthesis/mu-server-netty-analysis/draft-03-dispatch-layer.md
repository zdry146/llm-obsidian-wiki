---
title: "mu-server 分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)"
category: synthesis
tags: [java, mu-server, dispatch, builder-pattern, routing]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "核心调度器 NettyHandlerAdapter 在独立 ExecutorService 跑 user handler, MuServerBuilder fluent API (864 行, 34.7 KB), 路由系统 PathMatch/Matcher"
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

# Draft 03 — Dispatch Layer

> Source files: `NettyHandlerAdapter.java`, `MuServerBuilder.java`, `MuServerImpl.java`, `Routes.java`,
> `RouteHandler.java`, `MuHandler.java`, `ContextHandlerBuilder.java`.

## 1. `MuHandler.handle(req, resp)` chain — `NettyHandlerAdapter.onHeaders()`

* `NettyHandlerAdapter.onHeaders(HttpExchange muCtx)` (NettyHandlerAdapter.java:30-60) is the central
  dispatch routine. It runs **on the worker thread pool**, not the Netty event loop.
* Step-by-step:
  1. Submit a task to `executor.execute(() -> { ... })` (line 32). Bails out early if the exchange is
     already in an end state (line 33-35).
  2. Walks the registered `MuHandler`s in registration order (line 40-48). First one to return `true`
     wins. After `handleAsync()` is called, a `false` return throws (line 45-47) — async handlers must
     finish the response themselves.
  3. If nothing handled the request, throws `NotFoundException` (line 50).
  4. If the handler returned without going async AND response not already closed,
     calls `response.flushAndCloseOutputStream()` then `muCtx.block(muCtx::complete)` (line 53-54) — the
     `block()` makes the worker thread wait for the event loop to actually write the response and end
     the exchange.
* `useCustomExceptionHandlerOrFireIt(exchange, ex)` (line 62-74) — wraps any thrown exception:
  * If `server.unhandledExceptionHandler != null` and didn't return a `RedirectionException`, run it,
    flush, then `block(complete)`.
  * Otherwise fire `exchange.fireException(ex)` which writes a 500 with an `ERR-<uuid>` token.

## 2. `Routes` — the routing helper (Routes.java:54)

* `Routes.route(method, uriTemplate, handler)` compiles the template into a `UriPattern` (URI regex),
  and returns a `MuHandler` whose `handle` checks method + path. If both match, invokes the user's
  `RouteHandler` with `(req, resp, pathParams)`.
* `UriPattern` (rest/UriPattern.java) implements RFC 6570-style URI templates with named groups,
  custom regex constraints (`{id : [0-9]+}`), and matrix parameters (line 20).
* `PathMatch.fullyMatches()` (rest/PathMatch.java) — distinguishes "fully matched" vs "prefix matched"
  (the latter is what JAX-RS sub-resource locators use).

## 3. `MuHandler` (MuHandler.java:26) — the contract

* `boolean handle(MuRequest, MuResponse)` — first-handler-true semantics (line 17-23).
* Returning `false` from a non-async handler continues the chain; returning `true` (or going async)
  short-circuits the remaining handlers.
* All `Routes.route`, `ContextHandlerBuilder`, `RestHandlerBuilder.build()`, and the various
  `*HandlerBuilder` classes produce `MuHandler`s, which are all run through the same loop in step 1.

## 4. `ContextHandlerBuilder` — context mounting (file not yet read)

* Provides `withContext(String contextPath, MuHandler handler)` so a handler is only invoked when
  the request path begins with that context, and the `MuRequest.contextPath()` / `relativePath()`
  are then adjusted (`NettyRequestAdapter.addContext` line 373-377).

## 5. `MuServerBuilder.start()` (MuServerBuilder.java:650-749) — bootstrap

* Picks an executor: by default `ThreadPoolExecutor(8, 400, 60s, SynchronousQueue, "muhandler")`
  (MuServerBuilder.java:660). A `SynchronousQueue` means threads are created on demand up to 400
  (then `RejectedExecutionException` triggers a 503 — see `HttpExchange.create` line 301-305).
* Two `NioEventLoopGroup`s: `bossGroup(1)` + `workerGroup(nioThreads)` (line 664-665). Default
  `nioThreads = min(16, availableProcessors * 2)` (line 51).
* `GlobalTrafficShapingHandler(workerGroup, 0, 0, 1000)` (line 668) — note `writeLimit=0, readLimit=0`
  means **no** traffic shaping at all. The 1000 ms is just the check interval.
* `MuStatsImpl` is backed by `trafficShapingHandler.trafficCounter()` (line 669) so `MuStats` can
  surface global bytes-sent / bytes-received.
* `shutdown = (duration) -> { ... }` (line 672-701) — a `Function<Duration, Boolean>` that:
  1. Stops the `wheelTimer` if any rate limiters were registered.
  2. Closes the listening `Channel`s.
  3. Shuts down the boss group.
  4. Calls `gracefulWait(...)` — polls until either `stats.activeRequests().isEmpty()` or timeout.
  5. Shuts down the worker group and the handler executor.
  6. Returns `!hasInFlightRequests` — false if any requests were still in flight.
* `gracefulWait` (MuServerBuilder.java:757-763) — naive `Thread.sleep(100)` polling loop. No condition
  variable / `CountDownLatch`. Adequate because the handler executor will finish pending work itself.

## 6. `createChannel` — the per-connection Netty pipeline factory

(MuServerBuilder.java:771-818 — see draft 01 for the diagram.)

* Pipeline is built **per connection** in `initChannel(SocketChannel)` (line 787).
* `idleTimeoutMills` defaults to **10 minutes** (line 66).
* `writeBufferWaterMark` is configurable (line 380, default = `WriteBufferWaterMark.DEFAULT`).
* SSL path adds `MuSniHandler` (line 796) which uses `DomainWildcardMappingBuilder` for SNI-based
  cert switching.
* A small "conerror" `ChannelInboundHandlerAdapter` (line 803) increments `server.stats.onFailedToConnect()`
  on connection-failure exceptions.

## 7. Re-entrancy and exception paths

* When `executor.execute(...)` is called and the queue is full, `RejectedExecutionException` is caught
  in `HttpExchange.create` (HttpExchange.java:301-305) **for the synchronous version** and in
  `Http2Connection.onHeadersRead` (Http2Connection.java:296-301) **for the HTTP/2 version** and
  converted to a 503. This is what protects mu-server from collapsing when overloaded.
* Async paths (`@Suspended`, `CompletionStage` returns, SSE) skip the `block(muCtx::complete)` step —
  the handler is responsible for calling `AsyncHandle.complete()` (see `RestHandler.sendResponse`
  RestHandler.java:216-227 and `AsyncHandleImpl.complete` NettyRequestAdapter.java:502-510).

## 8. Listener hookpoints

* `ResponseCompleteListener` (interface) — invoked from `NettyHandlerAdapter.onResponseComplete`
  (line 76-88) **on the worker thread** that just finished, before stats are incremented. Used for
  audit logging, metrics, etc.
* `RequestRejectListener` — invoked from `NettyHandlerAdapter.onRequestRejected` (line 90-100) when
  the protocol layer rejects (4xx from `InvalidHttpRequestException`).
* `addResponseCompleteListener(...)` / `addRequestRejectListener(...)` live on `MuServerBuilder`.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
