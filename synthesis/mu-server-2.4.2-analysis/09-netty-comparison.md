---
title: "§9 Netty 原生 vs mu-server 对照表"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "Netty 原生 vs mu-server 对照表：30+ 行 API 对照 (配置 / 接收请求 / 线程模型 / 跨线程写 / HTTP/1+2 / HTTPS / ALPN / 路由 / 中间件 / 异常 / 限流 / 统计 / SSE / WebSocket / JAX-RS / OpenAPI / Let's Encrypt / HAProxy / 优雅关停 / 背压)。"
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


# Netty 原生 vs mu-server 对照表

| 能力 | Netty 原生 | mu-server |
|---|---|---|
| 配置 server | `ServerBootstrap.group().channel().childHandler()...` 链式 API | `MuServerBuilder.httpsServer().addHandler(...)...start()` fluent API |
| 接收请求 | `extends SimpleChannelInboundHandler<HttpRequest>` 重写 `channelRead0` | 实现 `MuHandler.handle(req, resp)` |
| 线程模型 | 所有 handler 跑 event loop | handler 跑独立 executor，event loop 只做 I/O |
| 跨线程写响应 | `ChannelFuture.addListener`（异步回调） | `HttpExchange.block()`（同步等） |
| HTTP/1 | `HttpServerCodec` + `HttpObjectAggregator` | 自定义拆装 `Http1Connection`（更细粒度背压） |
| HTTP/2 | `Http2FrameCodec` + 默认流控 | 自实现 `Http2ConnectionFlowControl`（避免大 body deadlock） |
| HTTPS | `SslHandler` + 手动配置 | `HttpsConfigBuilder`（keystore / SNI / 客户端证书） |
| ALPN | `ApplicationProtocolNegotiationHandler` 自己装 pipeline | `AlpnHandler.configurePipeline()` 自动切换 |
| 路由 | 完全没有 | `Routes.route(method, uriTemplate, handler)` + URI 模板参数 |
| 中间件 | `ChannelPipeline.addLast(handler)` 链 | `addHandler(MuHandler)` 链（按顺序第一个 `true` 消费） |
| 异常处理 | `exceptionCaught(ctx, ex)` | `UnhandledExceptionHandler` + `ExceptionMapper`（JAX-RS 风格） |
| 请求/响应抽象 | `HttpRequest` / `FullHttpResponse`（底层） | `MuRequest` / `MuResponse`（handler 友好的高层 API） |
| Body 读取 | `ByteBuf` + 手动 release | `inputStream()` / `readBodyAsString()` |
| Cookies | `ServerCookieDecoder` + 自己构造 | `request.cookies()` / `response.cookie(builder)` |
| Forwarded header | 自己解析 RFC 7239 | `Headers.forwarded()` 自动解析 |
| 限流 | 完全自己写 | `RateLimiter` + `RateLimitSelector` 接口 |
| 统计 | 完全自己写 | `MuStats` + `MuStatsImpl`（connectionsOpen / requestsHandled / bytesSent） |
| SSE | 自己写 text/event-stream 帧 | `SsePublisher` / `AsyncSsePublisher`（自动心跳） |
| WebSocket | `WebSocketServerProtocolHandler` + 自己处理 frame | `WebSocketHandlerBuilder` + `BaseWebSocket` 同步 API |
| JAX-RS | 完全不支持 | `rest.MuRuntimeDelegate` + 80+ 文件完整实现 |
| OpenAPI | 完全不支持 | `OpenApiGenerator` 自动生成 |
| Let's Encrypt | 完全不支持 | `letsencrypt/` 子包 |
| HAProxy | 自己装 `netty-codec-haproxy` | `HAProxyMessageHandler` 一行启用 |
| 优雅关停 | `EventLoopGroup.shutdownGracefully()` | `MuServer.stop(duration, unit)` 等 in-flight |
| 背压 | `channel.write()` 默认可写检查 | `BackPressureHandler` 自定义队列 + `MuFlowControlHandler` Netty fork |

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
- [[05-jax-rs|§5 JAX-RS 支持]]
- [[06-features|§6 功能特性]]
- [[07-handler-library|§7 Handler 库]]
- [[08-design-patterns|§8 关键设计模式]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
