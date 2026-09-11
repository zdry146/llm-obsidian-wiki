---
title: "mu-server 功能模块 (SSE/TLS/限流/统计/WebSocket)"
category: synthesis
tags: [java, mu-server, sse, tls, rate-limit, websocket, stats]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "SsePublisher/AsyncSsePublisher 长连接, HttpsConfigBuilder SSL/TLS 配置 + 证书热重载, RateLimiter, MuStatsImpl 统计, WebSocket 支持"
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

# Draft 06 — Feature Modules

## 1. SSE — `SsePublisher` + `AsyncSsePublisher`

* Two flavours:
  * `SsePublisher.start(req, resp)` (SsePublisher.java:107-111) — returns an `SsePublisherImpl` backed
    by `req.handleAsync()`. Used in classic handlers.
  * `AsyncSsePublisher.start(req, resp)` (separate file) — used by the JAX-RS `SseEventSink` path
    (`RestHandler.getContextParam` line 467-469).
* `SsePublisherImpl.send(...)` (SsePublisher.java:114-196) serialises to `text/event-stream` format
  with optional `id:`, `event:`, multi-line `data:` (line 187-193). Uses `sendChunk(...)` which
  writes via `asyncHandle.write(buf).get()` (line 155) — every send blocks on the event-loop
  write future.
* `setClientReconnectTime(...)` writes `retry: <ms>\n` (line 203-205) — clients honour it.
* `JaxSseEventSinkImpl` (rest/) and `SseBroadcasterImpl` (rest/) implement the JAX-RS `SseEventSink`
  and `SseBroadcaster` interfaces.

## 2. TLS — `HttpsConfigBuilder` + `SslContextProvider`

* `HttpsConfigBuilder` (file) — wraps `SslContextBuilder`. Defaults to a self-signed localhost cert
  (`HttpsConfigBuilder.unsignedLocalhost()`, MuServerBuilder.java:715).
* `changeHttpsConfig(...)` (MuServerImpl.java:128-139) — hot-reloadable. The new
  `SslContextProvider.set(nettySslContext)` (line 134) swaps the live `SslContext` used by all
  subsequent TLS handshakes. In-flight requests keep using the old context.
* `MuSniHandler` (file) — uses Netty's `DomainWildcardMappingBuilder` to choose a context per SNI
  hostname.
* `SSLCipherFilter.java` — restricts the cipher suite list.

## 3. Rate limiting — `RateLimiterImpl`

* Built on Netty's `HashedWheelTimer` (RateLimiterImpl.java:18).
* Per-bucket `AtomicLong` counter; when the timer fires, decrements and removes the bucket if it
  drops to zero (line 41-46). So this is a **fixed-window counter** rate limiter, not a leaky bucket.
* `record(MuRequest)` (line 25-49) is called from `ServerSettings.block(MuRequest)` (note: not
  directly read yet, but referenced from `HttpExchange.create` line 291-293 and
  `Http2Connection.onHeadersRead` line 264-266).
* `RateLimitRejectionAction` (file) — either `SEND_429` (default) or simply *log* without rejecting.

## 4. Stats — `MuStats` / `MuStatsImpl`

* `MuStats` (file) is the public interface: `requests(), activeConnections(), activeRequests(),
  completedRequests(), invalidHttpRequests(), rejectedDueToOverload(), currentReceivedBytes(),
  currentSentBytes()`.
* `MuStatsImpl` (file) tracks counts and reads `trafficShapingHandler.trafficCounter()` for global
  bytes.
* `HttpConnection.completedRequests()` etc. expose per-connection counters.

## 5. Exceptions — `UnhandledExceptionHandler`

* Set on `MuServerBuilder.withExceptionHandler(...)` (MuServerBuilder.java:443-446).
* Signature: `boolean handle(MuRequest, MuResponse, Throwable)`. Returning `true` means the
  handler consumed the exception.
* `NettyHandlerAdapter.useCustomExceptionHandlerOrFireIt` (NettyHandlerAdapter.java:62-74) wraps the
  call: if the handler returns true, it flushes & completes. Otherwise fires the generic
  `HttpExchange.onException` path which renders the HTML 500 with `ERR-<uuid>`.

## 6. WebSocket — `WebSocketHandlerBuilder`

* `MuWebSocketFactory` builds `WebSocketServerHandshaker` (NettyRequestAdapter.java:403-426).
* `MuWebSocket` (interface) — text/binary message handlers, close handler, ping handler.
* `BaseWebSocket` — abstract convenience class.
* `WebSocketHandlerBuilder` — builder producing a `WebSocketHandler`.
* Handshake happens during request handling: `NettyRequestAdapter.websocketUpgrade(...)` (line
  403-426) replaces the `idle` handler with a more aggressive `IdleStateHandler(idleRead, ping,
  0)` and fires `ExchangeUpgradeEvent`. The connection handler then swaps `currentExchange` to
  the new `MuWebSocketSessionImpl` (Http1Connection.java:188-201).
* WebSocket callbacks run on the **NIO event loop** (per `MuServerBuilder.withNioThreads` doc
  MuServerBuilder.java:204).

## 7. Forwarded / X-Forwarded headers — `ForwardedHeader`

* Parsed by `Headers.forwarded()` (referenced from NettyRequestAdapter.java:82).
* `MuRequest.clientIP()` (line 333-342) checks `ForwardedHeader.forValue()` then `X-Forwarded-For`
  then falls back to socket address.
* `MuRequest.uri()` is rebuilt from forwarded scheme/host (line 80-96).

## 8. Multipart upload — `RequestBodyReader.MultipartFormReader`

* Parses `multipart/form-data` request bodies.
* `UploadedFile` — represents a single file part (stream + filename + content-type + size).
* `MuRequest.uploadedFiles(name)` / `uploadedFile(name)` (line 230-239) return them.
* Capped at `MuServer.maxRequestSize()` (default 24 MB).

## 9. Compression — `SelectiveHttpContentCompressor`

* Wired into the HTTP/1 pipeline (MuServerBuilder.java:828-830) when `gzipEnabled=true`.
* Inspects `Accept-Encoding` and the configured mime-type set; passes through otherwise.
* For HTTP/2, mu-server ships custom encoders: `MuCompressorHttp2ConnectionEncoder` and
  `MuGzipHttp2ConnectionEncoder`.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
