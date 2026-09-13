---
title: "Kotlin Multiplatform (KMP)"
category: synthesis
tags: [kotlin, kmp, multiplatform, ios, android, jvm, native, wasm]
sources:
  - "Kotlin Multiplatform Docs (kotlinlang.org/docs/multiplatform.html)"
  - "JetBrains KMP Blog"
  - "Touchlab KMP Resources"
summary: "KMP 架构：commonMain/jvmMain/iosMain、expect/actual、共享代码策略、与 Flutter/RN 对比"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §07 Kotlin Multiplatform（KMP）

## 1. 什么是 KMP？

**Kotlin Multiplatform（KMP）** = JetBrains 的跨平台方案，让 **同一份 Kotlin 代码** 编译到多个平台：

| 目标 | 成熟度 | 编译产物 |
|------|--------|---------|
| **JVM** | ✅ 完美 | `.jar` |
| **Android** | ✅ 完美 | `.aar` |
| **iOS** | ✅ 稳定 | `.framework` |
| **macOS** | ✅ 稳定 | `.framework` |
| **Linux** | ✅ 稳定 | `.kexe` (Kotlin/Native) |
| **Windows** | ✅ 稳定 | `.kexe` |
| **JavaScript** | ✅ 稳定 | `.js` |
| **WebAssembly** | ✅ v2.0+ 稳定 | `.wasm` |

**核心洞察**：**KMP 不是新的框架，是 Kotlin 编译器的扩展**——一份源码，多个 target。

## 2. 与跨平台方案对比

| 方案 | 语言 | 跨平台能力 | UI |
|------|------|----------|-----|
| **KMP** | Kotlin | 共享业务逻辑 | Compose Multiplatform（共享 UI） |
| **Flutter** | Dart | 共享 UI + 业务 | 自带 UI 框架 |
| **React Native** | JavaScript | 共享业务逻辑 + UI | React |
| **Cordova/Capacitor** | JavaScript | Web 套壳 | WebView |
| **Xamarin** | C# | 共享业务逻辑 | MAUI |

**KMP 的定位**：
- **KMP** = **共享业务逻辑** + 平台特定 UI（原生体验）
- **Flutter** = **共享 UI** + 业务逻辑
- **React Native** = **共享 UI + JS** + 业务逻辑

## 3. 架构：3 层 source set

```
my-kmp-project/
├── commonMain/             ← 所有平台共享
│   └── kotlin/
│       ├── Main.kt
│       └── Greeting.kt
├── jvmMain/                ← JVM 特定（追加）
├── androidMain/            ← Android 特定
├── iosMain/                ← iOS 特定
├── jsMain/                 ← JS 特定
└── nativeMain/             ← Native（iOS/Android/Linux/...）共享
```

**3 层**：
1. **commonMain**（100% 共享）—— 业务逻辑
2. **nativeMain**（部分共享）—— Kotlin/Native 共享
3. **平台特定**（jvmMain/iosMain/...）—— 平台特有代码

## 4. expect/actual 模式

```kotlin
// commonMain
expect class Platform() {
    fun name(): String
    fun version(): String
}

// jvmMain
actual class Platform {
    actual fun name() = "JVM"
    actual fun version() = System.getProperty("java.version")
}

// iosMain
actual class Platform {
    actual fun name() = "iOS"
    actual fun version() = UIDevice.currentDevice.systemVersion
}

// 使用（在 commonMain）
fun greet(): String {
    val p = Platform()
    return "Hello from ${p.name()} ${p.version()}"
}
```

**核心洞察**：**expect = 接口声明，actual = 平台实现**——编译器强制所有平台都实现。

### 4.1 expect/actual vs interface

```kotlin
// ❌ 用 interface 模拟（Java 也能做）
interface Platform {
    fun name(): String
}
class JvmPlatform : Platform { ... }
class IosPlatform : Platform { ... }

// ✅ KMP expect/actual（编译期强制）
expect class Platform {
    fun name(): String
}
actual class Platform { ... }
```

**优势**：
- 编译期检查所有平台实现
- commonMain 直接调用（无需工厂方法）

## 5. 共享代码策略

### 5.1 业务逻辑（推荐 100% 共享）

```kotlin
// commonMain
class UserRepository {
    suspend fun findById(id: String): User? = withContext(Dispatchers.IO) {
        // 数据库访问（SQLDelight 跨平台）
        database.userQueries.findById(id).executeAsOneOrNull()
    }
    
    suspend fun save(user: User) = withContext(Dispatchers.IO) {
        database.userQueries.insert(user.toEntity())
    }
}
```

### 5.2 数据模型（100% 共享）

```kotlin
// commonMain
@Serializable
data class User(
    val id: String,
    val name: String,
    val email: String
)
```

### 5.3 HTTP 客户端（Ktor，跨平台）

```kotlin
// commonMain
import io.ktor.client.*

class ApiClient {
    private val client = HttpClient()
    
    suspend fun fetchUsers(): List<User> = client.get("https://api.example.com/users").body()
}
```

### 5.4 平台特定（10%）

- Android: Activity / ViewModel / Compose
- iOS: UIView / SwiftUI / Combine
- JVM: Spring Boot
- JS: DOM 操作

## 6. 实战：iOS + Android 共享 80%

```
commonMain: 业务逻辑、数据模型、HTTP 客户端、协程   (60%)
nativeMain:  文件 I/O、日期、UUID、Kotlin/Native 通用 (10%)
androidMain: ViewModel、LiveData、Compose        (15%)
iOSMain:     UIKit 桥接、Swift 互调              (15%)
```

**80% 代码共享**，20% 平台特定 UI。

## 7. Compose Multiplatform（共享 UI）

```kotlin
// commonMain
@Composable
fun App() {
    MaterialTheme {
        Column {
            Text("Hello from KMP!")
            Button(onClick = { /* ... */ }) {
                Text("Click me")
            }
        }
    }
}

// Android: App.kt 调用 setContent { App() }
// iOS: SwiftUI 包装（[ComposeUIViewController] { App() }）
// Desktop: 直接使用
```

**状态**：Compose Multiplatform 在 v1.5+ 稳定，iOS beta。

## 8. Ktor 跨平台 HTTP

```kotlin
// commonMain
import io.ktor.client.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*

val client = HttpClient {
    install(ContentNegotiation) {
        json()
    }
}

suspend fun getUsers(): List<User> = client.get("/users").body()
```

**支持平台**：JVM、Android、iOS、JS、Wasm、Linux、Windows、macOS。

## 9. SQLDelight 跨平台数据库

```kotlin
// commonMain
class UserRepository(private val database: UserDatabase) {
    fun getUser(id: String) = database.userQueries.findById(id).executeAsOneOrNull()
}

// 生成的 SQL（UserDatabase.sq 文件）
// findById:
//   SELECT * FROM users WHERE id = ?
```

**优势**：类型安全 SQL，编译期生成 Kotlin 代码。

## 10. KMP 实战：网络栈

```kotlin
// commonMain
class HttpClientFactory {
    fun create(): HttpClient = HttpClient {
        install(ContentNegotiation) { json() }
        install(Logging) { level = LogLevel.INFO }
        install(HttpTimeout) {
            requestTimeoutMillis = 5000
            connectTimeoutMillis = 3000
        }
    }
}

// 拦截器也共享
class AuthInterceptor(private val token: String) {
    suspend fun intercept(request: HttpRequestBuilder) {
        request.header("Authorization", "Bearer $token")
    }
}

// netlib-common 适配 KMP 版本
// （详见 ~/.openclaw/workspace-developer/netlib-common/）
```

## 11. 实际采用案例

| 公司 | 用法 |
|------|------|
| **JetBrains** | JetBrains Toolbox（iOS + Android + Desktop） |
| **Square** | Cash App 部分模块 |
| **Netflix** | Polly 服务（部分） |
| **Mozilla** | Focus for iOS/Android |
| **VMware** | 多产品跨平台 |
| **百度** | 百度网盘 iOS/Android |
| **腾讯** | 多个 App 跨端 |

## 12. 与 Flutter/RN 的本质区别

```
Flutter / React Native:
  - Dart / JS 单一语言
  - 自带 UI 框架
  - 平台抽象层厚
  - 性能：接近原生

KMP:
  - Kotlin 共享业务逻辑
  - UI 用平台原生（iOS SwiftUI, Android Compose）
  - 平台抽象层薄
  - 性能：完全原生（UI 层）
```

**核心洞察**：**KMP 不是「统一 UI」方案，是「共享业务逻辑」方案**——UI 仍用平台原生。

## 13. KMP 实战建议

### 13.1 适合 KMP 的项目

✅ **新项目**（从零开始）—— 一开始就 KMP
✅ **iOS + Android 双端**——共享业务逻辑省一半代码
✅ **跨 SDK / 库**——KMP 写 SDK，平台用
✅ **算法 / 数据处理**——100% 共享

### 13.2 不适合 KMP

❌ **已有大型 App**——迁移成本高
❌ **重 UI 项目**——Compose Multiplatform 还在成熟
❌ **强平台特性**——如 ARKit（iOS only）

## 14. 关键设计原则

1. **commonMain 尽量纯净**——只依赖 Kotlin stdlib + Ktor/SQLDelight
2. **expect/actual**——平台差异接口化
3. **业务逻辑 100% 共享**——UI 平台原生
4. **Coroutine 全**——KMP 协程稳定
5. **依赖管理**——版本统一（Kotlin 2.0+ + Ktor + ...）

## 15. 反模式

❌ **commonMain 调平台 API**——破坏共享
❌ **expect/actual 太多**——可能需要抽象层
❌ **忽视性能**——跨平台抽象有成本
❌ **KMP 重 UI**——目前不稳定

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **协程**: [[draft-03-coroutines]]
- **Java 互操作**: [[draft-08-java-interop]]
- **应用场景**: [[draft-12-use-cases]]
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`
- **综合入口**: [[summary]]