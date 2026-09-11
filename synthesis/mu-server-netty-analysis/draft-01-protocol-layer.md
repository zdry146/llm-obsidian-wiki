---
title: "mu-server 协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)"
category: synthesis
tags: [java, netty, mu-server, protocol, http2, alpn, haproxy]
sources: ["mu-server mu-server-0.0.3.6 @ commit 4f0aa3c (https://github.com/3redronin/mu-server)"]
summary: "mu-server 在 Netty 之上的协议层封装: HTTP/1.1 Http1Connection 继承 SimpleChannelInboundHandler, HTTP/2 Http2Connection 继承 Http2ConnectionHandler, ALPN 协议协商, HAProxy 协议解析, 自实现 BackPressureHandler"
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

# Draft 01 — Protocol Layer

> Source root: `src/main/java/io/muserver/`
> Netty primitives used: `HttpRequestDecoder`, `HttpResponseEncoder`, `HttpServerKeepAliveHandler`,
> `IdleStateHandler`, `Http2ConnectionHandler`, `Http2FrameListener`, `HAProxyMessageDecoder`,
> `ApplicationProtocolNegotiationHandler`, `SslHandler`, `HttpServerCodec` siblings.

## 1. HTTP/1.1 — `Http1Connection.java` (322 lines)

* Extends `SimpleChannelInboundHandler<Object>` and implements `HttpConnection` (Http1Connection.java:31).
* Single mutable `currentExchange` field — only one in-flight exchange per HTTP/1 connection (Http1Connection.java:42).
* Reads on the event loop, then dispatches to a separate worker pool via `NettyHandlerAdapter.onHeaders(exchange)`.
* `handlerAdded` (line 57) records `remoteAddress`, fires `serverStats.onConnectionOpened()` and
  calls `ctx.channel().read()` to start the auto-read loop.
* `channelRead0` (line 78) routes:
  * `HttpRequest` → builds an `HttpExchange` (`HttpExchange.create(...)`), wires two change listeners
    (line 92-118):
    * `RequestStateChangeListener` that re-arms `ctx.channel().read()` when state moves to `RECEIVING_BODY`.
    * `HttpExchangeStateChangeListener` that calls `onResponseComplete(...)` when the exchange ends, then
      resets `currentExchange = null` and decides whether to close the channel or keep reading.
  * Any other `Object` while `currentExchange != null` → `exchange.onMessage(...)`.
* `userEventTriggered` (line 172) handles three custom events:
  * `IdleStateEvent` → closes channel (line 178-181) with note that a 408 cannot be sent.
  * `ExchangeUpgradeEvent` → WebSocket upgrade path.
  * `MuExceptionFiredEvent` → funneled through `exceptionCaught(...)`.
* Reads are explicitly pulled (`ctx.channel().read()`) rather than auto-read — see lines 64, 94, 134,
  143, 151, 192, 201. This is critical for back-pressure.
* HTTP/1 keep-alive is handled by Netty's `HttpServerKeepAliveHandler` added in the pipeline builder
  (MuServerBuilder.java:831).

## 2. HTTP/2 — `Http2Connection.java` (584 lines)

* Two layers:
  * `abstract Http2ConnectionFlowControl extends Http2ConnectionHandler implements Http2FrameListener` (line 28).
    Holds per-stream `buffer: Map<Integer, Queue<DataReadData>>` and `wantsToRead: Map<Integer, Boolean>`
    (line 42-43). Custom `read(ctx, streamId)` API (line 49) submits a task to the event loop and calls
    `sendItMaybe` (line 58) which only forwards a frame if the consumer wants it AND the queue is non-empty.
    This is mu-server's HTTP/2 back-pressure implementation.
  * `final Http2Connection extends Http2ConnectionFlowControl implements HttpConnection` (line 129).
* `exchanges: ConcurrentHashMap<Integer, HttpExchange>` — one exchange per HTTP/2 stream (line 134).
* `onHeadersRead` (line 229) is the dispatch entry per HTTP/2 stream:
  1. Validates method/URL via `HttpExchange.getMethod(...)` and `getRelativeUrl(...)`.
  2. Throws `InvalidHttpRequestException(414/413)` if URL or content-length exceeds limits.
  3. Wraps the netty headers via `Http2To1RequestAdapter` + `Http2Headers`.
  4. Creates `HttpExchange` + `Http2Response` and stores in `exchanges`.
  5. Calls `settings.block(muReq)` → throws 429 if rate-limit rejected.
  6. Registers change listeners — when state ends, calls `cleanStream(streamId)` and possibly `resetStream(...)`.
  7. On `endOfStream`, jumps directly to `RequestState.COMPLETE`.
  8. Hands off to `nettyHandlerAdapter.onHeaders(httpExchange)` which submits to the worker pool.
* `onDataRead0` (line 322) wraps each `ByteBuf` in a `DefaultHttpContent` or `DefaultLastHttpContent`,
  then on completion **explicitly consumes flow-control bytes** via
  `decoder().flowController().consumeBytes(stream, consumed)` (line 341). This is the manual HTTP/2
  flow-control handshake.
* `onStreamError` (line 371) catches `Http2Exception.HeaderListSizeException` and emits a 431 (line 389-399).
* `onGoAwayRead` (line 473) closes the entire connection; `onRstStreamRead` cancels the matching exchange.
* `cancelExchange(streamId)` (line 439) only calls `httpExchange.onCancelled(ResponseState.ERRORED)` —
  removal from the map happens via the listener's `cleanStream` callback.

## 3. HAProxy Protocol — `HAProxyMessageHandler.java` (20 lines)

* `extends SimpleChannelInboundHandler<HAProxyMessage>` (line 8).
* Stashes the parsed `ProxiedConnectionInfoImpl` in the channel attribute `HA_PROXY_INFO`
  (line 10, 14-15). Both `Http1Connection.proxyInfo()` (Http1Connection.java:296-298) and
  `Http2Connection.proxyInfo()` (Http2Connection.java:571-573) read it back.
* Plumbed via `MuServerBuilder.createChannel` (MuServerBuilder.java:791-794) — only added when
  `withHAProxyProtocolEnabled(true)` is set.

## 4. ALPN — `AlpnHandler.java` (46 lines)

* Extends Netty's `ApplicationProtocolNegotiationHandler` (line 7).
* After TLS handshake completes, `configurePipeline` (line 20):
  * `h2` → appends `Http2ConnectionBuilder.build()`.
  * `http/1.1` → first removes the inherited `BackPressureHandler.NAME` (because the HTTP/1 pipeline
    adds it in the right position later — line 27), then calls `MuServerBuilder.setupHttp1Pipeline(...)`.
* `exceptionCaught` (line 36) and `handshakeFailure` (line 41) close the channel silently — they
  intentionally skip Netty's noisy default warn log.

## 5. Back-pressure helper — `BackPressureHandler.java`

* Lives at the bottom of the HTTP/1 pipeline (MuServerBuilder.java:833) and inside the ALPN pipeline
  (MuServerBuilder.java:800).
* Threshold-based write-buffer high/low watermark gates outbound writes so a slow client cannot OOM
  the server.

## 6. Pipeline construction — `MuServerBuilder.createChannel` (MuServerBuilder.java:771)

Per-connection pipeline order (HTTP/2 + SSL):
```
idle                IdleStateHandler(0,0,idle)
traffic-shaping     GlobalTrafficShapingHandler
[HAProxy decoder]   HAProxyMessageDecoder      (optional)
[HAProxy handler]   HAProxyMessageHandler      (optional)
[sni]               MuSniHandler                (SSL only)
back-pressure       BackPressureHandler         (when ALPN)
alpn                AlpnHandler                 (HTTP/2+SSL)
conerror            ChannelInboundHandlerAdapter (stats on connect failure)
…                   then either setupHttp1Pipeline or HTTP/2 framing
```

`setupHttp1Pipeline` (MuServerBuilder.java:820-836):
```
decoder       HttpRequestDecoder(maxLine, maxHeaders, 8192)
encoder       HttpResponseEncoder (customised to treat NettyResponseAdaptor.EmptyHttpResponse as empty)
compressor    SelectiveHttpContentCompressor (when gzip enabled)
keepalive     HttpServerKeepAliveHandler
flowControl   MuFlowControlHandler
back-pressure BackPressureHandler
preread       PreReader
muhandler     Http1Connection
```

## 7. Snippets worth quoting in final report

* `HttpExchange.block(...)` (HttpExchange.java:63-98) — see draft 07.
* `Http2Connection.read(ctx, streamId)` (Http2Connection.java:49-56) — see draft 09.
* `Http2Connection.onDataRead0` flow-control consumption (Http2Connection.java:322-367) — see draft 09.

## 相关笔记

- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
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
