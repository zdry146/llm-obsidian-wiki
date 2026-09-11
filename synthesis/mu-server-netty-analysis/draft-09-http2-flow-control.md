---
title: "mu-server HTTP/2 自定义流控"
category: synthesis
tags: [java, netty, mu-server, http2, flow-control, back-pressure]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "Http2ConnectionFlowControl 自实现 buffer (Map<Integer, Queue<DataReadData>>) + wantsToRead, 解决 Netty 默认流控写大 body 卡住的问题, 手动 consumeBytes"
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

# Draft 09 — HTTP/2 Flow Control

> Source: `Http2Connection.java` (584 lines), `MuFlowControlHandler.java` (file),
> `MuCompressorHttp2ConnectionEncoder.java`, `MuGzipHttp2ConnectionEncoder.java`.

## 1. The custom per-stream back-pressure layer

`Http2ConnectionFlowControl` (Http2Connection.java:28-127) sits **above** Netty's HTTP/2 codec:

```java
private final Map<Integer, Queue<DataReadData>> buffer = new HashMap<>();
private final Map<Integer, Boolean> wantsToRead = new HashMap<>();

protected void read(ChannelHandlerContext ctx, int streamId) {
    if (!ctx.executor().inEventLoop()) {
        ctx.executor().execute(() -> read(ctx, streamId));
        return;
    }
    wantsToRead.put(streamId, true);
    ctx.executor().submit(() -> sendItMaybe(ctx, streamId));
}

private void sendItMaybe(ChannelHandlerContext ctx, int streamId) {
    if (ctx.channel().isActive()) {
        Boolean wantsIt = wantsToRead.get(streamId);
        if (wantsIt != null && wantsIt) {
            Queue<DataReadData> queue = buffer.get(streamId);
            if (queue != null) {
                DataReadData msg = queue.poll();
                if (msg != null) {
                    wantsToRead.put(streamId, false);
                    onDataRead0(ctx, streamId, msg.data, msg.padding, msg.endOfStream);
                    msg.data.release();
                }
            }
        }
    }
}

@Override
public int onDataRead(ChannelHandlerContext ctx, int streamId, ByteBuf data,
                      int padding, boolean endOfStream) {
    Queue<DataReadData> buf = buffer.computeIfAbsent(streamId, integer -> new LinkedList<>());
    buf.add(new DataReadData(data.retain(), padding, endOfStream));
    sendItMaybe(ctx, streamId);
    return 0;            // <-- mu-server does NOT consume flow-control bytes here
}
```

### What this buys you

* `onDataRead` is called by Netty's HTTP/2 frame listener. Mu-server **never consumes flow-control
  bytes in this path** (it returns `0`).
* Instead, the buffered frame is held in `buffer[streamId]` until `read(ctx, streamId)` is called.
* `read` flips `wantsToRead[streamId] = true` and submits `sendItMaybe` to the event loop.
* `sendItMaybe` only forwards the buffered frame if (a) the consumer wants it AND (b) the queue is
  non-empty.

### Why this is necessary

Naive Netty HTTP/2 will pump DATA frames as fast as the decoder can deliver them. If the consumer
(handler) is slow (because it's blocked in `exchange().block(...)`), the inbound buffer fills. With
flow-control properly consumed, the peer stops sending — but you need to consume bytes back to the
flow controller after you've **processed** the frame.

The pattern mu-server uses:

```
Netty HTTP/2 decoder.onDataRead
   → Http2ConnectionFlowControl.onDataRead          // buffer + maybe dispatch
     → Http2Connection.onDataRead0
       → httpExchange.onMessage(...)
         → requestBodyReader.onRequestBodyRead(...)
         → DoneCallback
            → decoder().flowController().consumeBytes(stream, consumed)  // line 341
            → ctx.flush()
```

The `consumeBytes(stream, consumed)` call returns the bytes to the flow controller — at which
point Netty sends a `WINDOW_UPDATE` frame upstream and the peer resumes sending.

## 2. The decision to consume bytes in the DoneCallback

`Http2Connection.onDataRead0` (Http2Connection.java:322-368):

```java
DoneCallback doneCallback = error -> {
    Http2Stream stream = this.connection().stream(streamId);
    if (stream != null && this.decoder().flowController().consumeBytes(stream, consumed)) {
        ctx.flush();
    }
    data.release();
    if (error != null) {
        ctx.fireUserEventTriggered(new MuExceptionFiredEvent(httpExchange, streamId, error));
    } else if (!endOfStream) {
        read(ctx, streamId);
    }
};
httpExchange.onMessage(ctx, msg, error -> {
    if (ctx.executor().inEventLoop()) {
        doneCallback.onComplete(error);
    } else {
        ctx.executor().execute(() -> { try { doneCallback.onComplete(error); } catch (Exception e) { ... } });
    }
});
```

* `consumed = dataSize + padding` (line 323-324).
* `ctx.flush()` only happens after the consume succeeds (line 342).
* If the body reader is still hungry (`!endOfStream`), `read(ctx, streamId)` re-arms the consumer
  for the next frame (line 349).
* If the executor is not the event loop, the consume is hopped back to the loop (line 358-364) so
  flow-controller mutation is always single-threaded.

## 3. Buffer hygiene

* `cleanStream(streamId)` (line 99-114 + Http2Connection.java:203-206) — called from the exchange
  end-state listener. Removes `wantsToRead[streamId]`, then drains & releases the buffered bytes
  for that stream.
* `cleanup()` (line 85-97 + 209-216) — `handlerRemoved0` / `channelInactive` path; clears all
  buffered data + cancels every live exchange.
* Failure modes covered: stream reset (`onRstStreamRead` line 430-432 → `cancelExchange`),
  client go-away (`onGoAwayRead` line 473-475 → `closeAllAndDisconnect`),
  HTTP/2 stream error (`onStreamError` line 371-403 → 431 + listener notification).

## 4. `MuFlowControlHandler` for HTTP/1

(MuServerBuilder.java:832)

* Placed between `keepalive` and `BackPressureHandler`. Likely a wrapper around Netty's
  `FlowControlHandler` to ensure `ctx.channel().read()` is called the right number of times for
  request body frames.

## 5. Netty HTTP/2 vs mu-server HTTP/2 — the difference

| Capability | Netty default | mu-server override |
|---|---|---|
| Decoder consumes flow-control bytes immediately | Yes | No — bytes returned only after the body reader accepts them |
| Back-pressure semantics | Per-connection window | Per-stream, gated by handler readiness |
| Stream cancellation cleanup | Netty internal | Explicit `cancelExchange` + listener chain |
| HTTP/2 push | Yes | `onPushPromiseRead` is a no-op (line 467-470) — push is **not** supported |

## 6. Why this matters for users

* Slow handler → server tells client to stop sending via `WINDOW_UPDATE=0`. Client stalls.
* Body reader failure → consume still happens (`consumeBytes` is called regardless of `error`) so
  the connection doesn't deadlock on a half-consumed window.
* Stream reset from peer → `cancelExchange` calls `HttpExchange.onCancelled(ResponseState.ERRORED)`
  which propagates through `NettyRequestAdapter.onCancelled` (line 394-401) and
  `NettyResponseAdaptor.onCancelled` (line 103-107), eventually closing the exchange.

## 7. Limitations in this mu-server-0.0.3.6

* No `WindowUpdateRequest` handling for individual streams is exposed to user code (line 478-479 is
  empty).
* No HTTP/2 push — see `onPushPromiseRead` no-op.
* The buffer is a plain `HashMap<Integer, ...>`, which is fine because all access is on the event
  loop, but if anything ever reads it off-loop there will be data corruption.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
