---
title: "mu-server 2.2.9 功能模块 (SSE/TLS/限流/统计/WebSocket)"
category: synthesis
tags: [java, mu-server, sse, tls, rate-limit, websocket, stats, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "SsePublisher/AsyncSsePublisher 长连接, HttpsConfigBuilder SSL/TLS 配置 + 证书热重载, RateLimiter, MuStatsImpl 统计, WebSocket 支持"
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

# 06 - 功能模块 (Features)

> 按特性分类梳理: HTTPS, SSE, WebSocket, Rate Limit, Stats, Exception, Async, Form/Multipart。

## HTTPS (HttpsConfigBuilder, 459 行)

`io.muserver.HttpsConfigBuilder` (459 行) 提供:

**输入源**:
- `withKeystore(File)` / `withKeystore(InputStream)` / `withKeystoreFromClasspath(String)`
- `withKeystore(KeyStore, char[] password)`
- `withSSLContext(SSLContext)` (内部, 用现成 SSLContext)
- `withKeyManagerFactory(KeyManagerFactory)`

**配置**:
- `withKeystoreType(String)` (JKS, PKCS12, JCEKS...)
- `withKeystorePassword(char[]/String)` / `withKeyPassword(char[]/String)`
- `withProtocols(String...)` (TLSv1.2, TLSv1.3 等)
- `withCipherSuite(String)` / `withCipherSuites(...)` (用 Netty `CipherSuiteFilter`)
- `withTrustManager(TrustManager)` (mTLS 用)

**快捷方式**:
- `HttpsConfigBuilder.unsignedLocalhost()` → 自签证书 (开发用)

**转换**: `toNettySslContext(boolean http2Enabled)` → `io.netty.handler.ssl.SslContext`。

**运行时换证书**: `MuServerImpl.changeHttpsConfig(HttpsConfigBuilder)` (120-129) 走
`SslContextProvider.set(newNettyCtx)`, 后续 TLS 握手用新证书。

**Let's Encrypt**: 通过外部模块 (muserver-letsencrypt) 配合, mu-server 本身不内嵌。

## SSE (SsePublisher, 200 行)

两个入口:
1. **直接用** (非 JAX-RS):
```java
SsePublisher.start(request, response);  // 静态工厂
sse.send("data");
sse.send("data", "event-name");
sse.send("data", "event-name", "event-id");
sse.sendComment("keepalive");
sse.setClientReconnectTime(3, TimeUnit.SECONDS);
sse.close();                              // 客户端会按 retry: 决定是否重连
```

2. **JAX-RS 用**: `@Context SseEventSink` (参见 05-jaxrs.md)

**消息帧构造** (`SsePublisherImpl.dataText`, 170-190):
```
id: <eventID>\n           (可选)
event: <eventName>\n      (可选)
data: <message line 1>\n
data: <message line 2>\n  (如果含换行符)
\n
```

**实现细节**: `SsePublisherImpl` 包了一个 `AsyncHandle`, 每次 `send()` 通过
`asyncHandle.write(buf).get()` **同步** 等待写入完成 (NettyResponseAdaptor.writeAndFlush)。
失败 → 调 `close()` + 抛 `IOException`。

`AsyncSsePublisher.java` 存在 (异步版本) 但需要查源码确认是否还在用。

## WebSocket

核心抽象:
- `MuWebSocket` (接口) — 用户实现, 提供 `onConnect/onBinary/onText/onPing/onPong/onClientClosed/onError`
- `BaseWebSocket` (类, 86 行) — 推荐基类, 已实现 ping→pong, error handling
- `MuWebSocketSession` — `sendText/sendBinary/sendPing/sendPong/close(state,reason)`
- `MuWebSocketSessionImpl` — Netty 实现
- `WebSocketHandler` — `MuHandler` 适配, 用户注册 `addHandler(wsHandler)`
- `WebSocketHandlerBuilder` — builder
- `WebsocketSessionState` — 状态枚举 (CONNECTED/CLOSED/TIMED_OUT/ERRORED)

**握手** (`NettyRequestAdapter.websocketUpgrade`, 391-414):
- 用 Netty 的 `WebSocketServerHandshakerFactory`
- 替换 pipeline 中的 `idle` handler 为新的 `IdleStateHandler(idleRead, pingAfterWrite, 0, ms)`
- 创建 `MuWebSocketSessionImpl`
- 触发 `ExchangeUpgradeEvent` (pipeline userEvent)

**Idle + ping**:
- HTTP/1 模式: `IdleStateHandler(0, 0, idleTimeout)` + 单独 ping after write timeout
- 配置在 `WebSocketHandlerBuilder` (`withIdleReadTimeout`, `withPingAfterWriteTimeout`, `withMaxFramePayloadLength`)

**升级后** (`Http1Connection.userEventTriggered`, 170-191):
- 收到 `ExchangeUpgradeEvent` → 把 `currentExchange` 从 `HttpExchange` 换成 `MuWebSocketSessionImpl`
- 调用 `httpExchange.response.setWebsocket()` 把 ResponseState 设为 `UPGRADED`
- 重新 read()

## Rate Limiting (RateLimiterImpl.java, 71 行 + RateLimitBuilder)

```java
.withRateLimiter(request -> RateLimit.builder()
    .withBucket(request.remoteAddress())
    .withRate(100)
    .withWindow(1, TimeUnit.SECONDS)
    .build())
```

**实现** (`RateLimiterImpl.record`, 25-49):
```java
boolean record(MuRequest request) {
    RateLimit rateLimit = selector.select(request);
    if (rateLimit == null || rateLimit.bucket == null) return true;
    String name = rateLimit.bucket;
    AtomicLong counter = map.computeIfAbsent(name, s -> new AtomicLong(0));
    long curVal = counter.get();
    if (curVal >= rateLimit.allowed) {
        if (rateLimit.action == RateLimitRejectionAction.SEND_429) return false;
        // ... CONTINUE 动作 (还没实现 503? 实际是"通过", 让业务决定)
    } else {
        counter.incrementAndGet();
        timer.newTimeout(timeout -> {
            long newVal = counter.decrementAndGet();
            if (newVal <= 0) map.remove(name);
        }, rateLimit.per, rateLimit.perUnit);
    }
    return true;
}
```

**特性**:
- 基于 `HashedWheelTimer` (Netty) 的窗口计数器
- **单 bucket 维度** (IP / user / path 等)
- **多 limiter 可叠加** (`withRateLimiter(...)` 可调多次)
- **拒绝动作**: `SEND_429` (直接拒) / `CONTINUE` (继续往下走, 让业务决定)
- **`ServerSettings.block(muRequest)`** 串所有 limiter, 任一 false → 429

## Stats (MuStats / MuStatsImpl)

- `MuStats` (接口) — `currentConnections()`, `currentRequests()`, `completedRequests()`, `invalidHttpRequests()`, `rejectedDueToOverload()`, `bytesRead()`, `bytesWritten()`, `currentRequestsByConnection(...)`
- `MuStatsImpl` — 实现, 包装 `GlobalTrafficShapingHandler.trafficCounter()`
- `MuStatsImpl.connectionStats` (per-connection) vs `serverStats` (server-wide)
- 调用方: `Http1Connection`/`Http2Connection` 的 lifecycle hook

```java
serverStats.onConnectionOpened();      // 新连接
connectionStats.onConnectionOpened();
serverStats.onRequestStarted(req);     // 新请求
connectionStats.onRequestStarted(req);
// 完成:
connectionStats.onRequestEnded(req);
serverStats.onRequestEnded(req);
// 错误:
serverStats.onInvalidRequest();        // 400/414/...
serverStats.onRejectedDueToOverload(); // 429/503
serverStats.onFailedToConnect();        // 早期失败
```

## Exception Handling
两层防御:
1. **handler 层** (`NettyHandlerAdapter.useCustomExceptionHandlerOrFireIt`):
   - 用户装 `MuServerBuilder.withExceptionHandler(handler)` → handler 可以返回 true 表示自己处理完了
   - 否则走标准 `fireException` 流程
2. **HTTP 协议层** (`HttpExchange.onException`):
   - **如果还没发响应**: 转 `WebApplicationException`, 用 JAX-RS `Response` 写状态码 + body
   - **如果已经发响应**: 只 log + close connection (无法补发)
   - 决定 `streamUnrecoverable` (429/408/413 → 直接断流)

**自定义错误处理模式** (MuServerBuilder.withExceptionHandler, 416-419):
```java
muServerBuilder.withExceptionHandler((request, response, exception) -> {
    if (response.hasStartedSendingData()) return false; // 无法自定义
    if (exception instanceof NotAuthorizedException) return false;
    response.contentType(ContentTypes.TEXT_PLAIN_UTF8);
    response.write("Oh I'm worry, there was a problem");
    return true;
});
```

## Async (AsyncHandle, NettyRequestAdapter.java:469-546)

`MuRequest.handleAsync()` 返回 `AsyncHandle`:
- `setReadListener(RequestBodyListener)` — body 流式 callback (Servlet 3.1 风格)
- `complete()` / `complete(Throwable)` — 标记响应完成
- `write(ByteBuffer)` / `write(ByteBuffer, DoneCallback)` — 异步写
- `addResponseCompleteHandler(ResponseCompleteListener)` — 完成通知

**模式**: 一旦 `handleAsync()`, 同步 API (write/sendChunk/writer/outputStream) 都抛 `IllegalStateException` (`NettyResponseAdaptor.throwIfAsync`, 255-259)。

## Form / Multipart / InputStream

`NettyRequestAdapter.ensureFormDataLoaded()` (338-355):
- `multipart/*` → `RequestBodyReader.MultipartFormReader`
- `application/x-www-form-urlencoded` → `RequestBodyReader.UrlEncodedBodyReader`

**RequestBodyReader** 抽象:
- `InputStreamRequestBodyReader` — 用于 `request.inputStream()`
- `StringRequestBodyReader` — 用于 `request.readBodyAsString()`
- `MultipartFormReader` — 解析 multipart/form-data, 暴露 `uploads(name)`
- `UrlEncodedBodyReader` — 解析 form-urlencoded
- `DiscardingReader` — 读完丢弃, 用于释放
- `ListenerAdapter` — 包装 `RequestBodyListener` (async 模式)

**单次读取**: `requestBodyReader` 字段只有一个 slot, 第二次读抛 `IllegalStateException`。
`inputStream()` 还通过 `claimingBodyRead(inputStreamReader).get()` **同步阻塞**等 body 完整到达
(会临时切到 event loop)。

## 设计观察

1. **HTTPS 配置比 Express/Vertx 简单**——一个 builder 类, 无 keystore 工厂 + 协议协商细节。
2. **WebSocket 支持 HTTP/1 upgrade, 不支持 HTTP/2 (RFC 8441)**。
3. **Rate limit 是简单内存版**——单实例够用, 多实例需借助外部 (Redis)。
4. **Stats 通过 `GlobalTrafficShapingHandler.trafficCounter()` 拿到全局字节计数**——轻量级,
   没有 Micrometer/Prometheus 集成, 需要自己 `addResponseCompleteListener` 写出去。
5. **Async 是真正的 Netty 异步**——但用户接口被 `AsyncHandle` 抽象过, 业务代码不感知 Netty。
6. **Multipart 上传**有内存 + 磁盘两种 backend (取决于实现细节, 待细查)。

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
