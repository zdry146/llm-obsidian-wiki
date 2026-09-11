---
title: "mu-server JAX-RS 3.0 支持"
category: synthesis
tags: [java, mu-server, jax-rs, rest, annotations]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
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

# Draft 05 — JAX-RS Layer (`io.muserver.rest`)

> 84 files, the largest single subsystem. Maps Jakarta REST 3.1 onto the mu-server primitives.

## 1. Entry point — `RestHandler implements MuHandler`

`RestHandler.java:41-489`. Implements `MuHandler.handle(req, resp)` — i.e. plugs into the normal
`NettyHandlerAdapter` chain as if it were any other handler.

### Flow inside `handle` (line 75-199)

1. If a `documentor` is registered and consumes the request (e.g. OpenAPI doc), short-circuit.
2. Build a `JaxRSRequest` (line 84) which wraps `MuRequest` + `MuResponse` + `LazyAccessInputStream`
   + relative path + security context + reader interceptors.
3. Parse `Accept` headers via `MediaTypeDeterminer`.
4. Run pre-matching filters via `filterManagerThing.onPreMatch(...)` (line 93). If a filter set an
   abort response, send it.
5. `requestMatcher.findResourceMethod(...)` — the JAX-RS router. Throws `NotAllowedException` for
   wrong method on a known path. For HEAD, fall back to GET. For OPTIONS, write an `Allow` header
   and return.
6. Run post-matching filters (`onPostMatch`).
7. If method has `@Suspended AsyncResponse`, wrap the rest in a callback so async completion calls
   `sendResponse(...)`.
8. Otherwise invoke the resource method synchronously.
9. If the method returns a `CompletionStage`, wait on it asynchronously via
   `AsyncHandle + whenComplete`. Otherwise send the response.
10. Exception mapping via `dealWithUnhandledException` which uses `customExceptionMapper` (user-
    provided) and falls back to a generic 500.

## 2. JAX-RS adapter — `JaxRSRequest` (rest/JaxRSRequest.java)

* Implements `ContainerRequestContext`.
* Lazy input stream (line 84) — the body is only consumed when an entity provider actually pulls.
* Supports `setProperty` / `getProperty` for filters to communicate.

## 3. JAX-RS response — `JaxRSResponse` (rest/JaxRSResponse.java)

* Implements `jakarta.ws.rs.core.Response`.
* Holds the entity stream + annotations + media type.

## 4. Resource introspection — `ResourceClass`, `ResourceMethod`, `ResourceMethodParam`,
`ResourceClassIntrospection`

* `ResourceClassIntrospection.scan(application, schemaCustomizer, providers, paramConverters)` —
  walks `Application.getSingletons()` and `getClasses()`.
* For each resource class, builds a `ResourceClass` holding:
  * `produces` (merged `@Produces` from class + methods).
  * `consumes`.
  * `path` (from `@Path`).
  * Sub-resource locators (`@Path` returning an instance).
* `ResourceMethod.methodHandle` — a `MethodHandle` (not `Method`) obtained via
  `MethodHandles.publicLookup().unreflect(...)` so the method can be invoked efficiently with
  parameter substitution.
* `ResourceMethodParam` describes each parameter: which `@XxxParam`, `@Context`, or message body.
  `getValue(requestContext, mm, collectionParameterStrategy)` is called per parameter to extract it.

## 5. URI matching — `UriPattern` + `PathMatch` + `RequestMatcher`

* `UriPattern.uriTemplateToRegex(template)` — converts an RFC 6570-style template into a regex,
  with named groups.
* `RequestMatcher` — top-level matcher. For an incoming `JaxRSRequest` it finds the deepest
  `@Path`-annotated resource whose template matches.
* Supports sub-resource locators: a method annotated with `@Path` but no HTTP verb returns an
  instance, and `RequestMatcher` recurses (line 99-114 of RestHandler).

## 6. Entity providers — `JaxRSProviders`, `StringEntityProviders`, `BinaryEntityProviders`,
`PrimitiveEntityProvider`, `SourceEntityProviders`

* `JaxRSProviders` — registry. Built-in readers/writers for `byte[]`, `String`, `InputStream`,
  `Reader`, `File`, `Source`, `MultivaluedMap<String,String>` (form-urlencoded), primitives, and
  `StreamingOutput` (writer only).
* `SchemaObjectCustomizer` — hook to tweak the OpenAPI schema.

## 7. Filters / interceptors — `FilterManagerThing`

* `onPreMatch` (pre-matching), `onPostMatch` (post-matching), `onBeforeSendResponse` (response
  filter phase).
* Built-in pre-match and post-match filters; user can register more via
  `RestHandlerBuilder.addRequestFilter` etc.

## 8. Built-in CORS — `CORSConfig` / `CORSConfigBuilder`

* Used by `RestHandler.handle` for `OPTIONS` and by the standalone `CORSHandler` for non-REST
  routes.

## 9. Content negotiation — `MediaTypeDeterminer`

* `parseAcceptHeaders`, `determine(...)` — chooses the best `MessageBodyWriter` based on the
  resource's `@Produces`, the request's `Accept`, and the registered providers.

## 10. SSE via JAX-RS — `JaxSseImpl`, `JaxSseEventSinkImpl`, `SseBroadcasterImpl`,
`AsyncSsePublisher`

* Resource methods with `@SseEventSink` parameter get a `JaxSseEventSinkImpl` (RestHandler.java:
  467-469) backed by `AsyncSsePublisher`.
* `SseBroadcaster` implementation (separate file) broadcasts events to multiple connected sinks.
* `MuRuntimeDelegate.connectedSinksCount(broadcaster)` returns the live count.

## 11. OpenAPI doc — `OpenApiDocumentor`, `HtmlDocumentor`

* `RestHandlerBuilder.withOpenApiDocumentor(...)` registers a `documentor` that serves
  `/openapi.json` + `/openapi.html`.

## 12. Spec compliance — rest/README.md is the authoritative spec checklist

Highlights:
* All resource classes must be **singletons**; per-request lifecycle is explicitly not supported.
* No automatic classpath scanning (`@Provider` annotation is ignored).
* Bean Validation is **not** implemented.
* `EntityPart` / multipart entity body reader/writer not implemented (the `multipart/form-data`
  whole-body path through `MuRequest.form()` works fine, but per-part injection into JAX-RS
  parameters does not).
* `Feature` / `DynamicFeature` / `Configuration` not implemented.
* `SeBootstrap` is implemented (file `MuSeBootstrap.java`).

## 13. JSON support

* **No built-in JSON entity provider** in this version. Users add their own via
  `RestHandlerBuilder.addCustomReader(...)` / `addCustomWriter(...)`. There's a `README` example
  referencing Jackson.

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
