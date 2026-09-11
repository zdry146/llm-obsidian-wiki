---
title: "mu-server 2.4.2 源码分析 — Spark 直接版 MOC"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, moc, index, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "mu-server 2.4.2 (Java 1.8 + Netty 4.1.137.Final) 全量源码分析入口 — Spark 直接读源码的中文模块化报告 (248 文件 / 31840 行)，已 2026-09-12 review 校正 16 项事实。"
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

# mu-server 2.4.2 源码分析 — Map of Content

> 本目录是 **Spark 直接读 mu-server 2.4.2 源码**的中文模块化分析报告（替换之前 OMO Sisyphus agent team 的合成 draft，原因：OMO evidence 僵死后是手动合成，不可靠）。
> 对比参考: [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析 (OMO)]] | [[mu-server-2.2.9-analysis/summary|2.2.9 中间版本 (OMO)]]

## 主入口
- **[[summary|综合报告]]** — 13 节模块化分析 (架构 → 协议层 → 抽象层 → 分发层 → JAX-RS → 功能 → 模式 → 对照 → 演化 → 限制 → 场景 → 文件清单)

## 章节速览（summary.md 内）

1. **整体架构**：六层叠加（Netty 原生 / 协议 / 抽象 / 分发 / 路由 / JAX-RS）
2. **协议层**：HTTP/1.1 + HTTP/2（含自实现流控）+ ALPN + HAProxy + 背压
3. **抽象层**：`MuRequest` / `MuResponse` / `NettyRequestAdapter` / `NettyResponseAdaptor` / `HttpExchange`
4. **分发层**：`NettyHandlerAdapter`（98 行核心调度器）+ `MuServerBuilder`（835 行 fluent builder）+ `Routes`
5. **JAX-RS**：`MuRuntimeDelegate` + 80+ 文件完整实现
6. **功能特性**：SSE / TLS / RateLimiter / MuStats / WebSocket / Async
7. **Handler 库**：CORS / CSRF / HttpsRedirector / ResourceHandler
8. **关键设计模式**：线程模型 + 状态机 + HTTP/2 流控 + 优雅关停
9. **Netty 原生 vs mu-server 对照表**（30+ 行）
10. **演化对比**（0.0.3 → 2.2.9 → 2.4.2）
11. **限制 / 已知问题**
12. **适用场景**（✅ / ❌）
13. **关键文件清单**（40+ 行速查表）

## 版本对比
- **本目录: mu-server 2.4.2**（最新 2.x，Java 11 + Netty 4.1.135.Final）
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9]]（中间版本，OMO 分析）
- [[mu-server-netty-analysis/summary|mu-server 0.0.3-SNAPSHOT]]（早期 snapshot，OMO 分析）
- 关键差异详见 summary.md §10 "演化对比" 表格

## 标签
`#java` `#netty` `#mu-server` `#framework` `#direct-analysis` `#spark` `#2.4.2`

## 元信息
- **代码版本**: mu-server 2.4.2 @ tag（commit `086a921` "Update Netty version to 4.1.135.Final"）
- **JDK**: Java 11 (source/target)
- **依赖**: Netty 4.1.135.Final + jakarta.ws.rs 3.0
- **规模**: 248 Java 文件 / 31,840 行
- **分析时间**: 2026-09-11 13:30-14:30
- **执行者**: **Spark 直接读源码**（拒绝 OMO——上次 evidence 僵死后手动合成不靠谱）
- **方法**: 直读 24+ 核心文件（`MuServer` / `MuServerBuilder` / `NettyHandlerAdapter` / `Http1Connection` / `Http2Connection` / `NettyRequestAdapter` / `NettyResponseAdaptor` / `HttpExchange` / `SsePublisher` / JAX-RS 入口等），全部用 `head` 限行数批量读
- **产出**: 1 个 summary.md（700+ 行 / 43.6 KB）

## 相关链接
- 仓库: https://github.com/3redronin/mu-server
- 文档: https://muserver.io/
