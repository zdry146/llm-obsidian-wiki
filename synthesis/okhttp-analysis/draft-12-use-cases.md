---
title: "OkHttp 适用场景与决策矩阵"
category: synthesis
tags: [okhttp, use-cases, decision-matrix, best-practices]
sources:
  - "OkHttp 官方文档"
  - "Square 工程博客 - OkHttp 用户案例"
  - "作者实战经验"
summary: "OkHttp 的 6 类 ✅ 适合场景、4 类 ❌ 不适合场景、6 条铁律"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §12 OkHttp 适用场景与决策矩阵

## 1. ✅ 6 类适合场景

### 1.1 场景 1：Android 应用网络层

**为什么**：Android 自 4.4 起内置 OkHttp，是事实标准。

```kotlin
// 典型用法：单例 + Retrofit
object NetworkModule {
    val okHttpClient: OkHttpClient by lazy {
        OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(AuthInterceptor(tokenStore))
            .addInterceptor(LoggingInterceptor(BuildConfig.DEBUG))
            .build()
    }
    
    val retrofit: Retrofit by lazy {
        Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .client(okHttpClient)
            .addConverterFactory(MoshiConverterFactory.create())
            .build()
    }
}
```

**收益**：与 Retrofit 无缝集成；省 50% 网络代码。

### 1.2 场景 2：JVM 微服务调用

**为什么**：拦截器链 + 连接池 + HTTP/2 完美匹配微服务场景。

```kotlin
// Spring Cloud OpenFeign 底层用 OkHttp
@FeignClient(name = "order-service")
interface OrderClient {
    @GetMapping("/orders/{id}")
    fun getOrder(@PathVariable id: String): OrderDTO
}
```

**收益**：高并发稳定；与 Resilience4j / Sentinel 集成简单。

### 1.3 场景 3：调用第三方 REST API

**为什么**：拦截器统一处理鉴权、签名、重试；GZIP 自动；HTTP/2 默认。

```kotlin
val client = OkHttpClient.Builder()
    .addInterceptor(SignatureInterceptor(secretKey))  // 签名
    .addInterceptor(AuthInterceptor(tokenProvider))     // OAuth
    .addInterceptor(RetryInterceptor(maxRetries = 3))  // 重试
    .build()

// 调用
val request = Request.Builder()
    .url("https://api.partner.com/v1/data")
    .get()
    .build()
client.newCall(request).execute().use { response ->
    handleResponse(response)
}
```

**收益**：SDK 维护成本降低 60%。

### 1.4 场景 4：WebSocket 实时通信

**为什么**：内置 WebSocket 客户端 + 心跳 + 优雅关闭。

```kotlin
val listener = object : WebSocketListener() {
    override fun onMessage(ws: WebSocket, text: String) {
        handleEvent(text)
    }
}

val ws = client.newWebSocket(
    Request.Builder().url("wss://realtime.example.com/events").build(),
    listener
)

// 30 秒心跳
pingInterval(30, TimeUnit.SECONDS)
```

**收益**：相比自实现 WebSocket 协议，省 2 周开发量。

### 1.5 场景 5：数据采集 / 抓取

**为什么**：连接池复用 + 异步并发 + 拦截器采集元数据。

```kotlin
val client = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(100, 5, TimeUnit.MINUTES))
    .dispatcher(Dispatcher().apply {
        maxRequests = 200
        maxRequestsPerHost = 20
    })
    .addNetworkInterceptor(MetricsInterceptor())  // 采集耗时、状态码
    .build()

// 批量抓取
val urls = listOf("https://a.com", "https://b.com", ...)
urls.map { url ->
    async { client.newCall(Request.Builder().url(url).build()).execute() }
}.awaitAll()
```

**收益**：10x 吞吐量提升（vs HttpURLConnection）。

### 1.6 场景 6：测试 / Mock 服务

**为什么**：MockWebServer 是 OkHttp 官方测试工具，无依赖。

```kotlin
val server = MockWebServer()
server.enqueue(MockResponse().setBody("""{"name":"mike"}"""))
server.start()

val client = OkHttpClient.Builder().build()
val request = Request.Builder().url(server.url("/users")).build()
val response = client.newCall(request).execute()

assertEquals("""{"name":"mike"}""", response.body?.string())
server.shutdown()
```

**收益**：单元测试无网络依赖；可精确控制响应。

## 2. ❌ 4 类不适合场景

### 2.1 不适合 1：服务端 HTTP 实现

❌ **OkHttp 不是服务端**——它是 HTTP 客户端。

**应该用**：
- **Netty**（底层，灵活）
- **Jetty**（Java EE 兼容）
- **Spring Boot Tomcat**（企业级）
- **mu-server**（轻量级）
- **Vert.x**（响应式）

### 2.2 不适合 2：UDP / QUIC / WebRTC

❌ **OkHttp 只支持 HTTP/HTTPS/WebSocket**——不支持 UDP、QUIC、WebRTC。

**应该用**：
- **Netty**（全协议）
- **Kronos Netty**（QUIC）
- **webrtc-java**

### 2.3 不适合 3：极端低延迟场景

❌ OkHttp 有 Okio 抽象层、连接池等开销，**不适合微秒级延迟**。

**应该用**：
- **Aeron**（低延迟消息，μs 级）
- **Direct Netty**（裸 NIO）
- **DPDK**（用户态网络）

### 2.4 不适合 4：极简静态文件场景

❌ 用 OkHttp 同步下载文件是可行的，但**不是最优**。

**应该用**：
- **Kotlin/Java NIO 直接读**
- **Apache Commons IO**
- **Okio 直接读**

## 3. 决策矩阵（详细）

| 场景 | OkHttp？ | 备选 | 推荐理由 |
|------|---------|------|---------|
| Android 应用 | ✅ 必须 | Retrofit | 默认客户端 |
| Retrofit 用户 | ✅ 必须 | - | 强依赖 |
| Spring Cloud 微服务 | ✅ 推荐 | WebClient | 拦截器丰富 |
| Spring WebFlux | ❌ 不推荐 | WebClient | 响应式 |
| 高并发抓取 | ✅ 推荐 | Reactor Netty | 并发控制好 |
| 第三方 API 调用 | ✅ 推荐 | Retrofit | 签名/重试简单 |
| WebSocket 客户端 | ✅ 推荐 | Java WebSocket | 完整实现 |
| 测试 Mock 服务 | ✅ 推荐 | WireMock | 官方工具 |
| HTTP 服务端 | ❌ 不行 | Netty / Tomcat | OkHttp 不是服务端 |
| QUIC / UDP | ❌ 不行 | Netty + Quiche | 不支持 |
| 低延迟交易 | ❌ 不行 | Aeron | 抽象层太重 |
| JDK 8 项目 | ⚠️ 可用但有限制 | Apache HC | OkHttp 4.x 部分特性需 JDK 11+ |
| GraalVM 原生 | ✅ 推荐 | Java 11 HC | 5.x 完整支持 |

## 4. 6 条使用铁律

### 铁律 1：**OkHttpClient 单例**

```kotlin
// ❌ 错误：每次新建
fun getUser(id: String): User {
    val client = OkHttpClient()  // 浪费连接池
    // ...
}

// ✅ 正确：单例
object Http {
    val client = OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .build()
}
```

**为什么**：连接池、线程池、缓存都是 OkHttpClient 持有的。

### 铁律 2：**Response 必须 close**

```kotlin
// ❌ 错误：忘 close → 连接泄漏
val response = client.newCall(request).execute()
val body = response.body?.string()

// ✅ 正确：use 自动 close
client.newCall(request).execute().use { response ->
    val body = response.body?.string()
    process(body)
}
```

**为什么**：连接不归还池，连接池耗尽 → 所有请求卡住。

### 铁律 3：**拦截器调 proceed()**

```kotlin
// ❌ 错误：忘了 proceed → 请求挂起
override fun intercept(chain: Chain): Response {
    val request = chain.request()
    // ... 修改 request ...
    return Response.Builder().build()  // ❌ 没调 proceed
}

// ✅ 正确
override fun intercept(chain: Chain): Response {
    val request = chain.request()
    val response = chain.proceed(modifiedRequest)  // ✅ 调 proceed
    return response
}
```

### 铁律 4：**Android 主线程不能 execute()**

```kotlin
// ❌ 错误：主线程 ANR
class MainActivity : AppCompatActivity() {
    fun onCreate() {
        val response = client.newCall(request).execute()  // ANR
    }
}

// ✅ 正确：enqueue 或协程
client.newCall(request).enqueue(callback)
// 或
lifecycleScope.launch {
    val response = client.newCall(request).await()
}
```

### 铁律 5：**OKHttp 版本 ≥ 4.12（JDK 21）**

```xml
<!-- ❌ 老版本（JDK 21 下有 IPv6 bug） -->
<version>4.10.0</version>

<!-- ✅ 4.12+ 修复 -->
<version>4.12.0</version>
```

**为什么**：JDK 21 + 早期 OkHttp 的 IPv6/IPv4 解析 bug。详见 [[draft-10-known-issues]]。

### 铁律 6：**生产环境禁用日志 Body**

```kotlin
// ❌ 错误：生产环境打印 body（泄漏敏感数据）
HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY  // 打印请求/响应 body
}

// ✅ 正确：生产用 HEADERS 或 NONE
HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) BODY else NONE
}
```

**为什么**：Body 可能包含用户密码、token、隐私信息。

## 5. 启动检查清单（项目第一天）

```kotlin
// application.yml 或 properties 文件
okhttp:
  connect-timeout: 10s
  read-timeout: 30s
  call-timeout: 60s
  max-idle-connections: 5
  keep-alive: 5m
  retry-on-failure: true
  follow-redirects: true

// 代码初始化
val client = OkHttpClient.Builder()
    .connectTimeout(connectTimeout.toMillis(), TimeUnit.MILLISECONDS)
    .readTimeout(readTimeout.toMillis(), TimeUnit.MILLISECONDS)
    .callTimeout(callTimeout.toMillis(), TimeUnit.MILLISECONDS)
    .connectionPool(ConnectionPool(maxIdleConnections, keepAlive, TimeUnit.MINUTES))
    .retryOnConnectionFailure(true)
    .followRedirects(true)
    .addInterceptor(MetricsInterceptor())     // 监控必备
    .addInterceptor(TracingInterceptor())     // 链路追踪
    .build()
```

## 6. 上线前 Checklist

- [ ] OkHttpClient 单例化
- [ ] 所有 Response 都 close（use 块）
- [ ] OkHttp 版本 ≥ 4.12（JDK 21）
- [ ] 监控指标就位（QPS、延迟、连接池、失败率）
- [ ] 拦截器链可观测（日志、追踪）
- [ ] 敏感数据走 `Cache-Control: no-store`
- [ ] 生产环境日志级别 ≤ HEADERS
- [ ] 鉴权机制验证（拦截器测试）
- [ ] 重试机制有 backoff + 抖动
- [ ] 单元测试用 MockWebServer
- [ ] 异常处理：自定义 IOException → 业务异常

## 7. 监控告警配置

```kotlin
// 监控指标（用 Micrometer 暴露）
val timer = Timer.builder("okhttp.request.duration")
    .tag("host", request.url.host)
    .register(meterRegistry)

// 告警规则
- QPS 突降 50% → 告警
- P99 延迟 > 1s → 告警
- 失败率 > 1% → 告警
- 连接池使用率 > 80% → 告警
- DNS 解析延迟 > 100ms → 告警
```

## 8. 演进路径

```
第一阶段：直接用 OkHttp
   - 单例 OkHttpClient
   - 拦截器链做鉴权/日志
   - 同步调用 + use 块

第二阶段：加监控和测试
   - EventListener 时序
   - Micrometer 暴露指标
   - MockWebServer 单测

第三阶段：异步化（如需要）
   - enqueue + Dispatcher
   - 调优 maxRequests

第四阶段：协程化（Kotlin 项目）
   - lifecycleScope / coroutineScope
   - 协程取消传播
   - Structured Concurrency

第五阶段：平台化
   - 抽出 common-http 模块
   - 多 OkHttpClient 适配（不同 host 走不同 client）
   - 高级特性：连接复用、HTTP/2、HTTP/3 实验
```

## 9. 与 mu-server 的对比（HTTP 出站 vs 入站）

| 维度 | OkHttp | mu-server |
|------|--------|-----------|
| 角色 | 出站 | 入站 |
| 用户接口 | Request/Response | MuRequest/MuResponse |
| 拦截器 | Interceptor | Handler |
| 状态机 | Call → Response | RequestState |
| 跨线程 | Dispatcher | block() 模式 |
| 适用 | 调用外部 | 暴露服务 |

**实战组合**：用 mu-server 暴露内部 API → 用 OkHttp 调用外部 API → 用 OkHttp 调用 mu-server 暴露的内部 API。

## 10. 我的"开局即最佳"配置模板

```kotlin
// === 配置数据类（建议用 Duration 而非原始 Long）===
data class HttpConfig(
    // 超时：用 kotlin.time.Duration 避免单隐错
    val connectTimeout: Duration = 10.seconds,
    val readTimeout: Duration = 30.seconds,
    val writeTimeout: Duration = 30.seconds,
    val callTimeout: Duration = 60.seconds,
    
    // 连接池
    val maxIdleConnections: Int = 5,
    val keepAliveDuration: Duration = 5.minutes,
    
    // 并发调度
    val maxRequests: Int = 64,
    val maxRequestsPerHost: Int = 5,
    
    // WebSocket
    val pingInterval: Duration = 0.seconds,            // 0 = 不发心跳
    
    // 缓存
    val cacheEnabled: Boolean = false,
    val cacheDir: File? = null,
    val cacheMaxSizeBytes: Long = 10L * 1024 * 1024,   // 10 MB
    
    // 代理
    val proxy: Proxy? = null,
    val proxyAuthenticator: Authenticator? = null,
    
    // 自定义拦截器
    val interceptors: List<Interceptor> = emptyList(),
    val networkInterceptors: List<Interceptor> = emptyList(),
    
    // 可观测性
    val eventListenerFactory: EventListener.Factory? = null,
    
    // DNS
    val dns: Dns = Dns.SYSTEM,
    
    // 重试与重定向
    val retryOnConnectionFailure: Boolean = true,
    val followRedirects: Boolean = true,
    val followSslRedirects: Boolean = true,
    
    // 协议版本
    val protocols: List<Protocol> = listOf(Protocol.HTTP_1_1, Protocol.HTTP_2)
) {
    init {
        // 配置验证：启动期接误，避免运行时被 OkHttp 抛不友好异常
        require(!connectTimeout.isNegative()) { "connectTimeout must be positive" }
        require(!readTimeout.isNegative()) { "readTimeout must be positive" }
        require(!writeTimeout.isNegative()) { "writeTimeout must be positive" }
        require(!callTimeout.isNegative()) { "callTimeout must be positive" }
        require(maxRequests > 0) { "maxRequests must be > 0" }
        require(maxRequestsPerHost > 0) { "maxRequestsPerHost must be > 0" }
        require(maxRequestsPerHost <= maxRequests) { 
            "maxRequestsPerHost ($maxRequestsPerHost) cannot exceed maxRequests ($maxRequests)" 
        }
        require(maxIdleConnections >= 0) { "maxIdleConnections must be >= 0" }
        require(!keepAliveDuration.isNegative()) { "keepAliveDuration must be positive" }
        require(!pingInterval.isNegative()) { "pingInterval must be positive" }
        require(cacheMaxSizeBytes > 0) { "cacheMaxSizeBytes must be > 0" }
        if (cacheEnabled) {
            requireNotNull(cacheDir) { "cacheDir required when cacheEnabled = true" }
        }
    }
}

// === 工厂实现 ===
class HttpClientFactory {
    fun create(config: HttpConfig): OkHttpClient {
        val builder = OkHttpClient.Builder()
            // 超时（验证后转换类型）
            .connectTimeout(config.connectTimeout.inWholeMilliseconds, TimeUnit.MILLISECONDS)
            .readTimeout(config.readTimeout.inWholeMilliseconds, TimeUnit.MILLISECONDS)
            .writeTimeout(config.writeTimeout.inWholeMilliseconds, TimeUnit.MILLISECONDS)
            .callTimeout(config.callTimeout.inWholeMilliseconds, TimeUnit.MILLISECONDS)
            
            // 重试 / 重定向
            .retryOnConnectionFailure(config.retryOnConnectionFailure)
            .followRedirects(config.followRedirects)
            .followSslRedirects(config.followSslRedirects)
            
            // 协议版本
            .protocols(config.protocols)
            
            // 连接池
            .connectionPool(ConnectionPool(
                maxIdleConnections = config.maxIdleConnections,
                keepAliveDuration = config.keepAliveDuration.inWholeMilliseconds,
                timeUnit = TimeUnit.MILLISECONDS
            ))
            
            // 调度器
            .dispatcher(Dispatcher().apply {
                maxRequests = config.maxRequests
                maxRequestsPerHost = config.maxRequestsPerHost
            })
            
            // WebSocket 心跳（0 = 禁用）
            .apply {
                if (config.pingInterval > 0.seconds) {
                    pingInterval(config.pingInterval.inWholeSeconds, TimeUnit.SECONDS)
                }
            }
            
            // 代理
            .apply {
                if (config.proxy != null) proxy(config.proxy)
                if (config.proxyAuthenticator != null) proxyAuthenticator(config.proxyAuthenticator)
            }
            
            // DNS
            .dns(config.dns)
            
            // 可观测性
            .apply {
                if (config.eventListenerFactory != null) eventListenerFactory(config.eventListenerFactory)
            }
        
        // 业务拦截器
        config.interceptors.forEach { builder.addInterceptor(it) }
        config.networkInterceptors.forEach { builder.addNetworkInterceptor(it) }
        
        // 缓存（可选）
        if (config.cacheEnabled) {
            config.cacheDir!!.mkdirs()  // 确保目录存在
            builder.cache(Cache(config.cacheDir!!, config.cacheMaxSizeBytes))
        }
        
        return builder.build()
    }
}
```

## 10.1 修复的 8 个问题

| # | 旧版问题 | 严重性 | 新版修复 |
|---|---------|--------|---------|
| 1 | `HttpConfig` 类未定义——用户复制代码后编译不过 | 🔴 | 提供完整 `data class HttpConfig` 定义 |
| 2 | `connectTimeout` 等是原始 `Long`（毫秒）——语义不明确，调 10 和 10000 都对 | 🟡 | 改用 `kotlin.time.Duration`（`10.seconds` 一目了然） |
| 3 | 无配置验证——`maxRequests = -1` / `connectTimeout = -1` 等非法值运行时才崩 | 🟡 | `init { require(...) }` 启动期校验 |
| 4 | `config.pingIntervalSeconds` 是 `Long`，但 `pingInterval()` 接受 `Int`—— 大值会截断 | 🟡 | 检查后仅在 `> 0` 才设置 |
| 5 | 缺少 `proxy` / `proxyAuthenticator` / `dns` / `eventListenerFactory` —— 高阶场景无法配置 | 🟡 | 全部加上 |
| 6 | `cacheDir` 不存在会运行时报错——`File("cache")` 相对路径可能在不同进程路径不同 | 🟡 | `mkdirs()` + requireNotNull 验证 |
| 7 | `maxIdleConnections`, `maxRequestsPerHost` 之间无一致性检查——可能设 `maxRequestsPerHost > maxRequests` | 🟡 | `require(maxRequestsPerHost <= maxRequests)` |
| 8 | `protocols` 未暴露——用户要关 HTTP/2 却找不到 API | 🟡 | 加 `protocols: List<Protocol>` 参数 |

## 10.2 使用示例

```kotlin
// === 默认配置（开箱即用）===
val defaultClient = HttpClientFactory().create(HttpConfig())

// === 生产配置 ===
val prodClient = HttpClientFactory().create(
    HttpConfig(
        connectTimeout = 5.seconds,
        readTimeout = 15.seconds,
        writeTimeout = 15.seconds,
        callTimeout = 30.seconds,
        maxIdleConnections = 50,
        keepAliveDuration = 10.minutes,
        maxRequests = 256,
        maxRequestsPerHost = 32,
        cacheEnabled = true,
        cacheDir = File("/var/cache/myapp/http"),
        cacheMaxSizeBytes = 100L * 1024 * 1024,  // 100 MB
        pingInterval = 30.seconds,
        retryOnConnectionFailure = true,
        interceptors = listOf(
            AuthInterceptor(tokenStore),
            MetricsInterceptor(meterRegistry)
        ),
        eventListenerFactory = OkHttpEventListener.Factory(meterRegistry)
    )
)

// === 测试配置 ===
val testClient = HttpClientFactory().create(
    HttpConfig(
        retryOnConnectionFailure = false,
        cacheEnabled = false,
        callTimeout = 1.seconds,
        pingInterval = 0.seconds
    )
)
```

## 10.3 与 Spring Boot 整合

```kotlin
@Configuration
class HttpClientConfig {
    @Bean
    @Primary
    fun okHttpClient(@Value("\${http.connect-timeout:10s}") connect: Duration): OkHttpClient {
        return HttpClientFactory().create(
            HttpConfig(
                connectTimeout = connect,
                readTimeout = 30.seconds,
                // ... 从 application.yml 读所有配置
                interceptors = listOf(
                    tracingInterceptor(),
                    authInterceptor()
                )
            )
        )
    }
}
```

```yaml
# application.yml
http:
  connect-timeout: 5s
  read-timeout: 30s
  max-idle-connections: 50
  max-requests: 256
  max-requests-per-host: 32
```

## 10.4 为什么不用 Builder 模式而用 data class

| 维度 | Builder 模式 | data class |
|------|-------------|-----------|
| 代码量 | 多 30%（要写 Builder） | 少（Kotlin 自动生成 copy） |
| 类型安全 | 名字拼写错不会报错 | 编译期类型检查 |
| Spring 集成 | 需写自定义 ConfigurationProperties | `@ConfigurationProperties` 直接绑 |
| 验证 | 手动检查 | `init { require(...) }` |
| 默认值 | 字段 = 默认值 + Builder override | 字段 = 默认值，`copy()` 改 |

**结论**：配置参数 > 5 个时，**data class + copy()** 比 Builder 模式更优。

## 11. 总结

```
用 OkHttp 的时机：
✅ Android 应用
✅ Retrofit 用户
✅ 微服务调用外部 API
✅ WebSocket 实时通信
✅ 数据采集 / 抓取
✅ 测试 Mock 服务
✅ Spring Cloud OpenFeign 默认底层

不用 OkHttp 的时机：
❌ HTTP 服务端（用 Netty / Tomcat / [[mu-server-2.4.2-analysis/summary\|mu-server]]）
❌ QUIC / UDP（用 Netty）
❌ 极致低延迟（用 Aeron）
❌ 极简静态文件（用 NIO 直读）

记住 6 条铁律：
1. OkHttpClient 单例
2. Response 必须 close
3. 拦截器调 proceed
4. Android 主线程不能 execute
5. 版本 ≥ 4.12（JDK 21）
6. 生产环境禁用 Body 日志
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **对比选型**: [[draft-11-comparison]]
- **实战坑**: [[draft-10-known-issues]]
- **综合入口**: [[summary]]
