---
title: "§8 关键设计模式"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "关键设计模式：3 层线程模型 (Netty event loop + 独立 muhandler 池 + HttpExchange.block()) + 3 状态机 + HTTP/2 自实现流控 + 优雅关停 (stop(duration, unit))。"
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


# 关键设计模式 (Critical Design Patterns)

### 8.1 线程模型 (Threading Model)

```
                   ┌─────────────────────┐
   TCP socket ──→  │ Netty event loop    │  ← NIO thread (默认 16 个)
                   │ - 拆 HttpRequest    │
                   │ - 装 LastHttpContent │
                   └─────────┬───────────┘
                             │ executor.execute(() -> handler.handle(req, resp))
                             ▼
                   ┌─────────────────────┐
                   │ Handler executor    │  ← 独立线程池（用户代码）
                   │ - user logic        │
                   │ - 调用 resp.write() │
                   └─────────┬───────────┘
                             │ HttpExchange.block(task)
                             │ ctx.executor().submit(task).get()  ← 同步等
                             ▼
                   ┌─────────────────────┐
                   │ Netty event loop    │  ← 同一 NIO thread
                   │ - 真正写 socket     │
                   │ - 触发 outputState  │
                   └─────────────────────┘
```

**为什么这样设计**：
1. Netty 默认要求 handler 全部跑在 event loop 上 → slow handler 会阻塞整个 channel 的 I/O
2. Mu-server 把"用户逻辑"扔到独立 executor → Netty 永远只做"拆消息 + 写 socket"
3. 但写 socket 又必须在 Netty thread（状态机 assert 限制）→ 用 `block()` 把 handler 线程**同步等** Netty 完成
4. 对用户而言：handler 写法保持同步（不需要 callback hell），但 Netty 永远不被阻塞

**权衡**：跨线程同步有微小开销（thread context switch），所以 mu-server 也支持 async API（`AsyncHandle` / `AsyncSsePublisher`）作为完全异步的 escape hatch。

### 8.2 状态机 (State Machines)

mu-server 有 **3 个独立状态机** + **4 个状态变化点**：

| 状态机 | 状态 | 触发点 |
|---|---|---|
| `RequestState` | `HEADERS_RECEIVED` → `RECEIVING_BODY` → `COMPLETE` / `ERRORED` | `NettyRequestAdapter.outputState()` |
| `ResponseState` | `NOTHING` → `STREAMING` → `COMPLETE` / `ERRORED` / `UPGRADED` | `NettyResponseAdaptor.outputState()` |
| `HttpExchangeState` | `IN_PROGRESS` → `COMPLETE` / `ERRORED` / `UPGRADED` | `HttpExchange.onReqOrRespStateChange()` |

**状态变化监听**：`CopyOnWriteArrayList<...StateChangeListener>`（读多写少，写不阻塞读）

**状态一致性约束**：`assert ctx.executor().inEventLoop()` —— 所有状态变更必须在 Netty event loop 上

### 8.3 HTTP/2 流控（已在 §2.2 详述）

简言之：自实现 buffer + wantsToRead + 手动 consumeBytes，绕过 Netty 默认流控的 deadlock 风险。

### 8.4 优雅关停 (Graceful Shutdown)

```java
boolean stop(long duration, TimeUnit unit) {
    // 1. 停止接受新连接：boss group shutdownGracefully
    // 2. 在 duration 内等现有请求完成
    // 3. 超时后强制 abort 所有 in-flight exchanges
    // 4. worker group shutdownGracefully
    return allCompleted;  // true 表示在 duration 内全部完成
}
```

`HttpExchange` 跟踪 in-flight 数，`MuServerImpl` 定期 poll。完整实现见 `MuServerImpl.stopInternal()`。

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
- [[09-netty-comparison|§9 Netty 原生 vs mu-server 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
