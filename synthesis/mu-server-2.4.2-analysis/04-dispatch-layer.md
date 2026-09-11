---
title: "§4 分发层"
category: synthesis
tags: [java, netty, mu-server, framework, direct-analysis, spark, 2.4.2, sub-page]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "分发层：NettyHandlerAdapter (98 行，核心调度器) + MuServerBuilder (835 行，最大单文件 fluent builder) + MuServer (168) + MuServerImpl (167) + Routes URI 模板路由。"
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


# 分发层 (Dispatch Layer)

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

## 相关笔记

**同目录其他章节**:
- [[01-architecture-overview|§1 整体架构 + 6 层架构图]]
- [[02-protocol-layer|§2 协议层]]
- [[03-abstraction-layer|§3 抽象层]]
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
