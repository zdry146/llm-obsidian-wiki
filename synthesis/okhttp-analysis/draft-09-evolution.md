---
title: "OkHttp 版本演进 (3.x → 4.x → 5.x)"
category: synthesis
tags: [okhttp, version-history, kotlin, migration, 3-x, 4-x, 5-x]
sources:
  - "OkHttp 官方 CHANGELOG.md"
  - "GitHub square/okhttp releases"
  - "OkHttp 4.x 迁移指南"
summary: "OkHttp 从 3.x (Java) 到 4.x (Kotlin 重写但 Java 兼容) 到 5.x (Kotlin 协程) 的完整演进"
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

# §09 OkHttp 版本演进（3.x → 4.x → 5.x）

## 1. 三大版本全景

| 版本 | 发布时间 | 主要语言 | 状态 | 关键特性 |
|------|----------|----------|------|---------|
| **3.x** | 2015-2020 | Java | 维护模式 | 经典稳定 |
| **4.x** | 2019-至今 | Kotlin 源码 + Java API | **当前 stable** | Kotlin 重写但 Java 100% 兼容 |
| **5.x** | 2024-alpha | Kotlin + 协程 | alpha | Kotlin 协程一等支持 |

## 2. 3.x → 4.x 迁移（最重要的一次）

### 2.1 为什么重写

**Square 的理由**：
1. **Kotlin 表达力更强** — `object` / 扩展函数 / data class 简化代码 30%
2. **Java API 二进制兼容** — 用户无感知
3. **可维护性** — 减少样板代码
4. **Kotlin 多平台潜力** — 4.x 起为 KMP 铺路

### 2.2 用户体验（Java 项目）

```java
// 3.x 代码（4.x 仍然能用，二进制兼容）
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .build();

Request request = new Request.Builder()
    .url("https://api.example.com")
    .get()
    .build();

Response response = client.newCall(request).execute();
String body = response.body().string();

// 4.x 编译时自动替换实现，但 API 完全一样
// ✅ 不用改一行代码
```

### 2.3 用户体验（Kotlin 项目）

```kotlin
// 4.x 起，推荐 Kotlin DSL（3.x 没有）
val client = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .build()

val request = Request.Builder()
    .url("https://api.example.com")
    .get()
    .build()

// 4.x 起 OkHttp 是 Kotlin 代码，但 Java API 完全保留
```

### 2.4 主要内部变化

| 维度 | 3.x | 4.x |
|------|-----|-----|
| 实现语言 | Java | **Kotlin** |
| API 暴露 | Java | Kotlin + Java |
| 二进制兼容 | - | ✅ |
| 源兼容 | - | ✅（Java 编译时 OK） |
| 反射 | 反射用得多 | 减少反射 |
| 包名 | `okhttp3` | **仍是 `okhttp3`**（不变） |
| Maven artifact | `com.squareup.okhttp3:okhttp` | **不变** |

### 2.5 4.x 主要更新

```
4.0.0 (2019-09):  Kotlin 重写，binary compat with 3.x
4.1.0 (2019-11):  Java 9+ 支持
4.2.0 (2020-01):  Kotlin 1.3.60
4.3.0 (2020-06):  Kotlin 1.3.72
4.4.0 (2020-08):  Kotlin 1.4
4.5.0 (2020-10):  CookieJar 改进
4.6.0 (2020-12):  Kotlin 1.4.10
4.7.0 (2021-01):  EventListener API
4.8.0 (2021-05):  Kotlin 1.4.30
4.9.0 (2021-09):  Kotlin 1.5.30
4.9.1 (2021-11):  Bug fixes
4.9.2 (2022-01):  Bug fixes
4.9.3 (2022-03):  Security patches
4.10.0 (2022-05): Kotlin 1.6.20
4.11.0 (2022-11): JDK 21 IPv6/IPv4 bug 修复（关键！）
4.12.0 (2023-10): 最后稳定版
```

### 2.6 JDK 21 IPv6/IPv4 Bug 修复时间线

⚠️ **这个 bug 影响所有 OkHttp 4.x 之前版本**：

```
4.10.0 (2022-05) → 触发率低
4.11.0 (2022-11) → 引入修复（但不完整）
4.12.0 (2023-10) → 彻底修复
```

详见 [[draft-10-known-issues]]。

## 3. 4.x → 5.x 迁移

### 3.1 5.x 的变化

| 维度 | 4.x | 5.x |
|------|-----|-----|
| Kotlin | 1.x | 1.9+ |
| JDK | 8+ | 8+ |
| 协程支持 | 第三方 | **官方 `okhttp-coroutines`** |
| WebSocket Push | 移除 | 重设计 |
| 异步 API | enqueue/Callback | **+ suspend `await()`** |
| `Call.executeAsync()` | ❌ | ✅ |

### 3.2 5.x 新 API

```kotlin
// 5.0 协程支持
suspend fun fetch(): Response {
    val request = Request.Builder().url("https://api.example.com").build()
    return client.newCall(request).await()  // ← 新 API
}

// 5.0 异步执行
suspend fun fetchAsync(): Response = client.newCall(request).executeAsync()

// 旧 API 仍然支持
client.newCall(request).enqueue(callback)
client.newCall(request).execute()
```

### 3.3 5.x 移除 / 重设计

- **Server Push 接收**：`PushHandle` API 简化（之前设计被认为太复杂）
- **MockWebServer**：5.x 起独立版本（5.0.0-alpha.14 有 `MockWebServer`）
- **Brotli 解码**：从 core 移到单独的 `okhttp-brotli` 模块

### 3.4 当前状态（2026-09）

⚠️ **5.0 仍是 alpha**（2024-09 alpha.14，2026 仍没正式 GA）——Square 在等 Kotlin 协程 API 稳定。

**生产建议**：
- 维护现有项目：**继续用 4.12.0**
- 新项目：**用 4.12.0**（稳定）+ 自行用 coroutine wrapper（`Call.await()`）
- 试验性：5.0.0-alpha.14

## 4. 升级路径建议

### 4.1 从 3.x 直接升 5.x

⚠️ **不推荐跳过 4.x**——4.x 是兼容层。建议：

```
3.x → 4.12.0 → 5.0.x（未来 GA）
```

### 4.2 升级 Checklist

- [ ] 把 `okhttp3` 依赖升到目标版本
- [ ] 重新编译（Java 代码二进制兼容，但 Kotlin 需要重新编译）
- [ ] 测试所有自定义 `Interceptor`（4.x 后部分内部类变化）
- [ ] 测试所有 `CookieJar` / `Authenticator` 实现
- [ ] 测试 `EventListener`（4.7 起 API 扩展）
- [ ] 升级 Okio 到对应版本（4.x 配 Okio 3.x）
- [ ] 跑兼容性测试套件

### 4.3 依赖冲突解决

```xml
<!-- Maven -->
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>logging-interceptor</artifactId>
    <version>4.12.0</version>
</dependency>
<dependency>
    <groupId>com.squareup.okio</groupId>
    <artifactId>okio</artifactId>
    <version>3.6.0</version>
</dependency>
```

⚠️ **Okio 版本**必须与 OkHttp 匹配——4.12.0 配 Okio 3.6.0。

## 5. JDK 兼容性矩阵

| OkHttp 版本 | Java 8 | Java 11 | Java 17 | Java 21 |
|-------------|--------|---------|---------|---------|
| 4.12.0 | ✅ | ✅ | ✅ | ✅ |
| 4.10.0 | ✅ | ✅ | ✅ | ⚠️（IPv6 bug） |
| 3.14.x | ✅ | ✅ | ✅ | ⚠️（IPv6 bug） |

## 6. 实战：3.x 升 4.x 经验

### 6.1 常见升级报错

```kotlin
// ❌ 3.x 的 raw type 用法
val body: ResponseBody = response.body  // 4.x 需要明确 nullable

// ✅ 4.x 写法
val body: ResponseBody? = response.body
```

```kotlin
// ❌ 3.x 的 SAM 转换 (Listener 作为函数)
// 旧 OkHttp WebSocketListener 是 abstract class，必须实现所有方法
// 4.x 起部分回调可 SAM
```

### 6.2 反射 / 序列化场景

如果代码用了反射访问 OkHttp 内部（如 `RealCall`），升级可能崩。**正确做法**：用公开 API。

### 6.3 拦截器实现

```kotlin
// 4.x 拦截器实现需要标注 @Throws(IOException::class)（Kotlin）
class MyInterceptor : Interceptor {
    @Throws(IOException::class)
    override fun intercept(chain: Chain): Response {
        // ...
    }
}
```

## 7. 历史里程碑

| 时间 | 事件 |
|------|------|
| 2013-05 | Square 发布 1.0 |
| 2014-12 | HTTP/2 支持（2.x） |
| 2016-01 | 3.0：Interceptor 重写 |
| 2016-12 | 3.5：TLS 1.3 支持 |
| 2017 | 3.x 成为 Android 默认 |
| 2019-09 | 4.0：Kotlin 重写 |
| 2022-11 | 4.11：JDK 21 修复开始 |
| 2023-10 | 4.12：JDK 21 修复完成 |
| 2024-09 | 5.0 alpha：协程支持 |

## 8. 未来方向

1. **Kotlin Multiplatform**（5.x）：JVM/Android/iOS/macOS 跨平台
2. **原生编译**（GraalVM）：5.x 完整支持
3. **QUIC / HTTP/3**（实验）：5.x alpha 部分支持
4. **更强协程**：structured concurrency 支持
5. **类型安全 DSL**：更 Kotlin 友好的配置 API

## 9. 选型决策树

```
需要 HTTP 客户端
    │
    ├─ 在 Kotlin Multiplatform 项目？
    │     ├─ 是 → OkHttp 5.x alpha 或 Ktor Client
    │     └─ 否 ↓
    │
    ├─ 现有 3.x 项目？
    │     ├─ 是 → 升 4.12.0（兼容）
    │     └─ 否 ↓
    │
    ├─ Android 项目？
    │     ├─ 是 → OkHttp 4.12.0（标准）+ Retrofit
    │     └─ 否 ↓
    │
    ├─ JVM 后端？
    │     ├─ Spring Cloud 微服务 → OpenFeign (底层 OkHttp 4.12)
    │     └─ 直接调用 → OkHttp 4.12.0
    │
    └─ 协程是核心？
          ├─ 是 → OkHttp 4.12.0 + 协程包装
          └─ 否 → OkHttp 4.12.0 同步 / 异步
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **已知坑**: [[draft-10-known-issues]]
- **综合入口**: [[summary]]
