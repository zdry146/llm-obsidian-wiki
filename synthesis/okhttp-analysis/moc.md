---
title: "OkHttp 全量分析 - MOC (Map of Content)"
category: synthesis
tags: [java, okhttp, http-client, square, analysis, moc, index]
sources:
  - "OkHttp 4.12.0 @ GitHub (https://github.com/square/okhttp)"
  - "OkHttp 5.0.0-alpha.14 (Kotlin coroutines)"
  - "Okio 3.x (https://github.com/square/okio)"
summary: "OkHttp 4.12+ 源码级全量分析 - 拦截器链、连接池、HTTP/2、缓存、Okio、版本演进、已知坑与对比"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# OkHttp 全量分析 - Map of Content

> 本目录是 OkHttp **4.12.0** (stable) 与 **5.0.0-alpha.14** (coroutines 预览) 的源码级全量分析，附带 3.x 历史对比、12 个核心机制拆解、实战坑点（包含作者亲历的 JDK 21 + OkHttp + SonarQube 401 bug）。

## 主入口
- **[[summary|综合报告]]** — 执行摘要 + 生态位 + 12 章导航 + 适用场景 + 关键决策表

## 12 章深度分析 (12 个 draft)

| § | 子页面 | 核心内容 |
|---|---|---|
| **§00** | **[[draft-00-background\|背景与生态位]]** | Square 出品、Android 默认客户端、Retrofit/Spring Cloud OpenFeign 都用它 |
| §01 | [[draft-01-architecture\|核心架构]] | OkHttpClient / Request / Response / Call / Dispatcher 全景 |
| §02 | [[draft-02-interceptors\|拦截器链 (核心)]] | 4 层内置 + 2 类自定义 + 关键代码 + 实战模式 |
| §03 | [[draft-03-connection-pool\|连接池机制]] | ConnectionPool / RealConnection / 驱逐策略 / 调优 |
| §04 | [[draft-04-http2\|HTTP/2 支持]] | 多路复用 / 帧 / 流 / 连接合并 |
| §05 | [[draft-05-cache\|缓存机制]] | HTTP 缓存语义 + DiskLruCache + CacheStrategy |
| §06 | [[draft-06-websocket\|WebSocket 支持]] | RealWebSocket + 帧解析 + 心跳 |
| §07 | [[draft-07-sync-async\|同步/异步模型]] | execute/enqueue + Dispatcher 线程池 + 并发上限 |
| §08 | [[draft-08-okio\|Okio 底层]] | Buffer/Source/Sink/ByteString + 为什么自造 |
| §09 | [[draft-09-evolution\|版本演进]] | 3.x → 4.x → 5.x 关键差异 |
| §10 | [[draft-10-known-issues\|已知坑 (实战)]] | JDK 21 + OkHttp + SonarQube 401/连接失败复盘 |
| §11 | [[draft-11-comparison\|对比选型]] | vs Java 11 HttpClient / Apache HC 5 / Reactor Netty / Netty |
| §12 | [[draft-12-use-cases\|适用场景决策]] | ✅ / ❌ 场景判断 + 6 条使用铁律 |

## 标签
`#java` `#okhttp` `#http-client` `#square` `#jvm` `#analysis`

## 元信息

- **分析对象**: OkHttp 4.12.0 (JVM stable) + 5.0.0-alpha.14 (coroutines 预览)
- **代码仓库**: https://github.com/square/okhttp
- **协议**: Apache 2.0
- **核心依赖**: Okio 3.x + Kotlin stdlib (4.x 起)
- **JDK 要求**: Java 8+ (4.x/5.x), Android 5.0+ API 21+
- **分析时间**: 2026-09-12
- **执行者**: Spark (直接读源码 + 社区资料交叉验证)
- **关键实战参考**: [[draft-10-known-issues|JDK 21 + SonarQube 401 踩坑复盘]]
- **总产出**: 12 个 draft + 1 个 moc + 1 个 summary ≈ 3500 行 markdown

## 跨笔记链接

- **HTTP 客户端生态对照**: 配合 [[draft-11-comparison]] 阅读
- **与 Netty 的关系**: OkHttp 是 Netty 在 HTTP 客户端场景的轻量级替代（不暴露 NIO）
- **Jenkins pipeline 实战**: 拦截器 + 401 鉴权 fix 见 [[draft-10-known-issues]]
