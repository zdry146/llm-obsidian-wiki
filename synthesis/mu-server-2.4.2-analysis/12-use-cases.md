---
title: "§12 适用场景"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "适用场景：✅ 微服务/小中型 REST API + JAX-RS 标准 API + SSE 长连接 + HTTPS+Let's Encrypt + 嵌入式 server; ❌ 企业级大应用 (比不过 Spring) + 极限性能 + 需直接控制 Netty + HTTP/3 + 多语言场景。"
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


# 适用场景

✅ **适合**：
- **微服务 / 小到中型 REST API**：启动快（亚秒）、资源占用低
- **JAX-RS 标准 API**：自动 OpenAPI 文档、注解驱动
- **SSE 长连接**：原生 publisher（无自动心跳，需自己 `ScheduledExecutorService` 周期调 `sendComment` 保活）
- **HTTPS + Let's Encrypt 自动续签**：内置集成
- **嵌入式 HTTP server**：作为 library 嵌入 Java 应用

❌ **不适合**：
- **企业级大应用**：Spring 全家桶生态深度比不上
- **极限性能**：mu-server 的跨线程同步有开销，裸 Netty + 自写更适合
- **需要直接控制 Netty pipeline**：mu-server 抽象层会卡你
- **HTTP/3 / QUIC**：当前不支持
- **多语言场景**：纯 JVM

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
- [[09-netty-comparison|§9 Netty 原生 vs mu-server 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
