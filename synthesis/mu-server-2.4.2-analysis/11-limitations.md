---
title: "§11 限制 / 已知问题"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "限制 / 已知问题：无 HTTP/3 (QUIC) / 自写 HTTP/1 拆装逻辑 (背压 vs 维护负担) / MuFlowControlHandler fork drift / 单进程无 cluster / Async API 复杂 / JAX-RS 紧耦合 mu-server 核心。"
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


# 限制 / 已知问题

1. **没有 HTTP/3 (QUIC)**：2.4.2 仍只支持 HTTP/1.1 + HTTP/2。Netty 4.1.x 的 QUIC codec 不成熟，mu-server 选择等上游稳定
2. **`HttpServerCodec` 没复用**：mu-server 自己重写了 HTTP/1 拆装逻辑，好处是背压细粒度控制，坏处是维护负担（Netty 上游 bug fix 不直接受益）
3. **MU 自己的 HTTP/2 流控**：fork 了 Netty 的 `FlowControlHandler`，跟上游版本会逐步 drift
4. **单进程 / 单 host**：没有内置 cluster / session replication，需要外部方案（Redis 等）
5. **Async API 比同步 API 复杂**：`AsyncHandle` 的回调模型容易出错，新手应该先用同步 `write()` + `block()`
6. **JAX-RS 子模块紧耦合 mu-server 核心**：不像 Jersey / RESTEasy 可以独立使用，mu-server 的 JAX-RS 必须通过 `MuServerBuilder` 注册

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
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
