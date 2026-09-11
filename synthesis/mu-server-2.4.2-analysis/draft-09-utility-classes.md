---
title: "mu-server 2.4.2 工具类"
category: synthesis
tags: [java, mu-server, utility, helpers, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "Mutils, CookieBuilder, ForwardedHeader, Headers, ContentTypes, ChunkedHttpOutputStream 等基础设施"
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

# Draft 09 — Utility classes & small types

> Scope: the rest of the `io.muserver` package — the helpers and small
> types that aren't in the protocol/dispatch/abstract layers.

## 1. `Mutils` (285 lines)

The static-method utility class. Notable methods:

- `urlEncode(s)` — UTF-8 URL-encoded; replaces `+` with `%20` and `%7E` with `~` (per RFC 3986 §2.3).
- `urlDecode(s)` — UTF-8 URL-decoded; throws `MuException` on error.
- `copy(from, to, bufferSize)` — manual byte-copy loop.
- `toByteArray(source, bufferSize)` — read all bytes, close stream.
- `nullOrEmpty(s)` / `hasValue(s)` — string empty checks.
- `join(one, sep, two)` — smart string join (handles both sides having the separator).
- `trim(value, toTrim)` — strip prefix/suffix occurrences.
- `notNull(name, value)` — throw `IllegalArgumentException` if null.
- `fullPath(file)` — `getCanonicalPath()` with fallback to `getAbsolutePath()`.
- `toHttpDate(date)` / `fromHttpDate(date)` — RFC 1123 / RFC 7231 date format.
- `htmlEncode(value)` — minimal `&<>"'/` escaping. The doc warns this is "very basic" and insufficient for arbitrary contexts.
- `coalesce(values...)` — first non-null.
- `closeSilently(closeable)` — swallow exceptions on close.
- `toByteBuffer(text)` — UTF-8 encode to a `ByteBuffer`.
- `pathAndQuery(uri)` — extract raw path + raw query.

## 2. `ForwardedHeader` (282 lines)

RFC 7239 parser + immutable holder.

### 2.1 Construction

```java
new ForwardedHeader(by, forValue, host, proto, extensions);
```

All four standard fields plus a `Map<String,String>` of extension params.

### 2.2 The parser (`fromString`, line 86)

A state-machine parser that handles:
- Quoted strings (with backslash escapes — though `\X` for any X is consumed verbatim by Netty-style semantics).
- `param=value` pairs separated by `;`.
- Multiple forwarded headers separated by `,`.
- Whitespace tolerance.
- Unknown params go into `extensions`.

The parser uses a single `StringBuilder buffer` and a single loop. The
state enum is `PARAM_NAME` / `PARAM_VALUE`. Quoted-string handling is
careful with backslash escapes.

### 2.3 `toString(List<ForwardedHeader>)` (line 245)

Round-trips a list into a single header value, separating entries with
`", "`. Each entry's fields are appended in `by;for;host;proto;<ext...>`
order.

## 3. `Cookie` + `CookieBuilder`

### 3.1 `Cookie` (107 lines)

Wraps Netty's `io.netty.handler.codec.http.cookie.DefaultCookie`. The
public API exposes:
- `name()`, `value()`, `domain()`, `path()`, `maxAge()`, `sameSite()`,
  `isSecure()`, `isHttpOnly()`.
- `equals`, `hashCode`, `toString` delegate to the Netty cookie.
- A static factory `Cookie.builder()` returns a **secure-by-default**
  builder (Secure flag, HttpOnly, SameSite=Strict).

`nettyToMu(Set<Cookie>)` is package-private — used by
`NettyRequestAdapter.cookies()`.

### 3.2 `CookieBuilder`

Full builder for `Cookie` with `withName`, `withValue`, `withDomain`,
`withPath`, `withMaxAge`, `withSecure`, `withHttpOnly`, `withSameSite`.
The `newSecureCookie()` static method defaults Secure/HttpOnly/SameSite
to safe values.

## 4. `ParameterizedHeader` + `ParameterizedHeaderWithValue`

For parsing `Accept`, `Accept-Charset`, `Accept-Encoding`,
`Accept-Language`, `Cache-Control` style headers — comma-separated
key[;param=value]* lists.

`ParameterizedHeader.fromString(value)` returns a `Map<String, ParameterizedHeaderWithValue>` where the key is the main value (e.g. `text/html`) and the value has the `q` and other params.

## 5. `Headtils` (small file)

The header utilities class. Provides:
- `getParameterizedHeaderWithValues(headers, name)` — parses any Accept*-style header.
- `getForwardedHeaders(headers)` — combines `Forwarded:` and `X-Forwarded-*` (auto-generates a pseudo Forwarded if only the latter is present).
- `getMediaType(headers)` — parses `Content-Type` into a JAX-RS `MediaType`.
- `toString(headers, toSuppress)` — formats the headers for logging, hiding `authorization`, `cookie`, `set-cookie` by default (or any names in `toSuppress`).

## 6. `MediaTypeParser` (small file)

A pure-Java RFC 6838 media-type parser. Doesn't depend on
`jakarta.ws.rs.core.MediaType` directly (uses string manipulation),
returns structured `MediaType`-like records.

## 7. `HeaderNames` (364 lines)

Centralised constants for well-known header names. Uses Netty's
`HttpHeaderNames` constants where available. Notable additions:
- `HOST`, `DATE`, `CONTENT_TYPE`, `CONTENT_LENGTH`, `CONNECTION`,
  `TRANSFER_ENCODING`, `UPGRADE`, `VARY`, `CACHE_CONTROL`,
  `SEC_WEBSOCKET_VERSION`, `IF_MODIFIED_SINCE`, `LAST_MODIFIED`,
  `ACCEPT_RANGES`, `CONTENT_RANGE`, `ACCEPT_ENCODING`, `COOKIE`,
  `SET_COOKIE`, `AUTHORIZATION`, `FORWARDED`, `X_FORWARDED_FOR`,
  `X_FORWARDED_HOST`, `X_FORWARDED_PROTO`.

## 8. `HeaderValues` (small)

Common header values: `GZIP`, `DEFLATE`, `CLOSE`, `BYTES`,
`WEBSOCKET`, `ZERO`, etc. — eliminates magic-string repetition.

## 9. `Method` (49 lines)

The HTTP method enum (GET, POST, HEAD, OPTIONS, PUT, DELETE, TRACE, CONNECT, PATCH). The package-private `fromNetty(HttpMethod)` does `Method.valueOf(method.name())` — supports custom methods.

## 10. `ContentTypes` (372 lines)

Centralised constants for `text/plain;charset=utf-8`,
`text/html;charset=utf-8`, `application/json;charset=utf-8`,
`text/event-stream`, etc. + a `mimeTypeForFilename(filename)` lookup
that delegates to `ResourceType`.

## 11. `RateLimit`, `RateLimitRejectionAction`, `RateLimiter`

Small immutable types:
- `RateLimit(bucket, allowed, action, per, perUnit)`.
- `RateLimitRejectionAction` enum: `SEND_429`, `READ_REQUEST_BUT_DONT_REPLY`.
- `RateLimiter` (interface) exposes `selector()` and `currentBuckets()` (a `Map<String, Long>` of bucket → current count).

## 12. `RejectedRequest`, `RejectedRequestImpl`

Wraps a request that was rejected at the protocol layer (e.g. 431
header size, 405 method). Exposes:
- `connection()` — the `HttpConnection` that received it.
- `method()`, `uri()` — captured before rejection (or null if unavailable).
- `status()` — the HTTP status.
- `message()` — the human-readable message.

## 13. `ProxiedConnectionInfo`, `ProxiedConnectionInfoImpl`

Holds the PROXY-protocol header info: source address, destination
address. Set by `HAProxyMessageHandler` in the channel attr
`HA_PROXY_INFO`, read by `HttpConnection.proxyInfo()`.

## 14. `RequestBodyListener`

```java
public interface RequestBodyListener {
    void onDataReceived(ByteBuffer buffer, DoneCallback onComplete);
    void onComplete();
    void onError(Throwable error);
}
```

Servlet-3-style async read listener. Used by
`AsyncHandle.setReadListener(...)` → `RequestBodyReader.ListenerAdapter`.

## 15. `ResponseInfo`, `ResponseCompleteListener`

```java
public interface ResponseInfo {
    MuRequest request();
    long duration();
    boolean completedSuccessfully();
}
```

The thing passed to `ResponseCompleteListener.onComplete(info)`. The
duration is `System.currentTimeMillis() - request.startTime()`. Used by
stats / logging.

## 16. `ServerSettings` (66 lines)

Immutable bundle of server-level settings (see Draft 03 §2). Provides
`shouldCompress(declaredLength, contentType)` (size + mime-type gate)
and `block(request)` (rate-limit gate).

## 17. `Exchange` (interface, ~50 lines)

The common interface that `HttpExchange` (HTTP) and `MuWebSocketSessionImpl`
(WebSocket) both implement. Defines the lifecycle methods
`onMessage/onException/onConnectionEnded/onIdleTimeout/onUpgradeComplete`.

## 18. `DoneCallback` (small interface)

```java
public interface DoneCallback {
    void onComplete(Throwable error);
}
```

Single-method callback used everywhere a future/listener is needed.

## 19. `UploadedFile` + `MuUploadedFile`

`UploadedFile` is the public interface (with `name()`, `filename()`,
`contentType()`, `size()`, `description()`, `asBytes()`,
`asStream()`). `MuUploadedFile` wraps a Netty `FileUpload`.

## 20. `ClientDisconnectedException` (small)

Thrown to signal that the request was cancelled because the client
disconnected. Caught in `RequestBodyReader` cleanup paths.

## 21. `MuException` (small)

The general-purpose exception for mu-server internal errors. Caught by
the dispatcher.

## 22. `WebsocketSessionState` (enum, ~50 lines)

`NOT_STARTED`, `OPEN`, `SERVER_CLOSING`, `SERVER_CLOSED`,
`CLIENT_CLOSING`, `CLIENT_CLOSED`, `TIMED_OUT`, `ERRORED`. Has
`endState()` (returns true for closed/errored) and `closing()`
(returns true for the two CLOSING states).

## 23. `RequestState` / `ResponseState`

The state-machine enums for the request and response halves of an
exchange. Already covered in Draft 02 §2.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
