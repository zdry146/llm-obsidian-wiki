---
title: "§5 JAX-RS 支持"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "JAX-RS 支持：io.muserver.rest ~70 文件，含 MuRuntimeDelegate + @Path/@GET/@POST 注解 + OpenAPI 3 自动生成 + Entity providers + Per-resource CORS + HTTP Basic Auth + ExceptionMappers。"
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


# JAX-RS 支持 (`io.muserver.rest.*`)

~80+ 文件，是 mu-server 最大的子模块。

### 5.1 入口 — `MuRuntimeDelegate.java`

```java
public class MuRuntimeDelegate extends RuntimeDelegate {
    public static synchronized RuntimeDelegate ensureSet() {
        if (singleton == null) {
            singleton = new MuRuntimeDelegate();
            RuntimeDelegate.setInstance(singleton);
        }
        return singleton;
    }

    private final Map<Class<?>, HeaderDelegate> headerDelegates = new HashMap<>();

    private MuRuntimeDelegate() {
        headerDelegates.put(MediaType.class, new MediaTypeHeaderDelegate());
        headerDelegates.put(CacheControl.class, new CacheControlHeaderDelegate());
        headerDelegates.put(NewCookie.class, new NewCookieHeaderDelegate());
        headerDelegates.put(Cookie.class, new CookieHeaderDelegate());
        headerDelegates.put(EntityTag.class, new EntityTagDelegate());
        headerDelegates.put(Link.class, new LinkHeaderDelegate());
        headerDelegates.put(Date.class, new DateHeaderDelegate());
    }
}
```

**`ensureSet()` 在 mu-server `io.muserver.rest` 包多个类首次加载时被调用**（`RequestMatcher:27` / `NewCookieHeaderDelegate:12` / `MuUriInfo:22` / `MuUriBuilder:25` 都调了它），用于注册 mu 自己的 JAX-RS RuntimeDelegate。这是 JDK JAX-RS SPI 机制：`RuntimeDelegate.setInstance()` 后所有 JAX-RS API 走 mu 的实现。

### 5.2 子包内容（节选）

```
rest/
├── MuRuntimeDelegate.java           # JAX-RS 入口
├── JaxRSRequest.java / JaxRSResponse.java  # Request/Response → JAX-RS 适配
├── JaxClassLocator.java / JaxMethodLocator.java  # 注解扫描
├── UriInfoImpl.java / UriPattern.java / PathMatch.java  # URI 模板匹配
├── EntityProviders.java / BinaryEntityProviders.java / BuiltInParamConverterProvider.java
│   └── JSON / XML / 二进制 / 自定义 entity 编解码
├── BasicAuthSecurityFilter.java / Authorizer.java  # HTTP Basic Auth
├── CorsFilter.java                  # per-resource CORS（不是顶层 CORSHandler）
├── JaxOutboundSseEvent.java / SseBroadcasterImpl.java / JaxSseEventSinkImpl.java
│   └── JAX-RS 标准 SSE 适配
├── HtmlDocumentor.java              # 自动生成 API 文档 HTML
├── OpenApiGenerator.java / SchemaGenerator.java / OpenApiProcessor.java
│   └── OpenAPI 3.x 自动生成（从 @ApiResponse / @Schema 等注解）
├── ApiResponse.java / ApiResponses.java / Description.java / DescriptionData.java
│   └── OpenAPI 注解支持
└── ... (更多 entity providers, filters, interceptors)
```

### 5.3 JAX-RS 关键特点

1. **完整支持 jakarta.ws.rs 3.0+**：`@Path` / `@GET` / `@POST` / `@PUT` / `@DELETE` / `@HEAD` / `@OPTIONS` / `@PATCH`，路径参数 / 查询参数 / 表单 / header / cookie 注入
2. **Async + SSE 内建**：`@Suspended AsyncResponse` 异步响应；`SseEventSink` 流式广播
3. **OpenAPI 3 自动生成**：`@OpenAPI` 注解 + `HtmlDocumentor` 自动生成 API 文档页
4. **Entity providers 自动协商 content-type**：JSON (Jackson) / XML (JAXB) / 二进制 / 自定义
5. **Per-resource CORS**：`@CorsFilter` 注解单独控制每个 resource 的 CORS 策略（区别于顶层 `handlers.CORSHandler` 全局策略）
6. **HTTP Basic Auth**：`@BasicAuthSecurityFilter` + `Authorizer` 函数式接口
7. **ExceptionMappers**：默认实现覆盖所有 JAX-RS 异常类型 → HTTP 状态码映射

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
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
