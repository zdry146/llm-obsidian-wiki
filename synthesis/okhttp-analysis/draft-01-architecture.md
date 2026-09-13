---
title: "OkHttp 核心架构"
category: synthesis
tags: [okhttp, architecture, okhttpclient, request, response, call, dispatcher]
sources:
  - "OkHttp 4.12.0 源码: OkHttpClient.kt / Request.kt / Response.kt / Call.kt / Dispatcher.kt"
  - "Square Engineering Blog - OkHttp 设计"
summary: "OkHttp 五件套 OkHttpClient/Request/Response/Call/Dispatcher 详解，附同步/异步调用时序图"
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

# §01 OkHttp 核心架构

## 1. 五件套

OkHttp 的所有能力都通过 5 个核心类暴露：

| 类 | 角色 | 关键属性 |
|----|------|---------|
| **`OkHttpClient`** | 工厂 + 配置中心 | 拦截器、连接池、Dispatcher、超时、TLS、协议 |
| **`Request`** | 不可变 HTTP 请求 | URL、Method、Headers、Body、Tags |
| **`Response`** | 不可变 HTTP 响应 | Code、Headers、Body、Handshake（TLS）、CacheResponse |
| **`Call`** | 单次请求/响应对话 | execute() / enqueue() / cancel() |
| **`Dispatcher`** | 异步调度器 | 线程池、并发上限、host 限流 |

## 2. OkHttpClient — 配置中心

**关键原则**：**一个应用只应该有一个 OkHttpClient 实例**（共享连接池、线程池、缓存）。

```kotlin
val client = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .callTimeout(60, TimeUnit.SECONDS)              // 整调用总超时（含重试）
    .retryOnConnectionFailure(true)                 // 默认 true
    .followRedirects(true)                          // 默认 true
    .followSslRedirects(true)
    .connectionPool(ConnectionPool(5, 5, TimeUnit.MINUTES))  // 5 空闲连接 / 5 min
    .dispatcher(Dispatcher())                       // 自定义异步调度
    .addInterceptor(AuthInterceptor(tokenProvider)) // 应用层拦截器
    .addNetworkInterceptor(LoggingInterceptor())    // 网络层拦截器
    .cache(Cache(File("cache"), 10L * 1024 * 1024)) // 10 MB 磁盘缓存
    .build()
```

### 2.1 4 级超时（关键概念）

OkHttp 提供 4 个超时维度，**互不冲突、层层递进**：

```
connectTimeout (TCP/TLS 握手)
        ↓
writeTimeout  (请求体写完)
        ↓
readTimeout   (响应读完)
        ↓
callTimeout   (整调用完成 - 包括重试)  ← 最外层
```

**最佳实践**：
- `connectTimeout` 设短（5-10s），连接失败应快速失败
- `readTimeout` 设长（30-60s），避免大文件下载误杀
- `callTimeout` 设上限（60s），防止无限重试

### 2.2 不可变 + Builder 模式

`OkHttpClient` 是**不可变对象**——一旦 build，所有配置冻结。如果需要改配置：

```kotlin
val newClient = client.newBuilder()
    .readTimeout(60, TimeUnit.SECONDS)
    .build()
```

`newBuilder()` 复用所有现有配置，只覆盖修改的部分。

## 3. Request / Response — 不可变数据

### 3.1 Request

```kotlin
val request = Request.Builder()
    .url("https://api.example.com/users/123")
    .header("Authorization", "Bearer $token")
    .header("X-Request-Id", UUID.randomUUID().toString())
    .get()                                          // 默认就是 GET
    .tag<String>("user:123")                        // 自定义标签，cancel 时用
    .build()
```

### 3.2 RequestBody

```kotlin
// JSON
val jsonBody = RequestBody.create(
    """{"name":"mike"}""",
    "application/json".toMediaType()
)

// Form
val formBody = FormBody.Builder()
    .add("username", "mike")
    .add("password", "secret")
    .build()

// Multipart (文件上传)
val multipart = MultipartBody.Builder()
    .setType(MultipartBody.FORM)
    .addFormDataPart("file", "test.txt",
        File("test.txt").asRequestBody("text/plain".toMediaType()))
    .build()
```

### 3.3 Response

```kotlin
val response: Response = client.newCall(request).execute()

response.code                      // 200
response.message                   // "OK"
response.headers                   // Headers (多值)
response.body?.string()            // 字符串（消耗 body 后不可再读）
response.body?.bytes()             // 字节数组
response.body?.byteStream()        // InputStream（大文件）
response.handshake                 // TLS 握手信息（含 cipher suite）
response.protocol                  // HTTP/1.1 / HTTP_2
response.receivedResponseAtMillis  // 接收时间戳（调试用）
response.sentRequestAtMillis       // 发送时间戳
```

⚠️ **必须 close Response** —— 不关会导致连接泄漏（连接池无法复用）。

```kotlin
client.newCall(request).execute().use { response ->
    // 自动 close
    println(response.body?.string())
}
```

## 4. Call — 一次请求的生命周期

```kotlin
// 同步
val call: Call = client.newCall(request)
val response = call.execute()           // 阻塞当前线程

// 异步
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) { /* 重试 */ }
    override fun onResponse(call: Call, response: Response) { /* 处理 */ }
})

// 取消
call.cancel()                           // 配合 tag 批量取消
```

### 4.1 tag 机制 — 批量管理 Call

```kotlin
// 打 tag
val request = Request.Builder()
    .url("https://api.example.com/users")
    .tag("query-user")
    .build()

// 批量取消（Activity 销毁时）
for (call in client.dispatcher.queuedCalls() + client.dispatcher.runningCalls()) {
    if ("query-user" == call.request().tag()) {
        call.cancel()
    }
}
```

## 5. Dispatcher — 异步调度器

**Dispatcher** 是 OkHttp 的"线程池管理员"，**所有异步请求都经过它**。

### 5.1 默认参数

```kotlin
class Dispatcher {
    var maxRequests = 64                 // 总并发上限
    var maxRequestsPerHost = 5           // 单 host 并发上限
    var idleCallback: Runnable? = null
    
    // 内部线程池
    private val executorService: ExecutorService =
        ThreadPoolExecutor(
            0, Int.MAX_VALUE,           // core=0, max=unlimited
            60, TimeUnit.SECONDS,
            SynchronousQueue(),
            "OkHttp Dispatcher"
        )
}
```

### 5.2 调度策略

```
新请求到达
    ↓
当前并发 < maxRequests (64) ?
    ├─ 是 → 当前 host 并发 < maxRequestsPerHost (5) ?
    │        ├─ 是 → 立即执行
    │        └─ 否 → 等待（queuedCalls）
    └─ 否 → 等待（queuedCalls）
    
有 call 完成 → 触发 idleCallback → 从队列取下一个
```

**为什么限制单 host 5 个？**——防止对单个 server 形成"连接雪崩"。

### 5.3 同步调用不走 Dispatcher

⚠️ `call.execute()` 是**同步**的，**不经过 Dispatcher**！它直接占用调用者线程。如果你在主线程（Android）或 Tomcat 请求线程里 `execute()`，会**阻塞业务**。

**最佳实践**：
- 后端：可以 `execute()`，但建议用虚拟线程（Java 21+）包装
- Android：必须用 `enqueue()`

## 6. 完整调用时序图

```
client.newCall(request)
   │
   ▼
RealCall (Call 实现)
   │
   ├─ execute() ──────────────────────┐
   │                                  │ (同步)
   │                                  ▼
   │                          InterceptorChain.proceed()
   │                                  │
   │                                  ▼
   │                          BridgeInterceptor
   │                                  │
   │                                  ▼
   │                          CacheInterceptor
   │                                  │
   │                                  ▼
   │                          ConnectInterceptor
   │                                  │
   │                                  ▼
   │                          CallServerInterceptor
   │                                  │
   │                                  ▼
   │                          Server (HTTP)
   │                                  │
   │                                  ▼
   │                          Response ← 原路返回
   │                                  │
   └─ enqueue(callback) ───────────────┤
                                      ▼
                                Dispatcher.executorService
                                      │
                                      ▼
                                (异步线程) → 同 execute 流程
                                      │
                                      ▼
                                callback.onResponse / onFailure
```

## 7. 关键设计原则总结

1. **不可变优先** —— Request/Response/OkHttpClient 都是不可变的，线程安全
2. **Builder 模式** —— 配置灵活，避免构造器爆炸
3. **拦截器链** —— 所有扩展点都通过它暴露（详见 [[draft-02-interceptors]]）
4. **Dispatcher 隔离同步/异步** —— 同步调用不走线程池，异步走
5. **连接池复用** —— TCP 连接跨请求复用（详见 [[draft-03-connection-pool]]）
6. **Okio 抽象** —— I/O 不直接用 NIO（详见 [[draft-08-okio]]）

## 8. 常见反模式

❌ **每次请求 new OkHttpClient()**——浪费连接池、线程池、配置  
❌ **不 close Response**——连接池泄漏，最终所有连接被占满  
❌ **在主线程 execute()**（Android）——ANR  
❌ **同步调用嵌套同步调用**——线程耗尽  
❌ **tag 用可变对象**——tag 用于 cancel 标识，必须稳定

## 相关笔记

- **拦截器链**: [[draft-02-interceptors]]
- **连接池**: [[draft-03-connection-pool]]
- **同步/异步**: [[draft-07-sync-async]]
- **Okio 底层**: [[draft-08-okio]]
- **综合入口**: [[summary]]
