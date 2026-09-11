---
title: "§3 抽象层"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "抽象层：MuRequest (237) / MuResponse (120) public interface + NettyRequestAdapter (549) + NettyResponseAdaptor (367) + HttpExchange (479 行，含跨线程同步 block() 和 3 状态机)。"
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


# 抽象层 (Abstraction Layer)

**核心目的**：把 Netty 的底层概念（`ChannelHandlerContext`、`ByteBuf`、`FullHttpRequest`）包装成 handler 友好的接口（`MuRequest`、`MuResponse`），让 user code 写起来像 servlet 而不是 Netty。

### 3.1 公开接口 — `MuRequest.java` + `MuResponse.java`

**`MuRequest`**（237 行 public interface）暴露给 handler 的 API：
- 元数据：`method()` / `uri()` / `serverURI()` / `headers()` / `contentType()`
- 查询/表单/cookie：`query()` / `form()` / `cookie(name)`
- 请求体：`inputStream()` / `readBodyAsString()`（**只能选其一读**）
- 会话状态：`startTime()` / `attribute(key, value)`（handler 间传值）

**`MuResponse`**（120 行 public interface）：
- 元数据：`status(int)` / `headers()` / `contentType()`
- body 写方式 4 选 1：`write(text)` / `sendChunk(text)` / `outputStream()` / `writer()`（**每响应只能用一种**）
- 重定向：`redirect(url)`
- 文档明确说明：`write` 只能调一次；想多次写用 `sendChunk`

**MuRequest 的注释**很有信息量：
> "You must close the input stream"（inputStream 用完必须关）
> "只能读一次"（不能再用 readBodyAsString）

这些约束在 Netty 原生 API 里没有，mu-server 在接口层面就讲清楚了。

### 3.2 Netty 实现 — `NettyRequestAdapter.java` (549 行)

```java
class NettyRequestAdapter implements MuRequest {
    private volatile RequestState state = RequestState.HEADERS_RECEIVED;
    final ChannelHandlerContext ctx;
    private final HttpRequest nettyRequest;
    private final URI serverUri;
    private final URI uri;
    private final Headers headers;
    private volatile RequestBodyReader requestBodyReader;
    private final RequestParameters query;
    private List<Cookie> cookies;
    private String contextPath = "";
    private String relativePath;
    private Map<String, Object> attributes;
    private volatile AsyncHandleImpl asyncHandle;
    ...
}
```

**关键实现细节**：

1. **状态机 `RequestState`**：`HEADERS_RECEIVED` → `RECEIVING_BODY` → `COMPLETE` / `ERRORED`，状态变更通过 `CopyOnWriteArrayList<RequestStateChangeListener>` 通知
2. **Forwarded header 处理**：`getUri()` 检查 `Forwarded` / `X-Forwarded-*` header，反向代理场景下用真实客户端 IP/host 构造 `uri()`，而 `serverURI()` 用 backend 真实 URI
3. **Cookie 解析**：用 Netty 自带的 `ServerCookieDecoder`
4. **Query 解码**：用 Netty 的 `QueryStringDecoder(uri, true)`（`true` 表示 useMode=NFC）
5. **Body 读取**：`RequestBodyReader` 抽象 + `RequestBodyReaderInputStreamAdapter` 把 Netty 的 chunked body 流式包成 `InputStream`

### 3.3 Netty 实现 — `NettyResponseAdaptor.java` (367 行)

```java
abstract class NettyResponseAdaptor implements MuResponse {
    protected final boolean isHead;
    private volatile ResponseState state = ResponseState.NOTHING;
    protected final NettyRequestAdapter request;
    protected int status = 200;
    ...

    protected void outputState(ResponseState state) {
        assert request.ctx.executor().inEventLoop() : "Status change to " + state + " not in event loop";
        // ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        // 强制约束：状态变更必须在 Netty event loop
        ResponseState oldStatus = this.state;
        if (oldStatus.endState()) {
            throw new IllegalStateException("Didn't expect to get a status update to " + state + ...);
        }
        this.state = state;
        for (ResponseStateChangeListener listener : listeners) {
            listener.onStateChange(httpExchange, state);
        }
    }
}
```

**状态机 `ResponseState`**：`NOTHING` → `STREAMING` → `COMPLETE` / `ERRORED` / `UPGRADED`

**关键约束**：`assert request.ctx.executor().inEventLoop()` —— 状态变更必须在 Netty event loop 线程。如果在 handler executor 里调 outputState，会抛 AssertionError。这是 §3.4 `HttpExchange.block()` 存在的原因。

`outputState(future, successState)` 重载版本：等 `ChannelFuture` 完成后自动切换状态。`addChangeListener` 允许 `HttpExchange` 监听响应状态变化。

### 3.4 协调者 — `HttpExchange.java` (479 行)

**整个 mu-server 抽象层的"中央协调器"**。一个 HttpExchange = 一个 NettyRequestAdapter + 一个 NettyResponseAdaptor + 一个 ChannelHandlerContext。

```java
class HttpExchange implements ResponseInfo, Exchange {
    final ChannelHandlerContext ctx;
    final NettyRequestAdapter request;
    final NettyResponseAdaptor response;
    private final int streamId; // -1 for HTTP/1, stream ID for HTTP/2
    private final HttpConnection connection;
    private volatile HttpExchangeState state = HttpExchangeState.IN_PROGRESS;
    private final List<HttpExchangeStateChangeListener> listeners = new CopyOnWriteArrayList<>();

    // ============ 关键方法：跨线程同步 ============
    void block(Runnable runnable) {
        assert !inLoop() : "Should not be blocking on the event loop";
        io.netty.util.concurrent.Future<?> task = ctx.executor().submit(runnable);
        try { task.get(); } catch (...) { ... }
    }

    void block(Callable<ChannelFuture> callable) {
        assert !inLoop() : "Should not be blocking on the event loop";
        io.netty.util.concurrent.Future<ChannelFuture> task = ctx.executor().submit(callable);
        try { task.get().sync(); } catch (...) { ... }
    }
}
```

**`block()` 是 mu-server 跨线程同步的核心**：

- `assert !inLoop()`：不能在 Netty event loop 里调 block（会死锁）
- `ctx.executor().submit(task)`：把任务 submit 回 Netty event loop
- `task.get()`：当前线程（handler executor）**阻塞等 Netty 完成**
- 这样 handler 可以在独立线程池里跑，但需要写响应时通过 block() 跨线程同步

**为什么不用 Netty 的 `ChannelFuture.addListener` 异步模型**：因为那会要求 handler 写成全异步回调地狱。`block()` 让 handler 可以保持同步写法，同时又不阻塞 Netty event loop（见 §8.1）。

### 3.5 辅助类 — Headers / Cookie / ForwardedHeader / Mutils

| 类 | 行数 | 作用 |
|---|---|---|
| `Headers.java` | 400+ | Multi-map 形式 HTTP headers，自带 `contentType()` / `forwarded()` / `authorization()` 等便利方法 |
| `Cookie.java` | 中 | Cookie 值对象，支持 `httpOnly()` / `secure()` / `sameSite()` |
| `ForwardedHeader.java` | 250+ | 解析 RFC 7239 Forwarded header（含 for/proto/host 字段） |
| `Mutils.java` | 中 | `notNull` / `coalesce` 等小工具 |

---

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
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
