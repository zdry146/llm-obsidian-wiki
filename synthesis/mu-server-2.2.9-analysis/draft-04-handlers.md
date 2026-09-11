---
title: "mu-server 2.2.9 内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)"
category: synthesis
tags: [java, mu-server, handlers, cors, csrf, static-resources, 2.2.9]
sources: ["mu-server 2.2.9 @ tag mu-server-2.2.9 (https://github.com/3redronin/mu-server)"]
summary: "CORSHandler / CSRFProtectionHandler / HttpsRedirector / ResourceHandler (静态资源) / DirectoryLister 等开箱即用 handler"
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

# 04 - Handler 库 (Handlers Library)

> `io.muserver.handlers` 包, 一组开箱即用的 `MuHandler` 实现。
> 8 个文件 / 2009 行。

## 文件清单

| 文件 | 角色 |
|---|---|
| `ResourceHandler.java` (175) | 静态文件服务 (类路径 + 文件系统) |
| `ResourceHandlerBuilder.java` (253) | ResourceHandler 构造器 |
| `ResourceProvider.java` | 资源源抽象 (filesystem / classpath / servlet ctx) |
| `ResourceType.java` | MIME 映射 + gzip 决策表 |
| `BytesRange.java` | HTTP Range 请求解析 |
| `DirectoryLister.java` | 目录列表 HTML 渲染 |
| `BareDirectoryRequestAction.java` | 裸目录行为枚举 (REDIRECT / 404) |
| `ResourceCustomizer.java` | 响应头钩子 |
| `CORSHandler.java` (36) | CORS 响应头 (委托给 rest/CORSConfig) |
| `CORSHandlerBuilder.java` | CORSHandler 构造器 |
| `CSRFProtectionHandler.java` (89) | CSRF 防护 (Sec-Fetch-Site / Origin 检查) |
| `CSRFProtectionHandlerBuilder.java` | 构造器 |
| `HttpsRedirector.java` (68) | HTTP → HTTPS 重定向 + HSTS |
| `HttpsRedirectorBuilder.java` | 构造器 |

## ResourceHandler (核心, 175 行)

最常用的 handler。**`handle()` 一次过走完**:
```java
public boolean handle(MuRequest request, MuResponse response) throws IOException {
    String requestPath = request.relativePath();
    if (requestPath.endsWith("/") && defaultFile != null) {
        requestPath += defaultFile;             // /web/ → /web/index.html
    }
    String decodedRelativePath = Mutils.urlDecode(requestPath);

    ResourceProvider provider = resourceProviderFactory.get(decodedRelativePath);
    if (!provider.exists()) {
        if (directoryListingEnabled) {
            provider = resourceProviderFactory.get(Mutils.urlDecode(request.relativePath()));
            if (!provider.isDirectory()) {
                return false;                   // 委托给链中下一个 handler
            }
        } else {
            return false;
        }
    }
    // ...
}
```

支持的特性:
- **默认文件** (默认 `index.html`)
- **目录列表** (默认关, 启用后 HTML 输出)
- **MIME 推断** (`ResourceType.DEFAULT_EXTENSION_MAPPINGS` + 自定义)
- **If-Modified-Since** → 304
- **Range 请求** (单 range → 206 + Content-Range, 多 range 不支持)
- **HEAD** 自动跳过 body
- **`Vary: Accept-Encoding`** 自动加 (NettyResponseAdaptor.getVaryWithAE)
- **`ResourceCustomizer`** 在响应头发出前 hook (用于加 ETag 等)
- **类路径 / 文件系统 / 两者 fallback** (`fileOrClasspath("src/main/resources/web", "/web")`)

**`ResourceProvider`** 抽象三种源:
- `fileBased(Path)` → `Files.exists()`, `Files.size()`, `Files.getLastModifiedTime()`
- `classpathBased(String)` → `ClassLoader.getResource(...)`
- `servletContextBased(...)` → 用 servlet context (通过 ContextHandler)

## CORS (36 行核心)

```java
public class CORSHandler implements MuHandler {
    private final CORSConfig corsConfig;
    private final Set<Method> allowedMethods;

    @Override
    public boolean handle(MuRequest request, MuResponse response) {
        corsConfig.writeHeaders(request, response, allowedMethods);
        return false;                          // 永远不"消耗"请求
    }
}
```

`return false` 意味着 CORS handler **总是把请求传递给链中下一个 handler**, 只是在 response 上加 `Access-Control-*` 头。预检 (OPTIONS) 也由 `CORSConfig.writeHeaders` 处理。

完整配置在 `rest/CORSConfigBuilder`:
- `withAllowedOrigin` / `withAllowedOrigins` / `withAllowAnyOrigin`
- `withAllowedHeaders` / `withExposedHeaders`
- `withAllowedMethods`
- `withAllowCredentials`
- `withMaxAge`

## CSRFProtectionHandler (89 行, 2.2.x 新增)

2.2.x 新增的现代 CSRF 防护 (基于 Sec-Fetch-Site 而非传统 token):

```java
public boolean handle(MuRequest request, MuResponse response) throws Exception {
    Method method = request.method();
    if (method == Method.GET || method == Method.HEAD || method == Method.OPTIONS) {
        return false;                          // 安全方法永远放行
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
            return false;                       // 非浏览器无 origin, 放行
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

配置 (`CSRFProtectionHandlerBuilder`):
- `withTrustedOrigins(String...)`
- `withBypassPaths(String...)`
- `withRejectionHandler(MuHandler)` (默认抛 400)

灵感来自 Filippo Valsorda 的 [Cross-Site Request Forgery](https://words.filippo.io/csrf/)。

## HttpsRedirector (68 行)

```java
public boolean handle(MuRequest request, MuResponse response) throws Exception {
    URI uri = request.uri();
    boolean isHttp = uri.getScheme().equals("http");
    if (!isHttp) {
        // HTTPS: 加 HSTS 头
        if (expireTimeInSeconds > 0) {
            String val = "max-age=" + expireTimeInSeconds;
            if (includeSubDomainsForHSTS) val += "; includeSubDomains";
            if (preload) val += "; preload";
            response.headers().set(HeaderNames.STRICT_TRANSPORT_SECURITY, val);
        }
        return false;                          // 委托给链中下一个 handler
    }
    // HTTP: 重定向到 HTTPS
    int port = httpsPort == 443 ? -1 : httpsPort;
    URI newURI = new URI("https", uri.getUserInfo(), uri.getHost(), port, uri.getPath(), uri.getQuery(), uri.getFragment());
    if (request.method() == Method.GET || request.method() == Method.HEAD) {
        response.status(301);
        response.redirect(newURI);
    } else {
        response.status(400);
        response.write("HTTP is not supported...");
    }
    return true;
}
```

非 GET/HEAD 请求直接 400 (避免 POST 重放)。

## 设计观察

1. **handler 必须实现 `boolean handle(req, resp)`**, 返回 `true` 表示已处理 (链停止), `false` 表示放过 (链继续)。这是 Express-style middleware。
2. **handler 完全跑在业务线程池** (`NettyHandlerAdapter` 通过 `executor.execute()` 调度),
   所以 handler 内可以**同步阻塞**而不阻塞 Netty event loop。
3. **handler 不限于 stateless**: 内部可以持有状态 (如 ResourceProviderFactory)。
4. **CORS / CSRF / HSTS 都采用"不消耗, 委托下一个"模式** (`return false`),
   所以可以同时挂多个安全 handler, 也会按注册顺序叠加响应头。
5. **CSRF handler 是 2.2.x 的设计亮点**——主动抛弃传统 CSRF token, 采用现代浏览器原生
   `Sec-Fetch-Site` 头检测, 并兼容老浏览器的 `Origin` 回退。

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstraction-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatch-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-05-jaxrs|JAX-RS 3.0 支持]]
- [[draft-06-features|功能模块 (SSE/TLS/限流/统计/WebSocket)]]
- [[draft-07-threading-model|【关键】线程模型 (event loop + executor + block())]]
- [[draft-08-state-machines|状态机 (RequestState/ResponseState/HttpExchangeState)]]
- [[draft-09-http2-flow-control|HTTP/2 自定义流控]]
- [[draft-10-graceful-shutdown|优雅关停 (stop with grace period)]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]] — 历史快照
