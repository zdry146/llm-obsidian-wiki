---
title: "mu-server 【关键】线程模型 (event loop + executor + block())"
category: synthesis
tags: [java, netty, mu-server, threading, async, concurrency]
sources: ["mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "mu-server 最重要的发明: Netty event loop 只负责拆消息, user handler 在独立 ExecutorService 跑, 通过 HttpExchange.block() 跨线程同步回写响应"
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

# Draft 07 — Threading Model (KEY)

> This is the keystone of mu-server: the `block()` pattern.

## 1. The two thread pools

* **NIO event loop (`NioEventLoopGroup` workerGroup, default `min(16, cores*2)` threads)** — runs all
  `Http1Connection` and `Http2Connection` handlers. Configurable via `MuServerBuilder.withNioThreads(int)`
  (MuServerBuilder.java:209-212).
* **Handler executor (default `ThreadPoolExecutor(8, 400, 60s, SynchronousQueue, "muhandler")`)** —
  runs user code (handlers, route matching, response writing, body reading callbacks).
  Configurable via `MuServerBuilder.withHandlerExecutor(ExecutorService)`.

The split is documented in the builder (MuServerBuilder.java:200-204):
> "Generally only a small number is required as NIO threads are only used for non-blocking reads and
> writes of data. Request handlers are executed on a separate thread pool… Note that websocket
> callbacks are handled on these NIO threads."

## 2. `HttpExchange.block(...)` — the bridge

`HttpExchange.java:63-98`:

```java
void block(Runnable runnable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<?> task = ctx.executor().submit(runnable);
    try {
        task.get();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new UncheckedIOException(new InterruptedIOException("Interrupted while writing"));
    } catch (ExecutionException e) {
        Throwable cause = requireNonNull(e.getCause(), "ExecutionException had no cause");
        if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
        } else {
            throw new MuException("Error while writing response", cause);
        }
    }
}

void block(Callable<ChannelFuture> callable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<ChannelFuture> task = ctx.executor().submit(callable);
    try {
        task.get().sync();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new UncheckedIOException(new InterruptedIOException("Interrupted while writing"));
    } catch (ExecutionException e) { /* same as above */ }
}
```

### Why this matters

* The handler thread is calling something like `response.write("hello")` (NettyResponseAdaptor.java:339).
  Internally that goes to `writeOnLoop(text)` which uses `ctx.writeAndFlush(...)` — but `writeAndFlush`
  is **only safe to call from the event loop**. So mu-server hops the call onto the event loop via
  `ctx.executor().submit(...)` and then `Future.get()` on the handler thread until the event loop
  has finished the write.
* The assertion `assert !inLoop()` (line 65) ensures nobody accidentally calls `block()` from inside
  the event loop (which would deadlock — the event loop thread would be waiting for itself).
* `block()` is synchronous from the caller's perspective. The whole `NettyHandlerAdapter.onHeaders`
  method effectively runs synchronously on the handler thread even though the I/O happens on the
  event loop.

### Where it is called

Every public method on `MuResponse` that wants to actually emit data ends in `block(...)`:
* `write(String)` → `exchange().block(() -> writeOnLoop(text).addListener(...))` (line 341).
* `sendChunk(String)` → `exchange().block(() -> ... writeAndFlush ...)` (line 219).
* `outputStream(int)` → `exchange().block(() -> { startStreaming(); ... })` (line 267).
* `redirect(URI)` is fire-and-forget when called from outside the loop (line 368-375).
* `NettyHandlerAdapter.onHeaders` itself ends with `muCtx.block(muCtx::complete)` (NettyHandlerAdapter.java:54).

### The pipeline round-trip

For a synchronous request, the flow per request is:

```
NIO thread                                Handler thread
---------                                  -------------
Http1Connection.channelRead0
  -> HttpExchange.create
    -> executor.execute(...) ----------------------------> NettyHandlerAdapter.onHeaders
                                                          for h in handlers: h.handle(req, resp)
                                                            -> response.write("hello")
                                                               -> exchange.block(() -> writeAndFlush)
                                                                  -> ctx.executor().submit(...)
                                                                  <- task.get()  [blocks]
                                                                  [write completes]
                                                            -> response.flushAndCloseOutputStream()
                                                          -> muCtx.block(muCtx::complete)
                                                            -> ctx.executor().submit(complete)
                                                            <- task.get()  [blocks]
                                                          handler thread exits
Http1Connection state change listener runs on event loop,
  nulls currentExchange, calls ctx.channel().read()
```

So **one synchronous request → two event-loop round trips** (write, complete). The async path skips
both.

## 3. Async path — `AsyncHandle`

When a handler calls `request.handleAsync()` (NettyRequestAdapter.java:317-326) it gets back an
`AsyncHandleImpl` (line 482-559) that holds a reference to the `HttpExchange`. The handler can then
return immediately and finish later via `asyncHandle.complete()` / `complete(Throwable)` / `write(...)`.
The state machine in `NettyHandlerAdapter.onHeaders` skips the `flushAndCloseOutputStream` and
`block(muCtx::complete)` calls (line 52-55) when `request.isAsync()` is true.

For chunked SSE output, `AsyncSsePublisher` uses the same handle with a callback pattern
(SsePublisher.java:152-164) — every `send()` blocks on `asyncHandle.write(buf).get()`, which itself
goes via `NettyResponseAdaptor.writeAndFlush(...)` which submits to the event loop (line 159-184).

## 4. Body reading — the dual-state dance

`NettyRequestAdapter.claimingBodyRead(reader)` (line 205-221) enforces:
* The body can only be consumed by **one** reader (`IllegalStateException("body cannot be read twice")`).
* Calling from a non-event-loop thread first hops to the loop (line 209-211): if not in the loop,
  `ctx.executor().submit(() -> claimingBodyRead(reader))` and returns a future. The caller (e.g.
  `inputStream()` line 149) then `.get()`s the future.
* Once a body reader is set, `setState(RequestState.RECEIVING_BODY)` fires — that triggers the
  `RequestStateChangeListener` registered by the connection (Http1Connection.java:92-96,
  Http2Connection.java:281-285) which calls `ctx.channel().read()` so Netty pumps another chunk.

## 5. Why the assertion `!inLoop()` is the contract

Calling `block()` from inside an event loop is a deadlock hazard. `HttpExchange.complete` is
explicitly `assert inLoop() : "Not in NIO event loop"` (line 139) — the **inverse** assertion for the
inverse direction. Together these two assertions encode the contract:

| Caller's thread | Allowed call | Blocked call |
|---|---|---|
| Event loop | `exchange.complete()`, `response.writeOnLoop(...)`, `response.setWebsocket()`, etc. | `block(...)`, `response.write(String)` |
| Worker thread | `response.write(String)`, `response.sendChunk(String)`, `response.outputStream()` | Direct write to channel |

`NettyResponseAdaptor.writeAndFlush(ByteBuffer)` (line 159-184) shows the dual implementation:
* If `inLoop()` → write immediately, return future.
* Otherwise → submit a task that calls itself and pipe the result back through a `ChannelPromise`.

## 6. Implications and limitations

* **Per-request throughput cap.** A 400-thread `muhandler` pool means at most ~400 blocking reads in
  flight. If a client opens 1000 slow connections, the pool saturates and Netty's `workerGroup`
  still has to buffer bodies.
* **No automatic context-propagation.** Because handlers run on a different thread, thread-locals set
  on the NIO thread (e.g. SLF4J MDC, OpenTelemetry context) do not transfer. The handler must
  propagate them explicitly if it cares.
* **Order is FIFO per connection.** The NIO event loop processes events in order per channel, so
  `read()` calls are effectively serialized per HTTP/1 connection.
* **Back-pressure is explicit.** `BackPressureHandler` (placed in the pipeline) plus manual
  `ctx.channel().read()` calls in `Http1Connection` (lines 64, 94, 134, 143, 151, 192, 201) keep the
  handler thread pool from being flooded by a client streaming more bytes than the handler can absorb.

## 7. Quote-worthy code

* `NettyHandlerAdapter.onHeaders` — the dispatcher (NettyHandlerAdapter.java:30-60).
* `HttpExchange.block` — the bridge (HttpExchange.java:63-98).
* `NettyResponseAdaptor.writeAndFlush(ByteBuffer)` — the dual-mode write (NettyResponseAdaptor.java:159-184).
* `NettyRequestAdapter.claimingBodyRead` — single-shot body reader (NettyRequestAdapter.java:205-221).

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
