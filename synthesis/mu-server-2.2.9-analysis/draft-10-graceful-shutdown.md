---
title: "mu-server 2.2.9 优雅关停 (stop with grace period)"
category: synthesis
tags: [java, mu-server, lifecycle, graceful-shutdown, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
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

# 10 - 优雅关闭 (Graceful Shutdown)

> mu-server 的关闭流程由 `MuServer.stop(duration, unit)` 触发。

## 入口

```java
// MuServerImpl.java:48-51
@Override
public boolean stop(long duration, TimeUnit unit) {
    return shutdown.apply(Duration.ofMillis(unit.toMillis(duration)));
}
```

`shutdown` 是个 `Function<Duration, Boolean>` 在 `MuServerBuilder.start()` (638-663) 里定义:

```java
ExecutorService finalHandlerExecutor = handlerExecutor;
Function<Duration, Boolean> shutdown = (gracefulDuration) -> {
    try {
        if (wheelTimer != null) {
            wheelTimer.stop();
        }
        for (Channel channel : channels) {
            channel.close().sync();
        }

        bossGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();

        boolean hasInFlightRequests = gracefulWait(gracefulDuration, stats);
        if (hasInFlightRequests) {
            log.info("Shutting down worker threads. Active requests: {}", stats.activeRequests());
        }

        workerGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();
        finalHandlerExecutor.shutdown();

        return hasInFlightRequests;

    } catch (Exception e) {
        log.info("Error while shutting down. Will ignore. Error was: {}", e.getMessage());
        return false;
    }
};
```

## 步骤分解

### 1. 停 `HashedWheelTimer` (用于 rate limit)
`wheelTimer.stop()` 后, 所有 pending 的 timeout 不会再触发。**已在执行中的 timer task 会完成**。

### 2. 关闭 listen socket
```java
for (Channel channel : channels) {           // 1 或 2 个 (http + https)
    channel.close().sync();
}
```
关闭 ServerSocketChannel, **不再接受新连接**。已建立的连接不受影响。

### 3. 关 boss group
```java
bossGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();
```
立即关 (quiet period = 0, timeout = 0)。**但 bossGroup 主要是 accept, 已经没什么事了**。

### 4. 等 in-flight 请求完成
```java
private boolean gracefulWait(Duration gracefulDuration, MuStatsImpl stats) throws InterruptedException {
    long endTime = System.currentTimeMillis() + gracefulDuration.toMillis();
    while (!stats.activeRequests().isEmpty() && System.currentTimeMillis() < endTime) {
        Thread.sleep(100);
    }
    return !stats.activeRequests().isEmpty();   // true = 仍有未完成请求
}
```

**轮询**: 每 100ms 看 `stats.activeRequests()`, 直到空或超时。

### 5. 关 worker group (强制)
```java
workerGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();
```
**quiet period = 0, timeout = 0** —— Netty 默认会等 2s + 15s, 这里强制立即关。
所有正在 channelRead 的 Netty worker 会被中断 (可能抛 `InterruptedException` / `IOException` 给业务)。

### 6. 关业务线程池
```java
finalHandlerExecutor.shutdown();
```
`shutdown()` 不中断正在执行的任务, 但**拒绝新任务**。业务线程池里 in-flight 的 handler 会跑完。

### 7. 返回
```java
return hasInFlightRequests;  // true = 仍有未完成请求 (即"非真正空闲")
```

## 问题分析

### 问题 1: worker 强制关闭会中断业务
```java
workerGroup.shutdownGracefully(0, 0, ms).sync();
```
quiet period 和 timeout 都是 0 → Netty 不会给在处理 I/O 的 worker 留时间。
- Netty worker 在 channelRead 时被中断 → channelRead 抛出 → 用户 handler 不会被调用
- 已经在业务线程池跑的 handler 不受影响 (`shutdown()` 不中断)

**改进建议**: 给 worker group 一个合理的 quiet period (例如 2-5s) 让 channelRead 自然结束。

### 问题 2: 没有 GOAWAY (HTTP/2)
HTTP/2 spec 要求 server 在关闭前发 GOAWAY 帧, 让客户端知道哪些 stream 会被处理。
mu-server 在 `Http2Connection.exceptionCaught()` (164-167) 和 `closeAllAndDisconnect()`
(169-175) 中会发 GOAWAY, **但 graceful shutdown 路径上没有**!

`Http2Connection.channelInactive()` (156-160) 只是减 stats + 调 super, 不发 GOAWAY。
HTTP/2 client 在 connection close 时收到的 RST_STREAM, 而**不是**有序的 GOAWAY → 可能让客户端误判丢数据。

### 问题 3: 没有"硬超时"硬切
```java
return !stats.activeRequests().isEmpty();   // true = 仍有未完成请求
```
返回值给调用者, 但调用者 (`stop(duration, unit)`) 只是返回这个值, **没有进一步动作**。
也就是说, 调 `MuServer.stop(10, SECONDS)` 后:
- 如果 10s 后还有 in-flight, **仍然继续关闭** (worker / 业务池都 shutdown)
- in-flight 请求可能:
  - 业务线程跑完, response 写到一半, 但 channel 已关 → 客户端收不完整
  - 业务线程被 interrupt → handler 抛异常 → 走 exception 处理
  - 业务线程没被 interrupt, 但 Netty channel 已关 → `writeAndFlush` 抛 `IOException`

### 问题 4: AddShutdownHook
```java
if (addShutdownHook) {
    Runtime.getRuntime().addShutdownHook(new Thread(server::stop));
}
```
如果用户开启了 shutdown hook, **JVM 退出前会调 `server.stop(0, MILLISECONDS)`**。
duration=0 → `gracefulWait(0, ms)` 立即返回 (没时间等)。

**结论**: 默认 `addShutdownHook(false)`, 推荐用户在容器 (k8s, systemd) 用 SIGTERM 触发停服,
然后调 `stop(30, SECONDS)` 这种。

## 与 0.0.3 的差异

| 维度 | 0.0.3 | 2.2.9 |
|---|---|---|
| 关闭 socket 后 wait 策略 | 同 | 同 (轮询 100ms) |
| Netty worker 关闭参数 | `shutdownGracefully(2, 15, s)` (有缓冲) | `shutdownGracefully(0, 0, ms)` (硬切) |
| HTTP/2 GOAWAY | 同 | 同 (无主动 GOAWAY) |
| ShutdownHook 默认 | off | off |
| RateLimit wheelTimer 关 | 同 | 同 |
| 业务线程池关 | `shutdown()` 不中断 | `shutdown()` 不中断 |

**2.2.9 在 worker 关闭上更激进**——这是 0.0.3 → 2.2.9 期间的可见回归 (或"硬化"),
但对于 K8s 这类 "30s graceful 然后 SIGKILL" 的环境反而是好事。

## 设计观察

1. **简洁但有缺口**: 不发 GOAWAY, 不硬超时取消——对于 HTTP/2 + 严格 SLO 场景不够。
2. **Netty worker 强制 0s 关闭**——在快速关闭容器时反而是个 feature (减少僵尸进程)。
3. **业务线程 `shutdown()` 不中断**——保证 in-flight 业务逻辑跑完, 即使响应发不出去。
4. **依赖外部 SLO 机制**——K8s readiness probe, SIGTERM, preStop hook 才是真正兜底。
5. **可观测性**: 返回 `hasInFlightRequests` 让监控知道"是否干净关闭", 但需要外部 polling 才知道。

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
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
