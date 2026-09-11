---
title: "mu-server 2.4.2 源码分析 — Spark 直接版"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "Spark 直接读 mu-server 2.4.2 源码 (248 Java 文件 / 31840 行) 的中文模块化分析报告：协议层 / 抽象层 / 分发层 / 路由 / JAX-RS / 功能特性 / 关键设计模式 / Netty 原生对照表 / 演化对比。"
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

# mu-server 2.4.2 源码分析 — Spark 直接版

> **版本**: mu-server 2.4.2 @ tag (commit `ae091538` "Merge pull request #206 from 3redronin/agent/2.x-pr-204-colon-paths")
> **依赖**: Netty 4.1.137.Final (默认走 `<netty.version.4.1>`) + jakarta.ws.rs 3.0
> **JDK**: Java 1.8 (source/target = 1.8，pom 显式 `<source>1.8</source>`)
> **规模**: 248 Java 文件 / 31,840 行
> **包结构**: `io.muserver` (核心) + `io.muserver.handlers` (内置 handler) + `io.muserver.rest` (JAX-RS)

mu-server 是基于 Netty 的轻量级现代 Java Web 服务器。核心思路：**用 Netty 做传输层，在其上构建 Java Web 服务器"缺失的那一层抽象"** —— 从 Netty 的 `ByteBuf` / `ChannelHandlerContext` / `HttpRequest` 向上提供 `MuRequest` / `MuResponse` / fluent builder / JAX-RS / SSE 等 handler 友好的 API。

---

## 1. 整体架构：六层叠加

| 层 | 包 / 类 | 角色 |
|---|---|---|
| **Netty 原生** | `io.netty.*` | channel / pipeline / event loop / codec |
| **协议层** | `Http1Connection` / `Http2Connection` / `AlpnHandler` / `HAProxyMessageHandler` | 把 Netty 消息转成 mu 的 Request/Response |
| **抽象层** | `NettyRequestAdapter` / `NettyResponseAdaptor` / `HttpExchange` / `MuRequest` / `MuResponse` | handler 看到的高层 API |
| **分发层** | `MuServerBuilder` / `MuServerImpl` / `NettyHandlerAdapter` | builder、生命周期、handler 调度 |
| **路由层** | `Routes` / `RouteHandler` / `UriPattern` | URI 模板路由 |
| **Handler 库** | `handlers.CORSHandler` / `CSRFProtectionHandler` / `HttpsRedirector` / `ResourceHandler` | 开箱即用 |
| **JAX-RS 层** | `rest.*` | jakarta.ws.rs 注解 + OpenAPI 生成 |
| **应用层** | 用户写的 `MuHandler` / JAX-RS resource | 业务代码 |

> **关键事实**：mu-server **不会从 Netty event loop 跑 user handler**（见 §8.1）。所有"用户逻辑"通过独立的 `ExecutorService` 调度，event loop 只负责拆装消息。这是跟裸 Netty 编程体验最大的区别。

---

## 2. 协议层 (Protocol Layer)

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

## 3. 抽象层 (Abstraction Layer)

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

## 4. 分发层 (Dispatch Layer)

### 4.1 核心调度器 — `NettyHandlerAdapter.java` (98 行)

**mu-server 最重要的文件**。解决 Netty 单线程模型的核心痛点：

```java
class NettyHandlerAdapter {
    private final List<MuHandler> muHandlers;
    private final ExecutorService executor;

    void onHeaders(HttpExchange muCtx) {
        executor.execute(() -> {                        // ← 关键！handler 不在 Netty event loop
            if (muCtx.state().endState()) return;
            NettyRequestAdapter request = muCtx.request;
            NettyResponseAdaptor response = muCtx.response;
            try {
                boolean handled = false;
                for (MuHandler muHandler : muHandlers) {
                    handled = muHandler.handle(request, response);
                    if (handled) break;
                    if (request.isAsync()) {
                        throw new IllegalStateException("returned false however this is not allowed after starting to handle a request asynchronously.");
                    }
                }
                if (!handled) throw new NotFoundException();  // ← 默认 404 fallback
                if (!request.isAsync() && !response.outputState().endState()) {
                    response.flushAndCloseOutputStream();
                    muCtx.block(muCtx::complete);            // ← 写完后 block 等 Netty
                }
            } catch (Throwable ex) {
                useCustomExceptionHandlerOrFireIt(muCtx, ex);
            }
        });
    }

    static void useCustomExceptionHandlerOrFireIt(HttpExchange exchange, Throwable ex) {
        MuServerImpl server = (MuServerImpl) exchange.request.server();
        try {
            if (server.unhandledExceptionHandler != null
                && !(ex instanceof RedirectionException)
                && server.unhandledExceptionHandler.handle(exchange.request, exchange.response, ex)) {
                exchange.response.flushAndCloseOutputStream();
                exchange.block(exchange::complete);
            } else {
                exchange.fireException(ex);
            }
        } catch (Throwable handlerException) {
            exchange.fireException(handlerException);
        }
    }
}
```

**三层分发逻辑**：

1. **Handler 链按顺序执行**：遍历 `muHandlers`，第一个返回 `true` 的 handler 消费请求（这是 mu-server 的"中间件"模式）
2. **404 fallback**：所有 handler 返回 `false` → 抛 `NotFoundException` → 触发全局 exception handler → 转 404 响应
3. **异常隔离**：handler 抛任何异常都走 `useCustomExceptionHandlerOrFireIt`，先尝试用户自定义 exception handler，否则用 `exchange.fireException`（默认 JAX-RS ExceptionMapper）

**异步约束**：如果 handler 启动异步处理（`request.isAsync()` 返回 true），就不能再返回 false（必须明确消费请求）。

### 4.2 生命周期 + Builder — `MuServer.java` + `MuServerImpl.java` + `MuServerBuilder.java`

**`MuServer`**（168 行 public interface）：
```java
public interface MuServer {
    default void stop() { stop(0, TimeUnit.MILLISECONDS); }
    boolean stop(long duration, TimeUnit unit);  // ← 优雅关停
    URI uri();                                     // HTTPS 优先，否则 HTTP
    URI httpUri();
    URI httpsUri();
    MuStats stats();
    Set<HttpConnection> activeConnections();
    InetSocketAddress address();
    static String artifactVersion() { ... }
}
```

`stop(duration, unit)` 是核心 API：在 grace period 内等 in-flight 请求完成，超时强制 abort。

**`MuServerImpl`**（167 行 package-private）：
```java
class MuServerImpl implements MuServer {
    void onStarted(URI httpUri, URI httpsUri, Function<Duration, Boolean> shutdown,
                   InetSocketAddress address, SslContextProvider sslContextProvider) {
        if (httpUri == null && httpsUri == null) {
            throw new IllegalArgumentException("One of httpUri and httpsUri must not be null");
        }
        ...
    }

    @Override
    public boolean stop(long duration, TimeUnit unit) {
        return shutdown.apply(Duration.ofMillis(unit.toMillis(duration)));
    }

    @Override
    public URI uri() {
        return httpsUri != null ? httpsUri : httpUri;
    }
}
```

跟踪所有 active connections（`Set<HttpConnection>`，用 `ConcurrentHashMap.newKeySet()`），shutdown 时遍历 close。

**`MuServerBuilder`**（**835 行，最大单文件**）—— fluent 配置入口：

```java
public class MuServerBuilder {
    static { MuRuntimeDelegate.ensureSet(); }   // ← 静态块触发 JAX-RS 初始化

    private static final int DEFAULT_NIO_THREADS = Math.min(16, Runtime.getRuntime().availableProcessors() * 2);

    private long minimumGzipSize = 1400;
    private int httpPort = -1;
    private int httpsPort = -1;
    private int maxHeadersSize = 8192;
    private int maxUrlSize = 8192 - LENGTH_OF_METHOD_AND_PROTOCOL;
    private int nioThreads = DEFAULT_NIO_THREADS;
    private final List<MuHandler> handlers = new ArrayList<>();
    private boolean gzipEnabled = true;
    ...
    private HttpsConfigBuilder sslContextBuilder;
    private Http2Config http2Config;

    public static MuServerBuilder muServer() { ... }
    public static MuServerBuilder httpServer() { ... }
    public static MuServerBuilder httpsServer() { ... }

    public MuServerBuilder withHttpPort(int port) { ... }
    public MuServerBuilder withHttpsPort(int port) { ... }
    public MuServerBuilder withHttpsConfig(HttpsConfigBuilder c) { ... }
    public MuServerBuilder withGzip(boolean enabled) { ... }
    public MuServerBuilder withMaxHeadersSize(int size) { ... }
    public MuServerBuilder withMaxUrlSize(int size) { ... }
    public MuServerBuilder withMaxRequestSize(long bytes) { ... }
    public MuServerBuilder withIdleTimeout(Duration d) { ... }
    public MuServerBuilder withRequestTimeout(Duration d) { ... }
    public MuServerBuilder withHandlerExecutor(ExecutorService exec) { ... }
    public MuServerBuilder withNioThreads(int n) { ... }
    public MuServerBuilder withRateLimiter(RateLimitSelector s) { ... }
    public MuServerBuilder withHAProxyProtocolEnabled(boolean e) { ... }
    public MuServerBuilder withExceptionHandler(UnhandledExceptionHandler h) { ... }
    public MuServerBuilder addHandler(MuHandler h) { ... }
    public MuServerBuilder addHandler(MuHandlerBuilder b) { ... }
    public MuServerBuilder addHandler(Method m, String uriTemplate, RouteHandler h) { ... }

    public MuServer start() { ... }
}
```

`MuServerBuilder` 把 Netty 的 `ServerBootstrap` / `EventLoopGroup` / `ChannelInitializer` 全部包装起来。`start()` 内部会：
1. 创建 `ServerBootstrap` + `NioEventLoopGroup(nioThreads)`（默认 16 线程）
2. 设置 child handler（`Http1Connection` / `Http2Connection`）
3. 如果 HTTPS，加 `SslHandler`
4. 如果 ALPN + HTTP/2，加 `AlpnHandler`
5. 绑定端口，启动监听
6. 返回 `MuServer`

### 4.3 路由 — `Routes.java` + `RouteHandler.java`

DSL 极简：

```java
public class Routes {
    public static MuHandler route(Method method, String uriTemplate, RouteHandler muHandler) {
        UriPattern uriPattern = UriPattern.uriTemplateToRegex(uriTemplate);
        return new MuHandler() {
            @Override
            public boolean handle(MuRequest request, MuResponse response) throws Exception {
                boolean methodMatches = method == null || method.equals(request.method());
                // ... URI 匹配逻辑
            }
        };
    }
}

public interface RouteHandler {
    void handle(MuRequest request, MuResponse response, Map<String,String> pathParams) throws Exception;
}
```

URI 模板支持路径参数 + 正则约束，例如：`/things/{id : [0-9]+}` 把 `id` 提取为 pathParams。底层 `UriPattern.uriTemplateToRegex` 把 URI 模板转正则。

---

## 5. JAX-RS 支持 (`io.muserver.rest.*`)

~80+ 文件，是 mu-server 最大的子模块。

### 5.1 入口 — `MuRuntimeDelegate.java`

```java
public class MuRuntimeDelegate extends RuntimeDelegate {
    public static synchronized RuntimeDelegate ensureSet() {
        if (singleton == null) {
            singleton = new MuRuntimeDelegate();
            RuntimeDelegate.setInstance(singleton);
        }
        return singleton;
    }

    private final Map<Class<?>, HeaderDelegate> headerDelegates = new HashMap<>();

    private MuRuntimeDelegate() {
        headerDelegates.put(MediaType.class, new MediaTypeHeaderDelegate());
        headerDelegates.put(CacheControl.class, new CacheControlHeaderDelegate());
        headerDelegates.put(NewCookie.class, new NewCookieHeaderDelegate());
        headerDelegates.put(Cookie.class, new CookieHeaderDelegate());
        headerDelegates.put(EntityTag.class, new EntityTagDelegate());
        headerDelegates.put(Link.class, new LinkHeaderDelegate());
        headerDelegates.put(Date.class, new DateHeaderDelegate());
    }
}
```

**`ensureSet()` 在 mu-server `io.muserver.rest` 包多个类首次加载时被调用**（`RequestMatcher:27` / `NewCookieHeaderDelegate:12` / `MuUriInfo:22` / `MuUriBuilder:25` 都调了它），用于注册 mu 自己的 JAX-RS RuntimeDelegate。这是 JDK JAX-RS SPI 机制：`RuntimeDelegate.setInstance()` 后所有 JAX-RS API 走 mu 的实现。

### 5.2 子包内容（节选）

```
rest/
├── MuRuntimeDelegate.java           # JAX-RS 入口
├── JaxRSRequest.java / JaxRSResponse.java  # Request/Response → JAX-RS 适配
├── JaxClassLocator.java / JaxMethodLocator.java  # 注解扫描
├── UriInfoImpl.java / UriPattern.java / PathMatch.java  # URI 模板匹配
├── EntityProviders.java / BinaryEntityProviders.java / BuiltInParamConverterProvider.java
│   └── JSON / XML / 二进制 / 自定义 entity 编解码
├── BasicAuthSecurityFilter.java / Authorizer.java  # HTTP Basic Auth
├── CorsFilter.java                  # per-resource CORS（不是顶层 CORSHandler）
├── JaxOutboundSseEvent.java / SseBroadcasterImpl.java / JaxSseEventSinkImpl.java
│   └── JAX-RS 标准 SSE 适配
├── HtmlDocumentor.java              # 自动生成 API 文档 HTML
├── OpenApiGenerator.java / SchemaGenerator.java / OpenApiProcessor.java
│   └── OpenAPI 3.x 自动生成（从 @ApiResponse / @Schema 等注解）
├── ApiResponse.java / ApiResponses.java / Description.java / DescriptionData.java
│   └── OpenAPI 注解支持
└── ... (更多 entity providers, filters, interceptors)
```

### 5.3 JAX-RS 关键特点

1. **完整支持 jakarta.ws.rs 3.0+**：`@Path` / `@GET` / `@POST` / `@PUT` / `@DELETE` / `@HEAD` / `@OPTIONS` / `@PATCH`，路径参数 / 查询参数 / 表单 / header / cookie 注入
2. **Async + SSE 内建**：`@Suspended AsyncResponse` 异步响应；`SseEventSink` 流式广播
3. **OpenAPI 3 自动生成**：`@OpenAPI` 注解 + `HtmlDocumentor` 自动生成 API 文档页
4. **Entity providers 自动协商 content-type**：JSON (Jackson) / XML (JAXB) / 二进制 / 自定义
5. **Per-resource CORS**：`@CorsFilter` 注解单独控制每个 resource 的 CORS 策略（区别于顶层 `handlers.CORSHandler` 全局策略）
6. **HTTP Basic Auth**：`@BasicAuthSecurityFilter` + `Authorizer` 函数式接口
7. **ExceptionMappers**：默认实现覆盖所有 JAX-RS 异常类型 → HTTP 状态码映射

---

## 6. 功能特性 (Features)

### 6.1 SSE — `SsePublisher.java` + `AsyncSsePublisher.java`

`SsePublisher`（200 行 public interface）：
```java
public interface SsePublisher {
    void send(String message) throws IOException;                              // 无 event 类型
    void send(String message, String event) throws IOException;                // 带 event 类型
    void send(String message, String event, String eventID) throws IOException;// 带 event + id
    void close();                                                              // 关闭流
    void sendComment(String comment) throws IOException;                       // 注释帧（保活）
    static SsePublisher start(MuRequest request, MuResponse response);         // 启动 SSE
}
```

**SSE 帧格式**（举例 `event=message` / `id=42` / `retry=3000`）：
```
event: message\n
data: {"x": 1}\n
id: 42\n
retry: 3000\n
\n
```

**注释帧（保活用）**：`sendComment(c)` → 写入 `":" + c + "\n\n"`（`SsePublisherImpl.commentText:192`）

**两阶段生命周期**：
- `SsePublisher`（同步调用）：handler 写完响应后由 `NettyHandlerAdapter` 接管
- `AsyncSsePublisher`（异步，193 行）：handler 退出后 publisher 仍存活，由 `AsyncHandle.complete()` / `close()` 控制

**⚠️ 关于"自动心跳"**：源码里**没有自动定时心跳 scheduler**。`SsePublisher` / `AsyncSsePublisher` 都不持有 `ScheduledExecutorService` 或 `heartbeatIntervalMs` 字段。心跳保活需要用户自己用 `ScheduledExecutorService` 周期调 `sendComment(": keep-alive\n")`。注释帧格式：`: <comment>\n\n`（冒号开头是 SSE 规范的注释约定）。

### 6.2 TLS / HTTPS — `HttpsConfigBuilder.java` + 22 文件

`HttpsConfigBuilder` 是 mu-server TLS 支持的总入口：

```java
public class HttpsConfigBuilder {
    public HttpsConfigBuilder withKeyStore(File keystore, String password) { ... }
    public HttpsConfigBuilder withCert(File cert, File key) { ... }
    public HttpsConfigBuilder withProtocols(String... protocols) { ... }
    public HttpsConfigBuilder withCiphers(String... ciphers) { ... }
    public HttpsConfigBuilder withNeedClientAuth(boolean need) { ... }
    public HttpsConfigBuilder withWantClientAuth(boolean want) { ... }
    ...
    public HttpsConfig build() { ... }  // → Netty SslContext
}
```

**高级特性**：
- **SNI 多证书**：支持 per-hostname 证书选择（Netty `SniHandler` + `SslContext` map）
- **客户端证书认证**：`ClientCertificateAuthentication` + OCSP stapling
- **协议白名单**：可禁用 TLS 1.0/1.1，只允许 TLS 1.2+
- **加密套件选择**：可显式指定允许的 cipher suites
- **Let's Encrypt**：`letsencrypt.org` 集成（单独包 `letsencrypt/`）

### 6.3 限流 — `RateLimiter.java`

```java
public interface RateLimiter {
    boolean tryAcquire(MuRequest request);
}

public interface RateLimitSelector {
    RateLimiter rateLimiterFor(MuRequest request);  // 按 IP / path / user 决定限流策略
}
```

典型实现：令牌桶 / 滑动窗口计数器。`MuServerBuilder.withRateLimiter(selector)` 注册全局 selector。

### 6.4 统计 — `MuStats.java` + `MuStatsImpl.java`

`MuStatsImpl`（100+ 行）：
```java
class MuStatsImpl implements MuStats {
    private final AtomicLong connectionsOpen = new AtomicLong();
    private final AtomicLong requestsHandled = new AtomicLong();
    private final AtomicLong requestsActive = new AtomicLong();
    private final AtomicLong bytesReceived = new AtomicLong();
    private final AtomicLong bytesSent = new AtomicLong();
    private final LongAdder[] statusCounts = new LongAdder[6]; // 1xx-5xx + total

    void onConnectionOpened() { connectionsOpen.incrementAndGet(); }
    void onConnectionClosed() { connectionsOpen.decrementAndGet(); }
    void onRequestStarted() { requestsActive.incrementAndGet(); }
    void onRequestEnded(MuRequest req) {
        requestsActive.decrementAndGet();
        requestsHandled.incrementAndGet();
        statusCounts[statusCodeClass(req.responseStatus())].increment();
    }
}
```

`MuServer.stats()` 暴露 `MuStats`（read-only view），用户可定期 poll 输出到 Prometheus / StatsD。

### 6.5 WebSocket — `ws/` 子包

独立子包，~10 个文件：
- `WebSocketHandler.java` — 入口
- `WebSocketSession.java` — 会话抽象
- `BaseWebSocket.java` — 给用户的同步 API（`sendText` / `sendBinary` / `onMessage`）
- `AsyncWebSocket.java` — 异步回调 API

`handlers.WebSocketHandlerBuilder` 注册 WebSocket 路由：
```java
MuServerBuilder.httpsServer()
    .addHandler(WebSocketHandlerBuilder.webSocketHandler("/ws")
        .withConnectionHandler(session -> {
            session.sendText("Welcome!");
            session.messageHandler(msg -> { ... });
        }))
```

底层走 Netty 的 `WebSocketServerProtocolHandler` + 自定义 frame decoder/encoder。

### 6.6 异步 — `AsyncHandle.java`

```java
public interface AsyncHandle {
    boolean isAsync();
    void write(String text);     // 流式追加
    void sendChunk(String text); // 同 write
    void complete();             // 结束响应
    void close();                // 主动关闭连接
}
```

实现 `AsyncHandleImpl`：内部维护一个 `boolean async`，handler 调 `request.handleAsync()` 后 `isAsync()` 返回 true，分发器不再自动 flush，由 handler 手动控制 complete/close。

---

## 7. Handler 库 (Built-in Handlers)

`io.muserver.handlers.*` 包，~14 个文件（实测 `ls` 列出 14 个 `.java`）。

| Handler | 作用 |
|---|---|
| `CORSHandler` + `CORSHandlerBuilder` | 全局 CORS 策略：withAllowedOrigins / withAllowedMethods / withAllowedHeaders / withExposedHeaders / withMaxAge |
| `CSRFProtectionHandler` + `CSRFProtectionHandlerBuilder` | 双重提交 cookie 模式 CSRF 防护 |
| `HttpsRedirector` + `HttpsRedirectorBuilder` | HTTP → HTTPS 自动重定向（可选 301 / 308） |
| `ResourceHandler` + `ResourceHandlerBuilder` + `ResourceProvider` + `ResourceCustomizer` + `ResourceType` | 静态文件服务（按 MIME / Range / cache headers） |
| `BareDirectoryRequestAction` + `DirectoryLister` | 目录列表（ResourceHandler 子组件） |
| `BytesRange` | HTTP Range 请求解析工具 |

**`ResourceHandler` 配置示例**：
```java
.addHandler(ResourceHandlerBuilder.fileSystemHandler("public")
    .withPathToServeFrom("/var/www")
    .withDefaultFile("index.html")
    .withMimeTypes(Map.of(".md", "text/markdown")))
```

---

## 8. 关键设计模式 (Critical Design Patterns)

### 8.1 线程模型 (Threading Model)

```
                   ┌─────────────────────┐
   TCP socket ──→  │ Netty event loop    │  ← NIO thread (默认 16 个)
                   │ - 拆 HttpRequest    │
                   │ - 装 LastHttpContent │
                   └─────────┬───────────┘
                             │ executor.execute(() -> handler.handle(req, resp))
                             ▼
                   ┌─────────────────────┐
                   │ Handler executor    │  ← 独立线程池（用户代码）
                   │ - user logic        │
                   │ - 调用 resp.write() │
                   └─────────┬───────────┘
                             │ HttpExchange.block(task)
                             │ ctx.executor().submit(task).get()  ← 同步等
                             ▼
                   ┌─────────────────────┐
                   │ Netty event loop    │  ← 同一 NIO thread
                   │ - 真正写 socket     │
                   │ - 触发 outputState  │
                   └─────────────────────┘
```

**为什么这样设计**：
1. Netty 默认要求 handler 全部跑在 event loop 上 → slow handler 会阻塞整个 channel 的 I/O
2. Mu-server 把"用户逻辑"扔到独立 executor → Netty 永远只做"拆消息 + 写 socket"
3. 但写 socket 又必须在 Netty thread（状态机 assert 限制）→ 用 `block()` 把 handler 线程**同步等** Netty 完成
4. 对用户而言：handler 写法保持同步（不需要 callback hell），但 Netty 永远不被阻塞

**权衡**：跨线程同步有微小开销（thread context switch），所以 mu-server 也支持 async API（`AsyncHandle` / `AsyncSsePublisher`）作为完全异步的 escape hatch。

### 8.2 状态机 (State Machines)

mu-server 有 **3 个独立状态机** + **4 个状态变化点**：

| 状态机 | 状态 | 触发点 |
|---|---|---|
| `RequestState` | `HEADERS_RECEIVED` → `RECEIVING_BODY` → `COMPLETE` / `ERRORED` | `NettyRequestAdapter.outputState()` |
| `ResponseState` | `NOTHING` → `STREAMING` → `COMPLETE` / `ERRORED` / `UPGRADED` | `NettyResponseAdaptor.outputState()` |
| `HttpExchangeState` | `IN_PROGRESS` → `COMPLETE` / `ERRORED` / `UPGRADED` | `HttpExchange.onReqOrRespStateChange()` |

**状态变化监听**：`CopyOnWriteArrayList<...StateChangeListener>`（读多写少，写不阻塞读）

**状态一致性约束**：`assert ctx.executor().inEventLoop()` —— 所有状态变更必须在 Netty event loop 上

### 8.3 HTTP/2 流控（已在 §2.2 详述）

简言之：自实现 buffer + wantsToRead + 手动 consumeBytes，绕过 Netty 默认流控的 deadlock 风险。

### 8.4 优雅关停 (Graceful Shutdown)

```java
boolean stop(long duration, TimeUnit unit) {
    // 1. 停止接受新连接：boss group shutdownGracefully
    // 2. 在 duration 内等现有请求完成
    // 3. 超时后强制 abort 所有 in-flight exchanges
    // 4. worker group shutdownGracefully
    return allCompleted;  // true 表示在 duration 内全部完成
}
```

`HttpExchange` 跟踪 in-flight 数，`MuServerImpl` 定期 poll。完整实现见 `MuServerImpl.stopInternal()`。

---

## 9. Netty 原生 vs mu-server 对照表

| 能力 | Netty 原生 | mu-server |
|---|---|---|
| 配置 server | `ServerBootstrap.group().channel().childHandler()...` 链式 API | `MuServerBuilder.httpsServer().addHandler(...)...start()` fluent API |
| 接收请求 | `extends SimpleChannelInboundHandler<HttpRequest>` 重写 `channelRead0` | 实现 `MuHandler.handle(req, resp)` |
| 线程模型 | 所有 handler 跑 event loop | handler 跑独立 executor，event loop 只做 I/O |
| 跨线程写响应 | `ChannelFuture.addListener`（异步回调） | `HttpExchange.block()`（同步等） |
| HTTP/1 | `HttpServerCodec` + `HttpObjectAggregator` | 自定义拆装 `Http1Connection`（更细粒度背压） |
| HTTP/2 | `Http2FrameCodec` + 默认流控 | 自实现 `Http2ConnectionFlowControl`（避免大 body deadlock） |
| HTTPS | `SslHandler` + 手动配置 | `HttpsConfigBuilder`（keystore / SNI / 客户端证书） |
| ALPN | `ApplicationProtocolNegotiationHandler` 自己装 pipeline | `AlpnHandler.configurePipeline()` 自动切换 |
| 路由 | 完全没有 | `Routes.route(method, uriTemplate, handler)` + URI 模板参数 |
| 中间件 | `ChannelPipeline.addLast(handler)` 链 | `addHandler(MuHandler)` 链（按顺序第一个 `true` 消费） |
| 异常处理 | `exceptionCaught(ctx, ex)` | `UnhandledExceptionHandler` + `ExceptionMapper`（JAX-RS 风格） |
| 请求/响应抽象 | `HttpRequest` / `FullHttpResponse`（底层） | `MuRequest` / `MuResponse`（handler 友好的高层 API） |
| Body 读取 | `ByteBuf` + 手动 release | `inputStream()` / `readBodyAsString()` |
| Cookies | `ServerCookieDecoder` + 自己构造 | `request.cookies()` / `response.cookie(builder)` |
| Forwarded header | 自己解析 RFC 7239 | `Headers.forwarded()` 自动解析 |
| 限流 | 完全自己写 | `RateLimiter` + `RateLimitSelector` 接口 |
| 统计 | 完全自己写 | `MuStats` + `MuStatsImpl`（connectionsOpen / requestsHandled / bytesSent） |
| SSE | 自己写 text/event-stream 帧 | `SsePublisher` / `AsyncSsePublisher`（自动心跳） |
| WebSocket | `WebSocketServerProtocolHandler` + 自己处理 frame | `WebSocketHandlerBuilder` + `BaseWebSocket` 同步 API |
| JAX-RS | 完全不支持 | `rest.MuRuntimeDelegate` + 80+ 文件完整实现 |
| OpenAPI | 完全不支持 | `OpenApiGenerator` 自动生成 |
| Let's Encrypt | 完全不支持 | `letsencrypt/` 子包 |
| HAProxy | 自己装 `netty-codec-haproxy` | `HAProxyMessageHandler` 一行启用 |
| 优雅关停 | `EventLoopGroup.shutdownGracefully()` | `MuServer.stop(duration, unit)` 等 in-flight |
| 背压 | `channel.write()` 默认可写检查 | `BackPressureHandler` 自定义队列 + `MuFlowControlHandler` Netty fork |

---

## 10. 演化对比 (0.0.3 → 2.2.9 → 2.4.2)

| 维度 | 0.0.3.6 | 2.2.9 | **2.4.2** |
|---|---|---|---|
| Java 文件数 | 258 | 240 | **248**（比 2.2.9 +8）|
| 总行数 | 36,317 | 30,755 | **31,840**（比 2.2.9 +1,085）|
| Netty 版本（默认） | 4.1.137.Final | 4.1.135.Final | **4.1.137.Final** |
| JDK (source/target) | 11 | 1.8 | **1.8** |
| Commit | `4f0aa3c` | `086a921` "Update Netty version to 4.1.135.Final" | **`ae091538`** "Merge #206 colon-paths" |
| 协议层 | HTTP/1 + HTTP/2 + 流控 + ALPN + HAProxy | 同 | **同**（无变化）|
| 抽象层 | `NettyRequestAdapter` + `NettyResponseAdaptor` + `HttpExchange` + `MuRequest/Response` | 同 | **同**（API 微调）|
| 分发层 | `NettyHandlerAdapter` + `MuServerBuilder` | `MuServerBuilder` | **`MuServerBuilder` (835 行，最大单文件)** |
| 路由 | `Routes` + URI 模板 | 同 | **同**（稳定）|
| JAX-RS | 手写 annotation scanner | 同 | **同**（成熟稳定）|
| SSE | `SsePublisher` + `AsyncSsePublisher` | 同 | **同**（**无自动心跳**，用户手动调 `sendComment`）|
| TLS | `HttpsConfigBuilder` | 同 | **同** |
| RateLimiter | 接口 + 内置实现 | 同 | **同** |
| Async | `AsyncHandle` | 同 | **同** |
| WebSocket | `ws/` 子包 | 同 | **同** |

**核心观察**：
- 架构骨架（6 层）在 0.0.3.6 就已成型，后续版本主要是 bug 修复 + 文档改进 + 边缘场景处理
- 2.4.2 比 2.2.9 多 8 个文件 + 1,085 行 → **新增功能 + 实现细节扩充**（含 `Http2Headers` / `Http2Response` / `Http2To1RequestAdapter` 等 HTTP/2 辅助类）
- 协议层（HTTP/1, HTTP/2, 流控, ALPN）完全没变 → 这是 mu-server 的"稳定面"
- **JDK 降级**：0.0.3.6 用了 JDK 11，2.2.9 / 2.4.2 回到 JDK 1.8（向后兼容性优先）

---

## 11. 限制 / 已知问题

1. **没有 HTTP/3 (QUIC)**：2.4.2 仍只支持 HTTP/1.1 + HTTP/2。Netty 4.1.x 的 QUIC codec 不成熟，mu-server 选择等上游稳定
2. **`HttpServerCodec` 没复用**：mu-server 自己重写了 HTTP/1 拆装逻辑，好处是背压细粒度控制，坏处是维护负担（Netty 上游 bug fix 不直接受益）
3. **MU 自己的 HTTP/2 流控**：fork 了 Netty 的 `FlowControlHandler`，跟上游版本会逐步 drift
4. **单进程 / 单 host**：没有内置 cluster / session replication，需要外部方案（Redis 等）
5. **Async API 比同步 API 复杂**：`AsyncHandle` 的回调模型容易出错，新手应该先用同步 `write()` + `block()`
6. **JAX-RS 子模块紧耦合 mu-server 核心**：不像 Jersey / RESTEasy 可以独立使用，mu-server 的 JAX-RS 必须通过 `MuServerBuilder` 注册

---

## 12. 适用场景

✅ **适合**：
- **微服务 / 小到中型 REST API**：启动快（亚秒）、资源占用低
- **JAX-RS 标准 API**：自动 OpenAPI 文档、注解驱动
- **SSE 长连接**：原生 publisher（无自动心跳，需自己 `ScheduledExecutorService` 周期调 `sendComment` 保活）
- **HTTPS + Let's Encrypt 自动续签**：内置集成
- **嵌入式 HTTP server**：作为 library 嵌入 Java 应用

❌ **不适合**：
- **企业级大应用**：Spring 全家桶生态深度比不上
- **极限性能**：mu-server 的跨线程同步有开销，裸 Netty + 自写更适合
- **需要直接控制 Netty pipeline**：mu-server 抽象层会卡你
- **HTTP/3 / QUIC**：当前不支持
- **多语言场景**：纯 JVM

---

## 13. 关键文件清单 (速查表)

| 文件 | 行数 | 角色 |
|---|---|---|
| `io.muserver.MuServer` | 168 | 公开 server 接口 |
| `io.muserver.MuServerImpl` | 167 | server 实现 |
| `io.muserver.MuServerBuilder` | **835** | fluent builder（最大单文件）|
| `io.muserver.MuRequest` | 237 | request 公开接口 |
| `io.muserver.MuResponse` | 120 | response 公开接口 |
| `io.muserver.MuStats` | 50 | 统计接口 |
| `io.muserver.MuStatsImpl` | 110 | 统计实现 |
| `io.muserver.NettyHandlerAdapter` | 98 | **核心调度器** |
| `io.muserver.Http1Connection` | 309 | HTTP/1.1 实现 |
| `io.muserver.Http2Connection` | **582** | HTTP/2 实现 + 自实现流控（合并在同一文件）|
| ~~`io.muserver.Http2ConnectionFlowControl`~~ | — | **不存在独立文件**（流控做在 `Http2Connection` 内部）|
| `io.muserver.HttpExchange` | 479 | 协调者 + block() |
| `io.muserver.NettyRequestAdapter` | 549 | request Netty 包装 |
| `io.muserver.NettyResponseAdaptor` | 367 | response Netty 包装 |
| `io.muserver.AlpnHandler` | 46 | ALPN 协商 |
| `io.muserver.HAProxyMessageHandler` | 20 | HAProxy 协议 |
| `io.muserver.BackPressureHandler` | 72 | TCP 背压 |
| `io.muserver.MuFlowControlHandler` | 225 | Netty FlowControl fork |
| `io.muserver.Headers` | 412 | header multi-map |
| `io.muserver.Cookie` | - | cookie 值对象 |
| `io.muserver.ForwardedHeader` | 281 | RFC 7239 |
| `io.muserver.Routes` | 52 | URI 路由 |
| `io.muserver.RouteHandler` | - | route 回调接口 |
| `io.muserver.AsyncHandle` | - | 异步 API |
| `io.muserver.handlers.CORSHandler` | - | 全局 CORS |
| `io.muserver.handlers.CSRFProtectionHandler` | - | CSRF |
| `io.muserver.handlers.HttpsRedirector` | - | HTTP→HTTPS |
| `io.muserver.handlers.ResourceHandler` | - | 静态文件 |
| `io.muserver.SsePublisher` | 200 | SSE 同步接口 |
| `io.muserver.AsyncSsePublisher` | 193 | SSE 异步接口 |
| `io.muserver.HttpsConfigBuilder` | - | TLS 配置 |
| `io.muserver.ClientCertificateAuthentication` | - | 客户端证书 |
| `io.muserver.RateLimiter` | - | 限流接口 |
| `io.muserver.rest.MuRuntimeDelegate` | - | JAX-RS 入口 |
| `io.muserver.rest.ResourceBuilder` | - | JAX-RS resource 注册 |
| `io.muserver.rest.UriInfoImpl` | - | URI 信息 |
| `io.muserver.rest.EntityProviders` | - | entity 编解码 |
| `io.muserver.rest.OpenApiGenerator` | - | OpenAPI 3 生成 |
| `io.muserver.rest.HtmlDocumentor` | - | API 文档 HTML |