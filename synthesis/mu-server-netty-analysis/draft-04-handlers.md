---
title: "mu-server 内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)"
category: synthesis
tags: [java, mu-server, handlers, cors, csrf, static-resources]
sources: ["mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "CORSHandler / CSRFProtectionHandler / HttpsRedirector / ResourceHandler (静态资源) / DirectoryLister 等开箱即用 handler"
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

# Draft 04 — Built-in Handlers (`io.muserver.handlers`)

| File | Purpose |
|---|---|
| `CORSHandler.java` + `CORSHandlerBuilder.java` | Wraps a `CORSConfig` to emit CORS headers; uses JAX-RS internals to enumerate allowed methods. Defaults to all methods except TRACE/CONNECT (CORSHandlerBuilder.java:79-82). |
| `CSRFProtectionHandler.java` + `Builder` | CSRF token check (cross-origin unsafe methods require a valid token). |
| `HttpsRedirector.java` + `Builder` | Redirects plain-HTTP requests to HTTPS by replying 301 with a `Location: https://...` header. |
| `ResourceHandler.java` + `ResourceHandlerBuilder.java` | Serves static files from filesystem / classpath / webjars. 361-line builder with `withExtensionToResourceType`, `withDefaultFile`, `withResourceProviderFactory`, `withDirectoryListing`, `withRangeSupport` (via `BytesRange`). |
| `ResourceType.java` | Mime-type → gzip-decision table (`gzippableMimeTypes`). |
| `ResourceProvider.java` | Strategy interface for resolving a path to a `URL` (filesystem vs classpath vs webjar). |
| `BytesRange.java` | Parses `Range: bytes=...` headers for partial content (HTTP 206). |
| `DirectoryLister.java` | Generates HTML directory listings when `withDirectoryListing(true)`. |
| `BareDirectoryRequestAction.java` | Enum: what to do when `/dir` (no trailing slash) is requested — redirect-with-slash, serve-with-slash, serve-without-slash. |
| `ResourceCustomizer.java` | Hook for adding custom headers / caching rules per-resource. |
| `package-info.java` | Package documentation. |

## Static file serving

* `ResourceHandlerBuilder.fileOrClasspath(...)` (presumed; not read line-by-line) — tries filesystem
  first, falls back to classpath. Useful for dev/prod parity.
* Webjars: `WEBJAR_GROUP_IDS = {org.webjars.npm, org.webjars}` (ResourceHandlerBuilder.java:36) —
  looks up Maven-style `META-INF/resources/webjars/<name>/<version>/...`.
* `withRangeSupport()` (not yet read) — wires in `BytesRange`.

## CORS

* Implementation is backed by `io.muserver.rest.CORSConfig` (the same config used by the JAX-RS
  handler) so a server can share the same CORS policy across routes.
* `CORSHandler.handle` delegates most of the header writing to `corsConfig.writeHeadersInternal(...)`
  (RestHandler.java:125).

## CSRF

* Skim only. Standard double-submit / origin-check pattern.
* Tokens are typically tied to a session cookie; not all cookie sessions are tracked by mu-server
  out-of-the-box (no built-in session manager), so the handler is for apps that bring their own.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
