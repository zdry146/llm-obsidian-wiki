---
title: "§13 关键文件清单"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "关键文件清单 (速查表)：40+ 文件行数 + 角色，核心 11 个文件实测行数 (MuServer 168 / MuServerImpl 167 / MuServerBuilder 835 / MuRequest 237 / MuResponse 120 / NettyHandlerAdapter 98 / Http1Connection 309 / Http2Connection 582 / HttpExchange 479 / NettyRequestAdapter 549 / NettyResponseAdaptor 367)。"
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


# 关键文件清单 (速查表)

| 文件 | 行数 | 角色 |
|---|---|---|
| `io.muserver.MuServer` | 168 | 公开 server 接口 |
| `io.muserver.MuServerImpl` | 167 | server 实现 |
| `io.muserver.MuServerBuilder` | **835** | fluent builder（最大单文件）|
| `io.muserver.MuRequest` | 237 | request 公开接口 |
| `io.muserver.MuResponse` | 120 | response 公开接口 |
| `io.muserver.MuStats` | 50 | 统计接口 |
| `io.muserver.MuStatsImpl` | 110 | 统计实现 |
| `io.muserver.NettyHandlerAdapter` | 98 | **核心调度器** |
| `io.muserver.Http1Connection` | 309 | HTTP/1.1 实现 |
| `io.muserver.Http2Connection` | **582** | HTTP/2 实现 + 自实现流控（合并在同一文件）|
| ~~`io.muserver.Http2ConnectionFlowControl`~~ | — | **不存在独立文件**（流控做在 `Http2Connection` 内部）|
| `io.muserver.HttpExchange` | 479 | 协调者 + block() |
| `io.muserver.NettyRequestAdapter` | 549 | request Netty 包装 |
| `io.muserver.NettyResponseAdaptor` | 367 | response Netty 包装 |
| `io.muserver.AlpnHandler` | 46 | ALPN 协商 |
| `io.muserver.HAProxyMessageHandler` | 20 | HAProxy 协议 |
| `io.muserver.BackPressureHandler` | 72 | TCP 背压 |
| `io.muserver.MuFlowControlHandler` | 225 | Netty FlowControl fork |
| `io.muserver.Headers` | 412 | header multi-map |
| `io.muserver.Cookie` | - | cookie 值对象 |
| `io.muserver.ForwardedHeader` | 281 | RFC 7239 |
| `io.muserver.Routes` | 52 | URI 路由 |
| `io.muserver.RouteHandler` | - | route 回调接口 |
| `io.muserver.AsyncHandle` | - | 异步 API |
| `io.muserver.handlers.CORSHandler` | - | 全局 CORS |
| `io.muserver.handlers.CSRFProtectionHandler` | - | CSRF |
| `io.muserver.handlers.HttpsRedirector` | - | HTTP→HTTPS |
| `io.muserver.handlers.ResourceHandler` | - | 静态文件 |
| `io.muserver.SsePublisher` | 200 | SSE 同步接口 |
| `io.muserver.AsyncSsePublisher` | 193 | SSE 异步接口 |
| `io.muserver.HttpsConfigBuilder` | - | TLS 配置 |
| `io.muserver.ClientCertificateAuthentication` | - | 客户端证书 |
| `io.muserver.RateLimiter` | - | 限流接口 |
| `io.muserver.rest.MuRuntimeDelegate` | - | JAX-RS 入口 |
| `io.muserver.rest.ResourceBuilder` | - | JAX-RS resource 注册 |
| `io.muserver.rest.UriInfoImpl` | - | URI 信息 |
| `io.muserver.rest.EntityProviders` | - | entity 编解码 |
| `io.muserver.rest.OpenApiGenerator` | - | OpenAPI 3 生成 |
| `io.muserver.rest.HtmlDocumentor` | - | API 文档 HTML |

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
- [[09-netty-comparison|§9 Netty 原生 vs mu-server 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
