---
title: "§2 协议层"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "协议层：HTTP/1.1 (Http1Connection 309 行) + HTTP/2 (Http2Connection 582 行内嵌流控) + ALPN 协议协商 (AlpnHandler 46 行) + HAProxy 协议 (20 行) + 背压流控 (BackPressureHandler 72 行 + MuFlowControlHandler 229 行 Netty fork)。"
provenance:
  extracted: 0.95
  inferred: 0.03
  ambiguous: 0.02
  base_confidence: 0.95
lifecycle: reviewed
lifecycle_changed: 2026-09-12
created: 2026-09-11
updated: 2026-09-12
---


# 协议层 (Protocol Layer)

### 2.1 HTTP/1.1 — `Http1Connection.java`

```java
class Http1Connection extends SimpleChannelInboundHandler<Object> implements HttpConnection {
    private final NettyHandlerAdapter nettyHandlerAdapter;
    private final MuStatsImpl serverStats;
    private final MuStatsImpl connectionStats = new MuStatsImpl(null); // 每连接独立 stats
    private final MuServerImpl server;
    private final String proto;
    private Exchange currentExchange = null; // HTTP/1.1 串行，一个 exchange 在飞
```

**要点**：
1. 直接继承 Netty 的 `SimpleChannelInboundHandler<Object>`，自己处理 `channelRead0`，**不依赖 Netty 默认的 `HttpObjectAggregator`** —— 走自己的 `HttpRequest` / `HttpContent` / `LastHttpContent` 拆装逻辑
2. 单 active exchange 字段：HTTP/1.1 串行语义（一个请求完成前不能开始下一个）
3. **显式 `ctx.channel().read()` 拉读**（不是 auto-read）—— 这是背压的关键，不调 read 就不会触发更多解码
4. `handlerAdded` 时记录 `remoteAddress`、注册 stats、触发首次 `read()`
5. `IdleStateEvent` 处理 idle timeout（HTTP/1.1 没在 keep-alive 窗口内发新请求 → 关闭 channel）
6. 反向代理协议（`HAProxyMessageHandler`）通过 channel attribute `HA_PROXY_INFO` 传真实客户端 IP

### 2.2 HTTP/2 — `Http2Connection.java`（582 行，含自实现流控）

mu-server 最值得注意的代码：**自实现 HTTP/2 流控**（直接做在 `Http2Connection` 内部，没有拆成独立文件），因为 Netty 默认流控写大 body 会卡住。流控核心片段大致如下（合并自 `Http2Connection` 内部多个 inner class）：

```java
// Http2Connection extends Http2ConnectionHandler implements Http2FrameListener
// 内部 per-stream buffer 字段（简化）：
private final Map<Integer, Queue<DataReadData>> buffer = new ConcurrentHashMap<>();
private final Map<Integer, Boolean> wantsToRead = new ConcurrentHashMap<>();

protected void read(ChannelHandlerContext ctx, int streamId) {
    wantsToRead.put(streamId, true);
    ctx.executor().submit(() -> sendItMaybe(ctx, streamId));
}

// sendItMaybe: 只在 wantsToRead=true 且 buffer 非空时才投递
// onDataRead0 后调: decoder().flowController().consumeBytes(stream, consumed)
```

**设计要点**：
1. **自管 buffer**：每个 stream 一个 `Queue<DataReadData>`，存 `data + padding + endOfStream`
2. **wantsToRead 模型**：consumer 必须显式 `read()` 表示想读，否则 `sendItMaybe` 不投递
3. **手动 consumeBytes**：在 `onDataRead0` 后调用 `decoder().flowController().consumeBytes(stream, consumed)`，主动告诉 Netty "我处理完了"
4. **解决 Netty 默认流控的痛点**：Netty 默认严格按窗口投递，写大 body 时容易 deadlock；mu-server 自己的 buffer 解耦了"帧到达"和"消费"
5. `Http2Connection` 内部增加 `Map<Integer, HttpExchange> exchanges`（每 stream 一个 exchange）
6. `onStreamError` 捕获 `HeaderListSizeException` → 431 Request Header Fields Too Large
7. `onGoAwayRead` 关整条 connection；`onRstStreamRead` 取消单个 stream

### 2.3 ALPN 协议协商 — `AlpnHandler.java`

```java
class AlpnHandler extends ApplicationProtocolNegotiationHandler {
    AlpnHandler(...) { super(ApplicationProtocolNames.HTTP_1_1); } // 默认 HTTP/1.1 fallback

    @Override
    protected void configurePipeline(ChannelHandlerContext ctx, String protocol) {
        if ("h2".equals(protocol)) {
            ctx.pipeline().addLast(new Http2ConnectionBuilder(server, nettyHandlerAdapter).build());
        } else if ("http/1.1".equals(protocol)) {
            ctx.pipeline().remove(BackPressureHandler.NAME);
            MuServerBuilder.setupHttp1Pipeline(ctx.pipeline(), nettyHandlerAdapter, server, proto);
        }
    }
}
```

**要点**：TLS 握手后根据 ALPN 协商结果**动态切换 pipeline**。HTTP/2 时清掉 HTTP/1 专属的 `BackPressureHandler`（HTTP/2 自带流控）。`exceptionCaught` 和 `handshakeFailure` 静默关闭 channel（不调 super，避免 Netty 默认 warn 日志）。

### 2.4 HAProxy 协议 — `HAProxyMessageHandler.java`

仅 20 行：

```java
class HAProxyMessageHandler extends SimpleChannelInboundHandler<HAProxyMessage> {
    static final AttributeKey<ProxiedConnectionInfo> HA_PROXY_INFO =
        AttributeKey.valueOf("HA_PROXY_INFO");

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HAProxyMessage msg) {
        ProxiedConnectionInfoImpl proxyConnectionInfo = ProxiedConnectionInfoImpl.fromNetty(msg);
        ctx.channel().attr(HA_PROXY_INFO).set(proxyConnectionInfo);
        if (!ctx.channel().config().isAutoRead()) ctx.read();
    }
}
```

**用途**：反向代理（HAProxy / nginx）后面部署时，TCP socket 的 `remoteAddress` 是代理的 IP，不是真实客户端 IP。`HAProxyMessageHandler` 解析 HAProxy 协议头，把真实 IP 存到 channel attribute，`Http1Connection.proxyInfo()` / `Http2Connection.proxyInfo()` 再读出来。

### 2.5 背压与流控 — `BackPressureHandler.java` + `MuFlowControlHandler.java`

**双层防护**：

`BackPressureHandler`（72 行）：
```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    if (!ctx.channel().isWritable()) {
        toSend.add(new Delivery(msg, promise)); // 不可写就排队
        return;
    }
    ...
}
```
当 channel TCP buffer 满了（不可写），消息进队列，等 `channelWritabilityChanged` 触发再排空。

`MuFlowControlHandler`（225 行，**从 Netty 4.1.136 / 4.2.15+ 复制过来的修改版**）。原因：Netty 新版的 `FlowControlHandler` 在 `channelReadComplete` 时可能 consume 一个 outstanding read 但不投递消息 —— mu-server 要求"每次 read 必须投递至少一个解码消息"，所以 fork 了这个 handler。

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[03-abstraction-layer|§3 抽象层]]
- [[04-dispatch-layer|§4 分发层]]
- [[05-jax-rs|§5 JAX-RS 支持]]
- [[06-features|§6 功能特性]]
- [[07-handler-library|§7 Handler 库]]
- [[08-design-patterns|§8 关键设计模式]]
- [[09-netty-comparison|§9 Netty 原生 vs mu-server 对照表]]
- [[10-evolution|§10 演化对比]]
- [[11-limitations|§11 限制 / 已知问题]]
- [[12-use-cases|§12 适用场景]]
- [[13-file-manifest|§13 关键文件清单]]

**跨版本对照**:
- [[mu-server-netty-analysis/summary|mu-server 0.0.3.6 (OMO 合成)]]
- [[mu-server-2.2.9-analysis/summary|mu-server 2.2.9 (OMO 合成)]]

**主入口**: [[summary]]
