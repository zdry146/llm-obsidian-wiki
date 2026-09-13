---
title: "OkHttp 对比选型 (vs Java 11 HC / Apache HC 5 / Reactor Netty / Netty)"
category: synthesis
tags: [okhttp, comparison, java-11-http-client, apache-httpclient, reactor-netty, netty]
sources:
  - "Java 11 HttpClient (java.net.http)"
  - "Apache HttpClient 5.x"
  - "Reactor Netty 1.x"
  - "Netty 4.1 / 4.2"
summary: "OkHttp 与 4 大 HTTP 客户端的全方位对比：API、性能、生态、适用场景"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §11 OkHttp 对比选型

## 1. 五大 HTTP 客户端总览

| 客户端 | 作者 | 协议 | 优势 | 劣势 |
|--------|------|------|------|------|
| **OkHttp** | Square | Apache 2.0 | 生态最好、拦截器链、KMP | 同步 API 较底层 |
| **Java 11 HttpClient** | OpenJDK | OpenJDK | JDK 自带、纯异步 | 生态弱、拦截器扩展差 |
| **Apache HttpClient 5** | Apache | Apache 2.0 | 老牌稳定、企业特性 | API 笨重、性能略差 |
| **Reactor Netty** | Pivotal | Apache 2.0 | 响应式、生态齐全 | 复杂度高、学习曲线陡 |
| **Netty**（[[mu-server-2.4.2-analysis/summary\|mu-server 服务端底层]]） | Netty 社区 | Apache 2.0 | 极致性能、灵活 | 用作客户端太重 |

## 2. 对比维度矩阵

### 2.1 同步 API 易用性

```kotlin
// OkHttp（4.x）
val request = Request.Builder().url(url).build()
val response = client.newCall(request).execute()
val body = response.body?.string()

// Java 11 HttpClient
val request = HttpRequest.newBuilder().uri(URI(url)).build()
val response = client.send(request, HttpResponse.BodyHandlers.ofString())
val body = response.body()

// Apache HC 5
val request = ClassicHttpRequest(HttpGet(url))
val response = client.execute(request)
val body = response.entity.content.reader().use { it.readText() }

// Reactor Netty
val response = httpClient.get().uri(url).response().block()

// Netty（直接）
val bootstrap = Bootstrap()
    .group(EventLoopGroup)
    .channel(NioSocketChannel::class.java)
    .handler(HttpClientInitializer)
val channel = bootstrap.connect(host, port).sync().channel()
val request = DefaultFullHttpRequest(...)
channel.writeAndFlush(request).sync()
val response = channel.readInbound<FullHttpResponse>()
```

**OkHttp 胜出**：API 简洁、Kotlin/Java 都友好。

### 2.2 异步 API

```kotlin
// OkHttp
client.newCall(request).enqueue(object : Callback { ... })

// Java 11 HC（纯异步返回 CompletableFuture）
val future: CompletableFuture<HttpResponse<String>> = 
    client.sendAsync(request, HttpResponse.BodyHandlers.ofString())

// Reactor Netty（响应式）
httpClient.get().uri(url).retrieve().bodyToMono<String>()

// Netty（Future + Listener）
channel.writeAndFlush(request).addListener { ... }
```

**Java 11 HC 胜出**：原生 `CompletableFuture` 是现代风格。
**Reactor Netty**：响应式流，Spring WebFlux 一等公民。
**OkHttp**：Callback 风格不算最现代，但生态最好。

### 2.3 拦截器/扩展性

| 客户端 | 拦截器 | 评分 |
|--------|--------|------|
| OkHttp | ✅ 6 层 | ⭐⭐⭐⭐⭐ |
| Java 11 HC | ❌ 无拦截器 | ⭐ |
| Apache HC 5 | ✅ HttpRequestInterceptor/ResponseInterceptor | ⭐⭐⭐ |
| Reactor Netty | ✅ ExchangeFilterFunction | ⭐⭐⭐⭐ |
| Netty | ✅ ChannelHandler 链（完全自由） | ⭐⭐⭐⭐⭐ |

**OkHttp / Netty 胜出**。

### 2.4 HTTP/2 支持

| 客户端 | 客户端 HTTP/2 | 备注 |
|--------|--------------|------|
| OkHttp | ✅ 默认 | 完善 |
| Java 11 HC | ✅ 默认 | 完善 |
| Apache HC 5 | ✅（需额外配置） | 文档不全 |
| Reactor Netty | ✅ | 与 Reactor 深度集成 |
| Netty | ✅ | 最灵活 |

### 2.5 性能（基准对比）

```
基准测试（来自社区 benchmark，2024 年）

GET 请求 1000 次（HTTP/1.1，保持连接）：

  Java 11 HC        : 8.2s
  OkHttp 4.12       : 8.5s   ← 几乎相同
  Apache HC 5       : 12.4s  ← 慢 50%
  Reactor Netty     : 9.1s
  Netty (raw)       : 7.8s   ← 最快但代码最长

并发 100 请求：

  Reactor Netty     : 2.1s   ← 响应式最擅长
  OkHttp            : 2.4s
  Java 11 HC        : 2.5s
  Netty (raw)       : 1.9s
  Apache HC 5       : 4.8s   ← 最差
```

⚠️ **基准测试因场景差异大**——具体数据仅供参考。

### 2.6 体积

```
okhttp-4.12.0.jar              : 1.2 MB
okio-3.6.0.jar                  : 0.3 MB
java.net.http (JDK 内置)         : 0
apache-httpclient-5.x.jar       : 1.7 MB
reactor-netty-1.x.jar           : 4.5 MB
netty-all-4.1.x.jar             : 8 MB
```

**Java 11 HC 胜出**：JDK 自带零依赖。
**OkHttp 第二**：1.5 MB（含 Okio），合理。

### 2.7 协程支持

| 客户端 | 协程支持 |
|--------|---------|
| OkHttp 5.x | ✅ `await()` |
| OkHttp 4.x | 第三方包装 |
| Java 11 HC | ✅ `sendAsync()` + `await()` |
| Reactor Netty | ✅ 原生 Mono/Flux |
| Apache HC 5 | ❌ 需自己包装 |
| Netty | 需配合 reactor-netty 或自写 |

## 3. 各客户端详解

### 3.1 OkHttp（推荐度 ⭐⭐⭐⭐⭐）

**优势**：
- ✅ Android 默认客户端
- ✅ Retrofit 依赖 → 几乎所有 Android 项目的 HTTP 栈
- ✅ 拦截器链设计优雅
- ✅ 文档完善、社区活跃
- ✅ Square 持续维护 12+ 年

**劣势**：
- ❌ 同步 API 较底层（需手动处理 cookie、超时）
- ❌ 5.x 协程支持还在 alpha

**适用**：
- Android 应用（首选）
- JVM 微服务（首选）
- Square 生态（必须）
- Retrofit 用户（必须）

### 3.2 Java 11 HttpClient（推荐度 ⭐⭐⭐）

**优势**：
- ✅ JDK 内置，零依赖
- ✅ 纯异步 API（CompletableFuture）
- ✅ HTTP/2 / WebSocket / SSE 全支持
- ✅ 现代 API 设计

**劣势**：
- ❌ 拦截器扩展差
- ❌ 生态弱（不如 OkHttp）
- ❌ Java 8 项目不能用

**适用**：
- JDK 11+ 项目，不想引入依赖
- 简单 HTTP 调用
- 不需要复杂拦截器

### 3.3 Apache HttpClient 5（推荐度 ⭐⭐）

**优势**：
- ✅ 老牌稳定（20+ 年）
- ✅ 企业特性齐全（OAuth、NTLM、Kerberos）
- ✅ 兼容 Apache 2.x（迁移成本低）

**劣势**：
- ❌ API 笨重
- ❌ 性能弱
- ❌ 社区活跃度下降

**适用**：
- 老项目维护（已有 HC 代码）
- 需要 NTLM/Kerberos 等企业认证
- 银行 / 政府内部系统

### 3.4 Reactor Netty（推荐度 ⭐⭐⭐⭐）

**优势**：
- ✅ Spring WebFlux 默认底层
- ✅ 响应式编程范式
- ✅ 高并发场景优秀
- ✅ 与 Project Reactor 深度集成

**劣势**：
- ❌ 学习曲线陡（响应式概念）
- ❌ 调试困难
- ❌ 体积大

**适用**：
- Spring WebFlux 项目（首选）
- 高并发流式处理
- 微服务响应式架构

### 3.5 Netty（推荐度 ⭐⭐⭐⭐⭐，但仅限服务端）

**优势**：
- ✅ 极致性能
- ✅ 完全自由（ChannelHandler）
- ✅ 异步、零拷贝

**劣势**：
- ❌ API 复杂
- ❌ 用作客户端太重
- ❌ 学习曲线最陡

**适用**：
- 高性能服务端（首选）
- 自定义协议
- 嵌入式网络库

## 4. 选型决策树

```
需要 HTTP 客户端
    │
    ├─ Android 应用？
    │     └─ → OkHttp + Retrofit（默认）
    │
    ├─ Spring Boot 项目？
    │     ├─ Spring MVC → OkHttp (替换默认) 或 WebClient
    │     └─ Spring WebFlux → WebClient (Reactor Netty)
    │
    ├─ 微服务调用外部 API？
    │     ├─ 高并发 → Reactor Netty (WebClient)
    │     └─ 中小规模 → OkHttp
    │
    ├─ Kotlin 协程项目？
    │     ├─ KMP 跨平台 → OkHttp 5.x alpha 或 Ktor Client
    │     └─ JVM only → Java 11 HC 或 OkHttp
    │
    ├─ JDK 11+ 且不想引入依赖？
    │     └─ → Java 11 HttpClient
    │
    ├─ 老项目（Apache HC 2.x）维护？
    │     └─ → Apache HC 5（迁移成本低）
    │
    └─ 极致性能 / 自定义协议？
          └─ → Netty（直接用）
```

## 5. Spring Boot 默认 vs 显式选型

Spring Boot 默认使用 `ClientHttpRequestFactory`：
- 2.x 之前：`HttpURLConnection`
- 3.x 之后：可以配置 OkHttp / Reactor Netty / Apache HC

```yaml
# application.yml
spring:
  http:
    client:
      factory: okhttp  # 或 apache, jetty, reactor
```

**实战推荐**：微服务用 OkHttp（替换默认）+ Spring Cloud OpenFeign。

## 6. Spring Cloud OpenFeign 底层

OpenFeign 默认底层用 `HttpURLConnection`，**不推荐**。

```yaml
# 推荐：替换为 OkHttp
feign:
  okhttp:
    enabled: true
  httpclient:
    enabled: false
```

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

⚠️ OpenFeign + OkHttp 是 Spring Cloud 微服务的**事实标准组合**。

## 7. Ktor Client（备选）

Kotlin Multiplatform 项目可考虑 Ktor：

```kotlin
val client = HttpClient(CIO)  // 或 OkHttp engine
val response: HttpResponse = client.get("https://api.example.com")
val body: String = response.body()
```

| 维度 | Ktor | OkHttp |
|------|------|--------|
| KMP 支持 | ✅ 全平台 | ⚠️ JVM/iOS |
| 生态 | Kotlin 圈内 | JVM 主流 |
| 拦截器 | Pipeline | Interceptor chain |
| 协程 | ✅ 原生 | ✅ 5.x |

## 8. 性能优化技巧（通用）

无论选哪个客户端：

1. **启用 HTTP/2** —— 多路复用显著提速
2. **连接池调优** —— 复用 TCP 连接
3. **启用 GZIP** —— 减少流量
4. **超时分级** —— connect/read/write/call 分开
5. **缓存** —— 减少重复请求
6. **DNS 缓存** —— 减少解析开销
7. **请求合并** —— 减少网络往返

## 9. 切换成本对比

| 从 → 到 | 切换成本 | 备注 |
|---------|---------|------|
| OkHttp → Java 11 HC | 🟢 低 | API 相似 |
| OkHttp → Reactor Netty | 🟡 中 | 需重写为响应式 |
| OkHttp → Apache HC 5 | 🟡 中 | API 风格不同 |
| Apache HC → OkHttp | 🔴 高 | API 风格差异大 |
| Netty → OkHttp | 🟢 低（同步场景） | Netty 本来就低层 |
| Netty → Reactor Netty | 🟡 中 | 配合 Mono/Flux |

## 10. 实战推荐组合

### 10.1 Android 应用

```
OkHttp 4.12 + Retrofit + Moshi + OkHttp Logging Interceptor
```

### 10.2 Spring Cloud 微服务

```
Spring Cloud OpenFeign + OkHttp + Resilience4j + Micrometer
```

### 10.3 Spring WebFlux 响应式

```
Spring WebClient + Reactor Netty + Project Reactor
```

### 10.4 Kotlin Multiplatform

```
Ktor Client + kotlinx.serialization + Ktor Logging
```

### 10.5 高频交易 / 量化

```
Netty (raw) + 自定义二进制协议
```

## 11. 监控指标（不管用哪个都要监控）

```kotlin
// 1. 连接池使用率
val usage = pool.connectionCount().toDouble() / pool.maxIdleConnections
if (usage > 0.8) alert("连接池使用率高")

// 2. 请求 P99 延迟
val p99 = histogram.getSnapshot().get99thPercentile()
if (p99 > 500) alert("P99 延迟高")

// 3. 失败率
val failureRate = failures / total
if (failureRate > 0.01) alert("失败率超 1%")

// 4. EventListener 时序
val slowEvents = listOf(
    EventListener::callStart,
    EventListener::dnsStart,
    EventListener::connectStart,
    EventListener::responseHeadersEnd,
    EventListener::callEnd
).zipWithNext { a, b -> ... }
```

## 12. 总结

```
我的推荐优先级：

1. Android / JVM 微服务         → OkHttp + Retrofit
2. Spring WebFlux / 响应式      → WebClient (Reactor Netty)
3. JDK 11+ 不想引入依赖         → Java 11 HttpClient
4. 老项目维护 / 企业认证需求    → Apache HttpClient 5
5. KMP / 跨平台                → Ktor Client 或 OkHttp 5.x
6. 极致性能 / 自定义协议        → Netty (raw)
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **实战场景**: [[draft-12-use-cases]]
- **综合入口**: [[summary]]
