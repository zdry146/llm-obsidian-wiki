---
title: "mu-server 2.2.9 JAX-RS 3.0 支持"
category: synthesis
tags: [java, mu-server, jax-rs, rest, annotations, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "内置 jakarta.ws.rs 注解支持 (@Path/@GET/@POST), MuRuntimeDelegate 入口, ResourceBuilder 路由, UriInfo 参数绑定, Filters/Interceptors, EntityProviders"
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

# 05 - JAX-RS 实现 (io.muserver.rest)

> mu-server 自带的 **JAX-RS 3.x (Jakarta EE 9+)** 实现, **不使用 Jersey / RESTEasy**。
> 约 70 个文件 / 9761 行。`rest/README.md` 列出了对 Jakarta REST 3.1 spec 各章节的支持矩阵。

## 设计哲学

来自 `rest/README.md` 的开场:
> "Following the principle of only supporting programmatic config, Mu-Server only supports singletons"
> "Mu Server does not instantiate classes for the API user"
> "No classpath scanning" / "No `@Provider` auto-discovery" / "No priorities (would add another dependency)"

**所有 JAX-RS 资源类都是用户自己 `new` 出来, 然后 `addResource(MyResource.class)` 注入到 builder。**
mu 不做反射实例化、不做 DI、不做 annotation processor。

## 文件清单 (核心)

| 文件 | 角色 |
|---|---|
| `RestHandler.java` | `MuHandler` 适配, 把 JAX-RS 请求桥接到 mu 抽象 |
| `RestHandlerBuilder.java` (598) | 资源 + provider + filter + OpenAPI 配置 |
| `JaxRSRequest.java` | JAX-RS `ContainerRequestContext` 实现 |
| `JaxRSResponse.java` | JAX-RS `ContainerResponseContext` 实现 |
| `UriPattern.java` | URI template → regex 编译器 |
| `PathMatch.java` | 匹配结果 |
| `RequestMatcher.java` | 选择匹配的 resource method |
| `ResourceClass.java` / `ResourceMethod.java` / `ResourceMethodParam.java` | 反射元数据缓存 |
| `JaxClassLocator.java` / `JaxMethodLocator.java` | 注解扫描 |
| `EntityProviders.java` | MessageBodyReader/Writer 总入口 |
| `BinaryEntityProviders.java` | byte[]/InputStream/Reader/File 提供器 |
| `StringEntityProviders.java` | String 提供器 |
| `PrimitiveEntityProvider.java` | 基本类型/Number/Boolean 提供器 |
| `BuiltInParamConverterProvider.java` | 路径/查询/表单参数转换 |
| `CORSConfig.java` / `CORSConfigBuilder.java` | JAX-RS CORS 共享配置 (handler 也复用) |
| `AsyncResponseAdapter.java` | `@Suspended AsyncResponse` 适配 |
| `JaxSseImpl.java` / `JaxSseEventSinkImpl.java` / `SseBroadcasterImpl.java` | JAX-RS SSE |
| `JaxOutboundSseEvent.java` / `JaxOutboundSseEventBuilder.java` | 出站 SSE 事件 |
| `OpenApiDocumentor.java` | OpenAPI 文档生成器 |
| `HtmlDocumentor.java` | OpenAPI Swagger UI HTML |
| `MuRuntimeDelegate.java` | `RuntimeDelegate` SPI 实现 |
| `MuUriBuilder.java` / `MuUriInfo.java` / `MuPathSegment.java` | URI 工具 |
| `MuSecurityContext.java` | `SecurityContext` 实现 |
| `MuVariantListBuilder.java` | 内容协商 variant |
| `MediaTypeHeaderDelegate.java` / `MediaTypeDeterminer.java` | 媒体类型 |
| `CacheControlHeaderDelegate.java` / `CookieHeaderDelegate.java` / `DateHeaderDelegate.java` / `EntityTagDelegate.java` / `LinkHeaderDelegate.java` / `NewCookieHeaderDelegate.java` | 各种 header 的 `HeaderDelegate` |
| `LowercasedMultivaluedHashMap.java` / `ReadOnlyMultivaluedMap.java` | 多值 map |
| `Required.java` / `NotMatchedException.java` / `NotImplementedException.java` | 内部异常 |
| `CustomExceptionMapper.java` | `@Provider` 异常映射 (手动注册) |
| `Authorizer.java` / `BasicAuthSecurityFilter.java` / `UserPassAuthenticator.java` | Basic Auth |
| `FilterManagerThing.java` | request/response filter 链管理 |
| `JaxRsHttpHeadersAdapter.java` | HttpHeaders 适配 |
| `ObjWithType.java` | 类型擦除辅助 |
| `LazyAccessInputStream.java` / `LazyAccessOutputStream.java` / `EmptyInputStream.java` / `NullOutputStream.java` | 各种 stream 适配 |
| `ResourceClass.java` 等 | JAX-RS 反射元数据 |
| `CollectionParameterStrategy.java` | `List<T>` / `Set<T>` 参数策略 |
| `SchemaObjectCustomizer.java` / `SchemaObjectCustomizerContext.java` / `SchemaObjectCustomizerTarget.java` | OpenAPI 自定义 |
| `CombinedMediaType.java` | 媒体类型与 q 值组合 |
| `ResponseHeader.java` | 响应头枚举 |

## RestHandlerBuilder (598 行, 关键配置 API)

```java
RestHandlerBuilder.restHandler(MyResource.class, MyOtherResource.class)
    .addCustomWriter(new MyCustomWriter())
    .addCustomReader(new MyCustomReader())
    .addCustomParamConverterProvider(...)
    .addExceptionMapper(MyException.class, new MyExceptionMapper())
    .addRequestFilter(new MyFilter())
    .addResponseFilter(new MyFilter())
    .addWriterInterceptor(new MyInterceptor())
    .addReaderInterceptor(new MyInterceptor())
    .addOpenApiDocUrl("/openapi.json")
    .addOpenApiDocHtmlUrl("/openapi.html")
    .addOpenApiCustomizer(new MySchemaCustomizer())
    .withCollectionParameterStrategy(CollectionParameterStrategy.SetConvertToUsingSet)
    .build();
```

## 资源注册与反射 (核心)

- `addResource(Object... resources)` 接受**已实例化**的对象。
- `JaxClassLocator` 扫描 `@Path`, 方法上的 `@GET/@POST/@PUT/@DELETE/@HEAD/@OPTIONS`。
- 缓存到 `ResourceClass` → `ResourceMethod` → `ResourceMethodParam` 元数据。
- 不支持字段注入 (`@Context` 字段无效), 仅支持**方法参数** `@Context`。

## URI 模板

`UriPattern.uriTemplateToRegex("/foo/{id}")`:
- `{id}` → `(?<id>[^/]+)`
- `{id:[0-9]+}` → `(?<id>[0-9]+)`
- 支持 `:` 后的正则约束
- 编译成 `java.util.regex.Pattern`, 在请求时调 `matcher(relativePath)`

## 参数解析

`BuiltInParamConverterProvider` 处理:
- 字符串
- 基本类型 + 包装类
- 枚举
- `static fromString(String)` / `static valueOf(String)` 方法
- 单字符串 String constructor
- `List<T>` / `Set<T>` / `SortedSet<T>` (CollectionParameterStrategy 决定)
- `@DefaultValue` / `@Encoded`

不支持 `@BeanParam`, 不支持 `@FormParam` (用 `MultivaluedMap<String,String> form` 替代)。

## 异步支持 (`@Suspended AsyncResponse`)

```java
@GET @Path("/async")
public void async(@Suspended AsyncResponse resp) {
    new Thread(() -> {
        try { Thread.sleep(1000); }
        catch (InterruptedException ignored) {}
        resp.resume("done");
    }).start();
}
```

`AsyncResponseAdapter` (内部类) 包装 mu 的 `AsyncHandle`, 提供:
- `resume(Object)` / `resume(Response)` / `resume(Throwable)`
- `setTimeout(long, TimeUnit)` + `setTimeoutHandler(TimeoutHandler)`
- `register(CompletionCallback)` / `register(ConnectionCallback)`
- `cancel(...)` / `isCancelled()` / `isSuspended()` / `isDone()`

## SSE (SsePublisher 之上)

- `SsePublisher.start(req, resp)` (在 mu 主包) → 返回 `SsePublisher`
- JAX-RS 也有 `SseEventSink` / `SseBroadcaster` (`JaxSseImpl`):
  - `JaxSseImpl` 实现 `Sse` (创建 EventSink / Broadcaster factory)
  - `JaxSseEventSinkImpl` 实现 `SseEventSink`, 委托给 `SsePublisher`
  - `SseBroadcasterImpl` 实现 `SseBroadcaster`, 多订阅 fan-out

## Entity Providers

| 类型 | 支持 |
|---|---|
| `byte[]` | ✓ `*/*` |
| `String` | ✓ `*/*` |
| `InputStream` | ✓ `*/*` |
| `Reader` | ✓ `*/*` |
| `File` | ✓ `*/*` |
| `MultivaluedMap<String,String>` | ✓ `application/x-www-form-urlencoded` |
| `StreamingOutput` | ✓ `*/*` writer only |
| `Boolean`/`Character`/`Number` | ✓ `text/plain` |
| 原生类型 (经装箱) | ✓ |
| `DataSource` | ✗ (Java 9 移除) |
| `Source` / `JAXBElement` / JAXB | ✗ |
| `JAXBContext` | ✗ |

## CORS

共享配置 `CORSConfig` (在 rest 包), `handlers/CORSHandler` 也复用它。

## OpenAPI 自动生成

`OpenApiDocumentor` 通过反射:
1. 遍历 resource class 的 `@Operation`, `@Parameter`, `@ApiResponse`, `@Schema` 等
2. 对方法签名反推参数 (path/query/header/cookie/body)
3. 生成 `OpenAPIObject` JSON
4. `HtmlDocumentor` 输出 Swagger UI HTML

支持自定义: `SchemaObjectCustomizer` 在 schema 构造过程中修改生成的 schema (例如加 example, 加 format)。

`addOpenApiCustomizer(SchemaObjectCustomizer)` / `addOpenApiCustomizerToTargets(...)`。

## 过滤器 (ContainerRequestFilter / ContainerResponseFilter)

- 注册方式: `addRequestFilter(MyFilter)` / `addResponseFilter(MyFilter)`
- `@PreMatching` 通过 `addPreMatchRequestFilter(MyFilter)` 单独注册
- 不支持 `@NameBinding` 通过 `addRequestFilter` 注册多个 (注册顺序即执行顺序)
- 异常映射: `addExceptionMapper(NotFoundException.class, new MyMapper())`

## Basic Auth

- `BasicAuthSecurityFilter` + `UserPassAuthenticator` + `Authorizer`
- 用户实现 `Authorizer.authorize(username, password)` 返回 `Optional<SecurityContext>`

## 不支持的部分 (节选自 README)

- ❌ `@Provider` 自动发现
- ❌ 类路径扫描 (`getClasses()` / `getSingletons()`)
- ❌ 自动实例化 (用户必须自己 `new`)
- ❌ `Application` 子类
- ❌ 字段注入 (`@Context` 字段)
- ❌ 优先级 (`@Priority`)
- ❌ Bean Validation (`@Valid`, `@NotNull`)
- ❌ `@BeanParam`
- ❌ XML / JAXB
- ❌ Client API

## 设计观察

1. **手工实例化**让 mu 不需要 reflection 的类加载器 hack, 启动更快、APK 友好、native-image 友好。
2. **没有优先级**是个明确的取舍: 简化实现, 但要求用户自己排序。
3. **没有 Bean Validation**是个明确缺失 (REST 写大型应用时常被抱怨)。
4. **不实现 Client API**——专注服务端, 这是 mu-server 的设计取舍。
5. **OpenAPI 内置**且可定制, 比 Jersey (需要 swagger-jersey2-jaxrs) / RESTEasy (需要 micro-profile-openapi) 简单很多。
6. **`rest/README.md` 把 spec 一节节列了支持矩阵**——这是项目对用户负责的体现。

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
