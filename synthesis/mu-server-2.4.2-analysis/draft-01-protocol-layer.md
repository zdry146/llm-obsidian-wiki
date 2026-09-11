---
title: "mu-server 2.4.2 协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)"
category: synthesis
tags: [java, netty, mu-server, protocol, http2, alpn, haproxy, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "mu-server 2.4.2 在 Netty 之上的协议层封装: HTTP/1.1 Http1Connection 继承 SimpleChannelInboundHandler, HTTP/2 Http2Connection 继承 Http2ConnectionHandler, ALPN 协议协商, HAProxy 协议解析, 自实现 BackPressureHandler"
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

# Draft 01 — Protocol Layer (Netty Adapters)

> Scope: how mu-server wraps Netty's HTTP/1 and HTTP/2 codecs into a uniform
> `HttpConnection` interface; the ALPN-based protocol negotiation; HA Proxy
> protocol support; SNI-based multi-cert; custom flow-control.

## 1. Where the protocol boundary lives

mu-server deliberately exposes **only** a small set of public types —
`MuRequest`, `MuResponse`, `HttpConnection`, `MuServer`. Everything else that
references Netty is package-private. This means the Netty pipeline is hidden
from application code.

The two protocol-specific concrete handlers are:

| Concrete class | File | LOC | Public surface |
|---|---|---|---|
| `Http1Connection` | `src/main/java/io/muserver/Http1Connection.java` | 309 | package-private |
| `Http2Connection` | `src/main/java/io/muserver/Http2Connection.java` | 582 | package-private |
| `Http2ConnectionFlowControl` (parent of `Http2Connection`) | same file, lines 25-124 | — | package-private |
| `Http2To1RequestAdapter` | `src/main/java/io/muserver/Http2To1RequestAdapter.java` | — | package-private |
| `Http1Response` / `Http2Response` | `Http1Response.java` / `Http2Response.java` | 103 / 99 | package-private |
| `AlpnHandler` | `AlpnHandler.java` | 46 | package-private |
| `MuSniHandler` | `MuSniHandler.java` | 37 | package-private |
| `HAProxyMessageHandler` | `HAProxyMessageHandler.java` | 20 | package-private |
| `MuFlowControlHandler` | `MuFlowControlHandler.java` | 225 | package-private |
| `BackPressureHandler` | `BackPressureHandler.java` | 72 | package-private |

The common contract `HttpConnection` is at `src/main/java/io/muserver/HttpConnection.java`.

## 2. Pipeline assembly (HTTP/1 + HTTP/2)

The pipeline is built once in `MuServerBuilder.createChannel()` and
`setupHttp1Pipeline()`. The order matters — every handler is added in a
specific slot to maintain Netty's `ChannelInboundHandler` /
`ChannelOutboundHandler` contract.

```
HTTP listener (no SSL, no H2)                   HTTPS listener with H2 enabled
─────────────────────────────────────           ─────────────────────────────────────
idle (IdleStateHandler)                          idle
traffic-shaping (GlobalTrafficShapingHandler)    traffic-shaping
[optional HAProxyMessageDecoder+Handler]         [optional HAProxyMessageDecoder+Handler]
                                                 sni (MuSniHandler → DomainWildcardMappingBuilder)
                                                 pressure (BackPressureHandler)
                                                 alpn (AlpnHandler → configurePipeline on negotiation)
conerror (catch-all)                             conerror
decoder (HttpRequestDecoder, max-url + 17)       | then either…
encoder (HttpResponseEncoder, EmptyHttpResponse) |
[compressor (SelectiveHttpContentCompressor)]    |
keepalive (HttpServerKeepAliveHandler)           |
flowControl (MuFlowControlHandler)               |
pressure (BackPressureHandler)                   |
preread (PreReader)                              |
muhandler (Http1Connection)                      …
                                                 If h2: configurePipeline adds Http2ConnectionBuilder.build()
                                                 Otherwise: setupHttp1Pipeline(...) inserts the same stack as left.
```

**Reference:** `MuServerBuilder.java:749-807`

Three notable customisations vs raw Netty:

1. **HTTP/1 response encoder override** (`MuServerBuilder.java:793-798`): the
   default `HttpResponseEncoder.isContentAlwaysEmpty` returns false, but mu
   uses a custom subclass `NettyResponseAdaptor.EmptyHttpResponse` whose
   `isContentAlwaysEmpty` must short-circuit HEAD / empty-body responses to
   avoid sending `Content-Length: 0` with extra framing. The override checks
   `msg instanceof NettyResponseAdaptor.EmptyHttpResponse`.

2. **Custom flow control** (`MuFlowControlHandler.java`): the file-level
   comment (lines 29-36) explains why a local copy exists rather than using
   Netty's `FlowControlHandler` directly:
   > "Netty 4.1.136 and 4.2.15+ changed that handler so that an upstream
   > read-complete event can consume an outstanding read without delivering a
   > message. Mu Server requires the outstanding read to stay active until a
   > decoded HTTP message is available."

   Behaviour summary:
   - Inbound: queue each decoded message; fire `channelRead` downstream only
     while `isAutoRead()` is true or `minConsume > 0`.
   - Outbound: when `read()` is called downstream and the queue is empty,
     remember `shouldConsume = true`, then forward the read upstream. Once a
     message arrives, drain up to `minConsume` of it.
   - On `channelReadComplete`, only fire completion if the queue was drained
     (otherwise the next dequeue round will produce one).
   - On `channelInactive`/`handlerRemoved`, destroy the queue and release
     messages.

3. **Back-pressure** (`BackPressureHandler.java`): intercepts outbound
   `write()` and queues if `!ctx.channel().isWritable()`. On
   `channelWritabilityChanged` (or removal/inactive), drains the queue. The
   comment on `channelInactive` is interesting:
   > "even though we know these will fail, by delivering them it gives
   > relevant handlers the opportunity to release bytebufs."

## 3. HTTP/1 connection lifecycle

`Http1Connection.java` extends `SimpleChannelInboundHandler<Object>` and
implements `HttpConnection`. Important fields:

```java
private final MuStatsImpl serverStats;          // global
private final MuStatsImpl connectionStats = new MuStatsImpl(null); // per-conn
private final MuServerImpl server;
private final String proto;                     // "http" or "https"
private final Instant startTime = Instant.now();
private ChannelHandlerContext nettyCtx;
private InetSocketAddress remoteAddress;
private Exchange currentExchange = null;        // exactly one in-flight
```

### handlerAdded (line 53)

```java
serverStats.onConnectionOpened();
connectionStats.onConnectionOpened();
super.handlerAdded(ctx);
server.onConnectionStarted(this);   // registers connection in ConcurrentHashMap.newKeySet()
ctx.channel().read();                // explicit read — auto-read is off
```

### onChannelRead (line 83) — the heart

```java
if (msg instanceof HttpRequest) {
    this.currentExchange = HttpExchange.create(server, proto, ctx, this, ...);
} else if (currentExchange != null) {
    currentExchange.onMessage(ctx, msg, callback);   // body chunks
} else {
    log.debug("Got a chunk ... for an unknown request");   // body of rejected req
    ctx.channel().read();
}
```

Notable: HTTP/1.1 pipelining is **NOT** supported — `currentExchange` is a
single field, so a second request on the same connection cannot start until
the first finishes. (See "Limitations" below.)

### userEventTriggered (line 163) — the dispatch table

Three events:

| Event | Handling |
|---|---|
| `IdleStateEvent` ALL_IDLE | If exchange exists → `exchange.onIdleTimeout`; otherwise `ctx.close()` and log |
| `ExchangeUpgradeEvent` | For WebSocket: install `MuWebSocketSessionImpl` as new exchange; on UPGRADED state, call `onUpgradeComplete` and `ctx.read()`; on ERRORED, end new exchange's connection |
| `MuExceptionFiredEvent` | `exceptionCaught` delegation |

### Rejected-request path (line 113-127)

```java
} catch (InvalidHttpRequestException ihr) {
    if (ihr.code == 429 || ihr.code == 503) {
        connectionStats.onRejectedDueToOverload();
        serverStats.onRejectedDueToOverload();
    } else {
        connectionStats.onInvalidRequest();
        serverStats.onInvalidRequest();
    }
    nettyHandlerAdapter.onRequestRejected(new RejectedRequestImpl(ihr.code, ...));
    sendSimpleResponse(ctx, ihr.getMessage(), ihr.code);
    ctx.channel().read();
}
```

`sendSimpleResponse` (`Http1Connection.java:146-153`) writes a fixed
`DefaultFullHttpResponse(HTTP_1_1, status, bytes)` with `Date`,
`Content-Type`, `Content-Length`.

## 4. HTTP/2 connection lifecycle

`Http2Connection.java` is far more complex because of multiplexing. Key
innovations:

### 4.1 Custom per-stream DATA flow control (lines 25-124)

`Http2ConnectionFlowControl` extends `Http2ConnectionHandler` AND implements
`Http2FrameListener`. It overrides `onDataRead` to **buffer** DATA frames per
stream in a `Map<Integer, Queue<DataReadData>>`, and only delivers them when
the mu-server application code calls `read(ctx, streamId)`.

```java
@Override
public int onDataRead(ChannelHandlerContext ctx, int streamId,
                     ByteBuf data, int padding, boolean endOfStream) {
    Queue<DataReadData> buf = buffer.computeIfAbsent(streamId, k -> new LinkedList<>());
    buf.add(new DataReadData(data.retain(), padding, endOfStream));
    sendItMaybe(ctx, streamId);
    return 0;   // don't decrement flow-control window yet
}
```

This is *extra* flow control on top of Netty's own HTTP/2 WINDOW_UPDATE
protocol — mu-server needs to pause reading the body until the handler calls
`request.inputStream()` or `request.form()` (which is what calls
`NettyRequestAdapter.claimingBodyRead` → `setState(RECEIVING_BODY)` → the
`RequestState.RECEIVING_BODY` listener calls `read(ctx, streamId)`).

### 4.2 Stream → Exchange map (line 131)

```java
private final ConcurrentHashMap<Integer, HttpExchange> exchanges = new ConcurrentHashMap<>();
```

`onHeadersRead` (`Http2Connection.java:226-319`) creates one `HttpExchange`
per stream and stores it in the map. On end-state, `cleanStream(streamId)`
removes it (and releases any buffered DATA for that stream).

### 4.3 HTTP/2 → HTTP/1 adapter (line 240)

```java
HttpRequest nettyReq = new Http2To1RequestAdapter(streamId, nettyMeth, uri, headers);
```

This adapter makes the rest of mu-server HTTP-version-agnostic: the
`NettyRequestAdapter` works on `HttpRequest` regardless of wire version.

### 4.4 onDataRead0 → onMessage (line 322)

After releasing the body fragment to the application code, the doneCallback
calls `decoder().flowController().consumeBytes(stream, consumed)` to
acknowledge the bytes back to the remote peer. This is the proper HTTP/2
flow-control dance: receive → consume → WINDOW_UPDATE.

### 4.5 onStreamError (line 371)

Handles two cases:
- Known exchange → call `exchange.onException` then `super.onStreamError` if
  unrecoverable.
- Unknown stream + `HeaderListSizeException` during decode → fire
  `nettyHandlerAdapter.onRequestRejected(431)` (the headers could not be
  decoded, so method/target are `null`).

### 4.6 exceptionCaught (line 164)

Any exception during HTTP/2 processing triggers
`closeAllAndDisconnect(ctx, Http2Error.INTERNAL_ERROR, ResponseState.ERRORED)`
which writes a `GOAWAY` frame and closes the channel. **All** in-flight
exchanges are cancelled (`cleanup()` → `cancelExchange` per streamId).

### 4.7 Compression signalling (line 405-416 + Http2Response.java:53-71)

`Http2Response.writeHeaders` sets `Content-Encoding: mu-gzip` (or
`mu-deflate`) when the response should be compressed. The "mu-" prefix is
the marker; the mu-specific encoder
`MuCompressorHttp2ConnectionEncoder`/`MuGzipHttp2ConnectionEncoder` strips
the prefix and applies real compression. This pattern keeps the mu logic
out of the Netty encoder chain — they intercept only "mu-" prefixed values.

## 5. ALPN handshake (AlpnHandler.java)

```java
class AlpnHandler extends ApplicationProtocolNegotiationHandler {
    AlpnHandler(...) { super(ApplicationProtocolNames.HTTP_1_1); }

    protected void configurePipeline(ChannelHandlerContext ctx, String protocol) {
        if (HTTP_2.equals(protocol)) {
            ctx.pipeline().addLast(new Http2ConnectionBuilder(server, nettyHandlerAdapter).build());
            return;
        }
        if (HTTP_1_1.equals(protocol)) {
            ctx.pipeline().remove(BackPressureHandler.NAME); // because the http1 pipeline adds it in the right place
            MuServerBuilder.setupHttp1Pipeline(ctx.pipeline(), nettyHandlerAdapter, server, proto);
            return;
        }
        throw new IllegalStateException("unknown protocol: " + protocol);
    }
}
```

The default in the constructor (`HTTP_1_1`) means "if the client didn't
negotiate h2, fall back to h1". The HTTP/1 branch has to **remove** the
`BackPressureHandler` that was added *before* ALPN (at
`MuServerBuilder.java:771`) because `setupHttp1Pipeline` adds a new one in
the correct position.

## 6. HAProxy protocol (HAProxyMessageHandler.java)

20-line handler that consumes `HAProxyMessage` from Netty's
`HAProxyMessageDecoder`, builds a `ProxiedConnectionInfoImpl`, stores it in
the channel attr `HA_PROXY_INFO`, then triggers a `ctx.read()` if auto-read
is off.

`Http1Connection.proxyInfo()` and `Http2Connection.proxyInfo()` simply read
this channel attr.

## 7. SNI / multi-cert (MuSniHandler.java)

Extends Netty's `SniHandler`. Two overrides:

1. `replaceHandler` — store the SNI hostname in the channel attr
   `SNI_HOSTNAME` *before* swapping the SSL handler (so it's available
   immediately after the handshake).
2. `newSslHandler` — set `SSLParameters.setUseCipherSuitesOrder(true)` so
   the **server** picks the cipher (not the client). This is a best-practice
   for PFS / RSA vs ECDHE preference control.

The `MuServerBuilder.java:767` install is:
```java
p.addLast("sni", new MuSniHandler(
    () -> new DomainWildcardMappingBuilder<>(sslContextProvider.get()).build()));
```
The `Supplier` indirection lets the SNI mapping be **live-reloaded** when
`MuServer.changeHttpsConfig(...)` is called (see `SslContextProvider`).

## 8. Per-handler vs Netty responsibilities

| Concern | Where | Why |
|---|---|---|
| Byte-level I/O | Netty `NioEventLoop` | Standard |
| HTTP parsing | `HttpRequestDecoder` / `Http2ConnectionHandler` | Standard |
| Per-message buffering | `MuFlowControlHandler` | Custom — Netty 4.1.136/4.2.15+ behaviour unsuitable |
| Outbound back-pressure | `BackPressureHandler` | Custom — gives explicit control over queue draining |
| Per-stream DATA flow control (h2) | `Http2ConnectionFlowControl` | Custom — bridges Netty's window-update protocol to app-level read requests |
| Connection stats | `MuStatsImpl` (per-conn + global) | Custom — application-visible counters |
| Manual `ctx.read()` pacing | `Http1Connection.channelRead0` | Custom — guarantees one exchange at a time |
| Upgrade (WebSocket) | `ExchangeUpgradeEvent` in userEventTriggered | Custom |
| Exception → 500 | `HttpExchange.onException` | Custom — UUID error ID |

## 相关笔记

- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
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
