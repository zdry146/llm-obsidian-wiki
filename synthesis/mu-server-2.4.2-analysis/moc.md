---
title: "mu-server 2.4.2 Netty 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [java, netty, mu-server, framework, analysis, opencode, omo, moc, index, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "mu-server 2.4.2 在 Netty 之上新增/包装的能力 - 全量分析的入口和导航"
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

# mu-server 2.4.2 Netty 全量分析 - Map of Content

> 本目录是 omo Sisyphus agent team 对 mu-server **2.4.2** (release tag) 源码的全量分析产出。
> 对比参考: [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] | [[mu-server-2.2.9-analysis/summary|2.2.9 中间版本]]

## 主入口
- **[[summary|综合报告]]** — mu-server 在 Netty 之上新增/包装了什么 (执行摘要 + 架构图 + 对照表 + 关键代码 + 使用场景 + **0.0.3 → 2.2.9 → 2.4.2 演进**)

## 分层分析 (10 个 draft)
- **[[draft-01-protocol-layer|协议层]]** — HTTP/1.1 + HTTP/2 + ALPN + HAProxy + 背压
- **[[draft-02-abstract-layer|抽象层]]** — MuRequest/Response/HttpExchange
- **[[draft-03-dispatcher-layer|分发层]]** — NettyHandlerAdapter + MuServerBuilder + 路由
- **[[draft-04-handlers-library|内置 Handler 库]]** — CORS/CSRF/静态资源/HttpsRedirect
- **[[draft-05-rest-jax-rs|JAX-RS / REST]]** — 注解路由 + 参数绑定 + Filter/Interceptor
- **[[draft-06-openapi|OpenAPI 集成]]** — 自动从 JAX-RS 生成 OpenAPI 3.x
- **[[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]** — 长连接 + 双向通信
- **[[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]** — 证书 + 流控
- **[[draft-09-utility-classes|工具类]]** — Mutils / Cookie / Headers / ContentTypes
- **[[draft-10-evolution-and-comparison|演进对比]]** — 0.0.3 → 2.2.9 → 2.4.2 关键变化

## 版本对比
- **本目录: mu-server 2.4.2** (最新 2.x, Java 11 + Netty 4.1.135.Final)
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9]] (中间版本)
- [[mu-server-netty-analysis/summary|mu-server 0.0.3-SNAPSHOT]] (早期 snapshot)
- 关键差异详见 summary.md "0.0.3 → 2.2.9 → 2.4.2 演进" 章节 + draft 10

## 标签
`#java` `#netty` `#mu-server` `#framework` `#opencode` `#omo` `#2.4.2`

## 元信息
- **代码版本**: mu-server 2.4.2 @ tag (commit 086a921, Netty 4.1.135.Final)
- **JDK**: Java 11 (source/target)
- **分析时间**: 2026-09-11 09:50-12:20
- **执行者**: omo Sisyphus agent team (single ultraworker session, ~10 分钟写 drafts + 合成 evidence)
- **总代码量**: 248 Java 文件 / 31840 行 (vs 2.2.9 240/30755, 0.0.3 258/36317)
- **分析产出**: 2518 行 draft + 878 行 final report = 3396 行 markdown

## 相关链接
- 仓库: https://github.com/3redronin/mu-server
- 文档: https://muserver.io/
