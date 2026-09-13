---
title: "OkHttp 同步/异步模型"
category: synthesis
tags: [okhttp, async, dispatcher, threading, executor]
sources:
  - "OkHttp 4.12.0 源码: RealCall.kt / Dispatcher.kt / AsyncCall.kt"
  - "Java 21 Virtual Threads (JEP 444)"
summary: "OkHttp 同步 vs 异步：execute()/enqueue()、Dispatcher 线程池、并发控制、Java 21 协程"
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

# §07 OkHttp 同步/异步模型

## 1. 两种调用方式

```kotlin
val call = client.newCall(request)

// 同步（阻塞当前线程）
val response: Response = call.execute()

// 异步（不阻塞）
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) { /* ... */ }
    override fun onResponse(call: Call, response: Response) { /* ... */ }
})
```

**最关键差异**：
- `execute()` 在**调用者线程**上阻塞
- `enqueue()` 在**Dispatcher 线程池**里执行

## 2. Dispatcher 详解

**Dispatcher 是 OkHttp 的异步调度中心**——所有 `enqueue()` 都走它。

### 2.1 默认配置

```kotlin
class Dispatcher {
    var maxRequests = 64                 // 总并发上限
    var maxRequestsPerHost = 5           // 单 host 并发上限
    var idleCallback: Runnable? = null
    
    private val executorService = ThreadPoolExecutor(
        0, Int.MAX_VALUE,                // core=0, max 无限
        60, TimeUnit.SECONDS,
        SynchronousQueue(),
        ThreadFactory { r -> 
            Thread(r, "OkHttp Dispatcher").apply { isDaemon = true }
        }
    )
}
```

### 2.2 两个队列

```kotlin
// 等待中的请求
private val readyAsyncCalls = ArrayDeque<AsyncCall>()

// 正在执行的请求
private val runningAsyncCalls = ArrayDeque<AsyncCall>()
```

### 2.3 promoteAndExecute 核心逻辑

```kotlin
private fun promoteAndExecute(): Boolean {
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning = AtomicBoolean(false)
    
    synchronized(this) {
        for (i in readyAsyncCalls.indices) {
            val asyncCall = readyAsyncCalls[i]
            
            // 检查：总并发上限
            if (runningAsyncCalls.size >= maxRequests) break
            // 检查：单 host 上限
            if (runningAsyncCalls.count { it.host == asyncCall.host } >= maxRequestsPerHost) break
            
            readyAsyncCalls.remove(asyncCall)
            executableCalls.add(asyncCall)
            runningAsyncCalls.add(asyncCall)
        }
        isRunning.set(true)
    }
    
    // 真正执行
    for (i in executableCalls.indices) {
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }
    
    return isRunning.get()
}
```

### 2.4 单 host 限流的必要性

```
❌ 不限流：对 1 个 host 发 100 个并发请求
   → 雪崩效应：服务端过载 → 大量超时 → 全部失败

✅ 限流：单 host 最多 5 并发
   → 渐进式：失败 5 个 → 重试 5 个 → 失败再重试
```

## 3. RealCall 的同步路径

```kotlin
override fun execute(): Response {
    synchronized(this) {
        check(!executed) { "Already Executed" }
        executed = true
    }
    
    // 1. 通知 Dispatcher 开始（追踪超时）
    client.dispatcher.executed(this)
    
    try {
        // 2. 同步执行整个拦截器链
        val result = getResponseWithInterceptorChain()
        // 3. 如果失败，自动回调 onFailure
        return result
    } catch (e: Throwable) {
        // 通知 dispatcher 失败
        throw e
    } finally {
        // 4. 通知 Dispatcher 完成
        client.dispatcher.finished(this)
    }
}
```

⚠️ **`execute()` 不走 Dispatcher 线程池**——它在调用者线程跑。

## 4. AsyncCall 的异步路径

```kotlin
internal inner class AsyncCall(...) : Runnable {
    override fun run() {
        var success: Boolean
        try {
            val response = getResponseWithInterceptorChain()
            // 4. 成功回调
            callback.onResponse(this@RealCall, response)
            success = true
        } catch (e: IOException) {
            // 5. 失败回调
            callback.onFailure(this@RealCall, e)
            success = false
        } finally {
            // 6. 通知 Dispatcher 完成
            client.dispatcher.finished(this)
        }
    }
    
    fun executeOn(executorService: ExecutorService) {
        var success = false
        try {
            executorService.execute(this)  // 在 Dispatcher 池里执行
            success = true
        } finally {
            if (!success) {
                client.dispatcher.finished(this)
            }
        }
    }
}
```

## 5. 时序图对比

### 5.1 同步

```
Thread-1
   │
   ▼ call.execute()
   │
   │  (Thread-1 阻塞)
   │
   ▼ InterceptorChain.proceed()
   │   ... 实际网络 I/O ...
   ▼ Response
   │
   ▼ (Thread-1 继续)
```

### 5.2 异步

```
Thread-1                    Dispatcher Pool (线程 X)
   │                                │
   ▼ call.enqueue(cb)                │
   │                                 │
   │                            ┌────▼────┐
   │                            │ InterceptorChain
   │                            │  ... I/O ...
   │                            └────┬────┘
   │                                 │
   ▼ (Thread-1 立即返回)              │ cb.onResponse(response)
   │                                 │
   │                                 ▼ cb.onFailure (if err)
   │                                 │
```

## 6. Android 上的血泪教训

```kotlin
// ❌ 致命错误：主线程 execute() → ANR
class MainActivity : AppCompatActivity() {
    fun onCreate() {
        val response = client.newCall(request).execute()  // 主线程阻塞 5s
        // 用户等待 5 秒 → "应用无响应" 弹窗 → 强制退出
    }
}

// ✅ 正确：用 enqueue
class MainActivity : AppCompatActivity() {
    fun onCreate() {
        client.newCall(request).enqueue(object : Callback {
            override fun onResponse(call: Call, response: Response) {
                runOnUiThread { updateUI(response.body?.string()) }
            }
            override fun onFailure(call: Call, e: IOException) {
                runOnUiThread { showError(e) }
            }
        })
    }
}

// ✅ 更好：用协程（kotlinx-coroutines + okhttp-coroutines）
class MainActivity : AppCompatActivity() {
    fun onCreate() {
        lifecycleScope.launch {
            try {
                val response = client.newCall(request).await()  // 协程 suspend
                updateUI(response.body?.string())
            } catch (e: IOException) {
                showError(e)
            }
        }
    }
}
```

## 7. 后端场景：JDK 21 虚拟线程

```kotlin
// JDK 21+: 用虚拟线程做 "同步" 调用
val executor = Executors.newVirtualThreadPerTaskExecutor()

fun handleRequest(req: HttpRequest) {
    executor.submit {
        try {
            // OkHttp 同步调用 → 虚拟线程挂起 → 不占平台线程
            val response = client.newCall(okRequest).execute()
            sendResponse(req, response.body?.string() ?: "")
        } catch (e: IOException) {
            sendError(req, e)
        }
    }
}
```

**关键点**：
- 虚拟线程下，**同步调用是首选**（代码更简单）
- 同步代码下没有回调地狱
- 虚拟线程**不被阻塞**——释放平台线程给其他请求

## 8. OkHttp 5.0 协程支持

```kotlin
// okhttp-coroutines 模块
implementation("com.squareup.okhttp3:okhttp-coroutines:5.0.0-alpha.14")

// 使用
suspend fun fetch(): String {
    val request = Request.Builder().url("https://api.example.com").build()
    return client.newCall(request).await().body!!.string()
}

// 异步转 suspend：await()
// 同步转 suspend：executeAsync()
```

⚠️ **5.0 仍是 alpha**，生产慎用。

## 9. 并发调优

### 9.1 高 QPS 微服务

```kotlin
val dispatcher = Dispatcher().apply {
    maxRequests = 256                    // 提高总并发
    maxRequestsPerHost = 64              // 提高单 host
}

val client = OkHttpClient.Builder()
    .dispatcher(dispatcher)
    .connectionPool(ConnectionPool(100, 5, TimeUnit.MINUTES))
    .build()
```

### 9.2 严格限流（防压垮下游）

```kotlin
val dispatcher = Dispatcher().apply {
    maxRequests = 10                     // 极低总并发
    maxRequestsPerHost = 2               // 严格单 host
}
```

### 9.3 突发场景

```kotlin
val dispatcher = Dispatcher().apply {
    maxRequests = 1000
    maxRequestsPerHost = 100
}
```

## 10. 监控指标

```kotlin
// 当前并发
val runningCount = dispatcher.runningCallsCount()
val queuedCount = dispatcher.queuedCallsCount()

// 监控线程（每 5s 输出）
val monitor = ScheduledExecutor()
monitor.scheduleAtFixedRate({
    logger.info("running=$runningCount, queued=$queuedCount")
}, 0, 5, TimeUnit.SECONDS)
```

## 11. 常见陷阱

### 11.1 异步回调里访问 UI

```kotlin
// ❌ Android 异步回调不在主线程
override fun onResponse(call: Call, response: Response) {
    textView.text = response.body?.string()  // 崩溃：UI 只能在主线程访问
}

// ✅ 切到主线程
override fun onResponse(call: Call, response: Response) {
    runOnUiThread { textView.text = response.body?.string() }
}
```

### 11.2 异步回调里抛异常被吞

```kotlin
// ❌ Callback 默认不会向上抛异常
override fun onResponse(call: Call, response: Response) {
    throw RuntimeException("oops")  // 被 OkHttp 静默捕获
}

// ✅ 自己包 try-catch
override fun onResponse(call: Call, response: Response) {
    try {
        process(response)
    } catch (e: Exception) {
        log.error("Process failed", e)
    }
}
```

### 11.3 同步嵌套同步

```kotlin
// ❌ 严重问题：8 个线程被占住
fun handleReq() {
    val r1 = client.newCall(req1).execute()  // 阻塞线程 1
    val r2 = client.newCall(req2).execute()  // 阻塞线程 1（串行）
    // 如果 Tomcat 只有 200 个线程，串行 200 个这样的调用 = 服务挂
}

// ✅ 用 enqueue 或 join
fun handleReq() {
    val c1 = client.newCall(req1).executeAsync()  // 不阻塞
    val c2 = client.newCall(req2).executeAsync()
    // 配合 CountDownLatch 等待完成
}
```

## 12. 线程安全总结

| 对象 | 线程安全？ | 说明 |
|------|-----------|------|
| `OkHttpClient` | ✅ | 不可变 |
| `Request` / `Response` | ✅ | 不可变 |
| `Call` | ⚠️ 单次执行安全 | 不能并发 execute + enqueue |
| `Dispatcher` | ✅ | 内部同步 |
| `ConnectionPool` | ✅ | 内部同步 |

## 13. 设计哲学总结

1. **同步简单，异步非必需** —— 小服务用同步足矣
2. **Dispatcher 是异步的** —— 不要自己创建 ExecutorService
3. **限流保护下游** —— `maxRequestsPerHost` 是底线
4. **协程是未来** —— 5.x 起 OkHttp 一等公民
5. **不要阻塞主线程** —— Android 上是红线

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **连接池**: [[draft-03-connection-pool]]
- **综合入口**: [[summary]]
