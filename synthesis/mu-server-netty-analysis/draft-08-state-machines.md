---
title: "mu-server 状态机 (RequestState/ResponseState/HttpExchangeState)"
category: synthesis
tags: [java, mu-server, state-machine, observer-pattern]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "三套状态机: RequestState (HEADERS_RECEIVED → RECEIVING_BODY → COMPLETE/ERRORED), ResponseState (NOTHING → HEADERS_SENT → ...), HttpExchangeState (IN_PROGRESS/COMPLETE/ERRORED/UPGRADED), CopyOnWriteArrayList 监听器"
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

# Draft 08 — State Machines

## 1. Request — `RequestState` (file: `RequestState.java`)

```
HEADERS_RECEIVED ─→ RECEIVING_BODY ─→ COMPLETE
                  ↘                 ↘
                                     ERRORED
```

* `HEADERS_RECEIVED` — initial state after `Http1Connection` / `Http2Connection` creates the
  request. `setState` is only called from inside the event loop (`assert exchange.inLoop()`
  NettyRequestAdapter.java:441).
* `RECEIVING_BODY` — set by `claimingBodyRead(reader)` (NettyRequestAdapter.java:205-221) when a
  handler asks for `inputStream()` / `readBodyAsString()` / `form()` / `uploadedFile()`.
  `Http1Connection` registers a `RequestStateChangeListener` that fires
  `ctx.channel().read()` on entry (Http1Connection.java:92-96), re-arming the body pump.
* `COMPLETE` — set by `HttpExchange.onMessage` when the last content chunk arrives
  (HttpExchange.java:205-206).
* `ERRORED` — set by `onCancelled(ResponseState.ERRORED, ex)` from `HttpExchange.onException`,
  `HttpExchange.onCancelled`, or `HttpExchange.onConnectionEnded`.

Transitions are guarded: `setState` throws if the current state is already an end state
(NettyRequestAdapter.java:443-445).

## 2. Response — `ResponseState` (file: `ResponseState.java`)

```
NOTHING ─→ STREAMING ─→ FULL_SENT ─→ FINISHING ─→ FINISHED
   │           │            │             │           │
   ▼           ▼            ▼             ▼           ▼
(streamed via sendChunk / write / outputStream / writer / chunked)

NOTHING ─→ FINISHING ─→ FINISHED (no body, just headers + content-length)
NOTHING ─→ UPGRADED    (WebSocket upgrade before body sent)

Any state can transition to ERRORED.
```

* `NOTHING` — initial. `outputStream()` / `sendChunk()` call `startStreaming()` to leave this
  state (NettyResponseAdaptor.java:129-139).
* `STREAMING` — at least one chunk written.
* `FULL_SENT` — content-length reached or the single `write(String)` finished
  (NettyResponseAdaptor.java:202-203).
* `FINISHING` — entered by `complete()` (NettyResponseAdaptor.java:318), which writes the final
  empty content marker if needed.
* `FINISHED` — terminal success.
* `UPGRADED` — WebSocket. `NettyResponseAdaptor.setWebsocket()` (line 99-101) sets this so the
  connection knows to swap modes.
* `ERRORED` — terminal failure. Set by `onCancelled(...)` (line 103-107) or via the listener chain
  after a failed write.
* `CLIENT_DISCONNECTED` / `TIMED_OUT` — also terminal (see `ResponseState.java`).

`endState()` is true for `FULL_SENT, FINISHING, FINISHED, UPGRADED, ERRORED, CLIENT_DISCONNECTED,
TIMED_OUT`.

## 3. Exchange — `HttpExchangeState` (HttpExchange.java:463-474)

```
IN_PROGRESS ─→ COMPLETE
            ↘ ERRORED
            ↘ UPGRADED
```

Computed in `HttpExchange.onReqOrRespStateChange` (line 114-125) whenever either `request` or
`response` emits a state change:

* If `request.endState()` and `response == UPGRADED` → `UPGRADED`.
* If both end states and (request was `ERRORED` OR response did not complete successfully) →
  `ERRORED`. Otherwise → `COMPLETE`.
* If response reaches end state and request is still in progress, the request is told to discard
  any unread body (`request.discardInputStreamIfNotConsumed()`).

## 4. Pipeline-level back-pressure states (Netty)

* `ctx.channel().read()` is the only way mu-server pulls bytes out of Netty for HTTP/1. The
  `BackPressureHandler` watches the write buffer watermark; if it exceeds `high`, it pauses new
  reads, resumes on drop below `low`. This pairs with mu-server's explicit
  `ctx.channel().read()` per state-change.

## 5. HTTP/2 stream states

* Managed by Netty's `Http2ConnectionHandler`. Mu-server just listens to frame events
  (`onHeadersRead`, `onDataRead`, `onRstStreamRead`, `onGoAwayRead`, `onWindowUpdateRead`).
* Stream lifecycle per stream:
  ```
  IDLE → (HEADERS) → OPEN → (END_STREAM) → HALF_CLOSED_LOCAL/HALF_CLOSED_REMOTE
                                            → (RST_STREAM) → CLOSED
  ```
* The mu-server `Http2ConnectionFlowControl.buffer[streamId]` queue is populated while in
  `OPEN` / `HALF_CLOSED_*`; `cleanStream` drains it on transition to `CLOSED`.

## 6. Listener notification topology

```
Request state change ───► Http1Connection / Http2Connection listener
                                  │
                                  ▼
                            ctx.channel().read()    (re-arm read)
                            OR                       (last chunk)
                            complete cleanup

Response state change ───► HttpExchange.onReqOrRespStateChange
                                  │
                                  ▼
                          HttpExchange state ───► Http1Connection / Http2Connection listener
                                                      │
                                                      ▼
                                              ctx.channel().read() / .close()
                                              muReq.cleanup()
                                              onResponseComplete() (stats)
                                              scheduleReadTimeout (re-arm)
```

Listeners are `CopyOnWriteArrayList`s in `NettyRequestAdapter` (line 52),
`NettyResponseAdaptor` (line 43), and `HttpExchange` (line 56), so mutations are rare and reads
are lock-free.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
