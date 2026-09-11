---
title: "mu-server 2.4.2 TLS / SNI / HTTP/2"
category: synthesis
tags: [java, mu-server, tls, sni, http2, flow-control, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "HttpsConfigBuilder SSL/TLS 配置 + 证书热重载, SNI 多证书支持, HTTP/2 自实现流控 (Http2ConnectionFlowControl)"
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

# Draft 08 — TLS / SSL / SNI / HTTP/2 plumbing

> Scope: every piece of code that handles encrypted / multiplexed
> connections.

## 1. Dependency stack

```
io.netty:netty-handler (4.1.137.Final)
  ├── SslHandler (Netty)
  ├── SniHandler (Netty)
  ├── ApplicationProtocolNegotiationHandler (Netty)
  ├── Http2ConnectionHandler (Netty)
  ├── CompressorHttp2ConnectionEncoder (Netty)
  └── IdleStateHandler (Netty)

io.netty:netty-codec-http2 (4.1.137.Final)
  └── Http2CodecBuilder, Http2FrameCodec, Http2Settings, etc.

io.netty:netty-codec-haproxy (4.1.137.Final)
  └── HAProxyMessageDecoder
```

The primary Netty version is `4.1.137.Final`; `4.2.17.Final` is also
configured (`<netty.version.4.2>4.2.17.Final</netty.version.4.2>`) but
not currently active — see Draft 10 "Limitations" for why mu-server
pinned to 4.1.

## 2. HttpsConfigBuilder (459 lines)

### 2.1 Configuration sources

| Source | Method | Notes |
|---|---|---|
| Pre-built `SSLContext` | `withSSLContext(...)` (package-private) | Sets `keyManagerFactory = null` |
| Keystore from stream | `withKeystore(InputStream)` | Reads bytes, doesn't close |
| Keystore from file | `withKeystore(File)` | Opens FileInputStream, closes after read |
| Keystore from classpath | `withKeystoreFromClasspath(...)` | Uses `HttpsConfigBuilder.class.getResourceAsStream` |
| Pre-built `KeyStore` | `withKeystore(KeyStore, char[])` | Stores the keystore in memory |
| Custom `KeyManagerFactory` | `withKeyManagerFactory(...)` | Direct factory |
| Trust manager | `withClientCertificateTrustManager(...)` | For mTLS |

### 2.2 `toNettySslContext(boolean http2)` (line 327)

The central conversion method. Decision tree:

1. If `sslContext != null` → wrap in `JdkSslContext` (no ALPN).
2. Else if `keystoreBytes != null` →
   - Load keystore.
   - Auto-detect default alias (first key entry).
   - Build SAN→alias map (`buildSanToAliasMap`).
   - Build `KeyManagerFactory`.
   - Find an `X509ExtendedKeyManager` (required for SNI).
   - Wrap in `SniKeyManager`.
   - Use `SslContextBuilder.forServer(sniKeyManager)`.
3. Else if `keyManagerFactory != null` → `SslContextBuilder.forServer(kmf)`.
4. Else → `IllegalStateException("No SSL info")`.

If `http2` → adds `ApplicationProtocolConfig(ALPN, NO_ADVERTISE, ACCEPT,
[h2, http/1.1])`.

Then sets `clientAuth` (NONE if no trust manager, OPTIONAL otherwise),
`protocols` (validated against `SSLContext.getDefault().getSupportedSSLParameters().getProtocols()`), and `ciphers` (via the cipher filter).

### 2.3 `unsignedLocalhost()` factory (line 449)

Loads `/io/muserver/resources/localhost.p12` (PKCS12, password
`"Very5ecure"`, cert validity 36500 days). Used as the **default** in
`MuServerBuilder.start()` when no HTTPS config is provided
(`MuServerBuilder.java:703`).

The cert was generated with `keytool` (the comment in the code includes
the exact command).

## 3. SniKeyManager (80 lines)

A custom `X509ExtendedKeyManager` that wraps an underlying key manager
and adds SNI-based alias selection. The `chooseEngineServerAlias` method
(line 49):

1. Casts to `ExtendedSSLSession` to access `getRequestedServerNames()`.
2. Picks the first `SNI_HOST_NAME`-typed name.
3. Looks up in `sanToAliasMap` (the SAN→alias map built in
   `toNettySslContext`).
4. If the alias exists in the keystore AND has both a cert chain and
   private key → use it.
5. Otherwise fall back to `defaultAlias` (or the keystore's first key
   entry).

The `chooseServerAlias(Socket)` method is intentionally
`UnsupportedOperationException` — Netty uses SSLEngine, not SSLSocket,
so this path is never called.

## 4. MuSniHandler (37 lines)

Wraps Netty's `SniHandler` with two overrides:

1. `replaceHandler` — store `hostname()` in the `SNI_HOSTNAME` channel
   attr **before** the swap, so subsequent handlers can read it.
2. `newSslHandler` — set `SSLParameters.setUseCipherSuitesOrder(true)`,
   forcing **server** cipher preference.

The pipeline install is:
```java
new MuSniHandler(() -> new DomainWildcardMappingBuilder<>(sslContextProvider.get()).build())
```

The `Supplier` is critical: `DomainWildcardMappingBuilder` builds a
mapping table from `SslContext`, which can be **swapped** at runtime via
`MuServer.changeHttpsConfig(...)` — every new connection picks up the
new cert.

## 5. MuGzipHttp2ConnectionEncoder (137 lines) — the "mu-" prefix hack

The HTTP/2 `writeHeaders` is intercepted to detect the `mu-` prefix:

```java
static CharSequence actualEncodingIfHasMuPrefix(CharSequence seq) {
    if (seq != null) {
        int len = seq.length();
        if (len > 3 && seq.charAt(0) == 'm' && seq.charAt(1) == 'u' && seq.charAt(2) == '-') {
            return seq.subSequence(3, len);  // strip "mu-"
        }
    }
    return null;
}
```

The flow is:

1. `Http2Response.writeHeaders` sets `Content-Encoding: mu-gzip` if the
   response should be compressed.
2. `MuGzipHttp2ConnectionEncoder.writeHeaders` calls `fixEncoding` which
   strips the `mu-` prefix → `gzip`.
3. The downstream `CompressorHttp2ConnectionEncoder` sees the real
   encoding name and compresses the body.

Why the prefix? The encoder chain doesn't know which responses to
compress (the mu-server application code decides, based on size and
mime type). By using a custom encoding name, mu-server signals intent
without affecting the wire format — the client sees a normal
`Content-Encoding: gzip` header.

## 6. MuCompressorHttp2ConnectionEncoder (24 lines)

A thin wrapper that overrides `newContentCompressor`. If the encoding
has the `mu-` prefix, return the compressor for the stripped encoding.
Otherwise return `null` (no compression).

## 7. Http2ConnectionBuilder (36 lines)

Subclasses Netty's `AbstractHttp2ConnectionHandlerBuilder`:

```java
initialSettings()
    .maxHeaderListSize(server.settings().maxHeadersSize)
    .maxConcurrentStreams(server.http2Config().maxConcurrentStreams);
```

Then `build(decoder, encoder, initialSettings)`:

- If `gzipEnabled` → wrap encoder in `MuGzipHttp2ConnectionEncoder` →
  wrap in `MuCompressorHttp2ConnectionEncoder`.
- Create `Http2Connection(decoder, encoder, settings, server, nettyHandlerAdapter)`.
- Set `frameListener(handler)` so the connection class is the frame
  listener too.

`Http2ConfigBuilder` (100 lines):

| Setting | Default | Notes |
|---|---|---|
| `enabled` | `false` | Set to true via `Http2ConfigBuilder.http2Enabled()` or `http2EnabledIfAvailable()` |
| `maxConcurrentStreams` | 200 | Initial SETTINGS frame value |

`http2EnabledIfAvailable()` is a heuristic that checks
`"1.8".equals(System.getProperty("java.specification.version"))` — Java 8
disables ALPN, so HTTP/2 won't work. Java 9+ enables. The doc warns
this is not reliable; users should explicitly enable if they know ALPN
is supported.

## 8. The full HTTPS+HTTP/2 pipeline

```
idle                           ← IdleStateHandler (idleTimeoutMills)
traffic-shaping                ← GlobalTrafficShapingHandler
HAProxy?                       ← optional
sni (MuSniHandler)             ← chooses cert by SNI
pressure (BackPressureHandler) ← for HTTP/2 (pre-ALPN)
alpn (AlpnHandler)             ← configures post-handshake
conerror                       ← catch-all
[Http2ConnectionBuilder.build() OR setupHttp1Pipeline()]
```

The `BackPressureHandler` is added *before* ALPN so it's in the
pipeline no matter which protocol wins. ALPN's `configurePipeline` for
HTTP/1 removes it and re-adds it in the right position (see Draft 01
§5).

## 9. Live SSL reload

`MuServer.changeHttpsConfig(HttpsConfigBuilder newHttpsConfig)` is the
public mutation point:

```java
public void changeHttpsConfig(HttpsConfigBuilder newHttpsConfig) {
    Mutils.notNull("newSSLContext", newHttpsConfig);
    try {
        SslContext nettySslContext = newHttpsConfig.toNettySslContext(http2Config.enabled);
        sslContextProvider.set(nettySslContext);
        ((SSLInfoImpl) sslContextProvider.sslInfo()).setHttpsUri(httpsUri);
    } catch (Exception e) {
        throw new MuException("Error while changing SSL Certificate. The old one will still be used.", e);
    }
}
```

`SslContextProvider.set` (line 28) does two things:
1. Build the `SSLInfo` (provider + protocols + ciphers) for the new context.
2. `nettySslContext.set(newValue)` — atomically swap the reference.

Because `MuSniHandler`'s mapping provider is a `Supplier`, it reads the
new `SslContext` on the next connection. **Existing connections keep
their old `SslHandler`** (because Netty installs the handler during
handshake) — this is fine because the old handler was valid at the
time.

## 10. SSLInfo (interface, 53 lines)

A read-only snapshot of the server's SSL configuration. Exposed via
`MuServer.sslInfo()`. Has:

- `provider()` — "JDK" or "OpenSSL"
- `protocols()` — enabled TLS versions
- `ciphers()` — enabled cipher suites

Note: `SSLInfo` does **not** include the keystore or trust manager — only
the **enabled** set, derived from `SSLEngine.getEnabledProtocols()` on a
transient engine created in `SslContextProvider.set`.

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-06-openapi|OpenAPI 集成]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
