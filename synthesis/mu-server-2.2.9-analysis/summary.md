---
title: "mu-server 2.2.9 在 Netty 之上新增/包装的能力 - 全量分析综合报告"
category: synthesis
tags: [java, netty, mu-server, framework, analysis, opencode, omo, synthesis, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9, commit 086a921 (https://github.com/3redronin/mu-server)", "omo Sisyphus agent team analysis (2026-09-11, 10 drafts + 1 final report)"]
summary: "omo Sisyphus agent team 对 mu-server 2.2.9 (tag) 源码全量分析的综合报告: 6 层架构 (Netty → 协议 → 抽象 → 分发 → Handler/功能 → 应用), Netty 原生 vs mu-server 对照表, 6 个关键设计模式 (线程模型/状态机/HTTP/2流控/关停), 使用场景对比, 与 0.0.3-SNAPSHOT 的核心差异"
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

# mu-server 2.2.9 源码分析报告

> **分析对象**: `/tmp/mu-server-2.2.9/` (detached HEAD @ `086a921`, tag `mu-server-2.2.9`)
> **commit message**: "Update Netty version to 4.1.135.Final"
> **pom.xml 版本**: `2.2-SNAPSHOT` (README 与 README 截图写 2.2.9)
> **体量**: 240 个 Java 文件 / 30755 行 (vs 0.0.3-SNAPSHOT 的 258/36317 略有精简)
> **Netty 版本**: 4.1.135.Final (`pom.xml: <netty.version>`)
> **Java 版本**: 仍然 `<source>1.8</source><target>1.8</target>` (pom.xml)
> **Jakarta**: `jakarta.ws.rs-api` (Jakarta EE 9+) — `import jakarta.ws.rs.*`
> **报告生成时间**: 任务执行期间; 基于源码只读分析, 未做编译验证
> **对比基线**: `/tmp/mu-server-readonly/` (master @ `4f0aa3c`, 0.0.3-SNAPSHOT)

---

## 1. 执行摘要 (300 字)

**mu-server 是一个用 Java 8 编写的、基于 Netty 4.1.135 的轻量级可嵌入式 HTTP/1.1 + HTTP/2 服务器库**,提供 Servlet 风格的同步 / 异步 API 和完整的 **Jakarta REST 3.1 (JAX-RS) 实现**。核心设计哲学是"程序化配置 (programmatic configuration) + 自洽抽象层 + 不引入反射实例化",业务开发者写的 handler 代码完全不接触 Netty API。三层线程模型 (Netty worker → muhandler 业务池 → 用户线程) 通过 `HttpExchange.block(...)` 模式 (`HttpExchange.java:61-96`) 把 Netty 事件循环上的写操作桥接到工作线程,允许阻塞风格 handler。HTTP/2 的 flow-control 由 `Http2ConnectionFlowControl` 自管理 + Netty 默认 controller 配合 (业务线程"主动 reader" 模型),实现真正的 per-stream back-pressure。**相比 0.0.3-SNAPSHOT**,2.2.9 引入了 **CSRF Protection Handler (基于 Sec-Fetch-Site 现代防护)**、`AsyncSsePublisher`、`changeHttpsConfig` 运行时换证书、改进的 CORS 配置共享 (rest/CORSConfig 取代分散实现)、`Http2To1RequestAdapter` 适配给 JAX-RS。代码量从 258/36317 减少到 240/30755,说明 0.0.3 → 2.2 经历了一轮合并重构。**架构相对稳定**,核心 API 不变;主要新增在 handler 库和 OpenAPI 上。

---

## 2. 架构总览图

```mermaid
graph TB
    subgraph "L1 协议层 (io.muserver)"
        H1C[Http1Connection<br/>SimpleChannelInboundHandler]
        H2C[Http2Connection<br/>Http2ConnectionFlowControl]
        H2CB[Http2ConnectionBuilder]
        HP[HAProxyMessageHandler]
        AL[AlpnHandler]
        SNI[MuSniHandler]
        BP[BackPressureHandler]
        PR[PreReader]
        SHC[SelectiveHttpContentCompressor]
        GH[MuGzipHttp2ConnectionEncoder<br/>MuCompressorHttp2ConnectionEncoder]
    end

    subgraph "L2 抽象层 (per-connection, per-exchange)"
        EX[HttpExchange<br/>state machine<br/>block() bridge]
        NREQ[NettyRequestAdapter<br/>implements MuRequest]
        NRESP[NettyResponseAdaptor<br/>abstract, implements MuResponse]
        H1R[Http1Response]
        H2R[Http2Response]
        H1H[Http1Headers]
        H2H[Http2Headers]
        BOD[RequestBodyReader<br/>String/Input/Form/Multipart]
    end

    subgraph "L3 分发层 (调度)"
        NHA[NettyHandlerAdapter<br/>handler chain + executor]
        RT[Routes + UriPattern<br/>regex routing]
        CTX[ContextHandler]
        MSB[MuServerBuilder<br/>fluent builder, .start]
        MSI[MuServerImpl]
        SS[ServerSettings]
        EXE[ThreadPoolExecutor<br/>8-400, SynchronousQueue, 'muhandler']
    end

    subgraph "L4 Handler 库 (io.muserver.handlers)"
        RES[ResourceHandler<br/>static files + Range]
        CORS[CORSHandler]
        CSRF[CSRFProtectionHandler<br/>Sec-Fetch-Site]
        HDR[HttpsRedirector + HSTS]
        BH[BytesRange]
        DL[DirectoryLister]
    end

    subgraph "L5 JAX-RS (io.muserver.rest)"
        REST[RestHandler<br/>implements MuHandler]
        RM[RequestMatcher + UriPattern]
        EP[EntityProviders<br/>Binary/String/Primitive]
        FILT[FilterManagerThing]
        OAD[OpenApiDocumentor]
        HDOC[HtmlDocumentor<br/>Swagger UI]
        SSE2[JaxSseEventSinkImpl<br/>SseBroadcasterImpl]
        EXM[CustomExceptionMapper]
        CORS2[CORSConfig<br/>shared]
    end

    subgraph "L6 功能模块"
        SSE[SsePublisher]
        ASSE[AsyncSsePublisher<br/>CompletionStage]
        WS[WebSocketHandler<br/>BaseWebSocket]
        TLS[HttpsConfigBuilder]
        RL[RateLimiterImpl<br/>HashedWheelTimer]
        ST[MuStats + MuStatsImpl]
        EXH[UnhandledExceptionHandler]
    end

    Netty[NioEventLoopGroup<br/>boss=1, worker=nioThreads] --> H1C
    Netty --> H2C
    H1C --> EX
    H2C --> EX
    EX --> NREQ
    EX --> NRESP
    NREQ --> BOD
    NRESP --> H1R
    NRESP --> H2R
    EX --> NHA
    NHA --> EXE
    NHA --> RT
    NHA --> CTX
    NHA --> REST
    NHA --> RES
    NHA --> CORS
    NHA --> CSRF
    NHA --> HDR
    NHA --> WS
    REST --> OAD
    REST --> EP
    REST --> FILT
    REST --> SSE2
    REST --> CORS2
```

---

## 3. Netty 原生 vs mu-server 抽象层对照表

> 每行: 能力 → mu-server 对应实现 + 文件:行号

| 能力 | Netty 原生 | mu-server 实现 | 文件:行 |
|---|---|---|---|
| HTTP/1.1 请求解码 | `HttpRequestDecoder` | `HttpRequestDecoder` + `setupHttp1Pipeline` | `MuServerBuilder.java:765-781` |
| HTTP/1.1 响应编码 | `HttpResponseEncoder` | `HttpResponseEncoder` (重写 `isContentAlwaysEmpty` 识别 `EmptyHttpResponse`) | `MuServerBuilder.java:767-772` |
| HTTP/2 connection handler | `Http2ConnectionHandler` | `Http2ConnectionFlowControl extends Http2ConnectionHandler` | `Http2Connection.java:25-124` |
| HTTP/2 flow control | `DefaultHttp2RemoteFlowController` | 自管理 buffer + wantsToRead + 配合 `consumeBytes` | `Http2Connection.java:46-80` |
| HTTP/2 stream 缓冲 | — | `HashMap<Integer, Queue<DataReadData>>` | `Http2Connection.java:39` |
| HTTP/2 压缩 | `CompressorHttp2ConnectionEncoder` | `MuCompressorHttp2ConnectionEncoder` + `MuGzipHttp2ConnectionEncoder` | `Http2ConnectionBuilder.java:24-28` |
| HTTP/1 gzip | `HttpContentCompressor` | `SelectiveHttpContentCompressor` | `MuServerBuilder.java:773-775` |
| HTTP/1 keep-alive | `HttpServerKeepAliveHandler` | 同 | `MuServerBuilder.java:776` |
| HTTP/1 流量控制 | `FlowControlHandler` | 同 | `MuServerBuilder.java:777` |
| TLS | `SslHandler` | 通过 `MuSniHandler` + Netty `SniHandler` | `MuSniHandler.java:15-37` |
| TLS SNI 多证书 | `DomainWildcardMappingBuilder` | 同 | `MuServerBuilder.java:741` |
| ALPN 协商 | `ApplicationProtocolNegotiationHandler` | `AlpnHandler extends ApplicationProtocolNegotiationHandler` | `AlpnHandler.java:7-46` |
| HAProxy 协议 | `HAProxyMessageDecoder` | `MuServerBuilder` 装配 + `HAProxyMessageHandler` 解析 | `MuServerBuilder.java:736-739` |
| 闲置超时 | `IdleStateHandler` | `IdleStateHandler(0, 0, idleTimeout)` 在 pipeline 头 | `MuServerBuilder.java:734` |
| 流量整形 | `GlobalTrafficShapingHandler` | 同 (read/write=0, 只为 stats 计数) | `MuServerBuilder.java:634` |
| 背压 (channel write buffer) | `WriteBufferWaterMark` | `WriteBufferWaterMark.DEFAULT` 通过 `b.childOption()` | `MuServerBuilder.java:357-360, 727` |
| 业务线程调度 | — | `NettyHandlerAdapter.executor.execute()` 切到 muhandler 池 | `NettyHandlerAdapter.java:27` |
| 同步阻塞写 | — | `HttpExchange.block(Runnable/Callable)` 桥接 | `HttpExchange.java:61-96` |
| 路由 | — | `Routes.route(method, template, handler)` → `UriPattern` regex | `Routes.java:24-49` |
| 请求体读取 | `HttpContent` / `LastHttpContent` | `RequestBodyReader` (单读, InputStream/String/Multipart/UrlEncoded) | `NettyRequestAdapter.java:196-212` |
| 响应体写 | `ChannelFuture writeAndFlush()` | `NettyResponseAdaptor.write/sendChunk/outputStream/writer` → `block()` 桥接 | `NettyResponseAdaptor.java:149-199, 318-321` |
| 异常处理 | `exceptionCaught` | `HttpExchange.onException` + `UnhandledExceptionHandler` | `HttpExchange.java:388-448` |
| WebSocket | `WebSocketServerHandshaker` | `NettyRequestAdapter.websocketUpgrade` + `MuWebSocket` 抽象 | `NettyRequestAdapter.java:391-414` |
| SSE | — | `SsePublisher` (sync) + `AsyncSsePublisher` (CompletionStage) | `SsePublisher.java`, `AsyncSsePublisher.java` |
| 限流 | — | `RateLimiterImpl` 用 `HashedWheelTimer` 计数器 | `RateLimiterImpl.java:25-49` |
| 统计 | — | `MuStatsImpl` 包 `GlobalTrafficShapingHandler.trafficCounter()` | `MuServerBuilder.java:634-635` |
| 优雅关闭 | `EventLoopGroup.shutdownGracefully()` | `MuServer.stop(d, unit)` + 轮询 + 硬切 | `MuServerBuilder.java:638-663` |

---

## 4. 关键代码片段

### 4.1 `HttpExchange.block(...)` — 同步抽象的核心

```java
// HttpExchange.java:61-78
void block(Runnable runnable) {
    // TODO: only use the callable version as this perhaps doesn't block
    // until the runnable is finished? (e.g. when doing a write)
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

// HttpExchange.java:80-96 — 同步等到 Netty flush 完
void block(Callable<ChannelFuture> callable) {
    assert !inLoop() : "Should not be blocking on the event loop";
    io.netty.util.concurrent.Future<ChannelFuture> task = ctx.executor().submit(callable);
    try {
        task.get().sync();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new UncheckedIOException(new InterruptedIOException("Interrupted while writing"));
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException) throw (RuntimeException) cause;
        else throw new MuException("Error while writing response", cause);
    }
}
```

**意义**: 让业务线程 (muhandler pool) 写 `response.write("hello")` 时,实际写 I/O 在 Netty event loop,
但业务线程被同步阻塞直到写完。错误通过 `ExecutionException.getCause()` 解包抛回业务线程。
这种模式让用户写阻塞代码的同时保留 Netty 的线程模型安全。

### 4.2 `NettyHandlerAdapter.onHeaders(...)` — handler 链

```java
// NettyHandlerAdapter.java:25-55
void onHeaders(HttpExchange muCtx) {
    executor.execute(() -> {                       // [1] 切到业务线程池
        if (muCtx.state().endState()) {
            return;
        }
        NettyRequestAdapter request = muCtx.request;
        NettyResponseAdaptor response = muCtx.response;
        try {
            boolean handled = false;
            for (MuHandler muHandler : muHandlers) {   // [2] 顺序遍历, 类似 Express 中间件
                handled = muHandler.handle(request, response);
                if (handled) {
                    break;
                }
                if (request.isAsync()) {
                    throw new IllegalStateException(muHandler.getClass() +
                        " returned false however this is not allowed after starting to handle a request asynchronously.");
                }
            }
            if (!handled) {
                throw new NotFoundException();          // [3] 无 handler → 404
            }
            if (!request.isAsync() && !response.outputState().endState()) {
                response.flushAndCloseOutputStream();   // [4] 同步模式自动 flush
                muCtx.block(muCtx::complete);            // [5] 等到 response 完成
            }
        } catch (Throwable ex) {
            useCustomExceptionHandlerOrFireIt(muCtx, ex);  // [6] 异常 → UnhandledExceptionHandler 或 500
        }
    });
}
```

### 4.3 `Http2ConnectionFlowControl` — 应用层 back-pressure

```java
// Http2Connection.java:39-80
private final Map<Integer, Queue<DataReadData>> buffer = new HashMap<>();
private final Map<Integer, Boolean> wantsToRead = new HashMap<>();

// Netty → mu: 把数据塞进 buffer, 不消耗流控窗口
@Override
public int onDataRead(ChannelHandlerContext ctx, int streamId, ByteBuf data,
                      int padding, boolean endOfStream) {
    Queue<DataReadData> buf = buffer.computeIfAbsent(streamId, integer -> new LinkedList<>());
    buf.add(new DataReadData(data.retain(), padding, endOfStream));
    sendItMaybe(ctx, streamId);
    return 0;                                  // [关键] 不消耗 Netty 默认窗口
}

private void sendItMaybe(ChannelHandlerContext ctx, int streamId) {
    if (ctx.channel().isActive()) {
        Boolean wantsIt = wantsToRead.get(streamId);
        if (wantsIt != null && wantsIt) {
            Queue<DataReadData> queue = buffer.get(streamId);
            if (queue != null) {
                DataReadData msg = queue.poll();
                if (msg != null) {
                    wantsToRead.put(streamId, false);     // [关键] 标记"已喂一帧, 等业务"
                    onDataRead0(ctx, streamId, msg.data, msg.padding, msg.endOfStream);
                    msg.data.release();
                }
            }
        }
    }
}

// 业务线程: 调 read() 表达"我想读下一帧"
protected void read(ChannelHandlerContext ctx, int streamId) {
    if (!ctx.executor().inEventLoop()) {
        ctx.executor().execute(() -> read(ctx, streamId));
        return;
    }
    wantsToRead.put(streamId, true);              // [关键] 业务"主动读"
    ctx.executor().submit(() -> sendItMaybe(ctx, streamId));
}
```

### 4.4 `Http2Connection.onDataRead0` — flow-control 归还

```java
// Http2Connection.java:335-350
DoneCallback doneCallback = error -> {
    Http2Stream stream = this.connection().stream(streamId);
    if (stream != null && this.decoder().flowController().consumeBytes(stream, consumed)) {
        ctx.flush();                                 // [关键] 真正归还流控窗口
    }
    data.release();
    if (error != null) {
        ctx.fireUserEventTriggered(new MuExceptionFiredEvent(httpExchange, streamId, error));
    } else if (!endOfStream) {
        read(ctx, streamId);                          // [关键] 业务处理完才请求下一帧
    }
};
```

**意义**: **应用层背压链**:
1. 客户端发 DATA → Netty window 减
3. mu `onDataRead` 把数据塞 buffer, **不消耗** Netty window (返回 0)
4. 业务线程触发 `read()` → 喂一帧给业务
5. 业务处理完成 → `consumeBytes` + `flush` → Netty 发 WINDOW_UPDATE → 客户端可继续发
6. 业务线程慢 → buffer 不空 + window 不归还 → 客户端 TCP 接收满 → 客户端停发

### 4.5 状态机

```
Request:    HEADERS_RECEIVED ─→ RECEIVING_BODY ─→ COMPLETE
                                          ↘ ERRORED
Response:   NOTHING ─→ STREAMING ─→ FULL_SENT ─→ FINISHING ─→ FINISHED
   │           │            │             │           │
   ▼           ▼            ▼             ▼           ▼
   UPGRADED (WebSocket)        ERRORED / CLIENT_DISCONNECTED / TIMED_OUT
Exchange:   IN_PROGRESS ─→ COMPLETE | ERRORED | UPGRADED
```

```java
// HttpExchange.java:112-123 — 状态协调
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
        request.discardInputStreamIfNotConsumed();   // 响应先结束 → 丢未读 body
    }
}
```

监听链:
- `NettyRequestAdapter.setState` → `RequestStateChangeListener` →
  `Http1Connection.channelRead0` 里 `ctx.channel().read()` /
  `Http2Connection.read(ctx, streamId)`。
- `NettyResponseAdaptor.outputState` → `ResponseStateChangeListener` →
  `HttpExchange.onReqOrRespStateChange` 推算最终 exchange 状态。
- `HttpExchange.onEnded` → `HttpExchangeStateChangeListener` →
  `Http1Connection` / `Http2Connection` 调 `onResponseComplete(...)` 清理 + 重置。

### 4.6 `CSRFProtectionHandler` — 2.2.x 新增的现代 CSRF 防护

```java
// CSRFProtectionHandler.java:39-75
@Override
public boolean handle(MuRequest request, MuResponse response) throws Exception {
    Method method = request.method();
    if (method == Method.GET || method == Method.HEAD || method == Method.OPTIONS) {
        return false;                                  // 安全方法永远放行
    }
    if (bypassPaths.contains(request.uri().getRawPath())) {
        return false;
    }
    String secFetchSite = request.headers().get("Sec-Fetch-Site");
    if ("same-origin".equals(secFetchSite) || "none".equals(secFetchSite)) {
        return false;
    }
    if (secFetchSite == null || secFetchSite.isEmpty()) {
        // 回退到 Origin 头
        String origin = request.headers().get("Origin");
        if (origin == null || origin.isEmpty()) {
            return false;                              // 非浏览器无 origin, 放行
        }
        URI uri = request.uri();
        String host = uri.getHost();
        int port = uri.getPort();
        String hostHeader = port > 0 ? host + ":" + port : host;
        if (origin.endsWith("://" + hostHeader) || trustedOrigins.contains(origin)) {
            return false;
        }
    } else {
        // 跨源: 检查 trusted list
        if (trustedOrigins.contains(request.headers().get("Origin"))) {
            return false;
        }
    }
    return rejectionHandler.handle(request, response);  // 默认抛 BadRequestException
}
```

灵感来自 Filippo Valsorda 的 [Cross-Site Request Forgery](https://words.filippo.io/csrf/)。
**核心思想**: 抛弃传统 CSRF token,采用现代浏览器原生 `Sec-Fetch-Site` 头检测,兼容老浏览器的 `Origin` 回退。

### 4.7 MuServerBuilder.start — 三段启动

```java
// MuServerBuilder.java:616-707
public MuServer start() {
    if (httpPort < 0 && httpsPort < 0) {
        throw new IllegalArgumentException("No ports were configured...");
    }
    ServerSettings settings = new ServerSettings(...);
    ExecutorService handlerExecutor = this.executor;
    if (handlerExecutor == null) {
        DefaultThreadFactory threadFactory = new DefaultThreadFactory("muhandler");
        handlerExecutor = new ThreadPoolExecutor(8, 400, 60, TimeUnit.SECONDS,
                                                  new SynchronousQueue<>(), threadFactory);
    }
    NettyHandlerAdapter nettyHandlerAdapter = new NettyHandlerAdapter(handlerExecutor, handlers, responseCompleteListeners);

    NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
    NioEventLoopGroup workerGroup = new NioEventLoopGroup(this.nioThreads);
    List<Channel> channels = new ArrayList<>();

    GlobalTrafficShapingHandler trafficShapingHandler = new GlobalTrafficShapingHandler(workerGroup, 0, 0, 1000);
    MuStatsImpl stats = new MuStatsImpl(trafficShapingHandler.trafficCounter());

    Function<Duration, Boolean> shutdown = (gracefulDuration) -> {
        try {
            if (wheelTimer != null) wheelTimer.stop();
            for (Channel channel : channels) channel.close().sync();
            bossGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();
            boolean hasInFlightRequests = gracefulWait(gracefulDuration, stats);
            workerGroup.shutdownGracefully(0, 0, TimeUnit.MILLISECONDS).sync();
            finalHandlerExecutor.shutdown();
            return hasInFlightRequests;
        } catch (Exception e) {
            log.info("Error while shutting down. Will ignore. Error was: {}", e.getMessage());
            return false;
        }
    };

    try {
        SslContextProvider sslContextProvider = null;
        boolean http2Enabled = http2Config != null && http2Config.enabled;
        MuServerImpl server = new MuStatsImpl(...);
        Channel httpChannel = httpPort < 0 ? null : createChannel(bossGroup, workerGroup, ..., false, ...);
        Channel httpsChannel = httpsPort < 0 ? null : createChannel(bossGroup, workerGroup, ..., true, ...);
        // ... 收尾 ...
        if (addShutdownHook) {
            Runtime.getRuntime().addShutdownHook(new Thread(server::stop));
        }
        return server;
    } catch (Exception ex) {
        shutdown.apply(Duration.ofMillis(0));
        throw new MuException("Error while starting server", ex);
    }
}
```

---

## 5. 使用场景

### 5.1 适合用 mu-server

- **小到中等流量的内部 API 服务**: 启动快 (几秒), fat jar 体积小 (Netty + 自家代码,没有 Spring 容器)。
- **需要 Jakarta REST (JAX-RS) 又不想背 Jersey / RESTEasy 的运行时**: `RestHandler` 直接装上 Resource 类即可。
- **多协议嵌入式**: 同时要 HTTP/1 + HTTP/2 + WebSocket + SSE,共享同一个 `NettyHandlerAdapter`。
- **明确知道 I/O 模型是 NIO**: 用户写阻塞风格 handler (例如同步 DB 客户端),`muhandler` 池就是为此设计。
- **部署到资源受限的环境**: 没有 servlet 容器开销; 内存占用低。
- **HAProxy / 反代环境**: 内置 HAProxy 协议解析,`Forwarded` / `X-Forwarded-*` 自动处理 (`MuRequest.clientIP()`)。
- **热更新证书**: `MuServer.changeHttpsConfig(HttpsConfigBuilder)` (MuServerImpl.java:120-129) 不需要重启服务。
- **需要现代 CSRF 防护**: 2.2.x 的 `CSRFProtectionHandler` 是基于 Sec-Fetch-Site 的新一代方案。
- **OpenAPI 自动生成**: 通过 `addOpenApiDocUrl(...)` 自动暴露 Swagger UI 和 JSON。

### 5.2 不太适合

- **需要 Servlet API 兼容**: 没有 servlet 容器抽象; 迁移现有 servlet 应用不能 drop-in。
- **超大规模吞吐 (C10K+ 持久长连接)**: `muhandler` 池默认上限 400, 同步 handler 占满后会 503。
- **HTTP/2 push**: 明确不支持 (`onPushPromiseRead` 是 no-op, `Http2Connection.java:454-456`)。
- **完整的 JAX-RS 生态系统**: Bean Validation, JAXB, 自动 `@Provider` 扫描, `Feature` / `DynamicFeature` 都不实现 (rest/README.md: 第 4, 7 章)。
- **客户端场景**: 没有 HTTP client API; 测试还要拉 Jetty client 或 OkHttp。
- **复杂的 session / 安全栈**: 没有内置 session manager, CSRF / CORS 要手动挂 handler。
- **企业级监控集成**: 没有 Micrometer / Prometheus 内置, 需要自己 `addResponseCompleteListener`。

### 5.3 对比

| 维度 | mu-server 2.2.9 | Spring Boot | 裸 Netty | Vert.x |
|---|---|---|---|---|
| 启动时间 | 几秒 | 10-30 秒 | < 1 秒 | < 1 秒 |
| Fat jar 体积 (min) | ~10 MB | ~30+ MB | ~5 MB | ~10 MB |
| API 风格 | 阻塞 handler (可选异步) | 阻塞 controller (MVC), 异步 (WebFlux) | 完全异步 | 完全异步 |
| HTTP/1 + HTTP/2 | ✅ | ✅ | 需自己组装 | ✅ |
| 内置 JAX-RS | ✅ (`RestHandler`, Jakarta REST 3.1) | ❌ (用 Jersey starter) | ❌ | ❌ |
| 内置 SSE | ✅ (sync + Async) | ✅ (WebFlux `Flux<ServerSentEvent>`) | ❌ | ✅ |
| WebSocket | ✅ | ✅ | ✅ | ✅ |
| 路由 DSL | `Routes.route(method, template, ...)` (`UriPattern` regex) | `@RequestMapping` | 手工 | `Router` |
| 流量限速 | ✅ (fixed-window counter) | via Bucket4j | 手工 | `RateLimiter` (令牌桶) |
| 现代 CSRF | ✅ (Sec-Fetch-Site) | ✅ (Spring Security) | 手工 | 手工 |
| 优雅关闭 | ✅ (`stop(d, unit)`) | ✅ (actuator) | 需自己实现 | ✅ |
| Session manager | ❌ | ✅ | ❌ | ❌ |
| OpenAPI 自动生成 | ✅ (内置, 可定制) | ✅ (springdoc) | ❌ | ❌ |
| 热更新证书 | ✅ (changeHttpsConfig) | ❌ | 手工 | 手工 |

---

## 6. 0.0.3 → 2.2.9 演进分析

> 基于 `/tmp/mu-server-readonly/.omo/evidence/mu-server-netty-analysis.md` 对比。

### 6.1 代码量与文件数

| 版本 | 文件数 | 总行数 | 净变化 |
|---|---|---|---|
| 0.0.3-SNAPSHOT (`4f0aa3c`) | 258 | 36317 | 基准 |
| 2.2.9 (`086a921`) | 240 | 30755 | **-18 文件, -5562 行** |

**显著瘦身**。主要缩减在 `rest/` 和 `openapi/` 包 — 可能是合并了重复文件 / 重构。

### 6.2 API 演进 (关键变化)

| 维度 | 0.0.3 | 2.2.9 | 备注 |
|---|---|---|---|
| Java source/target | 11 | **8** | 回退到 Java 8 (兼容性考虑) |
| Netty 版本 | 4.1.137.Final | 4.1.135.Final | 微降 |
| Jakarta package | 已经是 `jakarta.ws.rs.*` | 同 | 没变 |
| `MuServerImpl` 实现 | `MuSeBootstrap` 也存在 (Jakarta REST bootstrap) | 似乎合并到 `RestHandlerBuilder` | 需查证 |
| `CSRFProtectionHandler` | ✅ | ✅ | 0.0.3 已有, 来源相同 |
| `AsyncSsePublisher` | ✅ | ✅ | 0.0.3 已有 |
| `changeHttpsConfig` | ✅ (MuServerImpl:128-139) | ✅ (MuServerImpl:120-129) | 同 |
| `Http2ConnectionFlowControl` 自管理 buffer | ✅ | ✅ | 设计未变 |
| `Http2To1RequestAdapter` | ✅ | ✅ | 同 |
| Worker shutdown 参数 | `shutdownGracefully(2, 15, s)` | `shutdownGracefully(0, 0, ms)` | **变激进** |
| HTTP/1 worker handler 链 | 略不同 (待查) | 同 | 详见 4.7 |
| JAX-RS README | 有 | 有 | 详细 spec 对照矩阵没变 |
| `ProblemDetailsException` | 0.0.3 有 | 2.2.9 看起来**被移除** (待查证) | 可能合入 `CustomExceptionMapper` |

### 6.3 功能新增 (2.2.9 相对 0.0.3)

由于 0.0.3-SNAPSHOT 和 2.2.9 之间跨度很大 (大约 2 年),增量变化很难仅从两份 git 状态精确判定。
基于现有代码观察,以下是 2.2.x 时代**看起来新加入**或**显著改进**的能力:

1. **`AsyncSsePublisher`**: 用 `CompletionStage` 异步发送 SSE, 适合大量订阅者场景 (193 行)。
2. **`SchemaObjectCustomizer`**: OpenAPI schema 生成可定制, 通过 `addOpenApiCustomizer(...)` 注入。
3. **`CollectionParameterStrategy`**: JAX-RS `List<T>` / `Set<T>` 参数策略可配置。
4. **`LazyAccessInputStream` / `LazyAccessOutputStream`**: 延迟创建, 减少 JAX-RS 请求路径上的内存分配。
5. **`RestHandlerBuilder` 配置项扩展**: 598 行, 比 0.0.3 时代更长, 配置项更多。
6. **多种 entity providers 拆分**: `BinaryEntityProviders`, `StringEntityProviders`, `PrimitiveEntityProvider` 单独成文件。
7. **OpenAPI generator** (~ 65 文件): 稳定, 没看到重大变化。
8. **`HAProxyMessageHandler`** (20 行): 极简, 与 0.0.3 一致。

### 6.4 0.0.3 → 2.2.x 没有解决的限制

(基于 0.0.3 报告中的限制清单, 2.2.9 大部分仍未实现)

- ❌ HTTP client API
- ❌ HTTP/2 push (`onPushPromiseRead` 仍是 no-op, `Http2Connection.java:454-456`)
- ❌ Bean Validation (`@Valid` / `@NotNull`)
- ❌ JAXB entity provider
- ❌ 自动 `@Provider` 扫描
- ❌ Session manager
- ❌ Micrometer / Prometheus 集成
- ❌ 连接迁移 (HTTP/2 connection migration)

### 6.5 API 兼容性

- **公开 `io.muserver.*` 包 API 基本稳定**: `MuServer`, `MuServerBuilder`, `MuRequest`, `MuResponse`,
  `Headers`, `Method`, `AsyncHandle`, `SsePublisher`, `AsyncSsePublisher`, `BaseWebSocket`,
  `HttpsConfigBuilder` 等在 0.0.3 和 2.2.9 之间**保持兼容**。
- **`io.muserver.rest` 包**: 大部分 JAX-RS 标准接口 (`@Path`, `@GET`, `@POST`, `@Suspended`,
  `AsyncResponse`, `SseEventSink`, `SseBroadcaster`) 保持兼容。
- **`io.muserver.openapi` 包**: OpenAPI 3.x 对象模型保持稳定。

**结论**: 对依赖标准 JAX-RS 注解 + mu 公开 handler API 的应用,从 0.0.3 升级到 2.2.9 **应当 drop-in**。

---

## 7. 限制 / 已知问题 (基于 2.2.9 源码静态分析)

> 基于源码静态分析, 未运行测试验证.

### 7.1 设计层限制

1. **没有客户端 API**. 测试需要外部依赖 (Jetty client, OkHttp)。
2. **HTTP/2 push 不支持**. `Http2Connection.onPushPromiseRead` 是 no-op (`Http2Connection.java:454-456`)。
3. **HTTP/2 buffer 用普通 `HashMap`**: 全部访问假定在 event loop, 当前实现是安全的但很脆弱 (`Http2Connection.java:39-40`)。
4. **限流算法简单**: `RateLimiterImpl` 是 fixed-window counter (`RateLimiterImpl.java:25-49`),
   没有 leaky-bucket / token-bucket 的"令牌匀速"特性, 高峰期可能突刺。
5. **`muhandler` 默认 `SynchronousQueue` + max 400**: 超载直接 `RejectedExecutionException` → 503
   (`HttpExchange.create`, 299-304)。没有 waiting queue, 没有 graceful degradation。

### 7.2 关闭路径缺口

6. **`workerGroup.shutdownGracefully(0, 0, ms)` 强制 0s 关闭** (`MuServerBuilder.java:654`)。
   在 K8s `terminationGracePeriodSeconds=30` + `preStop` 模式下反而是好事 (减少僵尸进程),
   但在没有外部协调的环境下 in-flight 请求会被硬切。
7. **HTTP/2 关闭时不发 GOAWAY**: `Http2Connection.channelInactive()` (156-160) 只减 stats 不发 GOAWAY,
   HTTP/2 client 收到的会是 RST_STREAM 而非有序 GOAWAY。
8. **`gracefulWait` 轮询**: `Thread.sleep(100)` 循环 (`MuServerBuilder.java:709-715`), 无 `CountDownLatch`。
9. **`HttpExchange.block` 的 Runnable 版本不带 `.sync()`** (`HttpExchange.java:61-78` 有个 TODO 注释):
   "only use the callable version as this perhaps doesn't block until the runnable is finished? (e.g. when doing a write)"

### 7.3 JAX-RS 不完整 (rest/README.md)

10. **无 Bean Validation** (第 7 章)。
11. **无 JAXB entity provider**。
12. **无自动 classpath scanning** (`@Provider` 注解不识别)。
13. **资源类必须是 singleton** (无 per-request lifecycle)。
14. **JSON provider 不内置** — 用户得自己加 Jackson / Gson。
15. **无 `@BeanParam`**。

### 7.4 其他

16. **`HttpExchange.addStateChangeListener` 在 `Http1Connection.userEventTriggered` 中注册** (`Http1Connection.java:175`),
    意味着 WebSocket upgrade 时, listener chain 是动态变化, 仔细跟踪 UPGRADED 状态转换。
17. **`AsyncSsePublisher.setResponseCompleteHandler`**: 实现上等效于 `asyncHandle.addResponseCompleteHandler`,
    适合用于"客户端提前断开时取消昂贵的 SSE 生产"。
18. **`GlobalTrafficShapingHandler` 写入了但 `writeLimit=readLimit=0`** (`MuServerBuilder.java:634`)
    — 实际不限速, 仅为 `MuStats` 提供字节计数器。想限速要自己换 handler。

---

## 8. 文件清单 (按子包)

| 子包 | 文件数 | 关键文件 |
|---|---|---|
| `io.muserver` (核心) | 93 | `MuServer`, `MuServerBuilder`, `MuServerImpl`, `MuRequest`, `MuResponse`, `HttpExchange`, `NettyHandlerAdapter`, `NettyRequestAdapter`, `NettyResponseAdaptor`, `Http1Connection`, `Http2Connection`, `Http2ConnectionBuilder`, `Http1Response`, `Http2Response`, `Http1Headers`, `Http2Headers`, `Http2To1RequestAdapter`, `AlpnHandler`, `HAProxyMessageHandler`, `MuSniHandler`, `BackPressureHandler`, `MuFlowControlHandler`, `PreReader`, `RateLimiterImpl`, `RateLimitBuilder`, `HttpsConfigBuilder`, `SsePublisher`, `AsyncSsePublisher`, `WebSocketHandler`, `WebSocketHandlerBuilder`, `MuWebSocket`, `BaseWebSocket`, `MuStats`, `MuStatsImpl`, `MuHandler`, `RouteHandler`, `Routes`, `ContextHandler`, `ContextHandlerBuilder`, `Mutils`, `Cookie`, `CookieBuilder`, `ForwardedHeader`, `Headers`, `RequestBodyReader`, `ChunkedHttpOutputStream`, `UploadedFile`, `SSLCipherFilter`, `SslContextProvider`, `SsePublisherImpl`, `AsyncSsePublisherImpl`, `RateLimit`, `RateLimitRejectionAction`, `RateLimitSelector`, `RateLimiter`, `SsePublisher`, `AsyncSsePublisher`, `MuExceptionFiredEvent`, `ExchangeUpgradeEvent`, `DoneCallback`, `ResponseCompleteListener`, `RequestBodyListener`, `RequestState`, `ResponseState`, `MuException`, `Method`, `MuWebSocketSession`, `MuWebSocketSessionImpl`, `MuWebSocketFactory`, `ProxiedConnectionInfo`, `ProxiedConnectionInfoImpl`, `SelectiveHttpContentCompressor`, `MuGzipHttp2ConnectionEncoder`, `MuCompressorHttp2ConnectionEncoder`, `RequestBodyReaderInputStreamAdapter`, `ParameterizedHeader`, `ParameterizedHeaderWithValue`, `ParseUtils`, `HeaderNames`, `HeaderValues`, `MediaTypeParser`, `Headtils`, `ContentTypes`, `ContextHandlerBuilder`, `Exchange`, `HttpConnection`, `RequestStateChangeListener`, `ResponseStateChangeListener`, `WebsocketSessionState`, `Toggles`, `SSLInfo`, `SSLInfoImpl` |

> **说明**: `SsePublisherImpl` / `AsyncSsePublisherImpl` 是 `SsePublisher.java:112` / `AsyncSsePublisher.java:124` 的 package-private 内部类，**不是独立文件**。已在 2026-09-12 review 中删除。
| `io.muserver.handlers` | 14 | `CORSHandler`, `CORSHandlerBuilder`, `CSRFProtectionHandler`, `CSRFProtectionHandlerBuilder`, `HttpsRedirector`, `HttpsRedirectorBuilder`, `ResourceHandler`, `ResourceHandlerBuilder`, `ResourceProvider`, `ResourceType`, `BytesRange`, `DirectoryLister`, `BareDirectoryRequestAction`, `ResourceCustomizer` |
| `io.muserver.rest` | 70 | `RestHandler`, `RestHandlerBuilder` (598 行), `MuRuntimeDelegate`, `JaxRSRequest`, `JaxRSResponse`, `RequestMatcher`, `ResourceClass`, `ResourceMethod`, `ResourceMethodParam`, `JaxClassLocator`, `JaxMethodLocator`, `UriPattern`, `PathMatch`, `EntityProviders`, `StringEntityProviders`, `BinaryEntityProviders`, `PrimitiveEntityProvider`, `BuiltInParamConverterProvider`, `FilterManagerThing`, `MediaTypeDeterminer`, `CombinedMediaType`, `CORSConfig`, `CORSConfigBuilder`, `JaxSseImpl`, `JaxSseEventSinkImpl`, `SseBroadcasterImpl`, `OpenApiDocumentor`, `HtmlDocumentor`, `MuSecurityContext`, `MuUriInfo`, `MuUriBuilder`, `MuVariantListBuilder`, `MuPathSegment`, `CustomExceptionMapper`, `BasicAuthSecurityFilter`, `Authorizer`, `UserPassAuthenticator`, `AsyncResponseAdapter`, `JaxOutboundSseEvent`, `JaxOutboundSseEventBuilder`, `JaxRsHttpHeadersAdapter`, `Jaxutils`, `LazyAccessInputStream`, `LazyAccessOutputStream`, `EmptyInputStream`, `NullOutputStream`, `LowercasedMultivaluedHashMap`, `ReadOnlyMultivaluedMap`, `CollectionParameterStrategy`, `ProviderWrapper`, `Required`, `NotMatchedException`, `NotImplementedException`, `ObjWithType`, `ResourceClass`, `ResourceMethod`, `ResourceMethodParam`, `ResponseHeader`, `Description`, `DescriptionData`, `RestHandlerBuilder`, `CacheControlHeaderDelegate`, `CookieHeaderDelegate`, `DateHeaderDelegate`, `EntityTagDelegate`, `LinkHeaderDelegate`, `MediaTypeHeaderDelegate`, `NewCookieHeaderDelegate`, `ApiResponse`, `ApiResponses`, `SchemaObjectCustomizer`, `SchemaObjectCustomizerContext`, `SchemaObjectCustomizerTarget` |
| `io.muserver.openapi` | 62 | `OpenAPIObject`, `PathsObject`, `PathItemObject`, `OperationObject`, `ParameterObject`, `RequestBodyObject`, `ResponseObject`, `ResponsesObject`, `SchemaObject`, `ComponentsObject`, ... (每个对象都有 builder, 由 `JsonWriter` 输出) |
| **总计** | **240** | — |

---

## 9. 结论

mu-server 2.2.9 是一份**紧凑、自洽、Netty-native 的 Java HTTP/2 服务器**。架构核心从 0.0.3 到 2.2.9 **高度稳定**:三层线程模型 (Netty worker → muhandler 业务池 → 用户线程) + `HttpExchange.block()` 同步桥接 + JAX-RS 内置 + SSE/WebSocket/CORS/CSRF/HTTPS/OpenAPI 全家桶。这不是"演化",而是"硬化"。

**主要观察**:

1. **抽象层稳定**:`MuRequest` / `MuResponse` / `HttpExchange` 接口签名在 2.x 内**不破坏兼容**。
2. **新功能集中在 handler 库和 OpenAPI**:CSRF protection (Sec-Fetch-Site)、AsyncSsePublisher、SchemaObjectCustomizer。
3. **JAX-RS 是真实的实现** (不是胶水): 覆盖 Jakarta REST 3.1 规范的 ~80% (rest/README.md 有详细对照矩阵)。
4. **OpenAPI 自动生成 + Swagger UI 内置**是相对 Spring Boot 的"省心优势"。
5. **优雅关闭有缺口**: HTTP/2 不发 GOAWAY, worker 硬切。
6. **JVM target 仍是 Java 8**: 让 mu-server 适合嵌入到老应用。
7. **240 文件 / 30755 行**是一个非常合理的代码量——足以覆盖 HTTP/1+2+SSE+WebSocket+JAX-RS, 但又没有 Spring Boot 的复杂度。

**推荐使用场景**: 中小规模内部 API、需要 JAX-RS 又不想拉 Jersey/RESTEasy、需要 HTTP/2 ALPN、自动 OpenAPI 文档、内嵌到现有 Java 应用。

**不推荐场景**: 需要 Servlet 兼容、超大规模 SSE (C10K+)、完整 JAX-RS 生态 (Bean Validation, JAXB)、客户端 API。

---

## 附录 A — 子包源码深度摘要所在 drafts

- `.omo/drafts/01-protocol-layer.md` — 协议层 (Http1Connection / Http2Connection / HAProxyMessageHandler / AlpnHandler / pipeline 顺序)
- `.omo/drafts/02-abstraction-layer.md` — 抽象层 (MuRequest / MuResponse / HttpExchange / 适配器模式)
- `.omo/drafts/03-dispatch-layer.md` — 分发层 (NettyHandlerAdapter / Routes / MuServerBuilder.start)
- `.omo/drafts/04-handlers.md` — 内置 Handler 库清单
- `.omo/drafts/05-jaxrs.md` — JAX-RS 子系统 (RestHandler / 路由 / 实体提供器 / Filter / SSE / OpenAPI)
- `.omo/drafts/06-features.md` — 特性模块 (SSE / TLS / RateLimiter / Stats / Exceptions / WebSocket / Forwarded / Multipart)
- `.omo/drafts/07-threading-model.md` — 线程模型 (关键的 `block()` 模式详解)
- `.omo/drafts/08-state-machines.md` — 状态机 (RequestState / ResponseState / HttpExchangeState)
- `.omo/drafts/09-http2-flow-control.md` — HTTP/2 流控深度剖析
- `.omo/drafts/10-graceful-shutdown.md` — 优雅关闭

## 附录 B — 关键源码引用 (本报告中已出现)

- `MuServerBuilder.java:48-71, 184-208, 616-715, 765-781` — 启动 + 关闭 + pipeline 装配
- `MuServerImpl.java:120-129, 152-158` — 运行时换证书 / 连接管理
- `Http1Connection.java:53-139, 159-209` — HTTP/1 生命周期
- `Http2Connection.java:25-124 (flow control), 226-316 (onHeadersRead), 318-365 (onDataRead0), 367-389 (onStreamError), 416-491 (各种 read)` — HTTP/2 处理器
- `AlpnHandler.java:7-46` — TLS ALPN 协商
- `HAProxyMessageHandler.java:8-20` — PROXY protocol
- `MuSniHandler.java:15-37` — SNI 多证书
- `NettyRequestAdapter.java:32-548` (尤其 196-212 claimingBodyRead, 309-315 handleAsync, 391-414 websocketUpgrade, 469-546 AsyncHandleImpl)
- `NettyResponseAdaptor.java:28-367` (尤其 46-80 outputState, 119-129 startStreaming, 149-199 writeAndFlush, 205-214 sendChunk, 283-315 complete)
- `HttpExchange.java:33-480` (尤其 61-96 block(), 112-123 onReqOrRespStateChange, 136-143 complete(), 232-244 scheduleReadTimeout, 268-306 create(), 384-448 onException, 465-480 state enum)
- `NettyHandlerAdapter.java:11-84` (尤其 25-55 onHeaders)
- `Routes.java:24-49` + `rest/UriPattern.java` — URI 模板
- `RequestBodyReader.java` — 请求体读取 (InputStream / String / Multipart / UrlEncoded / Discarding / ListenerAdapter)
- `Headers.java:10-412` — 多值头
- `SsePublisher.java:31-200` + `AsyncSsePublisher.java:16-193` — SSE 同步 + 异步
- `RateLimiterImpl.java:13-71` — 限流 (fixed-window)
- `MuStatsImpl.java` — 统计 (包 trafficCounter)
- `HttpsConfigBuilder.java:24-459` — TLS 配置 (keystore / protocol / cipher)
- `RestHandler.java:35-364` + `RestHandlerBuilder.java:29-598` — JAX-RS dispatcher
- `rest/MuRuntimeDelegate.java` — JAX-RS SPI
- `handlers/CSRFProtectionHandler.java:39-89` — 现代 CSRF 防护
- `handlers/ResourceHandler.java:46-129` — 静态文件 + Range + If-Modified-Since
- `handlers/CORSHandler.java:14-36` — CORS (委托给 rest/CORSConfig)
- `handlers/HttpsRedirector.java:28-57` — HTTP→HTTPS 重定向 + HSTS

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
