---
title: "OkHttp 拦截器链 (核心机制)"
category: synthesis
tags: [okhttp, interceptor, chain-of-responsibility, auth, logging, retry]
sources:
  - "OkHttp 4.12.0 源码: RealInterceptorChain.kt / BridgeInterceptor.kt / CacheInterceptor.kt / ConnectInterceptor.kt / CallServerInterceptor.kt"
  - "Square Wiki - Interceptors"
summary: "OkHttp 6 层拦截器链详解：4 层内置 + 2 类自定义 + 关键代码 + 鉴权/日志/重试实战"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §02 OkHttp 拦截器链（核心机制）

## 1. 为什么拦截器链是 OkHttp 的灵魂

OkHttp **所有高级能力**——鉴权、日志、重试、缓存策略、签名、链路追踪——**都通过拦截器实现**。这是它区别于 `HttpURLConnection` 的关键设计：

> `HttpURLConnection` 把所有能力写死在代码里，**无法扩展**；  
> OkHttp 用责任链模式把每层都暴露给用户，**全可插拔**。

## 2. 6 层拦截器管道

OkHttp 一次请求经过的拦截器（顺序固定）：

```
Request
   │
   ▼
┌────────────────────────────────────────┐
│ ① Application Interceptors             │  ← 用户添加：addInterceptor()
│    不受重定向/重试影响，看不到中间响应    │
├────────────────────────────────────────┤
│ ② BridgeInterceptor                    │  ← 内置
│    补充 Cookie/Accept-Encoding/User-Agent│
├────────────────────────────────────────┤
│ ③ CacheInterceptor                     │  ← 内置
│    HTTP 缓存命中直接返回 / 写回缓存      │
├────────────────────────────────────────┤
│ ④ ConnectInterceptor                   │  ← 内置
│    从 ConnectionPool 取/建 TCP 连接     │
├────────────────────────────────────────┤
│ ⑤ Network Interceptors                 │  ← 用户添加：addNetworkInterceptor()
│    看得到重定向/重试，每个网络调用一次     │
├────────────────────────────────────────┤
│ ⑥ CallServerInterceptor                │  ← 内置
│    实际写 HTTP 帧到 socket，读响应        │
└────────────────────────────────────────┘
   │
   ▼
Response (原路返回)
```

## 3. 两类自定义拦截器对比

| 维度 | `addInterceptor()` | `addNetworkInterceptor()` |
|------|---------------------|---------------------------|
| 触发时机 | 最早，**只调用一次**（即便重试） | 每次网络请求都调用（含重定向/重试） |
| 看到中间响应 | ❌ 看不到 | ✅ 看得到 |
| 看得到重试 | ❌ 看不到 | ✅ 看得到 |
| 适合场景 | **鉴权、日志（业务级）** | **网络级日志、重试、连接级监控** |
| 数量 | 可多个 | 可多个 |
| 执行顺序 | 按添加顺序 | 按添加顺序 |

## 4. Interceptor 接口

```kotlin
interface Interceptor {
    @Throws(IOException::class)
    fun intercept(chain: Chain): Response
    
    interface Chain {
        fun request(): Request
        fun proceed(request: Request): Response  // 调用下一层
        fun connection(): Connection?             // 当前连接（仅 network interceptor）
        fun call(): Call
        fun readTimeoutMillis(): Int
        fun withReadTimeout(timeout: Int, unit: TimeUnit): Chain
        // ... writeTimeout, connectTimeout, etc.
    }
}
```

### 4.1 最小骨架

```kotlin
class MyInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        // 前置处理（修改 request）
        val newRequest = request.newBuilder()
            .header("X-Timestamp", System.currentTimeMillis().toString())
            .build()
        
        // 调用下一层
        val response = chain.proceed(newRequest)
        
        // 后置处理（修改 response）
        return response.newBuilder()
            .header("X-Processed-By", "MyInterceptor")
            .build()
    }
}
```

⚠️ **`chain.proceed()` 必须调用**，否则请求中断。**必须返回 Response**，否则下游拿不到结果。

## 5. 实战模式 1：动态 Token 鉴权

```kotlin
class AuthInterceptor(
    private val tokenProvider: () -> String?  // 闭包，懒加载
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        // 跳过不需要鉴权的请求
        if (request.header("No-Auth") != null) {
            return chain.proceed(request.newBuilder()
                .removeHeader("No-Auth")
                .build())
        }
        
        val token = tokenProvider() ?: throw IOException("No token")
        val authed = request.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        
        return chain.proceed(authed)
    }
}

// 使用：可注入动态 token（OAuth refresh / JWT）
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor { tokenStore.getAccessToken() })
    .build()
```

> **为什么这个拦截器不用 try/finally**：
> - **没有需要清理的资源**（没开 response、没启 span、没启计时器）
> - `tokenProvider()` 抛错时 `?: throw` 直接传播，链路下游自然处理
> - 唯一建议：`tokenProvider()` 内部应**自身做缓存**（避免每次请求都查 DB）

### 5.1 进阶：401 自动刷新 Token

```kotlin
class TokenRefreshInterceptor(
    private val tokenStore: TokenStore,
    private val refreshClient: OkHttpClient
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)
        
        if (response.code != 401) return response
        
        // 401 → 尝试刷新 token
        synchronized(this) {
            try {
                val newToken = tokenStore.refreshToken() ?: run {
                    throw IOException("Token refresh failed")
                }
                
                val retried = request.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(retried)
            } finally {
                // 任何路径下都要关闭原 401 response（成功重发、refresh 失败、retried 抛错都触发）
                response.close()
            }
        }
    }
}
```

⚠️ **必须 close 上一个 Response**——必须用 `finally` 包裹，否则 refreshToken 抛错时连接会泄漏。

**为什么这个版本更稳**：

| 场景 | 旧写法 | 新写法 |
|------|--------|--------|
| 重发成功 | ✅ close | ✅ close（finally） |
| refreshToken 抛错 | ❌ **response 泄漏** | ✅ close（finally） |
| retried 请求抛 IOException | ❌ response 没关 | ✅ close（finally） |
| 线程被中断 | ❌ 资源未清理 | ✅ close（finally） |

## 6. 实战模式 2：请求日志

```kotlin
class LoggingInterceptor(
    private val logger: Logger
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val start = System.nanoTime()
        
        logger.info("→ ${request.method} ${request.url}")
        request.headers.forEach { (name, value) ->
            logger.info("  $name: $value")
        }
        
        return try {
            val response = chain.proceed(request)
            val took = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start)
            
            logger.info("← ${response.code} ${request.url} (${took}ms, ${response.protocol})")
            response.headers.forEach { (name, value) ->
                logger.info("  $name: $value")
            }
            response
        } catch (e: Exception) {
            val took = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start)
            logger.error("✗ ${request.url} (${took}ms) ${e.javaClass.simpleName}: ${e.message}")
            throw e
        }
    }
}
```

**生产推荐用 network interceptor**（看得到重试次数和最终网络耗时）。

**为什么包 try/catch**：
- **不包**：异常路径没日志，监控上看不到失败请求的耗时
- **包了**：失败也有日志（错误级别）+ 耗时上报，监控能区分"慢失败"和"快失败"

## 7. 实战模式 3：智能重试

```kotlin
class SmartRetryInterceptor(
    private val maxRetries: Int = 3
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var responseToReturn: Response? = null   // 即将 return 给调用方的（finally 不关）
        var responseToCleanup: Response? = null   // 5xx 等待重试的（finally 必须关）
        
        try {
            while (true) {
                // 清理上一轮遗留
                responseToCleanup?.close()
                responseToCleanup = null
                
                val response: Response = try {
                    chain.proceed(chain.request())
                } catch (e: IOException) {
                    if (attempt >= maxRetries) throw e
                    logger.warn("Retry ${attempt + 1}/$maxRetries on ${e.message}")
                    attempt++
                    sleep(backoff(attempt))
                    continue
                }
                
                // 成功路径判断：非 5xx 或已达最大重试
                if (response.code !in 500..599 || attempt >= maxRetries) {
                    // 移交所有权给调用方
                    responseToReturn = response
                    return response
                }
                
                // 5xx + 还能重试 → 标记为待清理
                logger.warn("Retry ${attempt + 1}/$maxRetries for ${response.code}")
                responseToCleanup = response
                attempt++
                sleep(backoff(attempt))
            }
            @Suppress("UNREACHABLE_CODE")
            error("unreachable")
        } finally {
            // 关键：只有未成功 return 的 response 才需要关闭
            // 已 return 的（responseToReturn != null）由调用方 RealCall 负责关闭
            if (responseToReturn == null) {
                responseToCleanup?.close()
            }
        }
    }
    
    private fun sleep(ms: Long) {
        try {
            Thread.sleep(ms)
        } catch (e: InterruptedException) {
            Thread.currentThread().interrupt()
            throw IOException("Retry interrupted", e)
        }
    }
    
    private fun backoff(attempt: Int): Long {
        // 指数退避 + 抖动：100ms, 200ms, 400ms, ... ± 20%
        val base = 100L * (1L shl (attempt - 1))
        val jitter = (base * 0.2 * Random.nextDouble()).toLong()
        return base + jitter
    }
}
```

⚠️ **指数退避 + 抖动** 是关键，否则雷鸣群（thundering herd）。

**为什么这个版本更稳（5 个修复点）**：

| # | 旧写法问题 | 新写法修复 |
|---|-----------|-----------|
| 1 | `Thread.sleep` 抛 `InterruptedException` 直接绕过重试逻辑 | `sleep()` 包 try/catch，恢复中断状态后抛 `IOException` |
| 2 | `response?.close()` 在 `try` 开头，但若 `proceed` 抛 `RuntimeException` 不被 catch，response 可能未初始化 | `try/finally` 在所有异常路径确保清理 |
| 3 | 最后一次 5xx 时 `return response`（5xx 状态），但**之前所有 5xx response 都已 close，最新这个也由调用方 close**——其实 OK | 但用 `responseToReturn` / `responseToCleanup` 显式区分所有权更清晰 |
| 4 | `lastException` 变量在最终抛出时再用，**所有重试都失败时 `while` 循环退出**——`return`/`throw` 后 `response?.close()` 不会执行（OK） | finally 兜底，无论如何路径退出都安全 |
| 5 | 复杂的状态变量（`response` / `lastException` / `attempt`）混在一起 | 拆成 `responseToReturn` 和 `responseToCleanup`，语义清晰 |

## 8. 实战模式 4：链路追踪（Trace ID）

```kotlin
class TracingInterceptor(
    private val tracer: Tracer  // OpenTelemetry / Sleuth / 自定义
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val span = tracer.startSpan(chain.request().url.toString())
        try {
            val request = chain.request().newBuilder()
                .header("X-Trace-Id", span.traceId())
                .header("X-Span-Id", span.spanId())
                .build()
            return chain.proceed(request)
        } catch (e: Exception) {
            span.recordException(e)
            throw e
        } finally {
            span.finish()  // 成功和失败都会执行
        }
    }
}
```

**为什么用 `finally` 而不是 `.also { span.finish() }`**：

| 写法 | 成功路径 | 异常路径 | 评价 |
|------|---------|---------|------|
| `.also { span.finish() }` 跟在 `proceed` 后面 | ✅ finish | ❌ 抛出前没 finish，**span 泄漏** | 失败时 span 不结束，上下文悬挂 |
| `try { ... } catch { recordException } finally { finish }` | ✅ finish | ✅ recordException + finish | 任何路径都关闭 span |
| `try { ... } catch { finish + recordException }` | ❌ 成功路径忘了 finish | ✅ | 容易漏成功路径 |

⚠️ **`finally` 里的 `span.finish()` 是必需的**——不 finish 的 span 会一直挂在 tracer 里，导致内存泄漏 + trace 永远不完整。

## 9. 实战模式 5：响应体 Peek（peekBody）

### 9.1 是什么

`peekBody(byteCount: Long)` 是 `ResponseBody` / `Response` 上的方法：**读取前 N 字节，但不会消费原始 body**——下游仍可正常读取。

```kotlin
val response = chain.proceed(request)
val preview = response.peekBody(2048)  // 读前 2KB
println("Body preview: ${preview.string()}")

// ✅ response.body() 仍可用
val fullBody = response.body?.string()  // 可被下游消费
```

### 9.2 解决了什么问题

| 场景 | 不用 peekBody | 用 peekBody |
|------|--------------|------------|
| 拦截器里读 body 记日志 | ❌ 读了后面就没了 → 下游崩溃 | ✅ 读前 N 字节作 preview，原 body 仍可用 |
| 重试决策（看 error message 再重试） | ❌ 读了就要 close response | ✅ peek 看完不消费 |
| 响应校验（看 body 类型/格式） | ❌ 同样问题 | ✅ peek 不影响后续 |
| 调试输出 body 限长 | 需手动限长 | ✅ peekBody 天然限制 |

### 9.3 完整示例：日志拦截器 v2（记 body preview）

```kotlin
class LoggingInterceptor(
    private val logger: Logger,
    private val maxPeekBytes: Long = 4096  // 默认 peek 前 4KB
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val start = System.nanoTime()
        
        logger.info("→ ${request.method} ${request.url}")
        request.headers.forEach { (n, v) -> logger.info("  $n: $v") }
        
        return try {
            val response = chain.proceed(request)
            val took = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start)
            
            // ✅ peekBody 不影响下游
            val bodyPreview = response.body?.let { body ->
                if (body.contentLength() == 0L) "(empty)" 
                else try {
                    body.peekBody(maxPeekBytes).string()
                } catch (e: Exception) {
                    "(peek failed: ${e.message})"
                }
            } ?: "(no body)"
            
            logger.info(
                "← ${response.code} ${request.url} " +
                "(${took}ms, ${response.protocol})\n" +
                "  Body preview (${minOf(maxPeekBytes, response.body?.contentLength() ?: 0)} bytes): " +
                bodyPreview
            )
            response
        } catch (e: Exception) {
            val took = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start)
            logger.error("✗ ${request.url} (${took}ms) ${e.javaClass.simpleName}: ${e.message}")
            throw e
        }
    }
}
```

### 9.4 实战示例：智能重试决策（peek + retry）

```kotlin
class ConditionalRetryInterceptor(
    private val maxRetries: Int = 2
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var response: Response? = null
        
        while (true) {
            response?.close()
            response = try {
                chain.proceed(chain.request())
            } catch (e: IOException) {
                if (attempt >= maxRetries) throw e
                logger.warn("Network error, retry ${attempt + 1}")
                attempt++
                Thread.sleep(backoff(attempt))
                continue
            }
            
            // 只有 503 + body 里说了 Retry-After 才重试
            if (response.code == 503 && attempt < maxRetries) {
                val retryAfterSec = response.peekBody(1024).use { body ->
                    // 假设服务端返回：{"error":"rate_limit","retry_after":5}
                    val text = body.string()
                    val match = Regex("\"retry_after\"\\s*:\\s*(\\d+)").find(text)
                    match?.groupValues?.get(1)?.toIntOrNull()
                }
                
                if (retryAfterSec != null) {
                    logger.warn("Server requested retry after ${retryAfterSec}s")
                    response.close()
                    Thread.sleep(retryAfterSec * 1000L)
                    attempt++
                    continue
                }
            }
            
            return response
        }
    }
}
```

### 9.5 实战示例：响应校验（peek 后分类）

```kotlin
class ContentTypeValidationInterceptor(
    private val expectedContentTypes: List<String>
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        
        val contentType = response.header("Content-Type") ?: ""
        val isJson = expectedContentTypes.any { contentType.contains(it, ignoreCase = true) }
        
        if (!isJson) {
            // ✅ peek 看 body 是不是真的不是 JSON（防止服务端返回 HTML 错误页）
            val preview = response.peekBody(512).string()
            logger.warn(
                "Unexpected content type for ${chain.request().url}: " +
                "Content-Type=$contentType, preview=$preview"
            )
        }
        
        return response
    }
}
```

### 9.6 peekBody vs body 关键对比

```kotlin
val response = chain.proceed(request)

// ❌ 错误：读 body 会被消费，下游崩
val bodyText = response.body?.string()
logger.debug("Response: $bodyText")
processBody(response.body)  // ❌ body 已被消费，IOException

// ✅ 正确：peek 不影响原始 body
val preview = response.peekBody(2048).string()
logger.debug("Response preview: $preview")
val fullBody = response.body?.string()  // ✅ 仍可读
processBody(response.body)               // ✅ body 仍可用
```

### 9.7 5 条实战铁律

1. **peekBody 必须指定 byteCount**——全量 peek 等于 body.string()
2. **peekBody 返回新 ResponseBody**——调用方需要负责 close（或用 `.use { }`）
3. **peekBody 只能 peek 一次**——底层 buffer 被读完后再 peek 会抛 EOFException
5. **peekBody 仅用于预览/决策**——不要依赖它读全量业务数据
6. **peekBody 对大 body 友好**——只读 N 字节，内存峰值是 N 而非整个 body

### 9.8 什么时候**不**用 peekBody

| 场景 | 不用 peekBody |
|------|-------------|
| 拦截器只是读响应码/响应头 | 不需要 body，用 `response.header(...)` |
| body 很小（< 1 KB） | 直接 `body.string()` 也可，但要 close response |
| 需要读完整 body 做业务处理 | `body.string()` 或 `body.bytes()`，交给下游 |
| 流式处理（不能缓存全 body） | `body.byteStream()`，不要 peek |

## 10. 关键代码解读：BridgeInterceptor

BridgeInterceptor 干的事："把用户的 Request 桥接成符合 HTTP/1.1 协议规范的 Request"。

```kotlin
// 简化版（BridgeInterceptor.kt）
class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val userRequest = chain.request()
        val builder = userRequest.newBuilder()
        
        // 1. Body 处理
        val body = userRequest.body
        if (body != null) {
            val contentType = body.contentType()
            if (contentType != null) builder.header("Content-Type", contentType.toString())
            builder.header("Content-Length", body.contentLength().toString())
            
            // 2. 自动 GZIP 压缩请求体
            if (body.isOneShot()) builder.header("Transfer-encoding", "chunked")
        }
        
        // 3. Host 头（如果用户没设）
        if (userRequest.header("Host") == null) builder.header("Host", hostHeader(userRequest.url, false))
        
        // 4. Connection 头（HTTP/1.1 默认 keep-alive）
        if (userRequest.header("Connection") == null) builder.header("Connection", "Keep-Alive")
        
        // 5. Accept-Encoding: gzip（自动）
        if (userRequest.header("Accept-Encoding") == null) {
            builder.header("Accept-Encoding", "gzip")
        }
        
        // 6. Cookie（如果有 CookieJar）
        if (cookieJar === CookieJar.NO_COOKIES) {
            builder.removeHeader("Cookie")
        } else {
            val cookies = cookieJar.loadForRequest(userRequest.url)
            if (cookies.isNotEmpty()) builder.header("Cookie", cookieHeader(cookies))
        }
        
        // 7. User-Agent
        if (userRequest.header("User-Agent") == null) builder.header("User-Agent", "okhttp/${OkHttp.VERSION}")
        
        val networkResponse = chain.proceed(builder.build())
        
        // 8. 自动解压 GZIP
        if (networkResponse.header("Content-Encoding") == "gzip"
            && !networkResponse.promisesBody()) {
            // ... 解压并替换 body
        }
        
        return networkResponse
    }
}
```

**关键点**：
- 用户几乎**不需要手写**这些 header，BridgeInterceptor 都补齐
- GZIP **自动**解压，应用层零感知
- 如果你想接管这些 header，自己显式 `header()` 即可

## 10. 拦截器链的执行（伪代码）

```kotlin
class RealInterceptorChain(
    private val interceptors: List<Interceptor>,
    private val index: Int = 0,
    // ...
) : Chain {
    override fun proceed(request: Request): Response {
        // 递归调用下一个
        val next = RealInterceptorChain(interceptors, index + 1, ...)
        val interceptor = interceptors[index]
        return interceptor.intercept(next)  // ← 责任链模式
    }
}

// RealCall.kt 启动：
val chain = RealInterceptorChain(
    interceptors = listOf(
        appInterceptors[0],
        appInterceptors[1],
        BridgeInterceptor(cookieJar),
        CacheInterceptor(cache),
        ConnectInterceptor(client),
        networkInterceptors[0],
        CallServerInterceptor(forWebSocket = false)
    )
)
val response = chain.proceed(originalRequest)
```

## 11. 拦截器执行顺序总结

**关键约束**：
- Application Interceptors **最先执行**（add 顺序）
- Network Interceptors **在 ConnectInterceptor 之后**、CallServer 之前
- 内置拦截器顺序**固定**（不能改）
- **同一类型内**按 add 顺序执行

## 12. 性能与陷阱

| 陷阱 | 影响 | 解决 |
|------|------|------|
| 拦截器里做耗时操作 | 拖慢每次请求 | 异步处理 / 缓存 |
| 拦截器里修改 body 但忘了重置 | body 被消费后空指针 | 用 `peekBody` 复制 |
| 拦截器里忘记 `chain.proceed()` | 请求挂起 | 必须调用 |
| 拦截器里 `response.close()` 漏掉 | 连接池泄漏 | 用 `.use { }` 块 |
| 重试拦截器没退避 | 雪崩 | 指数退避 + 抖动 |
| Token refresh 拦截器并发触发 | refresh 风暴 | 用 `synchronized` |

## 13. 自定义拦截器 Checklist

- [ ] 调 `chain.proceed()` 且只调一次（除非主动重试）
- [ ] 上一个 Response 关闭后再发新请求（重试场景）
- [ ] 不修改 `Request.body` 后直接 proceed（已消费）
- [ ] 异常时正确传播（不要吞 IOException）
- [ ] 考虑 Application vs Network 的差异
- [ ] 高频路径避免锁和阻塞 IO

## 14. 资源所有权速查表

> OkHttp 拦截器最常见的 bug 都来自「资源所有权不清晰」——下面这张表把 6 类资源的「创建/移交/清理」模式一次性理清。

| 资源 | 创建 | 移交/消费 | 清理时机 | 模式 | 错误示例 |
|------|------|---------|---------|------|---------|
| **`Response`** | `chain.proceed()` 返回 | return 给调用方 | finally 关闭未移交的 | `try { proceed → return } finally { close 未移交 }` | 忘 close → 连接池泄漏 |
| **`Span`（链路追踪）** | `tracer.startSpan()` | （无外部消费者） | finally 必 finish | `try { ... } catch { record } finally { finish }` | catch 里 finish → 成功路径漏 |
| **计时器** | `System.nanoTime()` | 记日志 | finally 必打印 | `try { proceed } finally { log 耗时 }` | 异常时无日志 |
| **`RequestBody`（流式）** | `asRequestBody()` | 进 proceed | OkHttp 自动关 | 无需手动处理 | 错误手动 close |
| **锁** | `synchronized(lock)` | 临界区 | 块退出 | 标准 `synchronized` 块 | finally 里 unlock（不要用 Lock） |
| **`CookieJar.loadForRequest`** | interceptor 里调用 | 进 proceed | 无状态 | 纯函数 | 加缓存要 finally 清 |

### 14.1 通用模板（所有「有资源」的拦截器都按这个写）

```kotlin
class MyInterceptor(...) : Interceptor {
    override fun intercept(chain: Chain): Response {
        // 1. 创建资源（如果需要）
        val span = tracer.startSpan(...)
        var responseToReturn: Response? = null   // 所有权标记
        var responseToCleanup: Response? = null   // 待清理标记
        
        try {
            // 2. 业务逻辑 + 可能抛错
            val response = chain.proceed(modifiedRequest)
            
            // 3. 决定移交还是重试
            if (shouldReturn(response)) {
                responseToReturn = response   // 标记：交给调用方
                return response
            } else {
                responseToCleanup = response  // 标记：等下关掉
                // ... 重试逻辑 ...
            }
        } catch (e: Exception) {
            // 4. 异常：记录 + 传播
            span.recordException(e)
            throw e
        } finally {
            // 5. 收尾：清理所有未移交的资源
            span.finish()              // span 总是要 finish
            if (responseToReturn == null) {
                responseToCleanup?.close()  // 未移交才关
            }
        }
    }
}
```

### 14.2 一行判断法

```
你的拦截器持有哪些资源？
  ├─ Response     → 必须 try/finally 关闭未移交的
  ├─ Span/Trace   → 必须 try/catch/finally（三段式）
  ├─ 计时器        → finally 必打日志
  └─ 都没有       → 不需要 try/finally（直接 chain.proceed）

是否可能重试/二次 proceed？
  ├─ 是 → 显式区分 responseToReturn vs responseToCleanup
  └─ 否 → 一个 response 变量 + finally close 也行

sleep()/await() 是否可能抛 InterruptedException？
  ├─ 是 → 必须 catch + Thread.currentThread().interrupt() + 包装成 IOException
  └─ 否 → 直接 sleep
```

### 14.3 三条铁律

1. **`chain.proceed()` 调用一次**——除非主动重试（重试里也只在前一个 Response close 后再 proceed）
2. **资源所有权重于简洁**——多写两行 `responseToReturn = ...` 比出 bug 强
3. **异常路径也要 finish/close**——监控和稳定性都靠这条

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **连接池**: [[draft-03-connection-pool]]
- **同步/异步**: [[draft-07-sync-async]]
- **实战踩坑**: [[draft-10-known-issues]]（含 401 鉴权 interceptor 实战）
- **综合入口**: [[summary]]
