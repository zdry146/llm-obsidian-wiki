---
title: "mu-server 2.2.9 【关键】线程模型 (event loop + executor + block())"
category: synthesis
tags: [java, netty, mu-server, threading, async, concurrency, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
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

# 07 - 线程模型 (Threading Model)

> mu-server 的线程模型是它区别于裸 Netty 最关键的部分。
> **核心思想**: Netty event loop 跑 I/O, 业务线程池跑 handler。

## Netty 线程池

`MuServerBuilder.start()` (627-635):
```java
NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);                   // accept
NioEventLoopGroup workerGroup = new NioEventLoopGroup(this.nioThreads);   // I/O
// nioThreads default = min(16, cpus * 2) (MuServerBuilder:48)
```

- **boss (1 个)**: 只接 `accept()` 事件
- **worker (n 个)**: 跑 `Http1Connection` / `Http2Connection` 等 inbound handlers, 处理 channelRead/writeAndFlush
- `Http1Connection.channelRead0()` 是在 worker thread 调用的

## 业务线程池 (handler executor)

`MuServerBuilder.start()` (623-627):
```java
ExecutorService handlerExecutor = this.executor;
if (handlerExecutor == null) {
    DefaultThreadFactory threadFactory = new DefaultThreadFactory("muhandler");
    handlerExecutor = new ThreadPoolExecutor(8, 400, 60, TimeUnit.SECONDS,
                                              new SynchronousQueue<>(), threadFactory);
}
```

- **核心 8, 最大 400, idle 60s, SynchronousQueue** ("muhandler-N")
- SynchronousQueue = 任务直接交给线程, 没空闲线程就 new (到 max), 没有缓冲, 拒绝时抛 `RejectedExecutionException`
- `NettyHandlerAdapter.onHeaders(muCtx)` 第一个动作就是 `executor.execute(() -> ...)`,
  把整个 handler 链的执行切到业务线程池 (NettyHandlerAdapter.java:27)

**RejectedExecutionException 处理** (HttpExchange.create, 299-304):
```java
try {
    serverStats.onRequestStarted(httpExchange.request);
    connectionStats.onRequestStarted(httpExchange.request);
    nettyHandlerAdapter.onHeaders(httpExchange);
} catch (RejectedExecutionException e) {
    serverStats.onRequestEnded(httpExchange.request);
    connectionStats.onRequestEnded(httpExchange.request);
    log.warn("Could not service " + muRequest + " because the thread pool is full so sending a 503");
    throw new InvalidHttpRequestException(503, "503 Service Unavailable");
}
```
**线程池满 → 直接 503**, 不会卡住 Netty worker。

## block() 模式 (HttpExchange.java:61-78)

这是 mu-server **最核心的同步抽象**:
```java
void block(Runnable runnable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<?> task = ctx.executor().submit(runnable);
    try {
        task.get();                          // 业务线程同步等
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new UncheckedIOException(new InterruptedIOException("Interrupted while writing"));
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
        } else {
            throw new MuException("Error while writing response", cause);
        }
    }
}
```

**意图**: 用户在业务线程调 `response.write(String)`, 但实际写 I/O 必须在 Netty event loop
(Netty 要求)。`block()` 把工作 submit 到 event loop, 然后 `task.get()` 阻塞业务线程等完成。
这样:
- **业务线程看到的是同步阻塞 API** (`write/sendChunk/writer/outputStream`)
- **I/O 仍在 event loop**, 不破坏 Netty 线程模型
- **错误传播**: event loop 上抛的异常变成 `ExecutionException`, `block()` 解包后抛回业务线程

### block(Runnable) vs block(Callable<ChannelFuture>)
```java
void block(Runnable runnable) { ... }
void block(Callable<ChannelFuture> callable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<ChannelFuture> task = ctx.executor().submit(callable);
    try {
        task.get().sync();                       // 还 .sync() 等 Netty 写完成
    } catch (InterruptedException e) { ... }
}
```
- `block(Runnable)` — 适合"切到 event loop 干个事, 不需要写结果"
- `block(Callable<ChannelFuture>)` — 适合"提交写, 等到 channel 真正 flush 完"

### 调用点
所有用户可见的同步方法都通过 `block()` 桥接:
- `NettyResponseAdaptor.write(String)` → `block(() -> writeOnLoop(text))`
- `NettyResponseAdaptor.sendChunk(String)` → `block(() -> writeAndFlush(...))`
- `NettyResponseAdaptor.outputStream(int)` → `block(() -> startStreaming())`
- `NettyResponseAdaptor.redirect(URI)` → 在 event loop 时直接调, 否则 `executor.execute(...)` 异步 (注意没有 sync 等待!)
- `NettyRequestAdapter.claimingBodyRead(...)` → 如果不在 loop, 走 `ctx.executor().submit(...)`

**注意**:`HttpExchange.java:62` 有个 `TODO`:
```java
// TODO: only use the callable version as this perhaps doesn't block until the runnable is finished? (e.g. when doing a write)
assert !inLoop() : "Should not be blocking on the event loop";
```
作者意识到 `block(Runnable)` 不保证业务真正完成 I/O, 但仍保留以兼容 `outputStream()` 这种不需要写 future 的场景。

## 异步写 (AsyncHandle.write)

```java
// NettyRequestAdapter.AsyncHandleImpl.java:528-535
@Override
public Future<Void> write(ByteBuffer data) {
    NettyResponseAdaptor response = request.httpExchange.response;
    try {
        return response.writeAndFlush(data);
    } catch (Throwable e) {
        return request.ctx.channel().newFailedFuture(e);
    }
}
```
- **不再走 `block()`**: 直接在调用线程 (业务线程) 调 `writeAndFlush`
- 但内部 `NettyResponseAdaptor.writeAndFlush(ByteBuffer data)` (149-173) 仍会检查 event loop:
  ```java
  if (!httpExchange.inLoop()) {
      ChannelPromise promise = httpExchange.ctx.newPromise();
      httpExchange.ctx.executor().submit(() -> writeAndFlush(data).addListener(f -> {
          if (f.isSuccess()) promise.setSuccess();
          else promise.setFailure(f.cause());
      }));
      return promise;
  }
  ```
  不在 loop 时, submit 到 event loop 后**立即返回 promise**, 不阻塞业务线程。

## 业务线程视角: 一个 HTTP 请求的一生

```
1. Netty worker: Http1Connection.channelRead0(ctx, HttpRequest msg)         [NIO]
2. HttpExchange.create(...) → Http1Response/Http1Headers + adapter 构造       [NIO]
3. onHeaders(httpExchange) → executor.execute(() -> ...)                    [NIO → pool]
4. muhandler.handle(req, resp) — 用户 handler                                [pool]
   4.1 req.inputStream() / readBodyAsString() — 需要时 block 到 body 完整     [pool → NIO → pool]
   4.2 resp.write("hello") — block 到 event loop 写完                         [pool → NIO → pool]
   4.3 resp.outputStream() — block 一次, 后续 write 不需要再切                [pool → NIO → pool]
5. response.flushAndCloseOutputStream() — close writer/os                    [pool]
6. muCtx.block(muCtx::complete) — 切回 event loop 完成 exchange              [pool → NIO]
7. onResponseComplete(info) — 跑 user response complete listeners            [NIO]
8. exchange addStateChangeListener → cleanup → next read()                   [NIO]
```

## WebSocket 线程

来自 `MuServerBuilder.withNioThreads` 的注释 (196-208):
> "Note that websocket callbacks are handled on these NIO threads."

也就是说 **WebSocket 的 `onText/onBinary/onConnect/onClose` 都在 Netty worker thread** 触发。
这意味着用户在这些回调里做重活会**阻塞 I/O**, 建议在自己的线程池里调度。

## SSE 线程

`SsePublisher.send()` 调用 `asyncHandle.write(buf).get()`——**业务线程**同步等 I/O。
若业务线程需要"边产生边发送" (例如定时器 / 队列), 调用方在自己的线程跑。

## 优雅关闭

`MuServerBuilder.shutdown` lambda (638-663):
1. `wheelTimer.stop()`
2. 所有 `channel.close().sync()` — 关闭监听 socket (停止接受新连接)
3. `bossGroup.shutdownGracefully(0, 0, ms).sync()` — boss 不再 accept
5. **`gracefulWait(gracefulDuration, stats)`** — 轮询 100ms 等 `stats.activeRequests()` 清空
6. `workerGroup.shutdownGracefully(0, 0, ms).sync()` — 强制关 worker
7. `finalHandlerExecutor.shutdown()` — 关业务线程池

`gracefulWait` (709-715):
```java
private boolean gracefulWait(Duration gracefulDuration, MuStatsImpl stats) throws InterruptedException {
    long endTime = System.currentTimeMillis() + gracefulDuration.toMillis();
    while (!stats.activeRequests().isEmpty() && System.currentTimeMillis() < endTime) {
        Thread.sleep(100);
    }
    return !stats.activeRequests().isEmpty();  // true = 仍有未完成请求
}
```

**没有"强制 cancel in-flight"**: graceful timeout 到了之后只是关 worker, in-flight 请求会因 worker 关闭被中断 (`IOException`)。这是个**温和但有缺陷**的实现——可能丢失响应的最后一段。

## 设计观察

1. **三线程边界**清晰: Netty worker (I/O) / 业务 pool (handler) / 用户线程 (SSE / async 内部)。
2. **同步 API 友好**: `response.write(String)` 不会让用户接触 Netty 的 Future/Listener。
3. **异步 API 也有**: `AsyncHandle.write` 返回 `Future<Void>`, 用户可注册 listener。
4. **`block()` 是核心抽象**: 单一机制同时实现 (a) 切线程 (b) 等 I/O (c) 异常传播。
5. **业务线程满 → 503** (而不是 OOM 或卡死), 是个**正确的背压策略**。
6. **WebSocket 在 Netty 线程上跑回调**——用户需要小心, 这跟"handler 在业务线程"的抽象不一致。
7. **优雅关闭有缺口**: 等待超时后强制关闭 worker, in-flight 请求可能中断。生产环境可考虑加个"硬超时"硬切。

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
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
