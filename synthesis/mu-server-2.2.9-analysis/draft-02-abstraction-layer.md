---
title: "mu-server 2.2.9 抽象层 (Request/Response/HttpExchange)"
category: synthesis
tags: [java, netty, mu-server, abstraction, adapter-pattern, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "Netty NettyRequestAdapter/ResponseAdaptor 包装 Netty 底层 HttpRequest/ChannelFuture, 提供用户友好的 MuRequest/MuResponse API, HttpExchange 协调两件套 + 状态机"
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

# 02 - 抽象层 (Abstraction Layer)

> mu-server 把 Netty 的 `HttpRequest` / `HttpContent` / `FullHttpResponse` 包装成自己的
> `MuRequest` / `MuResponse` / `HttpExchange` 三个核心接口。
> 这层决定了**业务开发者**看到的 API。

## 文件清单

| 文件 | 角色 | 行数 |
|---|---|---|
| `MuRequest.java` | 用户面对的请求接口 | 237 |
| `MuResponse.java` | 用户面对的响应接口 | 121 |
| `HttpExchange.java` | 内部: 整合 request/response, 状态机 | 480 |
| `NettyRequestAdapter.java` | MuRequest 的 Netty 实现 | 549 |
| `NettyResponseAdaptor.java` | MuResponse 的 Netty 实现 (abstract) | 367 |
| `Http1Response.java` | HTTP/1.1 响应 | - |
| `Http2Response.java` | HTTP/2 响应 | - |
| `Http1Headers.java` / `Http2Headers.java` | 头适配 | - |
| `Headers.java` | 多值头接口 | 412 |
| `Method.java` | HTTP 方法枚举 | - |
| `RequestParameters.java` | query/form 解析结果 | - |
| `RequestBodyReader.java` | 请求 body 流式读取 | - |
| `RequestState.java` / `ResponseState.java` | 状态枚举 | 42 / 83 |
| `Exchange.java` | pipeline 行为契约 | - |
| `MuException.java` | 自定义运行时异常 | - |
| `Cookie.java` / `CookieBuilder.java` | Cookie 模型 | - |
| `ForwardedHeader.java` | RFC 7239 Forwarded 解析 | - |

## 核心抽象

### MuRequest (237 行, 用户接口)

业务开发者直接用的 API:
- `method()`, `uri()`, `serverURI()`, `headers()`
- `contentType()`, `query()`, `cookies()`, `cookie(name)`
- `form()`, `inputStream()`, `readBodyAsString()`, `uploadedFile(name)`, `uploadedFiles(name)`
- `contextPath()`, `relativePath()` (ContextHandler 改写)
- `attribute(k, v)` / `attribute(k)` / `attributes()` (请求作用域属性)
- `handleAsync()` (异步句柄)
- `remoteAddress()`, `clientIP()` (Forwarded/X-Forwarded-aware)
- `server()`, `connection()`, `protocol()`, `isAsync()`, `startTime()`

注意 **没有** Netty 风格的 `ChannelHandlerContext` 暴露, **没有** `ByteBuf` 暴露
(除了内部 `RequestBodyReader`)。这是 mu 抽象层的核心约束: 让业务代码完全脱离 Netty API。

### MuResponse (121 行, 用户接口)

- `status(int)` / `status()`
- `write(String)`, `sendChunk(String)`, `outputStream()`, `outputStream(int)`, `writer()`
- `redirect(String)`, `redirect(URI)`
- `headers()`, `contentType(cs)`, `addCookie(Cookie)`
- `hasStartedSendingData()`, `responseState()`

限制 (NettyResponseAdaptor 强制):
1. **只允许一种 body 写入方式** (`write` 或 `sendChunk` 或 `outputStream` 或 `writer`), 中途切换抛 `IllegalStateException`。
2. **写过一次**后 `status()` 不能再改。
3. **async 模式**下禁止用 `write/sendChunk/writer/outputStream` 同步方法 (`throwIfAsync()`)。

### HttpExchange (480 行, 内部核心)

同时实现 `ResponseInfo` + `Exchange` 接口 (Netty pipeline 用), 并把 request/response 关联起来。

**关键职责**:
1. **跨线程切换** (用户线程 → event loop): `block(Runnable)` / `block(Callable)` 把工作 submit 回 event loop 并 sync 等待。
2. **状态机驱动**: `onReqOrRespStateChange()` (112-123) 监听 req+resp 双方状态, 决定 `HttpExchangeState` (`IN_PROGRESS/COMPLETE/ERRORED/UPGRADED`)。
3. **请求创建工厂**: `HttpExchange.create()` (268-306) 是 HTTP/1.1 入口, 完成校验、`100-continue` 处理、`429/503` 限流。
4. **异常处理**: `onException()` (388-448) 实现"先响应 500, 再关闭连接"的标准流。
5. **backpressure & timeout**: `scheduleReadTimeout()`, `cancelReadTimeout()`。
6. **客户端断连**: `onConnectionEnded()` 触发 `CLIENT_DISCONNECTED`。

```java
// HttpExchange.java:61-78
void block(Runnable runnable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<?> task = ctx.executor().submit(runnable);
    try {
        task.get();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new UncheckedIOException(new InterruptedIOException("Interrupted while writing"));
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
        } else {
            throw new MuException("Error while writing response", cause);
        }
    }
}
```

这个模式贯穿整个 `NettyResponseAdaptor` 的所有阻塞方法: `write()`, `sendChunk()`, `writer()`,
`outputStream()`。详见 07-threading-model.md。

### NettyRequestAdapter (549 行)

- 持有 `ChannelHandlerContext ctx`, `HttpRequest nettyRequest`, `URI serverUri`, `URI uri` (client-facing)
- 通过 `ForwardedHeader` 决定 `uri()` 是返回 origin 还是 server 视角
- 请求体读取: `RequestBodyReader` (单一, 不可重入) — `inputStream()`, `readBodyAsString()`, `form()` 互斥
- 状态机: `RequestState` 4 状态 (HEADERS_RECEIVED → RECEIVING_BODY → COMPLETE/ERRORED)
- `AsyncHandleImpl` (469-546) 是 `handleAsync()` 返回的内部类, 提供 write/complete/writeListener

**重要**:`claimingBodyRead()` (196-212) 通过 event loop 把自己设为 `requestBodyReader` (CAS),
并立即 `scheduleReadTimeout()`, **必须有 `ctx.executor().inEventLoop()` 才允许写**。
这是 mu-server 的**请求体单读保证**。

### NettyResponseAdaptor (367 行, abstract)

- 状态机: `ResponseState` 9 状态 (NOTHING → STREAMING/FULL_SENT → FINISHING → FINISHED/ERRORED/TIMED_OUT/CLIENT_DISCONNECTED/UPGRADED)
- `outputState()` 双重签名: 立即设 或 future listener (NettyResponseAdaptor:65-80)
- `writeAndFlush(ByteBuffer)` (149-173): 不在 event loop 时通过 `submit` 切回 loop
- `writeFullResponse` / `writeLastContentMarker` / `sendEmptyResponse` / `writeAndFlushToChannel` 都是 abstract, 由 Http1Response / Http2Response 实现
- `complete()` (288-315) 是 "已经什么都没写就结束" 的兜底逻辑: 自动加 Content-Length 或写 304/204

### Headers (412 行)

继承 `Iterable<Map.Entry<String,String>>`, 提供:
- 基本: `get/getAll/contains/set/add/remove/clear`
- 类型化: `getInt/getLong/getFloat/getDouble/getBoolean/getTimeMillis`
- 解析型: `accept()` `acceptCharset()` `acceptEncoding()` `acceptLanguage()` `forwarded()` `cacheControl()`
- 工具: `hasBody()` 判断 Transfer-Encoding/Content-Length
- 安全: `toString(Collection<String> toSuppress)` 隐藏敏感头 (默认隐藏 authorization/cookie/set-cookie)

Http1Headers 包 `Netty HttpHeaders`, Http2Headers 包 `Netty Http2Headers`。

## 状态机对照表

| 实体 | 状态枚举 | 终态? |
|---|---|---|
| 请求 (request) | `HEADERS_RECEIVED, RECEIVING_BODY, COMPLETE, ERRORED` | COMPLETE / ERRORED |
| 响应 (response) | `NOTHING, STREAMING, FULL_SENT, FINISHING, FINISHED, ERRORED, TIMED_OUT, CLIENT_DISCONNECTED, UPGRADED` | 大部分 |
| 整体 (exchange) | `IN_PROGRESS, COMPLETE, ERRORED, UPGRADED` | COMPLETE / ERRORED / UPGRADED |

状态转移由 `HttpExchange.onReqOrRespStateChange()` (112-123) 统一协调。
`Http2Response.writeAndFlushToChannel()` / `Http1Response` 都是 abstract 实现。

## 设计观察

1. **mu 抽象层完全屏蔽 Netty**——业务代码不应该 import `io.netty.*`。
2. **请求 body 是有限资源** (24 MB 默认, 单次读取), 通过 `claimingBodyRead` 强制单读。
3. **同步方法靠 `block()` 阻塞用户线程, 同时把 I/O 切回 event loop**, 这是 mu-server
   唯一允许的同步写模式 (async 模式必须用 `AsyncHandle.write(ChannelFuture)`)。
5. **错误处理走 JAX-RS 异常体系** (`WebApplicationException` / `NotFoundException`),
   即使 mu 不是 JAX-RS 也能享受一致的错误响应生成 (参见 `HttpExchange.fireException()`)。

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
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
