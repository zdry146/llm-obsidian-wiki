---
title: "mu-server 2.2.9 HTTP/2 自定义流控"
category: synthesis
tags: [java, netty, mu-server, http2, flow-control, back-pressure, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
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

# 09 - HTTP/2 流控 (Flow Control)

> mu-server 在 HTTP/2 上做了一个**应用层背压** (application-level backpressure) 的实现,
> 跟 Netty 默认的 `Http2FlowController` 配合使用。

## Netty HTTP/2 默认流控回顾

Netty 的 HTTP/2 实现有两层流控:
1. **Netty 的 `DefaultHttp2RemoteFlowController`**: 维护每个 stream 的发送窗口, 客户端发送 DATA
   会消耗窗口, 服务器处理后调 `consumeBytes` 释放窗口, `flush()` 时给客户端发 `WINDOW_UPDATE`。
2. **应用层**: handler 通过 `onDataRead` 接收数据, 应用可以暂存/丢弃/限速。

## mu-server 的实现

`Http2ConnectionFlowControl` (Http2Connection.java:25-124) 是基类, 用 `HashMap<Integer, ...>` 维护两个状态:
```java
private final Map<Integer, Queue<DataReadData>> buffer = new HashMap<>();
private final Map<Integer, Boolean> wantsToRead = new HashMap<>();
```

`DataReadData` 持有 `ByteBuf data`, `padding`, `endOfStream`。

### 读路径

```java
// Http2Connection.java:75-80 — Netty 调用
public int onDataRead(ChannelHandlerContext ctx, int streamId, ByteBuf data, int padding, boolean endOfStream) {
    Queue<DataReadData> buf = buffer.computeIfAbsent(streamId, integer -> new LinkedList<>());
    buf.add(new DataReadData(data.retain(), padding, endOfStream));
    sendItMaybe(ctx, streamId);
    return 0;                           // 不消耗 Netty 默认的窗口
}

// Http2Connection.java:55-70
private void sendItMaybe(ChannelHandlerContext ctx, int streamId) {
    if (ctx.channel().isActive()) {
        Boolean wantsIt = wantsToRead.get(streamId);
        if (wantsIt != null && wantsIt) {
            Queue<DataReadData> queue = buffer.get(streamId);
            if (queue != null) {
                DataReadData msg = queue.poll();
                if (msg != null) {
                    wantsToRead.put(streamId, false);    // 标记"已喂一帧, 等业务回来"
                    onDataRead0(ctx, streamId, msg.data, msg.padding, msg.endOfStream);
                    msg.data.release();
                }
            }
        }
    }
}
```

### 业务路径

```java
// Http2Connection.java:319-365
@Override
protected void onDataRead0(ChannelHandlerContext ctx, int streamId, ByteBuf data, int padding, boolean endOfStream) {
    int dataSize = data.readableBytes();
    int consumed = dataSize + padding;
    // ... 构造 HttpContent, 调 httpExchange.onMessage() ...
    DoneCallback doneCallback = error -> {
        Http2Stream stream = this.connection().stream(streamId);
        if (stream != null && this.decoder().flowController().consumeBytes(stream, consumed)) {
            ctx.flush();                              // [关键] 真正释放流控窗口
        }
        data.release();
        if (error != null) {
            ctx.fireUserEventTriggered(new MuExceptionFiredEvent(...));
        } else if (!endOfStream) {
            read(ctx, streamId);                       // [关键] 请求下一帧
        }
    };
    httpExchange.onMessage(ctx, msg, error -> {
        // ... 调 doneCallback ...
    });
}

// Http2Connection.java:46-53 — 业务线程调用
protected void read(ChannelHandlerContext ctx, int streamId) {
    if (!ctx.executor().inEventLoop()) {
        ctx.executor().execute(() -> read(ctx, streamId));
        return;
    }
    wantsToRead.put(streamId, true);                   // [关键] 标记"业务想读下一帧"
    ctx.executor().submit(() -> sendItMaybe(ctx, streamId));  // [关键] 触发 sendItMaybe
}
```

### 触发 read() 的入口

```java
// Http2Connection.java:282-286
muReq.addChangeListener((exchange, newState) -> {
    if (newState == RequestState.RECEIVING_BODY) {
        read(ctx, streamId);                            // 请求体开始接收时触发
    }
});
```

也就是说, **业务线程每次准备读 body 时, 才会触发下一帧的投递**。

## 流控语义对照

| 阶段 | 谁负责 | 做什么 |
|---|---|---|
| 客户端发送 DATA | Netty | 消耗 Netty 默认窗口 |
| Netty 调 `onDataRead` | mu | 入 buffer, 返回 0 (不消耗 Netty 默认窗口) |
| 业务触发 `read(streamId)` | mu | 设 `wantsToRead[streamId]=true`, submit `sendItMaybe` |
| `sendItMaybe` 检查 | mu | 若 `wantsToRead` 且 buffer 有数据, 取一帧调 `onDataRead0` |
| 业务处理 frame | 业务线程 | `httpExchange.onMessage(...)` (切到业务线程) |
| 业务处理完成 | 业务线程 | `doneCallback` 调 `flowController.consumeBytes(...)` + `flush()` |
| Netty 发 WINDOW_UPDATE | Netty | 客户端可以继续发 |

## 内存压力

`buffer` 是个未限容的 `LinkedList` —— **没有上限**。
如果业务线程慢/卡死, `buffer[streamId]` 会无限堆积 → **OOM 风险**。

不过通常 `wantsToRead` 保持 `false`, 直到业务准备好读下一帧; 此时 buffer 最多堆积一帧,
所以实际上正常情况下不会爆内存。只有"业务一直没触发 read() 但 Netty 又往里塞"才会堆积——这通常意味着业务卡死了。

## 流的清理

```java
// Http2Connection.java:96-111
protected void cleanStream(int streamId) {
    wantsToRead.remove(streamId);
    cleanBuffer(streamId);
}

protected void cleanBuffer(int streamId) {
    Queue<DataReadData> removed = buffer.remove(streamId);
    if (removed != null) {
        for (DataReadData dat : removed) dat.data.release();   // 释放 buffer 中的 ByteBuf
        removed.clear();
    }
}

@Override
public void onDataRead(ChannelHandlerContext ctx, int streamId, ByteBuf data, int padding, boolean endOfStream) {
    if (!exchanges.containsKey(streamId)) {
        super.cleanBuffer(streamId);                             // exchange 已被取消 → 直接丢
        return data.readableBytes() + padding;                   // 报告消耗的字节数
    }
    return super.onDataRead(ctx, streamId, data, padding, endOfStream);
}
```

`cleanStream` 在 `Http2Connection` 的 exchange 状态监听器里调 (271-280):
```java
httpExchange.addChangeListener((exchange, newState) -> {
    if (newState.endState()) {
        muReq.cleanup();
        cleanStream(streamId);                                   // 删 wantsToRead + buffer
        if (newState == HttpExchangeState.ERRORED) {
            resetStream(ctx, streamId, Http2Error.INTERNAL_ERROR.code(), ctx.voidPromise());
            ctx.flush();
        }
    }
});
```

## connection-level 清理

```java
// Http2Connection.java:114-122
@Override
protected void handlerRemoved0(ChannelHandlerContext ctx) throws Exception {
    cleanup();           // 清空所有 stream 的 buffer
    super.handlerRemoved0(ctx);
}

@Override
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
    cleanup();
    super.channelInactive(ctx);
}

// Http2Connection.java:206-213
@Override
protected void cleanup() {
    super.cleanup();                            // 清所有 buffer + wantsToRead
    if (!exchanges.isEmpty()) {
        for (Integer streamId : exchanges.keySet()) {
            cancelExchange(streamId);            // 取消所有未完成的 exchange
        }
    }
}
```

## 设计观察

1. **应用层背压** = 业务线程是"主动 reader", 跟 Netty 默认的"被动 consumer" 是不同模式。
2. **`return 0` 不消耗 Netty 默认窗口**——这是关键, 让 Netty 默认流控暂不起作用,
   完全交给 mu 的 buffer + wantsToRead 控制。
3. **业务线程慢 → 数据留在 mu buffer → 流控窗口不释放 → 客户端发 WINDOW_UPDATE 不能
   被响应 → 客户端 TCP 接收窗口被填满 → 客户端停发**——背压链完整。
4. **没有 buffer 上限**是个潜在问题——若业务线程卡死, 数据会无限堆积。
5. **`consumeBytes` + `flush`** 在 `doneCallback` 中调, **确保 WINDOW_UPDATE 在业务处理完后才发出**——这是"严格流控"的体现。
6. **相对 0.0.3 而言**: 流控实现基本相同, 但 `cleanStream` 时机更明确, `buffer` 的清理路径更清晰。

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
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
