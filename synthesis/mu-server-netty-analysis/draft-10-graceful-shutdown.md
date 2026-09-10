---
title: "mu-server 优雅关停 (stop with grace period)"
category: synthesis
tags: [java, mu-server, lifecycle, graceful-shutdown]
sources: ["mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "MuServer.stop(duration, unit) 等 in-flight 请求完成, 超时后强制 abort 连接, 通过 ConcurrentHashMap tracks 活动连接"
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

# Draft 10 — Graceful Shutdown

## 1. User-visible contract (`MuServer.stop`)

`MuServer.java:23-40`:

```java
default void stop() { stop(0, TimeUnit.MILLISECONDS); }
boolean stop(long duration, TimeUnit unit);
```

> "Gracefully shuts down the server with a timeout. During the graceful shutdown period, the server
> will stop accepting new connections and wait for in-flight requests to complete. When timeout is
> reached and there are still in-flight requests, all the http connections will be aborted, no
> exception will be thrown. … returns false if there were in-flight requests not completed."

## 2. Implementation — `MuServerBuilder.shutdown` lambda (MuServerBuilder.java:672-701)

```java
Function<Duration, Boolean> shutdown = (gracefulDuration) -> {
    try {
        if (wheelTimer != null) {
            wheelTimer.stop();
        }
        for (Channel channel : channels) {
            channel.close().sync();   // (1) stop accepting
        }

        bossGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();   // (2)

        boolean hasInFlightRequests = gracefulWait(gracefulDuration, stats);  // (3)

        if (hasInFlightRequests) {
            log.info("Shutting down worker threads. Active requests: {}", stats.activeRequests());
        }

        workerGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();  // (4)
        finalHandlerExecutor.shutdown();                                     // (5)

        return !hasInFlightRequests;
    } catch (InterruptedException e) { ... }
};
```

## 3. Step-by-step

1. **`wheelTimer.stop()`** — stops the HashedWheelTimer used by rate limiters. After this point,
   no new rate-limit buckets can be created/cleaned.
2. **`channel.close().sync()`** for each bound channel — netty stops accepting new TCP
   connections; existing ones remain open until they finish or until step (4).
3. **`bossGroup.shutdownGracefully(0, 0, ...).sync()`** — releases the boss thread. This is
   non-graceful because by the time we are here, we have already closed the listening sockets.
4. **`gracefulWait(gracefulDuration, stats)`** (MuServerBuilder.java:757-763):

   ```java
   private boolean gracefulWait(Duration gracefulDuration, MuStatsImpl stats) throws InterruptedException {
       long endTime = System.currentTimeMillis() + gracefulDuration.toMillis();
       while (!stats.activeRequests().isEmpty() && System.currentTimeMillis() < endTime) {
           Thread.sleep(100);
       }
       return !stats.activeRequests().isEmpty();
   }
   ```

   Polls every 100 ms until either all active requests have ended (as reported by
   `MuStatsImpl.activeRequests` which is the `ConcurrentHashMap.newKeySet()` of live `MuRequest`s,
   MuStatsImpl.java:23) or the timeout elapses.
5. **`workerGroup.shutdownGracefully(0, 0, ...)`** — Netty waits up to 0 ms (immediate) for any
   outstanding tasks, then hard-stops. Because mu-server explicitly calls `ctx.channel().read()`
   (HTTP/1) / `read(ctx, streamId)` (HTTP/2), there shouldn't be unconsumed data queued.
6. **`finalHandlerExecutor.shutdown()`** — the user-handler `ThreadPoolExecutor` is told to drain
   remaining tasks (it does **not** call `shutdownNow`, so any tasks already submitted will run).
7. Returns `!hasInFlightRequests` — `false` if the timeout was hit with requests still in flight.

## 4. `MuRuntimeDelegate.ensureSet()` lifecycle hook

`HttpExchange` calls `MuRuntimeDelegate.ensureSet()` in a static block
(HttpExchange.java:39-42). This is required so that the JAX-RS layer can be loaded even after the
shutdown hook has detached some classes.

## 5. Channel-level connection drain

For HTTP/1 connections still open when the worker group shuts down:
* `Http1Connection.channelInactive` (Http1Connection.java:67-75) decrements `serverStats` and fires
  `onConnectionEnded`. If `currentExchange` is not null, calls `exchange.onConnectionEnded(ctx)`
  which propagates `ResponseState.CLIENT_DISCONNECTED`.
* For HTTP/2, `Http2Connection.channelInactive` (Http2Connection.java:159-163) calls `cleanup()`
  which iterates over every live exchange and cancels it.

So even when the graceful timeout hits, in-flight requests get a clean
`ResponseState.CLIENT_DISCONNECTED` notification.

## 6. Limitations in this 0.0.3-SNAPSHOT

* `gracefulWait` polls — no `CountDownLatch` per request to wake it immediately when the last
  exchange ends. At 100 ms granularity, shutdown may lag up to 100 ms even when there's nothing left
  to do.
* `workerGroup.shutdownGracefully(0, 0, ...)` uses a **0 ms** quiet period / timeout, so if any
  write is mid-flight when the timeout expires, it is dropped. This is what the docs mean by "all
  the http connections will be aborted".
* `finalHandlerExecutor.shutdown()` (not `shutdownNow()`) is intentional, but combined with the
  `workerGroup` immediate-shutdown it means a user handler could be blocked on `response.write()`
  while the worker group is gone — leading to `RejectedExecutionException` next time `block()`
  hops to the loop.
* The `wheelTimer` stop is best-effort: in-flight `timer.newTimeout(...)` calls already submitted
  will still run.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
