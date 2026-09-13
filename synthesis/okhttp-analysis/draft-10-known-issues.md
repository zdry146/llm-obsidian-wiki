---
title: "OkHttp 已知坑与实战 (含亲历复盘)"
category: synthesis
tags: [okhttp, jdk-21, ipv6, sonarqube, debugging, troubleshooting, real-world]
sources:
  - "OkHttp 4.12.0 CHANGELOG (IPv6/IPv4 fix)"
  - "OkHttp GitHub Issues #7079, #7080 (JDK 21)"
  - "作者 Jenkins pipeline 实战踩坑 (2026-06-27)"
  - "Jenkins SonarQube Quality Gates plugin 源码"
summary: "OkHttp 实战坑点：JDK 21 IPv6/IPv4 bug、连接池泄漏、回调线程、auth 风暴、真实复盘"
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.85
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §10 OkHttp 已知坑与实战（含亲历复盘）

> 本章是 **2026-06-27 作者亲历的 Jenkins + SonarQube + OkHttp 401 bug 全程复盘**，配合其他常见坑的归类清单。

## 1. 【亲历复盘】Jenkins + SonarQube + OkHttp：401 → 连接失败的连锁踩坑

### 1.1 背景

- **Jenkins Pipeline** 调用 SonarQube Quality Gates 等待结果
- 使用 `waitForQualityGate()` step
- SonarQube 9.x 部署在 `127.0.0.1:9000`
- JDK 21 + OkHttp 早期 4.x

### 1.2 Bug 链时间线

```
Build #10 启动
   │
   ▼
[阶段 1] Pipeline 编译失败
   "WorkflowScript: 102: Expected a step @ line 102"
   def qg = waitForQualityGate()
   │
   ▼
【修复 1】waitForQualityGate() 是 pipeline step，必须在 script {} 块里
   移到 script { ... } 内
   │
   ▼
[阶段 2] Build 失败：4.8 秒结束
   "Expected a step" → Groovy 编译错
   │
   ▼
【修复 2】改用 wrap [\$class: 'BuildUser'] 管理 user → 改用 timeout()
   │
   ▼
[阶段 3] Pipeline 编译通过，开始执行
   │
   ▼
[阶段 4] waitForQualityGate() 拿到 task ID，但请求 /api/ce/task?id=... 失败
   │
   ▼
[阶段 5] 🎯 找到根因：401 Unauthorized！
   │
   │  根因：SonarQube plugin 的 waitForQualityGate() 不传 credentialsId 时，
   │       插件用的是已弃用的 serverAuthenticationToken（这里为空），
   │       所以请求是 anonymous → 401
   │
   ▼
【修复 3】显式传 credentialsId 参数
   waitForQualityGate(credentialsId: 'sonarqube-token')
   │
   ▼
[阶段 6] 🎉 不再 401，但出现新错：
   "Failed to connect to /127.0.0.1:9000"
   │
   │  根因：JDK 21 + 早期 OkHttp 4.x 的 IPv6/IPv4 解析 bug！
   │
   ▼
【修复 4】OkHttp 升级到 4.12.0+（JDK 21 修复版）
   │
   ▼
[阶段 7] 🎉 Push 成功 + Build 成功
```

### 1.3 关键代码片段

```groovy
// ❌ Bug 代码（Pipeline 编译错 + 401 + 连接失败）
def qg = waitForQualityGate()  // 编译错：必须在 script {} 块

// ✅ 修复后
node {
    stage('SonarQube') {
        sh 'mvn sonar:sonar'
    }
    script {
        timeout(time: 1, unit: 'HOURS') {
            def qg = waitForQualityGate(credentialsId: 'sonarqube-token')
            if (qg.status != 'OK') {
                error "Quality Gate failed: ${qg.status}"
            }
        }
    }
}
```

### 1.4 复盘要点

| 阶段 | 症状 | 根因 | 修复 |
|------|------|------|------|
| 1 | "Expected a step" | Groovy 编译 | 加 `script { }` 块 |
| 2 | 4.8s 失败 | 编译错 → 没到 Sonar 阶段 | 同上 |
| 3 | "Expected a step" | 同上 | 同上 |
| 4 | API 401 | `credentialsId` 未传 → anonymous | 加参数 |
| 5 | "Failed to connect to /127.0.0.1" | **JDK 21 + OkHttp IPv6/IPv4 bug** | **升 OkHttp 4.12+** |
| 6 | Build 成功 | - | - |

**教训**：
1. **不要相信"老工具自动工作"**——每个组件版本都要在 CI 验证
2. **JDK 升级是大版本变更**——升级 JDK 21 时，所有 HTTP 客户端要重新测试
3. **错误信息是有层次的**——先看 syntax，再看 auth，再看连接
4. **SonarQube plugin 文档不全**——必须读 plugin 源码才知道 `credentialsId` 是必需

### 1.5 作者复盘中的其他细节

- **GitHub PAT 失效**导致 `git push` 失败，但 API 调用（创建/删除 ref）能用——最终用 Jenkins `git-cred` credential 解密出来的 PAT 解决
- **`git credential-store erase` 清掉旧的坏凭证**——这是清理的关键步骤
- **OkHttp 4.11 修复不完整**——必须 4.12.0 才彻底修复 IPv6 bug

## 2. 【高频坑】连接池泄漏

### 2.1 现象

```
- 应用启动正常
- 跑一段时间后所有请求卡住
- 日志显示 connection pool exhausted
- okHttp.connectionPool.connectionCount() = maxIdleConnections
```

### 2.2 根因

```kotlin
// ❌ 错误：没 close Response
val response = client.newCall(request).execute()
val data = response.body?.string()
// connection.close() 没被调用，连接不归还池

// ✅ 正确：用 .use 块
client.newCall(request).execute().use { response ->
    val data = response.body?.string()
    // 自动 close → 连接归还
}
```

### 2.3 检测

```kotlin
val pool = client.connectionPool
val idle = pool.idleConnectionCount()
val total = pool.connectionCount()
println("idle=$idle, total=$total")

// 如果 total 持续增长但 idle 不增长 → 泄漏
```

### 2.4 修复

1. 启用 linter 检测 `Response` 未关闭（Detekt / ErrorProne）
2. 用 `kotlinx-coroutines` 的 `use { }`
3. 监控 `connectionCount`，超过阈值告警

## 3. 【高频坑】回调里抛异常被吞

### 3.1 现象

```
- 应用日志没异常
- 但下游数据没更新
- 监控看不到失败
```

### 3.2 根因

```kotlin
// OkHttp 的 Callback 实现自己负责异常处理
override fun onResponse(call: Call, response: Response) {
    processData(response.body!!.string())  // 抛异常被 OkHttp 吞掉
}
```

### 3.3 修复

```kotlin
override fun onResponse(call: Call, response: Response) {
    try {
        processData(response.body!!.string())
    } catch (e: Exception) {
        log.error("Process failed", e)  // 自己记录
        // 或者向上抛（OkHttp 会捕获但记录）
    }
}
```

## 4. 【高频坑】Android 主线程 execute()

### 4.1 现象

```
- 用户报告"应用卡死"
- ANR (Application Not Responding) 弹窗
- 用户被迫重启
```

### 4.2 根因

```kotlin
// ❌ Android 主线程调用 execute() → 阻塞 UI
class MainActivity : AppCompatActivity() {
    fun onCreate() {
        val response = client.newCall(request).execute()  // ANR
        textView.text = response.body!!.string()
    }
}
```

### 4.3 修复

```kotlin
// ✅ 用 enqueue + 切主线程
client.newCall(request).enqueue(object : Callback {
    override fun onResponse(call: Call, response: Response) {
        runOnUiThread { textView.text = response.body?.string() }
    }
})

// ✅ 更好：用协程
lifecycleScope.launch {
    val response = client.newCall(request).await()
    textView.text = response.body?.string()
}
```

## 5. 【高频坑】同步嵌套同步（线程耗尽）

### 5.1 现象

```
- 服务刚开始正常
- 高负载下大量超时
- 监控显示线程池占满
```

### 5.2 根因

```kotlin
// ❌ Tomcat/Spring 线程池串行阻塞调用
@GetMapping("/order")
fun getOrder(): Order {
    val user = client.newCall(userRequest).execute()  // 占 1 个线程
    val inventory = client.newCall(invRequest).execute()  // 仍占同一个线程（串行）
    val price = client.newCall(priceRequest).execute()
    // 200 个 Tomcat 线程 → 只能并发 200/3 ≈ 66 个订单请求
}

// ✅ 用 enqueue 或 CompletableFuture
@GetMapping("/order")
fun getOrder(): CompletableFuture<Order> {
    return CompletableFuture.supplyAsync({
        val user = client.newCall(userRequest).execute()
        // ...
    }, executor)
}
```

### 5.3 JDK 21 优化

```kotlin
// ✅✅ 最佳：JDK 21 虚拟线程
@GetMapping("/order")
suspend fun getOrder(): Order {
    // 虚拟线程挂起，不占平台线程
    val user = client.newCall(userRequest).execute()
    // ...
}
```

## 6. 【坑】DNS 缓存与负载均衡

### 6.1 现象

```
- 同一个域名解析到多个 IP
- 部分 IP 故障但请求仍发到故障 IP
```

### 6.2 根因

```kotlin
// OkHttp 默认用 InetAddress.getAllByName() 解析 DNS
// 但不缓存 DNS 结果（依赖系统缓存）
```

### 6.3 修复

```kotlin
// ✅ 自定义 DNS
val client = OkHttpClient.Builder()
    .dns(object : Dns {
        override fun lookup(hostname: String): List<InetAddress> {
            // 自定义解析逻辑（如轮询、就近）
            return customResolve(hostname)
        }
    })
    .build()
```

## 7. 【坑】HTTPS 证书校验失败

### 7.1 现象

```
- javax.net.ssl.SSLHandshakeException
- "Trust anchor for certification path not found"
```

### 7.2 常见原因

```kotlin
// ❌ 用自签证书没加载
val client = OkHttpClient.Builder()
    .sslSocketFactory(insecureSslSocketFactory, insecureX509TrustManager)  // ❌
    .hostnameVerifier { _, _ -> true }  // ❌
    .build()
```

### 7.3 修复

```kotlin
// ✅ 加载自签证书到 trustStore
val keyStore = KeyStore.getInstance(KeyStore.getDefaultType()).apply {
    load(inputStream, password)
}
val trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm()).apply {
    init(keyStore)
}
val sslContext = SSLContext.getInstance("TLS").apply {
    init(null, trustManagerFactory.trustManagers, null)
}

val client = OkHttpClient.Builder()
    .sslSocketFactory(sslContext.socketFactory, trustManagerFactory.trustManagers[0] as X509TrustManager)
    .build()
```

## 8. 【坑】大文件上传 OOM

### 8.1 现象

```
- 上传 1GB 文件
- 应用崩溃 OOM
```

### 8.2 根因

```kotlin
// ❌ 一次性读全部到内存
val body = File("bigfile.bin").asRequestBody("application/octet-stream".toMediaType())
val request = Request.Builder().url("https://api.example.com/upload").post(body).build()
```

### 8.3 修复

```kotlin
// ✅ 流式上传
val requestBody = object : RequestBody() {
    override fun contentType(): MediaType? = "application/octet-stream".toMediaType()
    override fun contentLength(): Long = -1L  // 未知长度
    
    override fun writeTo(sink: BufferedSink) {
        File("bigfile.bin").source().use { source ->
            sink.writeAll(source)
        }
    }
}
```

## 9. 【坑】Content-Length 为 -1

### 9.1 现象

```
- 服务端收不到 body
- "InvalidContentLengthException"
```

### 9.2 根因

```kotlin
// 当 contentLength() 返回 -1，OkHttp 用 chunked transfer-encoding
// 某些服务端（特别是老旧系统）不支持
```

### 9.3 修复

```kotlin
// ✅ 显式指定长度
val body = object : RequestBody() {
    override fun contentLength(): Long = file.length()
    // ...
}
```

## 10. 【坑】retryOnConnectionFailure 不该乱用

### 10.1 现象

```
- POST 请求重复发送
- 数据被处理两次
```

### 10.2 根因

```kotlin
// 默认 retryOnConnectionFailure = true
// 会重试 idempotent 方法（GET/HEAD/PUT/DELETE 等）
// 但 POST 不在 idempotent 列表
// 但某些连接错误会触发所有方法重试 → POST 可能被重发
```

### 10.3 修复

```kotlin
// ✅ 对幂等性要求高的 API 显式控制
val request = Request.Builder()
    .url("https://api.example.com/payment")
    .post(paymentBody)
    .header("Idempotency-Key", UUID.randomUUID().toString())  // 服务端去重
    .build()

// 或自定义重试拦截器（只在 idempotent 方法重试）
```

## 11. 【坑】Cookie 持久化

### 11.1 现象

```
- Cookie 丢失
- 用户登录状态丢失
```

### 11.2 修复

```kotlin
// ✅ 实现持久化 CookieJar
class PersistentCookieJar(context: Context) : CookieJar {
    private val store = mutableListOf<Cookie>()  // 替换为 SharedPreferences / DataStore
    
    override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>) {
        cookies.forEach { store.add(it) }
        persist()
    }
    
    override fun loadForRequest(url: HttpUrl): List<Cookie> {
        return store.filter { it.matches(url) }
    }
}
```

## 12. 【坑】System Proxy 干扰

### 12.1 现象

```
- 本地开发正常
- CI 环境所有请求都失败
```

### 12.2 根因

```
CI 环境（如 Jenkins）设了 `http.proxyHost` 系统属性
OkHttp 默认走系统代理 → 失败
```

### 12.3 修复

```kotlin
val client = OkHttpClient.Builder()
    .proxy(Proxy.NO_PROXY)  // 明确禁用代理
    .build()
```

## 13. 排查清单（实战 checklist）

| 症状 | 第一查 |
|------|--------|
| 连接失败 | OkHttp 版本是否 ≥ 4.12（JDK 21 bug 修复） |
| 401 Unauthorized | Token 是否正确传递（看 [[draft-02-interceptors]]） |
| 慢 | `Dispatcher.runningCallsCount()` 是否打满 |
| 内存泄漏 | `connectionPool.connectionCount()` 是否持续增长 |
| ANR | 主线程是否有 `execute()` |
| 性能差 | 是否用 HTTP/2？是否启用 GZIP？ |
| SSL 失败 | 证书加载逻辑？hostnameVerifier？ |

## 14. 调试工具

```kotlin
// 1. EventListener - 详细时序
val listener = object : EventListener() {
    override fun callStart(call: Call) { println("→ ${call.request().url}") }
    override fun dnsStart(call: Call, domainName: String) { println("  DNS: $domainName") }
    override fun connectStart(call: Call, inetSocketAddress: InetAddress, proxy: Proxy) { println("  Connect") }
    override fun responseHeadersEnd(call: Call, response: Response) { println("← ${response.code}") }
    override fun callEnd(call: Call) { println("✓ done") }
    override fun callFailed(call: Call, ioe: IOException) { println("✗ ${ioe.message}") }
}

val client = OkHttpClient.Builder()
    .eventListenerFactory { listener }
    .build()

// 2. MockWebServer - 隔离测试
val server = MockWebServer()
server.enqueue(MockResponse().setBody("hello"))
server.start()
val url = server.url("/test")
// ... 测试 ...

// 3. http2 调试
addNetworkInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.HEADERS })
```

## 15. 推荐升级路径

```
发现以下 bug，立即升级 OkHttp：

1. JDK 21 IPv6/IPv4 bug → 升 4.12.0+
2. CVE-2023-3635 (NTLM 解析长字符串) → 升 4.12.0
3. CVE-2021-0341 (TLS 主机名验证) → 升 4.9.1+
4. BouncyCastle 兼容性 → 升 4.10.0+
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **版本演进**: [[draft-09-evolution]]
- **综合入口**: [[summary]]
