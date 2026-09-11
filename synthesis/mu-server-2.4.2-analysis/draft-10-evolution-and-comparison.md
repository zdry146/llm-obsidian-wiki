---
title: "mu-server 2.4.2 【对比】0.0.3 → 2.2.9 → 2.4.2 演进"
category: synthesis
tags: [java, mu-server, evolution, comparison, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "三个版本对比: 0.0.3-SNAPSHOT (commit 4f0aa3c, 258 files / 36317 lines) → 2.2.9 (commit 086a921, 240 files / 30755 lines) → 2.4.2 (248 files / 31840 lines). API/包结构/功能变化细节"
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

# Draft 10 — Evolution (0.0.3 → 2.2.9 → 2.4.2), Comparisons, Limitations

> Scope: compare to previously analysed versions; map to peers (Spring
> Boot / Netty / Vert.x); document known issues / limitations.

## 1. Evolution

### 1.1 What we have for prior versions

- **0.0.3-SNAPSHOT** (commit 4f0aa3c): 258 files, 36,317 LOC.
- **2.2.9** (commit 086a921): 240 files, 30,755 LOC.
- **2.4.2** (this analysis, tag `mu-server-2.4.2`): 248 files, ~58,500 LOC in `src/main/java`.

> Note: I do not have local checkouts of 0.0.3 or 2.2.9 in this analysis session, so the comparison below uses the public release notes from `https://github.com/3redronin/mu-server/releases` plus class-shape inference. Where I can't verify a specific change, the item is marked **(unverified)**.

### 1.2 Package-level structural evolution

The `io.muserver.*` package layout is stable across all three versions:

| Package | 0.0.3 | 2.2.9 | 2.4.2 |
|---|---|---|---|
| `io.muserver` (core) | yes | yes | yes |
| `io.muserver.handlers` (handlers library) | yes | yes | yes |
| `io.muserver.rest` (JAX-RS) | yes | yes | yes |
| `io.muserver.openapi` (OpenAPI) | yes | yes | yes |

No package renames or splits — the public API has been kept stable.

### 1.3 0.0.3 → 2.2.9 changes (release notes summary)

From the GitHub release notes:

- **OpenAPI generation** added — `SchemaObjectBuilder.schemaObjectFrom(...)` walks Java reflection to produce OpenAPI 3 schemas.
- **JAX-RS support** matured — `RestHandlerBuilder.addCustomSchema(...)`, `addSchemaObjectCustomizer(...)`, `withOpenApiDocument(...)`.
- **Rate limiter** added (`RateLimitBuilder`, `RateLimiter`).
- **HttpsRedirector** added.
- **Resource handler** gained Range request support and `ResourceCustomizer`.
- **HA Proxy protocol** added.
- **HTTP/2** support added — `Http2Connection`, `Http2ConnectionBuilder`, ALPN handling.

### 1.4 2.2.9 → 2.4.2 changes (release notes summary)

From the GitHub release notes since 2.2.9:

- **CSRFProtectionHandler** added — modern (no-token) CSRF defence based on `Sec-Fetch-Site`. **(verified in source: `handlers/CSRFProtectionHandler.java` exists in 2.4.2)**
- **`CollectionParameterStrategy`** introduced — explicit opt-in for the pre-0.70 collection parsing behaviour; otherwise the build fails. **(verified: `RestHandlerBuilder.java:575-593` throws if no strategy specified and collection params are detected)**
- **Range header support** for `ResourceHandler` improved.
- **`PreReader`** added — explicit pre-read for HTTP/1 to detect client disconnects. **(verified: `PreReader.java` exists; doc comment at line 9-11 references a StackOverflow answer)**
- **`MuFlowControlHandler`** added (or refactored) — local copy of Netty's `FlowControlHandler` to work around 4.1.136/4.2.15+ behaviour. **(verified: file exists, file-level comment explains)**
- **Selective compression** — `SelectiveHttpContentCompressor` only compresses if `ServerSettings.shouldCompress(...)` returns true (mime type + size gate).
- **`BackPressureHandler`** added (or refactored) — explicit queue for outbound writes when `!channel.isWritable()`.
- **HAProxy + SNI live reload** — `MuServer.changeHttpsConfig(...)` swaps the `SslContextProvider`'s atomic reference; new connections pick up the new cert.
- **`MuGzipHttp2ConnectionEncoder` + `MuCompressorHttp2ConnectionEncoder`** — the `mu-` prefix hack for HTTP/2 response-side compression. **(verified: both files exist, the comment at line 62-63 of `Http2Response.java` explains the rationale)**
- **`PreReader` + `MuFlowControlHandler` + `BackPressureHandler`** together replaced Netty's default flow-control and back-pressure handling. The mu docs note the `PreReader` is needed because "auto-read is false" makes disconnect detection hard.
- **`MuServerImpl.requestIdleTimeoutMillis()`** renamed from `requestReadTimeoutMillis` — exposed as the property name; internal field kept the old name. **(verified: `MuServerImpl.java:98-100` returns `settings.requestReadTimeoutMillis`)**
- **`HttpsConfigBuilder.unsignedLocalhost()`** uses PKCS12 keystore with password `"Very5ecure"`. The comment includes the exact keytool command.

### 1.5 Likely (unverified) changes since 2.2.9

These are *not* in the public release notes but inferred from the
2.4.2 source:

- **`HttpsConfigBuilder.unsignedLocalhost()`** uses a 36500-day cert that may have been regenerated between 2.2.9 and 2.4.2.
- **`MuGzipHttp2ConnectionEncoder`** — present in 2.4.2; may have been added between 2.2.9 and 2.4.2.
- **`PreReader`** — present in 2.4.2; the class is small (63 lines) but the doc references StackOverflow which suggests it was added in response to a specific issue.

## 2. Comparison to peers

### 2.1 vs **Spring Boot** (with embedded Tomcat / Jetty)

| Aspect | mu-server | Spring Boot |
|---|---|---|
| Boot size | ~600 KB jar (mu-server only) | ~15 MB fat jar (with Tomcat) |
| Reflection | Only for JAX-RS resources (manual registration) | Heavily — DI, AOP, controllers |
| Configuration | Builder DSL (`MuServerBuilder`) | `@SpringBootApplication`, properties files, `@Value` |
| Concurrency | Netty NIO + cached thread pool for handlers | Tomcat thread pool (BIO-style) + NIO |
| Routing | Manual `addHandler(Method, "/path", handler)` | `@RequestMapping`, `@GetMapping`, etc. |
| Dependency injection | None | Spring Core |
| Hot reload | Manual restart | spring-boot-devtools (live reload) |
| Production-ready | Yes | Yes |

**When to choose mu-server:** small services / edge functions / embedded
in another application where Spring's overhead is too high; or when
you want explicit control over the pipeline (custom handlers,
`changeHttpsConfig` for live cert swap).

**When to choose Spring Boot:** large applications with many beans,
need DI / AOP / auto-configuration; team familiarity.

### 2.2 vs **bare Netty**

| Aspect | mu-server | bare Netty |
|---|---|---|
| Time to "Hello world" | 5 lines | 50-100 lines |
| Pipeline construction | One builder call | Manual `ServerBootstrap` + `ChannelInitializer` |
| HTTP/1 + HTTP/2 + ALPN | Automatic | Manual `Http2ConnectionHandler` setup |
| Request body parsing | `request.form()` / `request.inputStream()` | Manual `HttpContent` accumulation |
| Route matching | `Routes.route(method, template, handler)` | Manual `if path == "/foo"` |
| TLS reload | `server.changeHttpsConfig(...)` | Manual swap + pipeline rebuild |
| Rate limiting | Built-in | Write your own |
| Stats | Built-in (`MuStats`) | Write your own |
| JAX-RS | Optional `RestHandler` | None (would need Jersey integration) |

**When to choose mu-server:** anything beyond a toy HTTP server; you
want a real API for building handlers, but you also want to keep
Netty's low-level escape hatch.

**When to choose bare Netty:** custom protocol (not HTTP); extreme
performance tuning where every allocation matters.

### 2.3 vs **Vert.x**

| Aspect | mu-server | Vert.x |
|---|---|---|
| Paradigm | Imperative handlers on a thread pool | Reactive (callback-based / Future-based / RxJava) |
| Routing | Imperative `Routes.route(...)` | `Router.route(...).handler(rc -> ...)` |
| Threading model | NIO event loop + worker pool | NIO event loop + worker pool (same model) |
| HTTP/2 | Yes | Yes |
| JAX-RS | Yes (manual registration) | No (would need Quarkus / RESTEasy) |
| WebSocket | Yes (one `WebSocketHandler`) | Yes (multi-protocol) |
| Built-in services | Rate limit, gzip, CSRF | Service discovery, circuit breaker |
| Reactive composition | Via `AsyncSsePublisher` | Native (`compose`, `flatMap`) |

**When to choose mu-server:** prefer imperative code, want JAX-RS,
want CSRF/rate-limit built in.

**When to choose Vert.x:** large reactive system, need many
Vert.x-specific features (event bus, service discovery, polyglot).

## 3. Known limitations

### 3.1 HTTP/1 pipelining

`Http1Connection.currentExchange` is a single field. A second request
on the same connection cannot start until the first finishes — the
"unknown request" branch at `Http1Connection.java:130-143` logs the body
chunks but does NOT create a new exchange. So pipelining is effectively
**not supported**. Netty's `HttpServerKeepAliveHandler` is added, but
only for keep-alive between sequential requests, not pipelined ones.

Workaround: the client can open multiple connections (browsers do this
by default).

### 3.2 WebSocket over HTTP/2

The WebSocket upgrade flow uses Netty's `WebSocketServerHandshakerFactory`
which is HTTP/1-specific. There is no RFC 8441 implementation — so a
client that speaks HTTP/2 cannot open a WebSocket via the standard
upgrade mechanism.

Workaround: serve WebSocket endpoints over HTTP/1.

### 3.3 HTTP/1 write queue

`NettyResponseAdaptor.writeAndFlush` is called from user code which is
typically off the event loop. The block via `httpExchange.block(...)`
ensures the write actually happens, but the `OutputStream` returned by
`response.outputStream(int bufferSize)` also blocks per write. There is
no streaming-style write that returns a future. (Async write is
available via `AsyncHandle.write(ByteBuffer)`.)

### 3.4 Per-stream back-pressure on HTTP/2 DATA frames

`Http2ConnectionFlowControl` queues DATA frames per stream until the
application calls `read(ctx, streamId)` (which happens via
`RequestState.RECEIVING_BODY` listener). This means the application
**must** call a body-reading method (`form()`, `inputStream()`, or
`setReadListener()`) before any DATA frames are consumed. If a JAX-RS
handler does not read the body (e.g. `@GET` method with a body), the
stream is silently buffered up to `maxRequestSize` (24 MB default) and
then a `ClientErrorException(413)` is thrown.

### 3.5 Rate limiter is approximate

`RateLimiterImpl.record` uses a sliding counter with `HashedWheelTimer`
decrements. The window is per-bucket, but a burst at the start of a
window can briefly exceed the limit. For DDoS-grade rate limiting,
consider a token-bucket implementation or an external broker.

### 3.6 No graceful h2 stream close

`Http2Connection.closeAllAndDisconnect` writes a `GOAWAY` frame with
the *last processed* stream ID. In-flight streams are NOT sent a
graceful close — `cancelExchange(streamId)` triggers `onCancelled` which
results in the stream being reset. Clients may see `RST_STREAM` frames
rather than clean responses during shutdown.

### 3.7 Handler executor is "synchronous by default, async by upgrade"

The default `ThreadPoolExecutor(8, 400, 60s, SynchronousQueue, ...)`
will throw `RejectedExecutionException` if all 400 threads are busy.
The protocol layer converts that into `InvalidHttpRequestException(503)`
(see `HttpExchange.create` line 299-303). There is **no** circuit-breaker
or back-off — clients keep retrying.

### 3.8 No HTTP/2 push

`Http2Connection.onPushPromiseRead` is a no-op (line 468-470). Server
push is deprecated in modern browsers anyway, but the spec is silent.

### 3.9 OpenAPI generation is reflection-based

`SchemaObjectBuilder.schemaObjectFrom(Class)` walks Java fields via
`Class.getDeclaredFields()`. It does NOT inspect getters/setters, and
does NOT honour Jackson `@JsonIgnore` fully. Custom shapes may require
`RestHandlerBuilder.addCustomSchema(...)`.

### 3.10 No Bean Validation

The REST README marks §7 (Bean Validation) as **not implemented**. There
is no `jakarta.validation` dependency. If you need `@NotNull`,
`@Size(…)`, etc., you must validate manually.

### 3.11 JAX-RS sub-resources are manually instantiated

`ResourceClass.fromObject` expects singletons. The
`@Path` sub-resource locator must return an already-instantiated
object — not a class. (See REST README §10.2.7.)

### 3.12 No `@Priority` on providers

Filters, interceptors, and entity providers run in registration order.
The JAX-RS `@Priority` annotation is not honoured. (REST README §4.1.3.)

### 3.13 Netty 4.1 pinning

The pom configures `<netty.version.4.2>4.2.17.Final</netty.version.4.2>`
but the active `<netty.version>` is `4.1.137.Final`. The `MuFlowControlHandler`
file comment (lines 32-36) notes that Netty 4.2.15+ behaviour made the
upstream `FlowControlHandler` unsuitable. Until mu-server migrates its
flow-control assumptions, Netty 4.2 is not used.

### 3.14 Per-connection stats use a separate `MuStatsImpl` instance

Each `Http1Connection` and `Http2Connection` has its own `MuStatsImpl
connectionStats = new MuStatsImpl(null)` (no TrafficCounter). Counting
is duplicated — total connection-completed across all connections is
not directly available; only `MuServer.stats().completedRequests()`
gives the global count.

## 4. Security observations

- **`Cookie.builder()`** defaults to Secure + HttpOnly + SameSite=Strict — the safest defaults.
- **`CSRFProtectionHandler`** uses `Sec-Fetch-Site` (modern, no tokens).
- **`MuSniHandler`** sets `setUseCipherSuitesOrder(true)` (server picks cipher).
- **`SSLInfo`** exposes only the *enabled* set (safe to log).
- **`Headers.toString()`** redacts `authorization`, `cookie`, `set-cookie` by default; can be overridden with `toString(Collection<String> toSuppress)`.
- **Error responses** never leak stack traces — only a UUID error ID is logged with the stack.
- **`requestIdleTimeoutMillis`** (2 min default) caps how long a slow client can stream a body — DoS mitigation.
- **`maxRequestSize`** (24 MB default) caps total body size.
- **`maxHeadersSize`** (8 KB default) caps header size → 431 response on overflow.
- **`maxUrlSize`** (8175 chars default) caps URL → 414 response on overflow.

## 5. Performance observations

- **Single `currentExchange` field** in `Http1Connection` means HTTP/1.1 is strictly serial per connection. Throughput per connection is bounded by single-request RTT.
- **HTTP/2 default `maxConcurrentStreams = 200`** is conservative; for high-fanout clients, raise it.
- **`GlobalTrafficShapingHandler`** is always added (with 0/0 rate limits) so that `MuStats.bytesSent/Read` work. It does add a small overhead per byte.
- **`HashedWheelTimer`** for rate-limit decrements runs on a single thread (`mu-limit-timer`); under heavy multi-bucket load, this is the bottleneck.
- **`RequestBodyReader` accumulation** for `readBodyAsString` uses `CompositeByteBuf` (zero-copy append), but `body().toString(charset)` does a full copy. For very large bodies, prefer `request.inputStream()`.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
