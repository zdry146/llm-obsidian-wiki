---
title: "OkHttp 缓存机制"
category: synthesis
tags: [okhttp, cache, http-cache, disk-lru, cache-strategy]
sources:
  - "OkHttp 4.12.0 源码: Cache.kt / CacheStrategy.kt / DiskLruCache.kt / CacheInterceptor.kt"
  - "RFC 7234 - HTTP/1.1 Caching"
summary: "OkHttp HTTP 缓存：DiskLruCache 存储 + CacheStrategy 决策 + HTTP 缓存语义"
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

# §05 OkHttp 缓存机制

## 1. 为什么需要缓存

| 场景 | 缓存收益 |
|------|---------|
| 重复请求相同 URL | 节省网络流量 + 提速 100x |
| 离线模式 | 核心场景 |
| 弱网 | 减少超时失败 |
| 节省服务器压力 | CDN 边缘节点同样思路 |

## 2. 三种缓存粒度

OkHttp 的"缓存"是 **HTTP 协议级缓存**（不是结果缓存）：

| 粒度 | 说明 | OkHttp 支持 |
|------|------|-------------|
| **内存缓存** | 单次进程内有效 | ❌（无显式 API） |
| **磁盘缓存** | 跨进程、跨重启 | ✅ `Cache` 类 |
| **HTTP 协议缓存** | 遵守 HTTP 头语义 | ✅ CacheInterceptor |

## 3. 快速启用

```kotlin
val cacheDir = File(context.cacheDir, "http-cache")
val cacheSize = 10L * 1024 * 1024  // 10 MB

val client = OkHttpClient.Builder()
    .cache(Cache(cacheDir, cacheSize))
    .build()
```

**就这样**——后续 GET 请求如果服务端响应有合适的 `Cache-Control` 头，OkHttp 自动缓存。

## 4. HTTP 缓存语义（30 秒回顾）

### 4.1 缓存新鲜度（Freshness）

| Cache-Control 头 | 含义 |
|------------------|------|
| `max-age=3600` | 3600 秒内用缓存 |
| `no-cache` | 必须重新验证（不一定不用缓存） |
| `no-store` | 完全不缓存 |
| `private` | 只浏览器缓存，CDN 不缓存 |
| `public` | 任何缓存都可保存 |

### 4.2 验证（Revalidation）

缓存过期后，客户端发请求时加 `If-None-Match`（ETag）或 `If-Modified-Since`（Last-Modified）：

```
客户端 → If-None-Match: "abc123"
服务端 → 304 Not Modified  (内容没变)
或
服务端 → 200 + 新内容
```

## 5. CacheInterceptor 流程（核心）

```kotlin
class CacheInterceptor(private val cache: Cache?) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val request = chain.request()
        
        // 1. 检查缓存是否命中
        val cacheCandidate = cache?.get(request)
        
        // 2. 决定策略：命中 / 网络 / 验证 / 不缓存
        val strategy = CacheStrategy.Factory(now, request, cacheCandidate).compute()
        
        // 3. 不缓存场景
        if (cacheCandidate != null && strategy.cacheResponse == null) {
            closeQuietly(cacheCandidate.body)  // 不缓存 → 关闭
        }
        
        // 4. 只用缓存，不发请求（理论场景）
        if (cacheCandidate != null && strategy.networkRequest == null) {
            return cacheCandidate.newBuilder()
                .cacheResponse(stripBody(cacheCandidate))
                .build()
        }
        
        // 5. 正常发请求
        val networkResponse = chain.proceed(strategy.networkRequest!!)
        
        // 6. 写入缓存（如果服务端允许）
        if (cache != null) {
            val cacheable = strategy.cacheResponse != null
            if (cacheable && networkResponse.body != null) {
                // 包装 body 写入磁盘
                val cacheRequest = cache.put(strategy.cacheResponse!!.combineHeaders(networkResponse))
                return cacheWritingResponse(cacheRequest, networkResponse)
            }
        }
        
        return networkResponse
    }
}
```

## 6. CacheStrategy 决策表

| 场景 | Cache-Control | 策略 |
|------|---------------|------|
| 完全新鲜 | `max-age=N` 未过期 | **只用缓存**，不发请求 |
| 过期但有 ETag | `max-age=0` + ETag | 发请求验证，`If-None-Match` |
| 过期但有 Last-Modified | `max-age=0` + Last-Modified | 发请求验证，`If-Modified-Since` |
| 服务端说不缓存 | `no-store` | 完全不用缓存 |
| 请求说不缓存 | `Cache-Control: no-store` | 完全不用缓存 |
| POST 等非幂等方法 | - | **永不缓存**（除非显式） |

⚠️ **重要**：POST/PATCH/DELETE **默认不缓存**——除非服务端显式声明。

## 7. CacheResponse 字段

Response 上的 `cacheResponse()` 让你区分响应来源：

```kotlin
val response = client.newCall(request).execute()

when {
    response.cacheResponse != null && response.networkResponse == null -> "纯缓存"
    response.cacheResponse != null && response.networkResponse != null -> "缓存+验证"
    response.networkResponse != null && response.cacheResponse == null -> "网络直取"
}
```

## 8. 强制使用网络 / 强制使用缓存

```kotlin
// 强制走网络（跳过缓存）
val networkRequest = request.newBuilder()
    .cacheControl(CacheControl.Builder().noCache().build())  // 或 .noStore()
    .build()

// 强制只用缓存（如果有），不发请求
val cacheOnlyRequest = request.newBuilder()
    .cacheControl(CacheControl.Builder()
        .onlyIfCached()
        .maxAge(Integer.MAX_VALUE, TimeUnit.SECONDS)  // 必须 maxStale 才用
        .build())
    .build()
```

⚠️ `onlyIfCached` 如果缓存没有 → 直接抛 `IOException`，**不会尝试网络**。

## 9. 缓存预热 / 主动清除

```kotlin
val cache = client.cache!!

// 主动删除单个 URL 的缓存
cache.remove(request)

// 清空所有缓存
cache.evictAll()  // 强制清空

// 遍历缓存（调试用）
cache.urls().forEach { url ->
    println(url)
}

// 关闭缓存（必须调用，否则进程退出时数据可能丢失）
cache.close()
```

## 10. 缓存目录结构

```
/data/data/your.app/cache/http-cache/
├── journal          ← 操作日志（类似 LSM 的 WAL）
├── 0/               ← 数据分片
├── 1/
├── 2/
├── ...
└── _other/
```

每个 URL 缓存一个 entry，包含：
- URL（key）
- 请求方法
- Vary headers（决定 key 是否唯一）
- 响应码、响应头
- 响应 body

### 10.1 journal 格式

```
libcore.io.DiskLruCache
1
100
2

CLEAN 1 1234
DIRTY 2
REMOVE 1
READ 2 5678
```

类似 LSM 树的 WAL，保证崩溃一致性。

## 11. 高级用法：自定义 CacheStrategy

如果默认 `CacheStrategy.Factory` 不够用，可以**自己实现策略**：

```kotlin
class CustomCacheInterceptor(private val cache: Cache) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val request = chain.request()
        
        // 自定义：POST 也缓存 1 分钟
        if (request.method == "POST") {
            val cacheControl = CacheControl.Builder()
                .maxAge(1, TimeUnit.MINUTES)
                .build()
            return chain.proceed(request.newBuilder().cacheControl(cacheControl).build())
        }
        
        return chain.proceed(request)
    }
}
```

## 12. 缓存陷阱

### 12.1 敏感数据泄漏

```kotlin
// ❌ 危险：把用户个人信息缓存到磁盘
val request = Request.Builder()
    .url("https://api.example.com/users/me")  // 包含用户隐私
    .build()

// ✅ 敏感请求加 no-store
val request = Request.Builder()
    .url("https://api.example.com/users/me")
    .cacheControl(CacheControl.Builder().noStore().build())
    .build()
```

### 12.2 缓存污染

服务端 Bug 导致返回 `max-age=31536000`（1 年）——永久缓存旧数据。**应对**：监控缓存命中率 + 强制更新机制。

### 12.3 跨用户缓存

OkHttp 的 Cache **不做用户隔离**——A 用户的数据可能被 B 用户读到（如果同一进程）。**应对**：在 URL 里加 token / 在 key 里加用户 ID。

## 13. 性能收益实测

| 场景 | 无缓存 | 有缓存 | 提升 |
|------|--------|--------|------|
| 重复请求 1000 次 | 50s | 0.5s | **100x** |
| 弱网请求 | 经常失败 | 命中就用 | 成功率 ↑ 90% |
| 流量 | 满量 | 减少 80-95% | 显著 |

## 14. 与 Retrofit / Coil 的关系

- **Retrofit** 没有自己的缓存——直接复用 OkHttp 的
- **Coil**（Android 图片库）有自己的内存 + 磁盘缓存，OkHttp 仅作网络层
- **Volley**（旧 Android）有自己的缓存，**不通用**

## 15. 最佳实践 Checklist

- [ ] 设置合理的缓存目录（`context.cacheDir`）
- [ ] 设置合理的缓存大小（10-100 MB 看应用）
- [ ] 敏感数据加 `no-store`
- [ ] 后端正确返回 `Cache-Control` 头
- [ ] 应用退出时 `cache.close()`（Android）
- [ ] 监控缓存命中率和磁盘占用

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **综合入口**: [[summary]]
