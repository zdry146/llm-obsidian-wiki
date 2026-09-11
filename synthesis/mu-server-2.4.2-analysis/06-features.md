---
title: "§6 功能特性"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "功能特性：SSE (SsePublisher 200 行 + AsyncSsePublisher 193 行, 无自动心跳) + TLS/HTTPS (HttpsConfigBuilder + 22 文件) + RateLimiter + MuStats + WebSocket + AsyncHandle。"
provenance:
  extracted: 0.95
  inferred: 0.03
  ambiguous: 0.02
  base_confidence: 0.95
lifecycle: reviewed
lifecycle_changed: 2026-09-12
created: 2026-09-11
updated: 2026-09-12
---


# 功能特性 (Features)

### 6.1 SSE — `SsePublisher.java` + `AsyncSsePublisher.java`

`SsePublisher`（200 行 public interface）：
```java
public interface SsePublisher {
    void send(String message) throws IOException;                              // 无 event 类型
    void send(String message, String event) throws IOException;                // 带 event 类型
    void send(String message, String event, String eventID) throws IOException;// 带 event + id
    void close();                                                              // 关闭流
    void sendComment(String comment) throws IOException;                       // 注释帧（保活）
    static SsePublisher start(MuRequest request, MuResponse response);         // 启动 SSE
}
```

**SSE 帧格式**（举例 `event=message` / `id=42` / `retry=3000`）：
```
event: message\n
data: {"x": 1}\n
id: 42\n
retry: 3000\n
\n
```

**注释帧（保活用）**：`sendComment(c)` → 写入 `":" + c + "\n\n"`（`SsePublisherImpl.commentText:192`）

**两阶段生命周期**：
- `SsePublisher`（同步调用）：handler 写完响应后由 `NettyHandlerAdapter` 接管
- `AsyncSsePublisher`（异步，193 行）：handler 退出后 publisher 仍存活，由 `AsyncHandle.complete()` / `close()` 控制

**⚠️ 关于"自动心跳"**：源码里**没有自动定时心跳 scheduler**。`SsePublisher` / `AsyncSsePublisher` 都不持有 `ScheduledExecutorService` 或 `heartbeatIntervalMs` 字段。心跳保活需要用户自己用 `ScheduledExecutorService` 周期调 `sendComment(": keep-alive\n")`。注释帧格式：`: <comment>\n\n`（冒号开头是 SSE 规范的注释约定）。

### 6.2 TLS / HTTPS — `HttpsConfigBuilder.java` + 22 文件

`HttpsConfigBuilder` 是 mu-server TLS 支持的总入口：

```java
public class HttpsConfigBuilder {
    public HttpsConfigBuilder withKeyStore(File keystore, String password) { ... }
    public HttpsConfigBuilder withCert(File cert, File key) { ... }
    public HttpsConfigBuilder withProtocols(String... protocols) { ... }
    public HttpsConfigBuilder withCiphers(String... ciphers) { ... }
    public HttpsConfigBuilder withNeedClientAuth(boolean need) { ... }
    public HttpsConfigBuilder withWantClientAuth(boolean want) { ... }
    ...
    public HttpsConfig build() { ... }  // → Netty SslContext
}
```

**高级特性**：
- **SNI 多证书**：支持 per-hostname 证书选择（Netty `SniHandler` + `SslContext` map）
- **客户端证书认证**：`ClientCertificateAuthentication` + OCSP stapling
- **协议白名单**：可禁用 TLS 1.0/1.1，只允许 TLS 1.2+
- **加密套件选择**：可显式指定允许的 cipher suites
- **Let's Encrypt**：`letsencrypt.org` 集成（单独包 `letsencrypt/`）

### 6.3 限流 — `RateLimiter.java`

```java
public interface RateLimiter {
    boolean tryAcquire(MuRequest request);
}

public interface RateLimitSelector {
    RateLimiter rateLimiterFor(MuRequest request);  // 按 IP / path / user 决定限流策略
}
```

典型实现：令牌桶 / 滑动窗口计数器。`MuServerBuilder.withRateLimiter(selector)` 注册全局 selector。

### 6.4 统计 — `MuStats.java` + `MuStatsImpl.java`

`MuStatsImpl`（100+ 行）：
```java
class MuStatsImpl implements MuStats {
    private final AtomicLong connectionsOpen = new AtomicLong();
    private final AtomicLong requestsHandled = new AtomicLong();
    private final AtomicLong requestsActive = new AtomicLong();
    private final AtomicLong bytesReceived = new AtomicLong();
    private final AtomicLong bytesSent = new AtomicLong();
    private final LongAdder[] statusCounts = new LongAdder[6]; // 1xx-5xx + total

    void onConnectionOpened() { connectionsOpen.incrementAndGet(); }
    void onConnectionClosed() { connectionsOpen.decrementAndGet(); }
    void onRequestStarted() { requestsActive.incrementAndGet(); }
    void onRequestEnded(MuRequest req) {
        requestsActive.decrementAndGet();
        requestsHandled.incrementAndGet();
        statusCounts[statusCodeClass(req.responseStatus())].increment();
    }
}
```

`MuServer.stats()` 暴露 `MuStats`（read-only view），用户可定期 poll 输出到 Prometheus / StatsD。

### 6.5 WebSocket — `ws/` 子包

独立子包，~10 个文件：
- `WebSocketHandler.java` — 入口
- `WebSocketSession.java` — 会话抽象
- `BaseWebSocket.java` — 给用户的同步 API（`sendText` / `sendBinary` / `onMessage`）
- `AsyncWebSocket.java` — 异步回调 API

`handlers.WebSocketHandlerBuilder` 注册 WebSocket 路由：
```java
MuServerBuilder.httpsServer()
    .addHandler(WebSocketHandlerBuilder.webSocketHandler("/ws")
        .withConnectionHandler(session -> {
            session.sendText("Welcome!");
            session.messageHandler(msg -> { ... });
        }))
```

底层走 Netty 的 `WebSocketServerProtocolHandler` + 自定义 frame decoder/encoder。

### 6.6 异步 — `AsyncHandle.java`

```java
public interface AsyncHandle {
    boolean isAsync();
    void write(String text);     // 流式追加
    void sendChunk(String text); // 同 write
    void complete();             // 结束响应
    void close();                // 主动关闭连接
}
```

实现 `AsyncHandleImpl`：内部维护一个 `boolean async`，handler 调 `request.handleAsync()` 后 `isAsync()` 返回 true，分发器不再自动 flush，由 handler 手动控制 complete/close。

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
- [[05-jax-rs|§5 JAX-RS 支持]]
- [[07-handler-library|§7 Handler 库]]
- [[08-design-patterns|§8 关键设计模式]]
- [[09-netty-comparison|§9 Netty 原生 vs mu-server 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
