---
title: "Kotlin 最佳实践"
category: synthesis
tags: [kotlin, best-practices, naming, packages, idioms, code-style]
sources:
  - "Kotlin Official Code Style (kotlinlang.org/docs/coding-conventions.html)"
  - "JetBrains Kotlin Best Practices"
  - "Effective Kotlin (Marcin Moskala)"
summary: "Kotlin 最佳实践：13 条铁律 + 命名约定 + 包结构 + 不可变优先 + null 安全 + 函数式风格"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §09 Kotlin 最佳实践

## 1. 13 条铁律

1. **不可变优先**——val > var
2. **空安全**——用 `?` 显式标记可空
3. **类型推断**——不写冗余类型
4. **data class**——POJO 用一行
5. **默认参数**——替代重载
6. **扩展函数**——增强类不破坏封装
7. **sealed class**——封闭继承
8. **协程**——异步编程首选
9. **结构化并发**——CoroutineScope
10. **lambda + trailing**——集合操作一行
11. **不可变集合**——默认 listOf/mapOf
12. **作用域函数**——apply/let/also 优先
13. **顶层函数**——替代 utility 类

## 2. 命名约定

### 2.1 包名

```kotlin
// ✅ 全小写，不连续下划线
package io.zdry.netlib.okhttp.interceptor
package com.example.feature

// ❌ 大写或下划线
package io.zdry.NetLib  // ❌
package io.zdry.net_lib  // ❌
```

### 2.2 类名

```kotlin
// ✅ PascalCase
class HttpClientFactory       // 类
interface ResponseHandler      // 接口
object DatabaseManager        // object
sealed class Result<out T>     // sealed class
enum class Status              // enum

// ❌
class http_client_factory     // snake_case
```

### 2.3 函数和变量

```kotlin
// ✅ camelCase
fun fetchUser(): User
val userName: String = "Mike"
var isActive: Boolean = false

// ✅ 工厂函数用名词（返回实例）
fun user(name: String, age: Int): User = User(name, age)

// ✅ 转换函数用 toXxx
fun toDto(): UserDto

// ✅ 强制转换用 asXxx
fun asJson(): String

// ✅ 副作用函数用动词
fun saveUser(user: User)
fun deleteUser(id: String)

// ✅ boolean 用 is/has/can
fun isAdult(age: Int): Boolean = age >= 18
fun hasChildren(): Boolean
fun canSend(): Boolean

// ❌
fun fetch_user()    // snake_case
fun user_name       // snake_case
```

### 2.4 常量

```kotlin
// ✅ 全大写下划线
const val MAX_CONNECTIONS = 100
const val DEFAULT_TIMEOUT_MS = 5000L

// ✅ 对象里也用大写
object Constants {
    const val API_BASE_URL = "https://api.example.com"
}
```

## 3. 包结构

```
com.example.myapp/
├── Main.kt                          入口
├── config/
│   ├── HttpConfig.kt                data class
│   └── HttpClientFactory.kt
├── interceptor/
│   ├── AuthInterceptor.kt           类
│   └── LoggingInterceptor.kt
├── repository/
│   ├── UserRepository.kt            接口
│   └── UserRepositoryImpl.kt        实现
├── service/
│   └── UserService.kt
└── util/
    └── Extensions.kt                扩展函数集合
```

**最佳**：
- ✅ 一个文件一个公开类（除非 sealed class 紧密相关）
- ✅ 文件名 = 类名（首字母大写）
- ✅ 测试文件 = `ClassNameTest.kt`
- ✅ 扩展函数集中在 `Extensions.kt`

## 4. 不可变优先

```kotlin
// ✅ 不可变
val name: String = "Mike"
val items: List<Int> = listOf(1, 2, 3)

// ❌ 可变（除非必要）
var name: String = "Mike"   // 忘了改？
val items = mutableListOf(1, 2, 3)  // 暴露可变接口

// 实战：data class 全 val
data class User(val name: String, val age: Int)  // ✅

// ❌ data class 用 var
data class User(var name: String, var age: Int)  // ❌（破坏 hashCode 一致性）
```

## 5. null 安全

```kotlin
// ✅ 显式可空 + 安全处理
fun process(name: String?) {
    val length = name?.length ?: 0
}

// ❌ 强制非空（!! 滥用）
fun process(name: String?) {
    val length = name!!.length  // NPE 风险
}

// ✅ requireNotNull
fun process(user: User?) {
    requireNotNull(user) { "User must not be null" }
    // user 现在是非空
    println(user.name)
}

// ✅ lateinit（非空 var）
class MyService {
    lateinit var db: Database   // 必须在使用前赋值
}

// ✅ lateinitvar 检查
if (::db.isInitialized) {
    db.query()
}
```

## 6. 默认参数 vs Builder

```kotlin
// ✅ 函数 + 默认参数（推荐）
fun connect(
    host: String = "localhost",
    port: Int = 50051,
    useTls: Boolean = true
) { ... }

// 调用
connect()                                  // 全默认
connect("api.example.com")                 // 命名/位置
connect("api.example.com", port = 443)     // 混合

// ❌ Builder 模式（Java 时代产物，Kotlin 不需要）
class ConnectBuilder { /* 5 个 builder 方法 */ }
```

**例外**：复杂的配置对象（如 HttpClient）仍可用 builder + data class：

```kotlin
data class HttpConfig(
    val connectTimeout: Duration = 10.seconds,
    val readTimeout: Duration = 30.seconds,
    // ... 10+ 字段
)
```

## 7. 集合选择

```kotlin
// ✅ 不可变优先
val items: List<String> = listOf("a", "b")
val map: Map<String, Int> = mapOf("a" to 1)

// ✅ 需要可变才用 MutableX
val mut = mutableListOf<Int>()    // 真要 add/remove

// ❌ 过度用 mutable
val mut = mutableListOf("a", "b")
mut.add("c")                        // 不可变也能 + 操作

// ✅ Sequence vs Iterable
//   小数据用 Iterable（简单）
//   大数据链 + 早终止用 Sequence（性能）

listOf(1..1000000)
    .asSequence()
    .filter { it > 100 }
    .map { it * 2 }
    .take(10)
    .toList()
```

## 8. 作用域函数选择

| 场景 | 推荐 |
|------|------|
| **配置对象** | `apply` |
| **多步返回结果** | `with` |
| **配置后计算** | `run` |
| **log/debug** | `also` |
| **null 安全** | `let` |

```kotlin
// ✅ 用 apply 配置
val client = OkHttpClient.Builder()
    .apply {
        connectTimeout(10, TimeUnit.SECONDS)
        readTimeout(30, TimeUnit.SECONDS)
    }
    .build()

// ✅ 用 let 处理 nullable
val length = name?.let { it.length } ?: 0

// ✅ 用 also 副作用
val user = fetchUser()
    .also { log.info("Got: $it") }
    .also { cache.put(it.id, it) }
```

## 9. 协程风格

```kotlin
// ✅ suspend 函数命名（不暴露异步细节）
suspend fun fetchUser(id: String): User

// ✅ suspend 函数末尾返回数据
suspend fun fetchAll(): List<User> = coroutineScope {
    val d1 = async { fetchUser1() }
    val d2 = async { fetchUser2() }
    listOf(d1.await(), d2.await())
}

// ✅ Dispatcher 显式切换
suspend fun loadUser() = withContext(Dispatchers.IO) {
    db.query(...)
}

// ❌ GlobalScope.launch
GlobalScope.launch { ... }  // 泄漏

// ❌ runBlocking 在生产代码
runBlocking { ... }  // 仅用于 main/test
```

## 10. 函数式 vs 命令式

```kotlin
// ✅ 函数式
val adults = users.filter { it.age >= 18 }
val names = adults.map { it.name }

// ❌ 命令式
val adults = mutableListOf<User>()
for (user in users) {
    if (user.age >= 18) {
        adults.add(user)
    }
}
val names = mutableListOf<String>()
for (user in adults) {
    names.add(user.name)
}
```

**例外**：
- 副作用操作（forEach 更合适）
- 性能关键（命令式可能更快，但通常不重要）

## 11. Lambda 风格

```kotlin
// ✅ trailing lambda
list.filter { it > 0 }
    .map { it * 2 }
    .forEach { println(it) }

// ✅ it 代替单参数
list.filter { it > 0 }

// ✅ 显式参数（多参数）
list.fold(0) { acc, x -> acc + x }

// ✅ 命名参数 lambda（可读性）
listOf(1, 2, 3).fold(
    initial = 0,
    operation = { acc, x -> acc + x }
)
```

## 12. when 表达式

```kotlin
// ✅ when 是表达式
val description = when (status) {
    is Status.Active -> "Active"
    is Status.Inactive -> "Inactive"
    is Status.Deleted -> "Deleted"
}

// ✅ sealed class 编译器穷尽检查
sealed class Status {
    object Active : Status()
    object Inactive : Status()
    object Deleted : Status()
}
val s = when (status) {  // 无需 else（编译器穷尽）
    Status.Active -> "Active"
    Status.Inactive -> "Inactive"
    Status.Deleted -> "Deleted"
}

// ❌ if-else 链
val s = if (status is Status.Active) "Active"
        else if (status is Status.Inactive) "Inactive"
        else "Deleted"
```

## 13. 测试

```kotlin
// JUnit 5 + Kotlin Test
class HttpClientFactoryTest {
    @Test
    fun `should create client with default config`() {
        val client = HttpClientFactory().create(HttpConfig())
        assertNotNull(client)
    }
    
    @Test
    fun `should throw on negative timeout`() {
        assertFailsWith<IllegalArgumentException> {
            HttpConfig(connectTimeout = Duration.ofSeconds(-1))
        }
    }
}

// Mockk（Kotlin 友好的 mock 框架）
import io.mockk.*

@Test
fun `should call mock interceptor`() {
    val mock = mockk<Interceptor>(relaxed = true)
    // ...
}
```

## 14. 实战检查清单

### 14.1 新项目

- [ ] Kotlin 2.0+ + K2 编译器
- [ ] 包名小写无下划线
- [ ] 默认 val
- [ ] 不可变集合
- [ ] 默认参数
- [ ] requireNotNull 而非 !!

### 14.2 重构 Java → Kotlin

- [ ] 一个文件一个类
- [ ] data class 替代 POJO
- [ ] 顶层函数替代 utility 类
- [ ] apply/let/also 替代匿名类
- [ ] sealed class 替代 enum + state pattern

### 14.3 Code Review

- [ ] 函数参数 > 3 个？用 data class
- [ ] var 多？改 val
- [ ] !! 多？处理 null 安全
- [ ] 嵌套深？用 scope function
- [ ] 副作用？用 also
- [ ] 链长？用 Sequence

## 15. 反模式（Top 10）

）

❌ **var 滥用**——优先 val
❌ **!! 滥用**——除非要 100% 确定
❌ **data class 用 var**——破坏一致性
❌ **过度嵌套 lambda**——提取为具名函数
❌ **GlobalScope.launch**——生命周期泄漏
❌ **runBlocking 在生产**——阻塞主线程
❌ **Java 风格 Builder**——用默认参数
❌ **混用 apply 和 also**——语义不清
❌ **可空 Boolean**——很难用
❌ **长函数**（>50 行）——拆

## 16. 与团队协作

1. **统一代码风格**——使用 IntelliJ 默认
2. **配置 detekt / ktlint**——CI 检查
3. **KDoc 注释**——公开 API
4. **测试覆盖率**——关键路径 >80%
5. **Code Review**——强制

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **协程**: [[draft-03-coroutines]]
- **DSL**: [[draft-04-dsl-builders]]
- **集合**: [[draft-05-collections]]
- **坑**: [[draft-10-pitfalls]]
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`
- **综合入口**: [[summary]]