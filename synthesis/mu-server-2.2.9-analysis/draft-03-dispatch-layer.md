---
title: "mu-server 2.2.9 分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)"
category: synthesis
tags: [java, mu-server, dispatch, builder-pattern, routing, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "核心调度器 NettyHandlerAdapter 在独立 ExecutorService 跑 user handler, MuServerBuilder fluent API (40KB), 路由系统 PathMatch/Matcher"
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

# 03 - 分发层 (Dispatch Layer)

> 把 Netty `channelRead` 事件转换为 `MuHandler.handle()` 调用。
> 这层是 mu-server 的"glue code", 决定了请求的整个生命周期。

## 文件清单

| 文件 | 角色 | 行数 |
|---|---|---|
| `NettyHandlerAdapter.java` | **核心**: handler 链 + executor 调度 | 84 |
| `Routes.java` | 路由 (method + URI template) | 52 |
| `RouteHandler.java` | 路由 handler 函数式接口 | - |
| `MuHandler.java` | handler 接口 | - |
| `MuHandlerBuilder.java` | handler 构造器 marker | - |
| `MuServer.java` | 公开的服务接口 | - |
| `MuServerImpl.java` | 服务实现 | 163 |
| `MuServerBuilder.java` | **fluent builder**, 启动入口 | 808 |
| `ServerSettings.java` | 不可变 settings 值对象 | - |
| `ContextHandler.java` / `ContextHandlerBuilder.java` | URL 前缀改写 | - |
| `ResponseCompleteListener.java` | 响应完成回调 | - |
| `UnhandledExceptionHandler.java` | 全局异常处理 | - |
| `Toggles.java` | (兼容性) | - |

## NettyHandlerAdapter (核心, 84 行)

这是分发层的**最短也最关键**的类:
```java
// NettyHandlerAdapter.java:25-55
void onHeaders(HttpExchange muCtx) {
    executor.execute(() -> {                        // [1] 切到业务线程池
        if (muCtx.state().endState()) {
            return;
        }
        NettyRequestAdapter request = muCtx.request;
        NettyResponseAdaptor response = muCtx.response;
        try {
            boolean handled = false;
            for (MuHandler muHandler : muHandlers) {   // [2] 顺序遍历
                handled = muHandler.handle(request, response);
                if (handled) {
                    break;
                }
                if (request.isAsync()) {
                    throw new IllegalStateException(muHandler.getClass() + " returned false however this is not allowed after starting to handle a request asynchronously.");
                }
            }
            if (!handled) {
                throw new NotFoundException();          // [3] 无 handler 匹配 → 404
            }
            if (!request.isAsync() && !response.outputState().endState()) {
                response.flushAndCloseOutputStream();   // [4] 同步模式: 自动 flush + close
                muCtx.block(muCtx::complete);            // [5] 等响应完全写出
            }
        } catch (Throwable ex) {
            useCustomExceptionHandlerOrFireIt(muCtx, ex);  // [6] 异常 → 自定义 handler 或 500
        }
    });
}
```

**模式**:
1. **线程切换**: 通过 `executor.execute()` 把工作从 Netty event loop 切到业务线程池。
   默认线程池是 `ThreadPoolExecutor(8, 400, 60s, SynchronousQueue, "muhandler")` (MuServerBuilder:626)。
2. **handler 链**: 按注册顺序遍历, 第一个返回 `true` 的胜出 (类似 Express/Spring filter chain)。
3. **同步 vs 异步**:
   - 同步: handler 返回 → 自动 flush+close → block 到写出完成
   - 异步: handler `handleAsync().complete()` 后才会触发结束
4. **异常**: 抛到 `useCustomExceptionHandlerOrFireIt()` (57-69): 用户装了 `UnhandledExceptionHandler` 就给它,
   否则 `exchange.fireException(ex)` 把异常作为 userEvent 走 `Http1Connection.exceptionCaught` → 走标准 500 流程。

## 路由

`Routes.route(Method, uriTemplate, handler)` (52 行):
- 把 `uriTemplate` 编译成 `UriPattern` (regex + 参数) — 实际工作在 `rest/UriPattern.java`
- 返回一个**匿名 `MuHandler`** 在链中: 检查 method + 调 `uriPattern.matcher(relativePath)` 是否 `fullyMatches()`, 是就调业务 handler 并返回 true。

支持的 URI 模板语法 (rest/UriPattern.java):
- `/foo/{bar}` → 单段捕获
- `/foo/{bar:[0-9]+}` → 带正则约束的捕获
- 完整 spec 见 `rest/UriPattern.java`

`RouteHandler` 接口签名:
```java
void handle(MuRequest req, MuResponse resp, Map<String, String> pathParams) throws Exception;
```

## ContextHandler

`ContextHandler` 是把一个或多个 handler 包装到 URL 前缀下的工具:
- 注册时把 `/api/v1` → handler 链
- 匹配后 `muReq.addContext("/api/v1")` → `contextPath()` 返回 `/api/v1`, `relativePath()` 返回剥掉前缀的部分
- 典型用法: REST API 部署在 `/api/v1`

## MuServerImpl (163 行)

实现 `MuServer` 接口, 持有:
- `MuStatsImpl stats` (全局 stats, 含流量整形)
- `Set<HttpConnection> connections` (活跃连接, `ConcurrentHashMap.newKeySet()`)
- `ServerSettings settings` (不可变)
- `UnhandledExceptionHandler unhandledExceptionHandler`
- `SslContextProvider sslContextProvider`

提供:
- `stop(duration, unit)` 走 `shutdown.apply(Duration)`
- `uri()` / `httpUri()` / `httpsUri()`
- `activeConnections()` / `stats()` / `address()`
- 配置只读访问: `minimumGzipSize()`, `maxRequestHeadersSize()`, `requestIdleTimeoutMillis()`, `maxRequestSize()`, `maxUrlSize()`, `gzipEnabled()`, `mimeTypesToGzip()`
- `changeHttpsConfig(HttpsConfigBuilder)` 运行时换证书 (`SslContextProvider.set(newNettyCtx)`)
- `sslInfo()` / `rateLimiters()`

## MuServerBuilder (808 行)

启动流程 (`MuServerBuilder.start()`, 616-707):
1. 校验 http/https 至少一个 ≥ 0
2. `ServerSettings` 不可变
3. `handlerExecutor` 默认创建 (8-400 弹性线程池, SynchronousQueue, "muhandler" 命名)
4. `NettyHandlerAdapter` 把 handler + executor 包起来
5. `NioEventLoopGroup(1)` (boss, 只接 accept) + `NioEventLoopGroup(nioThreads)` (worker)
6. `GlobalTrafficShapingHandler` (读 0, 写 0, 检查间隔 1000ms, 用于 stats 统计)
7. `MuStatsImpl` 用 `trafficCounter()` 包装
8. 关闭钩子 `shutdown` = `Function<Duration, Boolean>`:
   - 停 `HashedWheelTimer`
   - `channel.close().sync()` 关闭监听 socket
   - `bossGroup.shutdownGracefully(0, 0, ms).sync()`
   - `gracefulWait(gracefulDuration, stats)` 轮询 100ms 等 activeRequests 清空
   - `workerGroup.shutdownGracefully(0, 0, ms).sync()`
   - `finalHandlerExecutor.shutdown()`
   - 返回 `hasInFlightRequests` 标识是否"完全空闲关闭"
9. 创建 `MuServerImpl`
10. `createChannel()` (723-763):
    - HTTP 明文 (port >= 0): `setupHttp1Pipeline` 直装
    - HTTPS (port >= 0): 默认 `HttpsConfigBuilder.unsignedLocalhost()` 自签证书, 装配 SNI/ALPN
11. 注册 `addShutdownHook` (可选)
12. 返回 `MuServer` 实例

**关键默认**:
- `httpPort = -1`, `httpsPort = -1` (未配置)
- `nioThreads = min(16, cpus * 2)`
- `idleTimeoutMills = 10min`
- `requestReadTimeoutMillis = 2min`
- `maxRequestSize = 24 * 1024 * 1024` (24 MB)
- `minGzipSize = 1400`, gzip 默认开
- `WriteBufferWaterMark = Netty DEFAULT`
- `haProxyProtocolEnabled = false`

## ResponseCompleteListener

在 `NettyHandlerAdapter.onResponseComplete()` (71-83) 中, 在请求结束后回调:
```java
void onResponseComplete(ResponseInfo info, MuStatsImpl serverStats, MuStatsImpl connectionStats) {
    connectionStats.onRequestEnded(info.request());
    serverStats.onRequestEnded(info.request());
    if (completeListeners != null) {
        for (ResponseCompleteListener listener : completeListeners) {
            try {
                listener.onComplete(info);
            } catch (Exception e) {
                log.error("Error from completion listener", e);
            }
        }
    }
}
```
用于打点 / logging / trace。

## ServerSettings

不可变值对象, 持有:
- `minimumGzipSize`, `maxHeadersSize`, `requestReadTimeoutMillis`
- `maxRequestSize`, `maxUrlSize`
- `gzipEnabled`, `mimeTypesToGzip`
- `rateLimiters` (List<RateLimiterImpl>)

## 设计观察

1. **handler 链是简单的线性链**, 没有 Spring 那种 `@Order` / priority, 顺序就是注册顺序。
2. **业务代码全部跑在业务线程池** (默认 `muhandler-1..N`), 不在 Netty event loop。
3. **同步响应自动 flush**——这是用户友好性: 写完不用调 complete。但这也意味着 handler 阻塞会阻塞 worker 线程。
4. **JAX-RS handler 是另一个 `MuHandler`** (RestHandler), 可以跟普通路由共存 (但优先级按注册顺序)。
5. **优雅关闭**: 通过 `stats.activeRequests()` 监控, 100ms 轮询, 超时后强制关闭 worker group。

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
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
