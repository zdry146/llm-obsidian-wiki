---
title: "mu-server 2.4.2 内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)"
category: synthesis
tags: [java, mu-server, handlers, cors, csrf, static-resources, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
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

# Draft 04 — Handler Library (`io.muserver.handlers`)

> Scope: the production-ready handler bundles that ship with mu-server.

## 1. Files at a glance

| File | LOC | Purpose |
|---|---|---|
| `CORSHandler.java` | 36 | Adds CORS headers to responses (delegates to `CORSConfig`) |
| `CORSHandlerBuilder.java` | 86 | Builder for the above |
| `CSRFProtectionHandler.java` | 89 | Rejects cross-origin unsafe requests (GET/HEAD/OPTIONS safe) |
| `CSRFProtectionHandlerBuilder.java` | 90 | Builder; supports trusted origins + bypass paths + custom rejection |
| `HttpsRedirector.java` | small | 301 redirect HTTP→HTTPS for a path prefix |
| `HttpsRedirectorBuilder.java` | small | Builder |
| `ResourceHandler.java` | 175 | Static-file serving with Range, If-Modified-Since, directory listing |
| `ResourceHandlerBuilder.java` | 253 | Builder; file/classpath/fileOrClasspath factories |
| `ResourceProvider.java` | 386 | File/classpath resource lookup + `sendTo` + skip semantics |
| `ResourceType.java` | 435 | Extension→mime + headers (cache-control, etc.) |
| `BytesRange.java` | small | Parse HTTP Range header into (from, to) ranges |
| `DirectoryLister.java` | small | HTML directory listing |
| `BareDirectoryRequestAction.java` | enum | `REDIRECT_WITH_TRAILING_SLASH` vs `TREAT_AS_NOT_FOUND` |
| `ResourceCustomizer.java` | interface | Last-mile hook for `beforeHeadersSent(request, headers)` |

Total ≈ 1,600 LOC for the handlers package.

## 2. CORS

`CORSHandler.handle` is a one-liner: `corsConfig.writeHeaders(request, response, allowedMethods); return false;`

The actual CORS logic lives in `io.muserver.rest.CORSConfig` / `CORSConfigBuilder`
(≈ 200 lines). The `CORSHandler` is just a thin glue that lets the same
config be applied to non-REST handlers. The JAX-RS `RestHandlerBuilder`
also accepts a `CORSConfig` directly and applies it to REST routes only.

Default `allowedMethods` (from `CORSHandlerBuilder.build()`) is "all
methods except TRACE and CONNECT".

## 3. CSRF Protection

`CSRFProtectionHandler.handle` (89 lines) implements a **modern** CSRF
defence (no token, no cookie-based state) following Filippo Valsorda's
guidance (the class doc links to his blog post).

```
if method is GET/HEAD/OPTIONS → allow
if request URI is in bypass paths → allow
secFetchSite == "same-origin" or "none" → allow
if secFetchSite missing/empty:
    Origin header missing/empty → allow   (probably non-browser)
    Origin matches request host → allow   (same origin)
    Origin in trustedOrigins → allow
else:                                  // cross-origin
    if Origin in trustedOrigins → allow
else:
    return rejectionHandler.handle(...)
```

`Sec-Fetch-Site` is the modern header that browsers send to indicate
"this is a same-origin / cross-origin / no-cors request". This makes the
handler future-proof: it doesn't need CSRF tokens, but it still rejects
unsafe cross-origin POSTs from browsers.

The default rejection handler throws `BadRequestException` → 400.

## 4. ResourceHandler (static files)

`ResourceHandler.handle` (175 lines) is the most featureful of the handlers:

### 4.1 Path resolution

```java
String requestPath = request.relativePath();
if (requestPath.endsWith("/") && defaultFile != null) {
    requestPath += defaultFile;          // / → /index.html
}
String decodedRelativePath = Mutils.urlDecode(requestPath);
ResourceProvider provider = resourceProviderFactory.get(decodedRelativePath);
if (!provider.exists()) { ... }          // fall through to directory listing
```

### 4.2 Directory handling

```java
if (provider.isDirectory()) {
    if (!request.relativePath().endsWith("/")) {
        switch (bareDirectoryRequestAction) {
            case REDIRECT_WITH_TRAILING_SLASH: response.redirect(...); return true;
            case TREAT_AS_NOT_FOUND: return false;
        }
    }
    if (directoryListingEnabled) listDirectory(...);
    else return false;
}
```

The default `BareDirectoryRequestAction.REDIRECT_WITH_TRAILING_SLASH`
matches Apache/Nginx behaviour: `/foo` → `/foo/` (301).

### 4.3 Caching (If-Modified-Since → 304)

```java
String ims = request.headers().get(HeaderNames.IF_MODIFIED_SINCE);
if (ims != null) {
    long lastModTime = lastModified.getTime() / 1000;
    long lastAccessed = Mutils.fromHttpDate(ims).getTime() / 1000;
    if (lastModTime <= lastAccessed) {
        response.status(304);
        sendBody = false;     // HEAD-style body suppression
    }
}
```

Truncates to seconds to match HTTP-date precision.

### 4.4 Range requests (HTTP 206)

```java
String rh = request.headers().get("range");
if (rh != null && totalSize != null && response.status() != 304) {
    List<BytesRange> requestedRanges = BytesRange.parse(totalSize, rh);
    if (requestedRanges.size() == 1) {
        BytesRange range = requestedRanges.get(0);
        boolean couldSkip = provider.skipIfPossible(range.from);
        if (couldSkip) {
            response.status(206);
            response.headers().set(HeaderNames.CONTENT_RANGE, range.toString());
        }
    }
}
```

Only single-range requests are served. `provider.skipIfPossible` uses
`FileChannel.position(...)` for file-based resources (and falls back to
read+discard for classpath resources).

### 4.5 Customization

`ResourceCustomizer.beforeHeadersSent(request, headers)` is called once
per file response, letting the user inject `Cache-Control` /
`Access-Control-*` headers. The handler still sets its own headers first,
so user code can override but cannot remove them.

### 4.6 ResourceProvider (386 lines)

This is a fat interface with two impls:

- `FileResourceProvider` — wraps a `Path`; supports `skipIfPossible` via `FileChannel.transferTo`.
- `ClasspathResourceProvider` — uses `ClassLoader.getResourceAsStream`; cannot skip, so Range requests are served by reading and discarding.

The interface also handles the `Last-Modified` and `Content-Length`
headers (returning `null` when unknown, which disables caching).

## 5. HttpsRedirector

A trivial handler: 301 redirect from `http://host[:port]/prefix*` to
`https://host/prefix*`. Used when the server binds both HTTP (80) and
HTTPS (443) — visitors who hit HTTP get upgraded to HTTPS.

## 6. Other utilities

- `BytesRange` — parses `Range: bytes=0-499` / `bytes=500-` / `bytes=-500`
  syntax (RFC 7233).
- `DirectoryLister` — produces the HTML listing (uses `BufferedWriter`,
  supports CSS injection).
- `ResourceType.DEFAULT_EXTENSION_MAPPINGS` — the static map of 200+
  file extensions to mime types and per-extension caching headers.
- `BareDirectoryRequestAction` — enum with 2 values.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
