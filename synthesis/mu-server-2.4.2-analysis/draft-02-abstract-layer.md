---
title: "mu-server 2.4.2 抽象层 (Request/Response/HttpExchange)"
category: synthesis
tags: [java, netty, mu-server, abstraction, adapter-pattern, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "NettyRequestAdapter/ResponseAdaptor 包装 Netty 底层 HttpRequest/ChannelFuture, 提供用户友好的 MuRequest/MuResponse API, HttpExchange 协调两件套 + 状态机"
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

# Draft 02 — Abstract Layer (Mu API surface)

> Scope: the types that bridge Netty internals to user-visible code:
> `MuRequest`, `MuResponse`, `HttpExchange`, `NettyRequestAdapter`,
> `NettyResponseAdaptor`, `Headers` (with `Http1Headers`/`Http2Headers`
> implementations), `ForwardedHeader`, `RequestBodyReader` family,
> `SsePublisher` / `AsyncSsePublisher`, `AsyncHandle`.

## 1. The contract split

| Public-facing | Internal | Adapter |
|---|---|---|
| `MuRequest` | `NettyRequestAdapter` | (same class, `NettyRequestAdapter implements MuRequest`) |
| `MuResponse` | `NettyResponseAdaptor` (abstract), `Http1Response`, `Http2Response` | subclasses |
| `HttpConnection` | (impl'd by `Http1Connection` & `Http2Connection`) | — |
| `Headers` | `Http1Headers` (wraps `io.netty.handler.codec.http.HttpHeaders`), `Http2Headers` (wraps `io.netty.handler.codec.http2.Http2Headers`) | — |

The split allows the public API to be Netty-free. Internally every operation
delegates to the underlying Netty type.

## 2. HttpExchange — the central coordination object

`HttpExchange.java` (480 lines) owns:

- The raw `ChannelHandlerContext`
- The request/response pair
- An `int streamId` (HTTP/2 only; `-1` for HTTP/1)
- The `HttpConnection`
- A start time + end time
- A state machine via `HttpExchangeState` (enum: `IN_PROGRESS`,
  `COMPLETE`, `ERRORED`, `UPGRADED`)
- A `List<HttpExchangeStateChangeListener>` for state transitions
- A `ScheduledFuture<?> readTimer` for the per-request body-read idle timeout

### 2.1 The bridge methods — `block()`

```java
void block(Runnable runnable) { … }
void block(Callable<ChannelFuture> callable) { … }
```

Both wrap a user-mode operation onto the event loop, then `task.get()` (the
blocking version) or `task.get().sync()` (the future version). Asserts
`!inLoop()` to forbid blocking on the event loop thread. Failure modes:

| Cause | Action |
|---|---|
| `InterruptedException` | re-set interrupt flag, throw `UncheckedIOException(InterruptedIOException)` |
| `ExecutionException` | unwrap; throw `RuntimeException` cause or wrap in `MuException` |

The TODO comment on line 62 hints the maintainer considered replacing
`Runnable` overload with the `Callable<ChannelFuture>` one — the
`write(String)` path on `NettyResponseAdaptor` actually uses
`httpExchange.block(() -> writeOnLoop(text).addListener(...))` so the
listener-style completion is enough; the result future is discarded.

### 2.2 The exception-to-response pipeline (`onException`, lines 388-448)

This is the heart of mu-server's error UX:

1. If state is already ended → log and return true.
2. If response has **not** started → build a `Response` from either the
   `WebApplicationException` thrown or a fresh
   `InternalServerErrorException("Oops! … ErrorID=" + uuid)`.
3. Status 429/408/413 → mark `streamUnrecoverable = true` (close the
   connection on HTTP/1 by setting `Connection: close`).
4. Use `MuRuntimeDelegate.writeResponseHeaders` (jakarta.ws.rs bridge).
5. Send a tiny HTML body via `response.writeOnLoop("<h1>…</h1><p>…</p>")`.
6. If `streamUnrecoverable` → call `response.onCancelled(ERRORED)` and
   `request.onCancelled(ERRORED, cause)`.

The error ID (UUID) is the only thing the server logs that correlates to
the client-visible message. This is a clean "no stack-trace leak" pattern.

### 2.3 Body dispatch (`onMessage`, lines 186-230)

Called from both `Http1Connection` and `Http2Connection` for each body
chunk:

```java
cancelReadTimeout();
HttpContent content = (HttpContent) msg;
ByteBuf byteBuf = content.content().retain();
boolean last = msg instanceof LastHttpContent;

DoneCallback onDone = error -> {
    byteBuf.release();
    /* state transitions ... */
};
request.onRequestBodyRead(byteBuf, last, onDone);
```

The `retain()`/`release()` pair is critical — mu-server takes ownership of
the buffer from Netty and returns it when done. For HTTP/2 the same pattern
applies (see `Http2Connection.onDataRead0`).

### 2.4 Read timeout — `scheduleReadTimeout` / `cancelReadTimeout`

Each `HttpExchange` schedules a per-request body-read timer using
`ctx.executor().schedule(...)`. Every body fragment cancels and reschedules.
If the timer fires (`NettyRequestAdapter.onReadTimeout`), it propagates a
`TimeoutException` to the active `RequestBodyReader` which eventually
yields a `ClientErrorException(408)`.

## 3. NettyRequestAdapter (549 lines)

### 3.1 Body reading strategy

`NettyRequestAdapter.claimingBodyRead(reader)` (line 196) is the only legal
way to access the body. It:

1. Asserts `inLoop()` (only event loop can claim).
2. Sets `requestBodyReader = reader` (single-reader invariant).
3. Calls `setState(RequestState.RECEIVING_BODY)` which fires the listener
   that the dispatcher installed in `HttpExchange.create()` (and which
   calls `ctx.channel().read()` for HTTP/1 or `read(ctx, streamId)` for
   HTTP/2 — this is what kicks off the buffered DATA frames delivery).
4. Returns a Netty future.

`requestBodyReader` field is single-slot — calling `inputStream()` then
`form()` throws `IllegalStateException`.

### 3.2 Body reader types (`RequestBodyReader` family, 351 lines)

| Class | Use |
|---|---|
| `RequestBodyReader` (abstract) | base — manages `bytes` counter, `maxSize`, a `CompletableFuture<Throwable>` for completion |
| `StringRequestBodyReader` | `readBodyAsString()` — accumulates `CompositeByteBuf` |
| `UrlEncodedBodyReader` | `form()` for `application/x-www-form-urlencoded` |
| `MultipartFormReader` | `form()` for `multipart/*` — uses Netty's `HttpPostMultipartRequestDecoder` |
| `ListenerAdapter` | async `setReadListener()` — fires `RequestBodyListener.onDataReceived` per chunk |
| `DiscardingReader` | installed when the handler returns without reading — drains body silently |

`onRequestBodyRead` in the base class enforces `bytes > maxSize` →
`ClientErrorException(413)`. The base future is also what
`blockUntilFullyRead()` waits on; if it gets a `TimeoutException` it
converts to `ClientErrorException(408)` with `Connection: close`.

### 3.3 WebSocket upgrade

`NettyRequestAdapter.websocketUpgrade(...)` (line 391) is invoked from
`WebSocketHandler.handle` and:

1. Builds a `WebSocketServerHandshakerFactory` (max frame payload size
   configurable).
2. Replaces the `IdleStateHandler` in the pipeline (named `"idle"`) with a
   new one using the WebSocket idle/ping settings.
3. Creates a `MuWebSocketSessionImpl` and fires `ExchangeUpgradeEvent` so
   that `Http1Connection.userEventTriggered` swaps the active exchange to
   the WebSocket session.

### 3.4 Forwarded-header aware `clientIP()`

```java
public String clientIP() {
    List<ForwardedHeader> forwarded = headers.forwarded();
    for (ForwardedHeader f : forwarded) {
        if (f.forValue() != null) return f.forValue();
    }
    return this.connection().remoteAddress().getHostString();
}
```

`Headers.forwarded()` is computed lazily by `Headtils.getForwardedHeaders`
and combines `Forwarded:` (RFC 7239) with `X-Forwarded-For` /
`X-Forwarded-Proto` / `X-Forwarded-Host` fallback. The URI is also
re-computed against `Forwarded.proto` + `Forwarded.host` so redirects see
the original scheme.

## 4. NettyResponseAdaptor (367 lines) — the response state machine

State enum (`ResponseState`):

```
NOTHING → STREAMING → FINISHING → FINISHED
                       ↘ ERRORED ↗
        → FULL_SENT  (used by single-shot write())
        → UPGRADED   (WebSocket upgrade)
        → CLIENT_DISCONNECTED
        → TIMED_OUT
```

### 4.1 The contract for writing

| User call | Effect on state | Threading |
|---|---|---|
| `write(String)` | NOTHING → STREAMING → FULL_SENT | blocks via `httpExchange.block(...)` (off-loop) |
| `write(ByteBuffer)` | NOTHING → STREAMING | `ChannelFuture` returned; off-loop-safe |
| `sendChunk(String)` | NOTHING → STREAMING (then STREAMING stays) | blocks |
| `outputStream(int)` | NOTHING → STREAMING | lazy init via `httpExchange.block` |
| `redirect(URI)` | sets 302 + Location header | requires in-loop |
| `complete()` (called by dispatcher or `AsyncHandle`) | NOTHING → FINISHING → FINISHED, or STREAMING → FINISHING → FINISHED | must be in-loop |

### 4.2 Content-Length invariant

`writeAndFlush(ByteBuf)` (line 175) is the single byte path:

```java
bytesStreamed += size;
boolean isLast = bytesStreamed == declaredLength;
if (declaredLength > -1 && bytesStreamed > declaredLength) {
    onContentLengthMismatch();   // throws on HTTP/1, also on HTTP/2
    isLast = true;
}
ChannelFuture future = writeAndFlushToChannel(isLast, content);
if (isLast) {
    future.addListener(wf -> {
        outputState(wf.isSuccess() ? FULL_SENT : ERRORED);
    });
}
```

`onContentLengthMismatch` is implemented in both `Http1Response` and
`Http2Response` to throw `IllegalStateException`. This guarantees that an
honoured `Content-Length` from the handler actually matches what is
streamed. Note: only enforced if `Content-Length` was explicitly set;
chunked responses can stream indefinitely.

### 4.3 The `complete()` orchestration (line 288)

```java
void complete() {
    ResponseState finalState = ResponseState.FINISHED;
    ResponseState state = this.state;
    if (state.endState()) return;
    outputState(ResponseState.FINISHING);
    boolean isFixedLength = headers.contains(HeaderNames.CONTENT_LENGTH);
    ChannelFuture finishedFuture = null;
    if (state == ResponseState.NOTHING) {
        boolean addContentLengthHeader = !isHead && !isFixedLength
            && status != 204 && status != 205 && status != 304;
        finishedFuture = sendEmptyResponse(addContentLengthHeader);
    } else if (state == ResponseState.STREAMING) {
        boolean badFixedLength = !isHead && isFixedLength && declaredLength != bytesStreamed && status != 304;
        if (badFixedLength) {
            log.warn(...);
            finalState = ResponseState.ERRORED;
        }
        if (finalState == ResponseState.FINISHED) {
            finishedFuture = writeLastContentMarker();
        }
    }
    outputState(finishedFuture, finalState);
}
```

Note the `204/205/304` exclusions (responses that must not have a body) and
the `!isHead` guard (HEAD never has a body). `outputState(future, state)`
attaches a listener that switches to `ERRORED` on write failure.

## 5. Headers — the two implementations

Both `Http1Headers` (325 lines) and `Http2Headers` (347 lines) wrap a
Netty type. The mu layer adds:

- `accept()`, `acceptCharset()`, `acceptEncoding()`, `acceptLanguage()` —
  parsed via `ParameterizedHeaderWithValue.fromString`.
- `forwarded()` — RFC 7239 parser with `X-Forwarded-*` fallback.
- `cacheControl()` — `ParameterizedHeader.fromString`.
- `contentType()` — JAX-RS `MediaType` parsing.
- `toString()` — redaction for `authorization`, `cookie`, `set-cookie`.

Key differences between the two:

| Aspect | `Http1Headers` | `Http2Headers` |
|---|---|---|
| Backing | `io.netty.handler.codec.http.HttpHeaders` | `io.netty.handler.codec.http2.Http2Headers` |
| Header-name case | Netty preserves as-given | `toLower(name)` enforced (HTTP/2 mandates lowercase) |
| Pseudo-headers (`:method`, `:path` …) | n/a | filtered out of `iterator()`, `names()`, `entries()` |
| `hasBody()` | inspects `Transfer-Encoding` + `Content-Length` | uses `hasRequestBody` boolean (the HTTP/2 endStream flag on the headers frame) |

## 6. SSE (Server-Sent Events) — two flavours

### 6.1 `SsePublisher` (sync, 200 lines)

`SsePublisher.start(request, response)`:

1. Sets `Content-Type: text/event-stream`.
2. Sets `Cache-Control: no-cache, no-transform`.
3. Calls `request.handleAsync()` (puts the request in async mode).
4. Returns `SsePublisherImpl(asyncHandle)`.

Each `send(message, event, eventID)` builds the SSE text frame:

```
id: <id>\n
event: <event>\n
data: <message>\n       (one line per message line)
\n
```

`send` internally calls `asyncHandle.write(buf).get()` — **blocking** wait
on the write future. If the future fails (client disconnect), `close()` is
called and an `IOException` is thrown.

### 6.2 `AsyncSsePublisher` (193 lines)

Identical protocol but `send` returns a `CompletionStage<?>` instead of
blocking:

```java
private CompletionStage<?> write(String text) {
    CompletableFuture<?> stage = new CompletableFuture<>();
    if (closed) stage.completeExceptionally(...);
    else asyncHandle.write(Mutils.toByteBuffer(text), error -> {
        if (error == null) stage.complete(null);
        else stage.completeExceptionally(error);
    });
    return stage;
}
```

Also installs an auto-close via `addResponseCompleteHandler`. This is the
right choice when broadcasting to many subscribers because a slow client
doesn't block other publishers' send queues.

## 7. AsyncHandle — the bridge from sync handlers to async I/O

`AsyncHandle` is the handle returned by `MuRequest.handleAsync()`. It has
six methods (see `AsyncHandle.java`). The implementation
`NettyRequestAdapter.AsyncHandleImpl`:

- `setReadListener(RequestBodyListener)` → installs a
  `RequestBodyReader.ListenerAdapter` via `claimingBodyRead(...)`.
- `complete()` / `complete(Throwable)` → if not in-loop, schedules on the
  event loop; else calls `httpExchange.complete()`.
- `write(ByteBuffer)` → calls `NettyResponseAdaptor.writeAndFlush(data)`.
- `write(ByteBuffer, DoneCallback)` → wraps the future with the callback.
- `addResponseCompleteHandler(ResponseCompleteListener)` → subscribes to
  `HttpExchange` state changes.

## 8. Method, Cookie, ParameterizedHeader — the small types

`Method` (49 lines) is a plain enum (GET/POST/HEAD/OPTIONS/PUT/DELETE/TRACE/CONNECT/PATCH)
with `Method.fromNetty(HttpMethod)` that does `Method.valueOf(method.name())`
— so any method Netty knows about (custom methods) is supported.

`Cookie` + `CookieBuilder` (~150 lines combined) wraps Netty's
`io.netty.handler.codec.http.cookie.DefaultCookie`. Notable:
`Cookie.secureCookie(name, value)` factory for HTTPS-only cookies.

`ParameterizedHeader` parses cache-control / Accept-style headers into a
map of parameter→value pairs (preserving order for `LinkedHashMap`).

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
