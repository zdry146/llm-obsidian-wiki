---
title: "§10 演化对比"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "演化对比 (0.0.3.6 / 2.2.9 / 2.4.2)：文件数 258/240/248, 总行数 36317/30755/31840, Netty 4.1.137/4.1.135/4.1.137, JDK 11/1.8/1.8, commit 4f0aa3c/086a921/ae091538。"
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


# 演化对比 (0.0.3 → 2.2.9 → 2.4.2)

| 维度 | 0.0.3.6 | 2.2.9 | **2.4.2** |
|---|---|---|---|
| Java 文件数 | 258 | 240 | **248**（比 2.2.9 +8）|
| 总行数 | 36,317 | 30,755 | **31,840**（比 2.2.9 +1,085）|
| Netty 版本（默认） | 4.1.137.Final | 4.1.135.Final | **4.1.137.Final** |
| JDK (source/target) | 11 | 1.8 | **1.8** |
| Commit | `4f0aa3c` | `086a921` "Update Netty version to 4.1.135.Final" | **`ae091538`** "Merge #206 colon-paths" |
| 协议层 | HTTP/1 + HTTP/2 + 流控 + ALPN + HAProxy | 同 | **同**（无变化）|
| 抽象层 | `NettyRequestAdapter` + `NettyResponseAdaptor` + `HttpExchange` + `MuRequest/Response` | 同 | **同**（API 微调）|
| 分发层 | `NettyHandlerAdapter` + `MuServerBuilder` | `MuServerBuilder` | **`MuServerBuilder` (835 行，最大单文件)** |
| 路由 | `Routes` + URI 模板 | 同 | **同**（稳定）|
| JAX-RS | 手写 annotation scanner | 同 | **同**（成熟稳定）|
| SSE | `SsePublisher` + `AsyncSsePublisher` | 同 | **同**（**无自动心跳**，用户手动调 `sendComment`）|
| TLS | `HttpsConfigBuilder` | 同 | **同** |
| RateLimiter | 接口 + 内置实现 | 同 | **同** |
| Async | `AsyncHandle` | 同 | **同** |
| WebSocket | `ws/` 子包 | 同 | **同** |

**核心观察**：
- 架构骨架（6 层）在 0.0.3.6 就已成型，后续版本主要是 bug 修复 + 文档改进 + 边缘场景处理
- 2.4.2 比 2.2.9 多 8 个文件 + 1,085 行 → **新增功能 + 实现细节扩充**（含 `Http2Headers` / `Http2Response` / `Http2To1RequestAdapter` 等 HTTP/2 辅助类）
- 协议层（HTTP/1, HTTP/2, 流控, ALPN）完全没变 → 这是 mu-server 的"稳定面"
- **JDK 降级**：0.0.3.6 用了 JDK 11，2.2.9 / 2.4.2 回到 JDK 1.8（向后兼容性优先）

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
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
