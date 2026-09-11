---
title: "mu-server 2.4.2 JAX-RS 3.0 / REST 支持"
category: synthesis
tags: [java, mu-server, jax-rs, rest, annotations, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "内置 jakarta.ws.rs 注解支持 (@Path/@GET/@POST), MuRuntimeDelegate 入口, ResourceBuilder 路由, UriInfo 参数绑定, Filters/Interceptors, EntityProviders, OpenAPI 集成"
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

# Draft 05 — JAX-RS Implementation (`io.muserver.rest`)

> Scope: mu-server's JAX-RS 3.0 (jakarta.ws.rs) implementation — the largest
> sub-package. References the spec-conformance README at
> `src/main/java/io/muserver/rest/README.md`.

## 1. The "no classpath scanning, no DI" stance

From the README:

> "The Mu Jax-RS implementation does not support classpath scanning or
> definition of Resource classes, and as such the
> `jakarta.ws.rs.core.Application` class is not supported. All resources
> and optional providers are registered programmatically using the
> `io.muserver.rest.RestHandlerBuilder` class."

This is **the** defining philosophy of mu-server's REST layer — it
forbids:

- `Application` subclasses
- Per-request resource instantiation (only singletons)
- `Priority` annotation on providers
- Resource lifecycle (`@PostConstruct`, `@PreDestroy`)
- Bean Validation (§7) — entire chapter
- Context Providers (§4.3)
- `@Provider` auto-registration

This is what makes mu-server small (≈ 5,000 LOC for all of REST) versus
Jersey/RESTEasy (~hundreds of thousands).

## 2. `RestHandlerBuilder` (598 lines) — the entry point

### 2.1 What you can register

| Category | Method | Notes |
|---|---|---|
| Resources | `addResource(Object...)` | Class instances, must be singletons |
| Entity writers | `addCustomWriter(MessageBodyWriter<T>)` | Per-class |
| Entity readers | `addCustomReader(MessageBodyReader<T>)` | Per-class |
| Param converters | `addCustomParamConverterProvider(...)`, `addCustomParamConverter(Class, ParamConverter)` | For `@QueryParam` etc. |
| Reader/Writer interceptors | `addReaderInterceptor(...)`, `addWriterInterceptor(...)` | Order = execution order |
| Request filters | `addRequestFilter(ContainerRequestFilter)` | `@PreMatching` annotation splits them into pre-match / post-match buckets |
| Response filters | `addResponseFilter(ContainerResponseFilter)` | Post-method |
| Exception mappers | `addExceptionMapper(Class<T>, ExceptionMapper<T>)` | By class |
| CORS | `withCORS(CORSConfig)` | Defaults to `CORSConfigBuilder.disabled().build()` |
| OpenAPI | `withOpenApiDocument(OpenAPIObjectBuilder)`, `withOpenApiJsonUrl(...)`, `withOpenApiHtmlUrl(...)` | Generates both endpoints automatically |
| Schema customisation | `addCustomSchema(Class<?>, SchemaObject)`, `addSchemaObjectCustomizer(SchemaObjectCustomizer)` | For OpenAPI |
| Collection param parsing | `withCollectionParameterStrategy(...)` | See below |

### 2.2 The collection-parameter gotcha

`RestHandlerBuilder.build()` (lines 575-593) scans every resource method
parameter that is a `Collection` and is bound to `@HeaderParam` or
`@QueryParam`. If no `CollectionParameterStrategy` is set, it **fails to
build** with:

> "Please specify a string handling strategy for collections for
> querystring and header parameters. Please note that the behaviour of
> these parameters have changed since Mu Server 0.70.0 to follow the
> JAX-RS standard. Previously, a parameter values such as 'one,two,three'
> when passed to a collection parameter would be interpreted as 3
> values, however the JAX-RS standard is for this to be a single value."

This is a deliberate **breaking change** guard — users must opt into
`NO_TRANSFORM` (default) or `SPLIT_ON_COMMA`.

## 3. The matching pipeline

`RestHandler.handle` (364 lines) is the main `MuHandler`. The high-level
flow per request:

```
pre-match request filters (in order)         (annotation: @PreMatching)
  ↓
RequestMatcher.findRoute(request, resources)  (matches method+path+consumes+produces)
  ↓
matched = ResourceMethod + path params
  ↓
post-match request filters
  ↓
method.invoke(params...)                       (reflection)
  ↓
if return is CompletionStage: wait for completion
if return is AsyncResponse (@Suspended): wait for completion
  ↓
write response body via EntityProviders
  ↓
response filters
```

### 3.1 `UriPattern` (222 lines, draft 03 §3)

Compiled per `@Path` template at registration time. Reused across requests.

### 3.2 `RequestMatcher` (299 lines)

Walks every `ResourceClass.rootMethods + ResourceClass.locatorMethods`,
then every sub-resource method, doing:

1. Method match.
2. URI pattern match (via `UriPattern.matcher(...).fullyMatches()`).
3. Content-Type match (consumes).
4. Accept match (produces).
5. Picks the **first** fully-matching method.

Sub-resource locators return an object instance; the matcher recurses
into that object's `@Path`-annotated methods. Sub-resources must be
**already instantiated** — the spec's `ResourceContext` (which would
auto-instantiate) is **not** implemented (README §10.2.7).

### 3.3 `ResourceClass.fromObject(instance, ...)` 

Scans the instance class for `@Path` (class-level) + public methods
annotated with HTTP method annotations. Builds:

- `rootMethods: List<ResourceMethod>` — paths relative to class `@Path`.
- `locatorMethod: ResourceMethod?` — sub-resource locator (return type is not void/Response, no `@GET/POST/...`).
- `subResourceMethods: List<ResourceMethod>` — methods on the sub-resource class.
- `params: List<ResourceMethodParam>` — `@QueryParam`, `@PathParam`, `@HeaderParam`, `@CookieParam`, `@FormParam`, `@MatrixParam`, `@BeanParam`, `@Context`.

### 3.4 `ResourceMethod.invoke(params...)` (line 88)

Plain reflection: `methodHandle.invoke(resourceClass.resourceInstance, params)`. The `InvocationTargetException` is unwrapped to the user exception.

### 3.5 Async support

The README marks §8 (Asynchronous Processing) as fully implemented.
Three async patterns are supported:

1. `@Suspended AsyncResponse` parameter — mu-server keeps the exchange
   open until `asyncResponse.resume(...)` is called from another thread.
2. `CompletionStage<T>` return type — mu-server waits for the stage,
   then writes the result body.
3. `@Suspended AsyncResponse` + `CompletionCallback` / `ConnectionCallback`
   for completion notification.

The implementation lives in `AsyncResponseAdapter.java` (not detailed
here) — it bridges `AsyncResponse` to `AsyncHandle`.

### 3.6 SSE (Server-Sent Events)

§9 is implemented. Two interfaces:

- `SseEventSink` — per-connection sink, registered by JAX-RS
  `@Produces(SERVER_SENT_EVENTS)`.
- `SseBroadcaster` — fan-out from one event to many sinks.

Implementation files: `JaxSseImpl`, `JaxSseEventSinkImpl`,
`SseBroadcasterImpl`, `JaxOutboundSseEvent`, `JaxOutboundSseEventBuilder`.

## 4. Entity body codec (`EntityProviders`)

`EntityProviders` is the registry of `MessageBodyReader` / `MessageBodyWriter`
pairs. `builtInReaders()` / `builtInWriters()` produce the default set
(see README §4.2.4 — supports byte[], String, InputStream, Reader, File,
StreamingOutput, primitives, MultivaluedMap<String,String>, and
auto-boxed variants).

`StringEntityProviders` and `PrimitiveEntityProvider` are the
implementations. `BinaryEntityProviders` for `byte[]`/`InputStream`/
`Reader`/`File`. `BuiltInParamConverterProvider` for primitive/boxed/enum
path/query/header parameters.

## 5. `JaxRSRequest` / `JaxRSResponse` (492 / 736 lines)

`JaxRSRequest` adapts `MuRequest` to `jakarta.ws.rs.core.Request`:
- `selectVariant(List<Variant>)` — content negotiation
- `getCookies()`, `getDate()`, `getHeaderString()`, `getLanguage()`, `getAcceptableLanguages()`, `getAcceptableMediaTypes()`, `getMediaType()`, `getMethod()`, `getUri()`, `getRequestUri()`, `resolveUri()`, `getUserPrincipal()` / `isUserInRole()` (via `MuSecurityContext`)

`JaxRSResponse` is the reverse — builds a `Response` from JAX-RS into a
`MuResponse`. Has explicit handling for `GenericEntity`, `Response`
(status + headers + entity), `StreamingOutput`, SSE events.

## 6. Header delegates

mu-server ships its own `HeaderDelegate` implementations rather than
relying on JAX-RS providers (because the spec doesn't include them):

- `MediaTypeHeaderDelegate` (RFC parsing/serialization)
- `CacheControlHeaderDelegate`
- `CookieHeaderDelegate`
- `NewCookieHeaderDelegate`
- `DateHeaderDelegate`
- `EntityTagDelegate`
- `LinkHeaderDelegate`
- `CacheControlHeaderDelegate`

## 7. URI building (`MuUriBuilder`, 519 lines)

`UriBuilder` is required by JAX-RS for any HATEOAS-style response. mu-server
implements `MuUriBuilder` with full RFC 3986 template substitution:
`{var}`, `{?var1,var2}`, `{/var}`, `{var:regex}`, etc.

## 8. Runtime delegate

`MuRuntimeDelegate.ensureSet()` is called by every `HttpExchange` (static
block in `HttpExchange.java:38`) and by `MuServerBuilder` (line 42). It
registers `MuRuntimeDelegate` as the JAX-RS `RuntimeDelegate` via the
service-loader mechanism (`META-INF/services/jakarta.ws.rs.ext.RuntimeDelegate`).

## 9. CORS inside REST

`CORSConfig` (in this package) is the same code that `CORSHandler` uses.
`RestHandlerBuilder.withCORS(...)` applies it to REST routes only;
`CORSHandler` applies it to any handler chain. They cannot conflict
because each request only enters one — the `CORSHandler` (if any) runs
first; if a request matches a REST route, the REST handler's CORS
applies.

## 10. Security context

`MuSecurityContext` implements `jakarta.ws.rs.core.SecurityContext`. It
is populated by `BasicAuthSecurityFilter` if registered, and by
`UserPassAuthenticator` callbacks.

## 11. Problem Details (RFC 7807)

`ProblemDetailsException` + `ProblemDetailsExceptionBuilder` +
`ProblemDetailsExceptionMapper` provide RFC 7807-style
`application/problem+json` error responses. The mapper is registered
automatically (every `RestHandler` gets one by default in
`RestHandlerBuilder.build()` — `CustomExceptionMapper` wraps it).

## 12. README spec coverage summary

| Spec section | Status |
|---|---|
| §2 Application | N/A — not supported |
| §3 Resources (most) | ✅ |
| §3.1.1 Lifecycle (per-request) | ❌ — never |
| §3.3.4 Exceptions | ✅ |
| §4 Providers | ✅ (manual registration only) |
| §4.2.4 Standard Entity Providers | ✅ except JAXB / `DataSource` / `Source` |
| §4.3 Context Providers | ❌ |
| §4.5 Exception Mapping | ✅ |
| §6 Filters / Interceptors | ✅ (no priorities) |
| §7 Validation | ❌ — not implemented |
| §8 Async Processing | ✅ |
| §9 SSE | ✅ |
| §10.2.x `@Context` types | Most — Application / Providers / ResourceContext / Configuration unsupported |
| §11 Environment | N/A |
| §12 Runtime Delegate | ✅ |

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
