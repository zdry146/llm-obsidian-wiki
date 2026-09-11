---
title: "mu-server 2.2.9 协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)"
category: synthesis
tags: [java, netty, mu-server, protocol, http2, alpn, haproxy, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "mu-server 2.2.9 在 Netty 之上的协议层封装: HTTP/1.1 Http1Connection 继承 SimpleChannelInboundHandler, HTTP/2 Http2Connection 继承 Http2ConnectionHandler, ALPN 协议协商, HAProxy 协议解析, 自实现 BackPressureHandler"
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

# 01 - 协议层 (Protocol Layer)

> mu-server 2.2.9 在 Netty 4.1.135.Final 上构建协议层。
> 本层负责: TLS / HAProxy / ALPN 协商 / HTTP/1.1 / HTTP/2 帧编解码 / 流量整形 / 闲置超时。

## 文件清单

| 文件 | 角色 | 行数 |
|---|---|---|
| `src/main/java/io/muserver/Http1Connection.java` | HTTP/1.1 pipeline 末端处理器 | 305 |
| `src/main/java/io/muserver/Http2Connection.java` | HTTP/2 connection handler (含 flow control 子类) | 568 |
| `src/main/java/io/muserver/Http2ConnectionBuilder.java` | HTTP/2 帧监听器构造 | 34 |
| `src/main/java/io/muserver/AlpnHandler.java` | TLS ALPN 协商, 切换 H1/H2 pipeline | 46 |
| `src/main/java/io/muserver/HAProxyMessageHandler.java` | PROXY protocol v1/v2 解析 | 20 |
| `src/main/java/io/muserver/MuSniHandler.java` | TLS SNI 多证书选择 | 37 |
| `src/main/java/io/muserver/MuCompressorHttp2ConnectionEncoder.java` | HTTP/2 gzip 帧压缩(包装 Netty 默认 encoder) | - |
| `src/main/java/io/muserver/MuGzipHttp2ConnectionEncoder.java` | HTTP/2 gzip 选项 hack (hook writeHeaders) | - |
| `src/main/java/io/muserver/SelectiveHttpContentCompressor.java` | HTTP/1 gzip 选择性压缩 | - |
| `src/main/java/io/muserver/PreReader.java` | HTTP/1 启动时主动 read() 防止 hang | - |
| `src/main/java/io/muserver/BackPressureHandler.java` | Netty 写缓冲水位背压 | - |
| `src/main/java/io/muserver/Http2To1RequestAdapter.java` | Http2Headers → HttpRequest 适配 (REST 用) | - |

## Pipeline 装配 (来源: `MuServerBuilder.createChannel` 723-763)

**HTTP/1 明文连接**:
```
idle (IdleStateHandler) → trafficShaping → conerror → decoder → encoder →
[compressor] → keepalive (HttpServerKeepAliveHandler) → flowControl →
BackPressureHandler → preread → muhandler (Http1Connection)
```

**HTTPS(默认, 用 `unsignedLocalhost()` 证书)**:
```
idle → trafficShaping → sni (MuSniHandler) → alpn (AlpnHandler)
  → 由 AlpnHandler 在 configurePipeline() 中按 ALPN 结果切换:
       h2: BackPressure + Http2Connection
       h1: 移除 BackPressure + setupHttp1Pipeline
```

**启用 HAProxy** (`MuServerBuilder.withHAProxyProtocolEnabled(true)`):
```
idle → trafficShaping → HAProxyMessageDecoder → HAProxyMessageHandler → ...
```

`HAProxyMessageHandler` (20 行) 把 `ProxiedConnectionInfo` 塞进 channel attribute `HA_PROXY_INFO`,
连接结束时由 `Http1Connection.proxyInfo()` / `Http2Connection.proxyInfo()` 取回。
`request.clientIP()` (NettyRequestAdapter:323) 优先看 `Forwarded` 头, 再看 `X-Forwarded-*` (通过 `ForwardedHeader` 解析), 最后回退到 socket 地址。

## TLS / SNI / ALPN

`MuSniHandler` (37 行) 继承 Netty `SniHandler`, 通过 `DomainWildcardMappingBuilder` 把 host → `SslContext` 映射。
`replaceHandler()` 把 hostname 写入 channel attr `SNI_HOSTNAME` 让应用层可见。
`newSslHandler()` 强制 `setUseCipherSuitesOrder(true)` (服务端优先密码套件顺序)。

`AlpnHandler` (46 行) 默认构造参数 `ApplicationProtocolNames.HTTP_1_1`。
- ALPN 选 `h2` → 加 `Http2Connection` (通过 `Http2ConnectionBuilder.build()`)
- ALPN 选 `http/1.1` → 移除 BackPressure, 走 `MuServerBuilder.setupHttp1Pipeline`
- `handshakeFailure()` 直接 close() (避免 Netty 默认警告日志)

## HTTP/2 流控 (核心)

`Http2ConnectionFlowControl` (Http2Connection.java:25-124) 是 `Http2ConnectionHandler` 子类,
**自己实现了一个 per-stream 的 data buffer + wantsToRead 队列**, 而不直接依赖 Netty 默认 flow controller。

```java
// Http2Connection.java:75-80
@Override
public int onDataRead(ChannelHandlerContext ctx, int streamId, ByteBuf data,
                      int padding, boolean endOfStream) {
    Queue<DataReadData> buf = buffer.computeIfAbsent(streamId, integer -> new LinkedList<>());
    buf.add(new DataReadData(data.retain(), padding, endOfStream));
    sendItMaybe(ctx, streamId);
    return 0; // 不立即消耗流控窗口
}

// Http2Connection.java:46-53
protected void read(ChannelHandlerContext ctx, int streamId) {
    if (!ctx.executor().inEventLoop()) {
        ctx.executor().execute(() -> read(ctx, streamId));
        return;
    }
    wantsToRead.put(streamId, true);
    ctx.executor().submit(() -> sendItMaybe(ctx, streamId));
}
```

**核心设计**: `onDataRead` 先把 frame 放进 `buffer[streamId]`, 然后调用 `sendItMaybe`。
`sendItMaybe` 检查 `wantsToRead[streamId]` 是否为 true, 是的话从 buffer 取出一帧调 `onDataRead0` 并把
`wantsToRead` 置 false。**

应用层 (handler executor) 处理完后回调 (`Http2Connection.java:335-350`) 才真正消耗流控窗口:
```java
Http2Stream stream = this.connection().stream(streamId);
if (stream != null && this.decoder().flowController().consumeBytes(stream, consumed)) {
    ctx.flush();
}
```

> 这是一种 **应用层背压 (application-level backpressure)**: 除非业务线程显式调用 `read(ctx, streamId)`,
> 否则不会继续往下游喂数据, 这让慢消费者能自然限速而不会爆内存。

`wantsToRead` 由 `muReq.addChangeListener` 在 `RECEIVING_BODY` 时触发 (`Http2Connection.java:282-286`)。
`cleanStream(streamId)` 同时清 `wantsToRead` 和 `buffer`。

## HTTP/2 GoAway / 异常

`Http2Connection.exceptionCaught()` (164-167) 直接 `closeAllAndDisconnect(..., INTERNAL_ERROR, ERRORED)`:
```java
encoder().writeGoAway(ctx, lastStreamId, Http2Error.INTERNAL_ERROR.code(), EMPTY_BUFFER, ctx.voidPromise());
cleanup();
ctx.close();
```

`onGoAwayRead` (459-461) 同样关闭但不带 GOAWAY 帧。
`onRstStreamRead` (416-167) 只取消该 stream 的 exchange。

## HTTP/1 keep-alive + flow control

`MuServerBuilder.setupHttp1Pipeline` (765-781) 加入:
- `HttpRequestDecoder` (URL size = `maxUrlSize + 17`, max headers 8192 默认)
- `HttpResponseEncoder` (重写 `isContentAlwaysEmpty` 识别 `EmptyHttpResponse`)
- 可选 `SelectiveHttpContentCompressor`
- `HttpServerKeepAliveHandler` (Netty 提供的 keep-alive 决策)
- `FlowControlHandler` (Netty 提供, channel-level inbound 暂停恢复)
- `BackPressureHandler` (mu 自实现, 用 `BackPressureHandler.NAME` 标识)
- `PreReader` (mu 自实现, channelRegistered 时主动 read, 解决 channelInactive race)
- 末端 `Http1Connection`

## 闲置 / 读超时

`HttpExchange.scheduleReadTimeout()` (232-236) 在 `RECEIVING_BODY` 时启动一个定时器,
到期调 `request.onReadTimeout()` 抛 `TimeoutException` 给 `RequestBodyReader`。

`Http1Connection.userEventTriggered()` (159-197) 把 `IdleStateEvent` 转交给当前 exchange 的
`onIdleTimeout()`, 仅 `ALL_IDLE` 算超时 (读 + 写 + 全部空闲)。

`Http2Connection.userEventTriggered()` (473-490) 在非 `READER_IDLE` 时整体 GO_AWAY 关闭。

## HAProxy 协议

`HAProxyMessageHandler.channelRead0()` (13-19) 在解析完一条消息后:
```java
ctx.channel().attr(HA_PROXY_INFO).set(proxyConnectionInfo);
if (!ctx.channel().config().isAutoRead()) {
    ctx.read();
}
```
注意 `ProxiedConnectionInfoImpl` 存的是 **原始 (upstream) 客户端 IP** + 代理 ID, 后续 `request.clientIP()` 的优先级:
**Forwarded header → X-Forwarded-* → HAProxy attr → socket address**。

## 设计观察**
- 协议层是 mu-server 最"硬"的部分: 跟 Netty 4.x API 深度耦合 (尤其是 HTTP/2 的
  `Http2ConnectionHandler` 继承, `Http2ConnectionFlowControl` 自己实现 buffer)。
- 协议层代码相对低层 (粗略估算 ~1500 LOC), 但提供了 90% 业务所需的能力
  (H1/H2/ALPN/SNI/HAProxy/keepalive/gzip/backpressure/idle timeout)。
- 与 0.0.3 相比, 协议层 API **几乎未变**, 这部分 Netty 抽象相对稳定。

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
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
