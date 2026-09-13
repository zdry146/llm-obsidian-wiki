---
title: "Kotlin 已知坑与陷阱"
category: synthesis
tags: [kotlin, pitfalls, null-safety, lateinit, coroutines, gotchas]
sources:
  - "Kotlin Docs - Idioms"
  - "Effective Kotlin (Marcin Moskala)"
  - "Kotlin in Action - Common Pitfalls"
summary: "Kotlin 10 大陷阱：!! 滥用、lateinit、协程泄漏、Java 反射、数据类拷贝、可变集合、协程作用域"
provenance:
  extracted: 0.85
  inferred: 0.12
  ambiguous: 0.03
base_confidence: 0.85
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §10 Kotlin 已知坑与陷阱

## 1. 【最常见】!! 滥用 → NPE

```kotlin
// ❌ 危险：name 可能为 null，强行非空
val name: String? = getName()
val length = name!!.length        // NPE 风险

// ✅ 安全：处理 null
val length = name?.length ?: 0

// ✅ requireNotNull + 显式信息
val safeName = requireNotNull(name) { "Name must be set" }
val length = safeName.length
```

**核心洞察**：**`!!` 是 Kotlin 编译器的「逃生舱」**——除非要 100% 确定才用。

## 2. lateinit 陷阱

```kotlin
class MyService {
    lateinit var db: Database
    
    fun init() {
        db = Database.connect(...)
    }
    
    fun query() {
        db.query()   // ❌ 若 init() 没调，UninitializedPropertyAccessException
    }
}

// ✅ 检查是否初始化
fun query() {
    if (::db.isInitialized) {
        db.query()
    }
}

// ✅ 改用 lazy（线程安全）
class MyService {
    val db: Database by lazy { Database.connect(...) }
}
```

## 3. 【协程】GlobalScope 泄漏

```kotlin
// ❌ 严重泄漏：Activity 销毁后仍跑
class MyActivity : AppCompatActivity() {
    override fun onCreate() {
        GlobalScope.launch {
            while (true) {
                delay(1000)
                println("Tick")
            }
        }
    }
}

// ✅ 用 lifecycleScope（Activity 自动取消）
class MyActivity : AppCompatActivity() {
    override fun onCreate() {
        lifecycleScope.launch {
            // 自动随生命周期取消
        }
    }
}

// ✅ ViewModel 用 viewModelScope
class MyViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch { ... }
    }
}
```

## 4. 【协程】async 不 await → 异常吞没

```kotlin
// ❌ 异常被默默吞掉
suspend fun load() {
    coroutineScope {
        async {
            throw RuntimeException("oops")
        }   // 没 await → 异常传播不到父协程
    }
}

// ✅ 必须 await（或用 try/catch 启动）
suspend fun load() {
    coroutineScope {
        val deferred = async {
            try {
                throw RuntimeException("oops")
            } catch (e: Exception) {
                null
            }
        }
        deferred.await()
    }
}

// ✅ 或用 supervisorScope
suspend fun load() = supervisorScope {
    val d = async {
        throw RuntimeException("oops")
    }
    d.await()   // 异常向上传播但不取消其他协程
}
```

## 5. 【协程】runBlocking 在主线程 → ANR

```kotlin
// ❌ Android 主线程跑 runBlocking → ANR
class MyActivity : AppCompatActivity() {
    fun onCreate() {
        runBlocking {                   // ❌ 阻塞主线程
            val data = fetchData()
        }
    }
}

// ✅ 用 lifecycleScope.launch（不阻塞）
class MyActivity : AppCompatActivity() {
    fun onCreate() {
        lifecycleScope.launch {
            val data = fetchData()       // ✅ 不阻塞
            updateUI(data)
        }
    }
}
```

## 6. data class 字段不能是 var

```kotlin
// ❌ 危险：hashCode 不一致
data class User(var name: String, var age: Int)

val user = User("Mike", 30)
user.age = 31                          // hashCode 变了
val map = hashMapOf(user to "data")     // 找不到 key！

// ✅ 全 val
data class User(val name: String, val age: Int)
val older = user.copy(age = 31)         // 新对象，hashCode 一致
```

## 7. Java 互操作：平台类型不可控

```kotlin
// Java
public String findName(int id) { return null; }    // 可能 null

// Kotlin
val name = javaService.findName(1)        // String!（平台类型）
val length = name.length                    // ❌ 运行时 NPE

// ✅ 显式处理
val nullable: String? = javaService.findName(1)
val length = nullable?.length ?: 0
```

## 8. 可变集合共享 → 线程不安全

```kotlin
// ❌ 多线程共享 mutableList
val shared = mutableListOf(1, 2, 3)
Thread { shared.add(4) }.start()           // ConcurrentModificationException

// ✅ 不可变集合
val shared = listOf(1, 2, 3)              // 线程安全
val updated = shared + 4                  // 新对象

// ✅ 线程安全集合（Java 已有）
val safe = java.util.Collections.synchronizedList(mutableListOf<Int>())
// 或 ConcurrentHashMap 等
```

## 9. 【协程】withContext 误用 → 性能

```kotlin
// ❌ 频繁切换 dispatcher（开销大）
suspend fun process() {
    withContext(Dispatchers.IO) {
        val a = read1()                    // IO
        withContext(Dispatchers.Default) { // 又切换
            val b = process1(a)            // CPU
        }
        withContext(Dispatchers.IO) {     // 又切换
            val c = read2()                // IO
        }
    }
}

// ✅ 一次切换，分区
suspend fun process() = withContext(Dispatchers.IO) {
    val a = read1()
    val b = withContext(Dispatchers.Default) {
        process1(a)                        // 只切一次
    }
    val c = read2()
    // ...
}
```

## 10. 【类型推断】数字字面量推断

```kotlin
// ❌ 推断为 Int，溢出
val big = 1_000_000_000_000L + 5         // 报错：Int + Long

// ✅ 显式类型
val big: Long = 1_000_000_000_000L
```

```kotlin
// ❌ Long 运算后转 Int
val id = "123".toLong().toInt()           // 溢出

// ✅ 保持 Long
val id = "123".toLong()
```

## 11. 【扩展函数】访问 internal 失败

```kotlin
// ❌ Java 看不到 Kotlin 的 internal 扩展
internal fun String.internalGreet() = "Hi: $this"

// Java 调用
StringUtils.internalGreet("hello");    // ❌ 编译错
```

## 12. 【反射】data class 拷贝有坑

```kotlin
data class User(val name: String, val age: Int)

val user = User("Mike", 30)
val modified = user.copy(age = 31)         // 正常

// 但反射拷贝
val newUser = User::class.java
    .getDeclaredConstructor(String::class.java, Int::class.java)
    .newInstance("Mike", 31)
// 不触发 copy()，但等价
```

## 13. 【协程】取消传播不彻底

```kotlin
// ❌ 协程取消时 Thread.sleep 不响应
suspend fun slowOperation() {
    Thread.sleep(10000)              // ❌ 不响应取消
}

// ✅ 用 suspend delay
suspend fun slowOperation() {
    delay(10000)                     // ✅ 响应取消
}
```

## 14. 【Java 互操作】Array 类型混淆

```kotlin
// Java: String[] vs Collection<String>
val array: Array<String> = javaService.getArray()           // Array<String!>

// 调用 .toList() 转 Kotlin
val list: List<String> = array.toList()                    // List<String!>

// 过滤 null
val nonNull: List<String> = array.filterNotNull().toList()
```

## 15. 【作用域函数】this vs it 混淆

```kotlin
// apply：this 是接收者
val user = User("Mike").apply {
    println(this.name) // ✅ 显式 this
    println(name)     // ✅ 隐式 this
}

// also：it 是接收者
val user = User("Mike").also {
    println(it.name)   // ✅ 显式 it
}
```

## 16. 实战检查清单

- [ ] 不滥用 `!!`（除非要 100% 确定）
- [ ] lateinit 用前检查 `::prop.isInitialized`
- [ ] 不在生产用 GlobalScope.launch
- [ ] async 必须 await（否则异常吞没）
- [ ] 不在 Android 主线程用 runBlocking
- [ ] data class 字段全 val
- [ ] Java 类型当平台类型处理（可能 null）
- [ ] 多线程共享集合用不可变或 ConcurrentX
- [ ] withContext 不频繁切换 dispatcher
- [ ] 数字字面量加 L/F 后缀
- [ ] Thread.sleep 改 suspend delay
- [ ] 协程作用域绑定生命周期

## 17. 工具

```kotlin
// Detekt：静态检查
// KtLint：代码风格

// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt").version("1.23.0")
}

detekt {
    buildUponDefaultConfig = true
    allRules = true
    exclude = listOf("...")
}
```

## 18. 与 OkHttp / gRPC 的坑

### 18.1 OkHttp + Kotlin

```kotlin
// ❌ lateinit var 误用
class ApiClient {
    lateinit var client: OkHttpClient
    
    fun init() {
        client = OkHttpClient()                  // 可能忘调
    }
}

// ✅ 改用 lazy 或 val
class ApiClient {
    val client: OkHttpClient by lazy { OkHttpClient() }
}
```

### 18.2 gRPC + Kotlin

```kotlin
// ❌ BlockingStub 阻塞 Netty event loop
val stub = GreeterGrpc.newBlockingStub(channel)
val reply = stub.sayHello(req)                  // ❌ event loop 卡死

// ✅ 用 FutureStub 或 Stub（异步）
val stub = GreeterGrpc.newStub(channel)
stub.sayHello(req, observer)                     // ✅ 异步回调

// ✅✅ 更好：用 grpc-kotlin（await）
val stub = GreeterCoroutineGrpc.newStub(channel)
val reply = stub.sayHello(req)                   // ✅ suspend
```

## 19. 关键反思

**Kotlin 的"安全"是双刃剑**：
- ✅ 类型系统强制 null 安全（编译期发现 NPE）
- ❌ 但 `!!` 给了"逃生舱"，开发者易绕过

**核心建议**：
1. **永远不要**用 `!!` 除非逻辑保证
2. **永远不要** GlobalScope.launch
3. **永远不要** 在主线程 runBlocking
4. **永远不要** data class 字段用 var
5. **永远不要** 假设 Java 返回非 null

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **协程**: [[draft-03-coroutines]]
- **最佳实践**: [[draft-09-best-practices]]
- **OkHttp 坑**: [[okhttp-analysis/draft-10-known-issues]]
- **gRPC 坑**: [[grpc-analysis/draft-11-known-issues]]
- **综合入口**: [[summary]]