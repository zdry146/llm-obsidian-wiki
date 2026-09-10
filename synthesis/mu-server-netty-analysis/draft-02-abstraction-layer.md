---
title: "mu-server 抽象层 (Request/Response/HttpExchange)"
category: synthesis
tags: [java, netty, mu-server, abstraction, adapter-pattern]
sources: ["mu-server 0.0.3-SNAPSHOT @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "Netty NettyRequestAdapter/ResponseAdaptor 包装 Netty 底层 HttpRequest/ChannelFuture, 提供用户友好的 MuRequest/MuResponse API, HttpExchange 协调两件套 + 状态机"
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

# Draft 02 — Abstraction Layer

> Goal: hide Netty objects behind a small, framework-neutral API (MuRequest / MuResponse / HttpExchange).

## 1. `MuRequest` (241 lines) — public interface

What the user sees:
* `method()`, `uri()`, `serverURI()` (line 113-128) — URI built from `proto://host(headers)+requestUri`,
  with `Forwarded` / `X-Forwarded-*` headers overriding scheme/host when present (lines 80-96).
* `headers()` returns a `Headers` view over the parsed netty headers (line 132-134).
* `inputStream()`, `readBodyAsString()`, `form()`, `uploadedFiles(...)` — body readers, single-shot only
  (line 141-251). Body access is claim-once; calling `inputStream()` after `readBodyAsString()` throws.
* `query()` returns a `NettyRequestParameters` wrapping a `QueryStringDecoder(uri, true)` (line 61, 243-245).
* `cookies()`, `cookie(name)` (line 254-279) — decoded lazily by `ServerCookieDecoder.STRICT`.
* `attributes()` / `attribute(key[, value])` (line 291-315) — request-scoped map.
* `handleAsync()` (line 317-326) — returns an `AsyncHandleImpl` so a handler can yield and complete later.
* `remoteAddress()`, `clientIP()` (line 329-342) — `clientIP()` honours `Forwarded.for=` first.
* `protocol()` returns `"HTTP/1.1"` or `"HTTP/2"` from `nettyRequest.protocolVersion().text()` (line 71-73).
* `connection()` returns `HttpConnection` for stats / SNI / client cert / proxy info access.

## 2. `MuResponse` (123 lines) — public interface

User-facing surface:
* `status(int)` / `status()` (line 26-34) — defaults to 200.
* `write(String)` (single-shot, line 37-42), `sendChunk(String)` (chunked, line 45-50),
  `outputStream()` / `outputStream(int)` (line 88-102), `writer()` (line 105-110).
* `headers()` (line 67-71), `contentType(...)` (line 73-78), `addCookie(...)` (line 80-86).
* `redirect(String|URI)` (line 52-64).
* `hasStartedSendingData()` (line 113-117) — gates late mutation of status/headers.
* `responseState()` (line 119-122) — `ResponseState` enum.

The contract: "only one of write / outputStream / writer per response, aside from `sendChunk`"
(line 19-22). The implementation enforces this in `NettyResponseAdaptor`.

## 3. `HttpExchange` (478 lines) — internal coordination class

* Implements both `ResponseInfo` and `Exchange` (line 35).
* Holds `request: NettyRequestAdapter`, `response: NettyResponseAdaptor`, `streamId: int` (-1 for HTTP/1,
  ≥0 for HTTP/2), `state: HttpExchangeState`.
* `addChangeListener(HttpExchangeStateChangeListener)` (line 110-112) — used by `Http1Connection` and
  `Http2Connection` to know when the exchange ends.
* `onReqOrRespStateChange(...)` (line 114-125) — drives `state` to `COMPLETE`, `ERRORED`, or `UPGRADED`
  once both request and response reach end states.
* `block(Runnable)` / `block(Callable<ChannelFuture>)` (line 63-98) — **the keystone of the threading
  model**. Submits a task to `ctx.executor()` (the Netty event loop) and blocks the worker thread on
  `Future.get()`. Asserts `!inLoop()` first so a misuse inside the event loop fails fast.
  See draft 07 for full breakdown.
* `onMessage(ctx, msg, DoneCallback)` (line 188-232) — the body-pump entry; copies bytes into the
  request body reader and updates state.
* `scheduleReadTimeout()` / `cancelReadTimeout()` (line 234-246) — replaces Netty's `IdleStateHandler`
  while the body is being read; uses `ctx.executor().schedule(...)`.
* `onIdleTimeout(...)` (line 248-254) — closes if `ALL_IDLE`.
* `onException(...)` (line 386-446) — converts thrown exceptions into either a custom handler call or
  a 500 with a generated `ERR-<uuid>` token (line 406-409). Sends `Connection: close` for HTTP/1 when
  the stream is unrecoverable (line 421-423).
* `throwIfInvalid(...)` (line 346-380) — converts Netty decoder failures (TooLongFrameException,
  missing Host, oversized Content-Length) into 431/414/413/417 errors.

## 4. Adapter pattern — `NettyRequestAdapter` / `NettyResponseAdaptor`

* `NettyRequestAdapter implements MuRequest` (561 lines).
* Owns the request lifecycle state machine: `HEADERS_RECEIVED → RECEIVING_BODY → COMPLETE / ERRORED`
  (line 36, 439-450). Listeners (`RequestStateChangeListener`) get notified when `setState` is called
  from the event loop.
* `claimingBodyRead(reader)` (line 205-221) — guards the body reader to a single consumer and transitions
  to `RECEIVING_BODY`. Submits to event loop if not already there. Called from `inputStream()`,
  `readBodyAsString()`, `form()`, `uploadedFile()`.
* `websocketUpgrade(...)` (line 403-426) — performs the netty `WebSocketServerHandshaker.handshake`
  then fires `ExchangeUpgradeEvent` so the connection can switch modes.
* `AsyncHandleImpl` (line 482-559) — the user's handle for async handlers. `complete()` schedules
  `HttpExchange.complete()` on the event loop; `write(ByteBuffer)` bypasses the body-of-response
  write path and writes directly via `NettyResponseAdaptor.writeAndFlush`.

## 5. Body readers — `RequestBodyReader` (file), `RequestBodyReaderInputStreamAdapter`

The `RequestBodyReader` hierarchy (`InputStream`, `String`, `UrlEncodedBodyReader`, `MultipartFormReader`,
`ListenerAdapter`, `DiscardingReader`) is pluggable into `NettyRequestAdapter.claimingBodyRead(...)`.
The request body is owned by **one** reader — the second body access throws
`IllegalStateException("The body of the request message cannot be read twice...")` (line 207).

## 6. State enums

* `RequestState` (`RequestState.java`): `HEADERS_RECEIVED, RECEIVING_BODY, COMPLETE, ERRORED`.
* `ResponseState` (`ResponseState.java`): drives the response side.
* `HttpExchangeState` (HttpExchange.java:463): `IN_PROGRESS, COMPLETE, ERRORED, UPGRADED`.

## 7. Headers / Cookies / Form / Multipart

* `Headers` interface, with `Http1Headers` (wraps `HttpHeaders`) and `Http2Headers` (wraps netty
  `io.netty.handler.codec.http2.Http2Headers`) as the two implementations.
* `Cookie` + `CookieBuilder` — public API; converted to/from netty `io.netty.handler.codec.http.cookie.Cookie`.
* `ParameterizedHeader` / `ParameterizedHeaderWithValue` — q-value aware header parsing
  (used for `Accept`, `Accept-Encoding`, `Content-Type`).
* `UploadedFile` — represents a single file from a `multipart/form-data` request.

## 8. Connection interface — `HttpConnection`

* Per-connection stats: `completedRequests()`, `invalidHttpRequests()`, `rejectedDueToOverload()`,
  `activeRequests()`, `activeWebsockets()`.
* SSL info: `httpsProtocol()`, `cipher()`, `clientCertificate()`, `sniHostName()`.
* Network info: `protocol()`, `isHttps()`, `remoteAddress()`, `proxyInfo()`.
* Lifecycle: `startTime()`, `server()`.
* All `Http1Connection` and `Http2Connection` provide these.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
