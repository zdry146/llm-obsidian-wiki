---
title: "mu-server 2.2.9 状态机 (RequestState/ResponseState/HttpExchangeState)"
category: synthesis
tags: [java, mu-server, state-machine, observer-pattern, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
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

# 08 - 状态机 (State Machines)

> mu-server 通过三层状态机协同工作: RequestState / ResponseState / HttpExchangeState。

## 1. RequestState (RequestState.java, 42 行)

```java
enum RequestState {
    HEADERS_RECEIVED(false),   // 头收到, body 未开始
    RECEIVING_BODY(false),     // body 接收中
    COMPLETE(true),            // 完整 body 收完
    ERRORED(true);             // 出错
}
```

**转移图**:
```
HEADERS_RECEIVED ──requestBodyReader claim──→ RECEIVING_BODY
RECEIVING_BODY   ──LastHttpContent 收到──→ COMPLETE
[任意非终态] ──异常 / 客户端断开──→ ERRORED
```

**触发点** (`NettyRequestAdapter`):
- 构造时 `HEADERS_RECEIVED` (默认, 34)
- `claimingBodyRead(reader)` (196-212): `setState(RECEIVING_BODY)` + `scheduleReadTimeout()`
- `HttpExchange.onMessage(...)` 的 `onDone` 回调 (199-203): `if (last) request.setState(COMPLETE)`
- `onCancelled(reason, ex)` (382-388): `setState(ERRORED)`

**Listener 通知** (`setState()`, 427-437): `CopyOnWriteArrayList<RequestStateChangeListener>` 用于:
- HTTP/1: 在 `RECEIVING_BODY` 时调 `ctx.channel().read()` (自动读下一块)
- HTTP/2: 在 `RECEIVING_BODY` 时调 `read(ctx, streamId)` (应用层 backpressure)

**不能反向转移**: `setState` 检查 `oldState.endState()` 是 true 则抛异常 (430-432)。

## 2. ResponseState (ResponseState.java, 83 行)

```java
public enum ResponseState {
    NOTHING(false, false),
    FULL_SENT(true, true),       // 短响应一次性发完
    STREAMING(false, false),     // 块发送中
    FINISHING(false, false),     // 最后块写入中
    FINISHED(true, true),        // 全部完成
    ERRORED(true, false),
    TIMED_OUT(true, false),
    CLIENT_DISCONNECTED(true, false),
    UPGRADED(true, true);
}
```

每个值带两个 flag: `endState` (是否终态) + `fullResponseSent` (是否完整发送)。
`completedWithError()` = `endState && !fullResponseSent` (用于 `status()` 判断)。
`completedSuccessfully()` = `fullResponseSent`。

**转移图 (同步路径)**:
```
NOTHING ──write()/writeOnLoop()──→ FULL_SENT (短响应)
NOTHING ──outputStream() / sendChunk() / writeOnLoop-stream──→ STREAMING
STREAMING ──complete()──→ FINISHING ──(Netty 写完)──→ FINISHED
NOTHING ──complete() (no body)──→ FINISHED
```

**异常路径**:
```
[任意非终态] ──异常 / 客户端断开──→ ERRORED / CLIENT_DISCONNECTED / TIMED_OUT
NOTHING ──WebSocket upgrade──→ UPGRADED
```

**触发点** (`NettyResponseAdaptor`):
- 构造时 `NOTHING` (默认, 31)
- `writeAndFlushToChannel(isLast, content)` (175-199): 若是最后一块, listener 设 `FULL_SENT`, 失败设 `ERRORED`
- `outputState()` (46-56): 立即设置
- `outputState(future, successState)` (65-80): future 完成后设置
- `setWebsocket()` (91-93): `UPGRADED`
- `onCancelled(reason)` (95-99): 设成传入 reason
- `complete()` (288-315): 走到 `FINISHING` → 等 last chunk → `FINISHED`

## 3. HttpExchangeState (HttpExchange.java:465-476)

```java
enum HttpExchangeState {
    IN_PROGRESS(false), COMPLETE(true), ERRORED(true), UPGRADED(true);
}
```

**计算规则** (`HttpExchange.onReqOrRespStateChange()`, 112-123):
```java
private void onReqOrRespStateChange(RequestState requestChanged, ResponseState responseChanged) {
    RequestState reqState = request.requestState();
    ResponseState respState = response.responseState();
    if (reqState.endState() && respState == ResponseState.UPGRADED) {
        onEnded(HttpExchangeState.UPGRADED);
    } else if (reqState.endState() && respState.endState()) {
        HttpExchangeState newState = reqState == RequestState.ERRORED || !respState.completedSuccessfully()
            ? HttpExchangeState.ERRORED : HttpExchangeState.COMPLETE;
        onEnded(newState);
    } else if (responseChanged != null && responseChanged.endState()) {
        request.discardInputStreamIfNotConsumed();   // 响应结束 → 丢弃未读 body
    }
}
```

**简化规则**:
| req | resp | exchange |
|---|---|---|
| 完成 | UPGRADED | UPGRADED |
| 完成 | 终态 + 成功 | COMPLETE |
| 完成 | 终态 + 失败 | ERRORED |
| ERRORED | 终态 (不管成功失败) | ERRORED |
| 任意 | 终态, req 未完成 | discardInputStream |

## 4. 状态转移的协调: 状态监听器

`HttpExchange` 构造时 (98-106) 注册:
```java
request.addChangeListener((exchange, newState) -> onReqOrRespStateChange(newState, null));
response.addChangeListener((exchange, newState) -> onReqOrRespStateChange(null, newState));
```

`NettyHandlerAdapter.onHeaders()` 设置 req 的状态监听器 (Http1Connection.java:88-92):
```java
(exchange, newState) -> {
    if (newState == RequestState.RECEIVING_BODY) {
        ctx.channel().read();        // HTTP/1: 自动读下一块
    }
}
```

`Http2Connection` (282-286) 类似但调 `read(ctx, streamId)` (应用层 backpressure)。

`Http2Connection` 还设置了 exchange 状态监听器 (271-280):
```java
httpExchange.addChangeListener((exchange, newState) -> {
    if (newState.endState()) {
        muReq.cleanup();
        cleanStream(streamId);
        if (newState == HttpExchangeState.ERRORED) {
            resetStream(ctx, streamId, Http2Error.INTERNAL_ERROR.code(), ctx.voidPromise());
            ctx.flush();
        }
    }
});
```

`Http1Connection` 类似 (93-110):
```java
(exchange, newState) -> {
    if (newState.endState()) {
        nettyHandlerAdapter.onResponseComplete(exchange, serverStats, connectionStats);
        ctx.channel().eventLoop().execute(() -> {
            // ... cleanup, ctx.channel().read()
            if (exchange.state() == HttpExchangeState.ERRORED) {
                ctx.channel().close();
            } else {
                ctx.channel().read();
            }
        });
    }
}
```

## 5. 异常流的特殊状态机

`HttpExchange.onException()` (388-448) 在异常触发时:
1. **如果还没发响应 (`!response.hasStartedSendingData()`)**: 转 `WebApplicationException` 或创建
   `InternalServerErrorException`, 写状态码 + body; 错误状态码 429/408/413 → `streamUnrecoverable=true`
2. **如果已经发响应**: 只 log, 不尝试补发 (协议不允许)
3. **finally**:
   - `streamUnrecoverable` 为 true → `response.onCancelled(ERRORED)` + `request.onCancelled(ERRORED, cause)`
     把双方状态推到终态, 触发 exchange 收尾

`fireException()` (384-386) 把异常作为 userEvent 派发:
```java
void fireException(Throwable cause) {
    ctx.pipeline().fireUserEventTriggered(new MuExceptionFiredEvent(this, streamId, cause));
}
```

`Http1Connection.userEventTriggered()` (192-195) 收到 `MuExceptionFiredEvent` → `exceptionCaught(ctx, error)` →
`exchange.onException(ctx, cause)` (如果是 HttpExchange)。

## 6. WebSocket 升级状态

1. handler 调 `request.handleAsync()` 拿 AsyncHandle
2. 调 `NettyRequestAdapter.websocketUpgrade(...)` (391-414):
   - 用 Netty `WebSocketServerHandshakerFactory` 创建 handshaker
   - 替换 pipeline 中的 `idle` handler 为新 `IdleStateHandler(idleRead/ping/0, ms)`
   - 创建 `MuWebSocketSessionImpl`, 触发 `ExchangeUpgradeEvent`
3. `Http1Connection.userEventTriggered` 收到 `ExchangeUpgradeEvent`:
   - 把 `currentExchange` 从 `HttpExchange` 换成 `MuWebSocketSessionImpl`
   - 调 `httpExchange.response.setWebsocket()` → `ResponseState.UPGRADED`
   - `httpExchange.addChangeListener(...)` 监听 UPGRADED, 收到时 `currentExchange = eue.newExchange`
   - 重新 read()

**WebSocket session 内部状态** 由 `WebsocketSessionState` 枚举管理:
- `NOT_STARTED, CONNECTED, CLOSED, TIMED_OUT, ERRORED`

## 7. 状态转移图 (综合)

```
                          [REQUEST START]
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
   RequestState.HEADERS_RECEIVED      (body?)  
                  │                       │
       claimingBodyRead()                │
                  │                       ▼
                  ▼         RequestState.RECEIVING_BODY
       ResponseState.NOTHING              │
                  │                       │
       response.write/sendChunk/outputStream
                  │                       │
                  ▼                       │
   ResponseState.STREAMING/FULL_SENT     │
                  │                       │
       response.complete()                │
                  │                       │
                  ▼                       │
   ResponseState.FINISHING               │
                  │                       ▼
                  │         LastHttpContent received
                  │                       │
                  ▼                       ▼
   ResponseState.FINISHED         RequestState.COMPLETE
                  │                       │
                  └───── both end ────────┘
                                  │
                                  ▼
                  HttpExchangeState.COMPLETE
                  
   (异常路径) → ERRORED / TIMED_OUT / CLIENT_DISCONNECTED
   (WebSocket) → UPGRADED
```

## 设计观察

1. **状态机驱动**而不是回调地狱——状态变化自动触发清理和后续动作。
2. **RequestState / ResponseState 解耦**: 各自独立管理, HttpExchange 在双方都到终态时合并。
3. **`UPGRADED` 是特殊的 "成功" 终态**: 响应不算"完成" (没正常 body), 但也不算错误。
4. **listener 用 `CopyOnWriteArrayList`**: 写少读多, 加 listener 不阻塞现有 listener。
5. **状态转移单向**: 一旦终态就不能再转, 任何"再转移"会立即抛 `IllegalStateException`。
6. **`discardInputStreamIfNotConsumed()`**: 响应结束后, 自动清理未读的请求体, 避免连接挂着。

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
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
