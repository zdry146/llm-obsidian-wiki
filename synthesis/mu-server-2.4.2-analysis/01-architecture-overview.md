---
title: "§1 整体架构 + 6 层架构图"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "mu-server 6 层架构总览：Netty 原生 / 协议层 / 抽象层 / 分发层 / 路由层 / Handler 库 / JAX-RS / 应用层，含 6 层架构 mermaid 图。"
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


# 整体架构：六层叠加

| 层             | 包 / 类                                                                                        | 角色                                      |
| ------------- | -------------------------------------------------------------------------------------------- | --------------------------------------- |
| **Netty 原生**  | `io.netty.*`                                                                                 | channel / pipeline / event loop / codec |
| **协议层**       | `Http1Connection` / `Http2Connection` / `AlpnHandler` / `HAProxyMessageHandler`              | 把 Netty 消息转成 mu 的 Request/Response      |
| **抽象层**       | `NettyRequestAdapter` / `NettyResponseAdaptor` / `HttpExchange` / `MuRequest` / `MuResponse` | handler 看到的高层 API                       |
| **分发层**       | `MuServerBuilder` / `MuServerImpl` / `NettyHandlerAdapter`                                   | builder、生命周期、handler 调度                 |
| **路由层**       | `Routes` / `RouteHandler` / `UriPattern`                                                     | URI 模板路由                                |
| **Handler 库** | `handlers.CORSHandler` / `CSRFProtectionHandler` / `HttpsRedirector` / `ResourceHandler`     | 开箱即用                                    |
| **JAX-RS 层**  | `rest.*`                                                                                     | jakarta.ws.rs 注解 + OpenAPI 生成           |
| **应用层**       | 用户写的 `MuHandler` / JAX-RS resource                                                           | 业务代码                                    |

> **关键事实**：mu-server **不会从 Netty event loop 跑 user handler**（见 §8.1）。所有"用户逻辑"通过独立的 `ExecutorService` 调度，event loop 只负责拆装消息。这是跟裸 Netty 编程体验最大的区别。

### 1.1 6 层架构图

```mermaid
graph TB
    subgraph L0["L0 · Netty 原生 (io.netty.*)"]
        NETTY["channel / pipeline<br/>event loop / codec"]
    end

    subgraph L1["L1 · 协议层"]
        H1C["Http1Connection<br/>309 行"]
        H2C["Http2Connection<br/>582 行<br/>+ inner Http2ConnectionFlowControl"]
        H2AUX["Http2Headers<br/>Http2Response<br/>Http2To1RequestAdapter"]
        AL["AlpnHandler<br/>46 行"]
        HP["HAProxyMessageHandler<br/>20 行"]
        BP["BackPressureHandler<br/>72 行"]
        MFC["MuFlowControlHandler<br/>229 行 (Netty fork)"]
    end

    subgraph L2["L2 · 抽象层 (handler-facing API)"]
        EX["HttpExchange<br/>479 行<br/>block() + 3 状态机"]
        NREQ["NettyRequestAdapter<br/>549 行 · implements MuRequest"]
        NRESP["NettyResponseAdaptor<br/>367 行 · implements MuResponse"]
        MR["MuRequest (public · 237)"]
        MRE["MuResponse (public · 120)"]
    end

    subgraph L3["L3 · 分发层"]
        MSB["MuServerBuilder<br/>835 行 (最大单文件)"]
        MSI["MuServerImpl<br/>167 行"]
        NHA["NettyHandlerAdapter<br/>98 行 · 核心调度器"]
        RT["Routes + UriPattern<br/>52 行"]
    end

    subgraph L4["L4 · Handler 库 (io.muserver.handlers · 14 文件)"]
        CORS["CORSHandler"]
        CSRF["CSRFProtectionHandler"]
        HDR["HttpsRedirector"]
        RES["ResourceHandler<br/>(+ BareDirectoryRequestAction)"]
    end

    subgraph L5["L5 · JAX-RS (io.muserver.rest · 70 文件)"]
        MRD["MuRuntimeDelegate<br/>@Path / @GET / @POST"]
        OPEN["OpenApiGenerator<br/>+ HtmlDocumentor"]
    end

    subgraph L6["L6 · 应用层 (user code)"]
        UH["MuHandler"]
        JR["@Path resource class"]
    end

    subgraph TM["线程模型 (3 层)"]
        T1["Netty event loop<br/>16 NIO threads"]
        T2["muhandler 独立 Executor<br/>ThreadPoolExecutor(8, 400, 60s)"]
        T3["HttpExchange.block()<br/>跨线程同步"]
        T1 -.executor.execute.-> T2
        T2 -.ctx.executor().submit.-> T3
        T3 -.task.get().sync.-> T1
    end

    %% Netty → 协议
    NETTY --> H1C
    NETTY --> H2C
    H2C -.uses.-> H2AUX
    AL -.动态切换 pipeline.-> H1C
    AL -.动态切换 pipeline.-> H2C
    HP -.真实客户端 IP.-> EX
    BP -.pipeline 背压.-> H1C
    MFC -.fork 自 Netty 4.1.136+.-> H1C

    %% 协议 → 抽象
    H1C --> EX
    H2C --> EX
    EX --> NREQ
    EX --> NRESP
    NREQ -.implements.-> MR
    NRESP -.implements.-> MRE

    %% 抽象 → 分发
    EX --> NHA
    MSB --> MSI
    MSB --> NHA
    NHA --> RT
    RT --> L4
    RT --> L5

    %% 分发 → 应用
    NHA --> UH
    NHA --> JR

    %% 颜色
    classDef proto fill:#f0e6ff,stroke:#6600cc,color:#000
    classDef abs fill:#e6f3ff,stroke:#0066cc,color:#000
    classDef disp fill:#e6ffe6,stroke:#00cc66,color:#000
    classDef user fill:#fff4e6,stroke:#ff8c00,color:#000
    classDef tm fill:#ffe6e6,stroke:#cc0000,color:#000

    class H1C,H2C,H2AUX,AL,HP,BP,MFC proto
    class EX,NREQ,NRESP,MR,MRE abs
    class MSB,MSI,NHA,RT disp
    class UH,JR user
    class T1,T2,T3 tm
```

**图注**：
- 紫色 L1 = 协议层（Netty → mu Request/Response 翻译器）
- 蓝色 L2 = 抽象层（handler 看到的高层 API）
- 绿色 L3 = 分发层（builder + 调度器 + 路由）
- 橙色 L6 = 应用层（用户写的代码）
- 红色 TM = 线程模型（详见 §8.1）
- 跨层虚线表示「该类被另一层引用但非数据流主路径」

---

## 相关笔记

**同目录其他章节**:
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
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
