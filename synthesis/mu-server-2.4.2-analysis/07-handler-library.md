---
title: "§7 Handler 库"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "Handler 库 (io.muserver.handlers, 14 文件)：CORSHandler / CSRFProtectionHandler / HttpsRedirector / ResourceHandler / BareDirectoryRequestAction / BytesRange。"
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


# Handler 库 (Built-in Handlers)

`io.muserver.handlers.*` 包，~14 个文件（实测 `ls` 列出 14 个 `.java`）。

| Handler | 作用 |
|---|---|
| `CORSHandler` + `CORSHandlerBuilder` | 全局 CORS 策略：withAllowedOrigins / withAllowedMethods / withAllowedHeaders / withExposedHeaders / withMaxAge |
| `CSRFProtectionHandler` + `CSRFProtectionHandlerBuilder` | 双重提交 cookie 模式 CSRF 防护 |
| `HttpsRedirector` + `HttpsRedirectorBuilder` | HTTP → HTTPS 自动重定向（可选 301 / 308） |
| `ResourceHandler` + `ResourceHandlerBuilder` + `ResourceProvider` + `ResourceCustomizer` + `ResourceType` | 静态文件服务（按 MIME / Range / cache headers） |
| `BareDirectoryRequestAction` + `DirectoryLister` | 目录列表（ResourceHandler 子组件） |
| `BytesRange` | HTTP Range 请求解析工具 |

**`ResourceHandler` 配置示例**：
```java
.addHandler(ResourceHandlerBuilder.fileSystemHandler("public")
    .withPathToServeFrom("/var/www")
    .withDefaultFile("index.html")
    .withMimeTypes(Map.of(".md", "text/markdown")))
```

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
- [[05-jax-rs|§5 JAX-RS 支持]]
- [[06-features|§6 功能特性]]
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
