---
title: "mu-server Netty 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [java, netty, mu-server, framework, analysis, opencode, omo, moc, index]
sources: ["mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "mu-server 在 Netty 之上新增/包装的能力 - 全量分析的入口和导航"
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

# mu-server Netty 全量分析 - Map of Content

> 本目录是 omo Sisyphus agent team 对 mu-server 0.0.3-SNAPSHOT 源码的全量分析产出 (258 Java 文件 / 36317 行)。

## 主入口
- **[[summary|综合报告]]** — mu-server 在 Netty 之上新增/包装了什么 (执行摘要 + 架构图 + 对照表 + 关键代码 + 使用场景)

## 分层分析 (10 个 draft)
- **[[draft-01-protocol-layer|协议层]]** — HTTP/1.1 + HTTP/2 + ALPN + HAProxy + 背压
- **[[draft-02-abstraction-layer|抽象层]]** — MuRequest/Response/HttpExchange
- **[[draft-03-dispatch-layer|分发层]]** — NettyHandlerAdapter + MuServerBuilder + 路由
- **[[draft-04-handlers|内置 Handler 库]]** — CORS/CSRF/静态资源/HttpsRedirect
- **[[draft-05-jaxrs|JAX-RS 支持]]** — 注解路由 + 参数绑定 + Filter/Interceptor
- **[[draft-06-features|功能模块]]** — SSE/TLS/限流/统计/WebSocket
- **[[draft-07-threading-model|线程模型 (关键)]]** — NettyHandlerAdapter + block() 跨线程同步
- **[[draft-08-state-machines|状态机]]** — RequestState/ResponseState/HttpExchangeState
- **[[draft-09-http2-flow-control|HTTP/2 自定义流控]]** — 自实现 buffer + wantsToRead
- **[[draft-10-graceful-shutdown|优雅关停]]** — stop(duration, unit) + in-flight 等

## 标签
`#java` `#netty` `#mu-server` `#framework` `#opencode` `#omo` `#analysis`

## 元信息
- **代码版本**: mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (master 分支)
- **分析时间**: 2026-09-11 01:00-01:14
- **执行者**: omo Sisyphus agent team (single ultraworker session, ~14 分钟)
- **总代码量**: 258 Java 文件 / 36317 行
- **分析产出**: 1096 行 draft + 504 行 final report = 1600 行 markdown
