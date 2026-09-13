---
title: "Kotlin 函数与扩展"
category: synthesis
tags: [kotlin, functions, default-args, named-args, extension-functions, lambda]
sources:
  - "Kotlin Docs - Functions"
  - "Kotlin Docs - Lambdas"
  - "Kotlin Docs - Extensions"
summary: "Kotlin 函数：默认参数、命名参数、扩展函数、lambda、高阶函数、函数引用"
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

# §02 Kotlin 函数与扩展

## 1. 函数定义

```kotlin
// 基本定义
fun add(a: Int, b: Int): Int {
    return a + b
}

// 表达式函数体（自动推断返回类型）
fun add(a: Int, b: Int) = a + b

// 单表达式函数（public API 推荐）
fun greet(name: String): String = "Hello, $name!"

// Unit 返回（= Java void）
fun log(message: String): Unit = println(message)
// 或省略 Unit
fun log(message: String) = println(message)

// Nothing（永不返回）
fun fail(message: String): Nothing = throw IllegalStateException(message)
```

## 2. 默认参数与命名参数（杀手特性）

```kotlin
// ✅ 默认参数
fun connect(
    host: String = "localhost",
    port: Int = 50051,
    useTls: Boolean = true,
    timeout: Duration = 5.seconds
) { /* ... */ }

// 调用 - 多种方式
connect()                                               // 全默认
connect("api.example.com")                              // 只 host
connect("api.example.com", 443)                         // host + port
connect("api.example.com", port = 443)                  // host + 命名参数 port
connect(port = 443, host = "api.example.com")           // 全命名（任意顺序）
connect("api.example.com", useTls = false)              // 跳过 port
```

**vs Java**：
- Java 需要重载（5 个构造器组合 = 5 个函数）
- Kotlin 一个函数 + 默认参数 = 5 个构造器
- **Builder 模式不再必需**

```kotlin
// Java 等价写法（5 个构造器）
public ConnectService(String host, int port, boolean useTls, Duration timeout) {...}
public ConnectService(String host, int port, boolean useTls) { this(host, port, useTls, 5.seconds); }
public ConnectService(String host, int port) { this(host, port, true, 5.seconds); }
public ConnectService(String host) { this(host, 50051, true, 5.seconds); }
public ConnectService() { this("localhost", 50051, true, 5.seconds); }
```

## 3. 扩展函数（核心创新）

```kotlin
// 给 String 类添加方法（不修改源码！）
fun String.greet(): String = "Hello, $this!"
fun String.addExclaim(): String = "$this!"

// 使用
val name = "Mike"
println(name.greet())           // "Hello, Mike!"
println("Hi".addExclaim())       // "Hi!"

// 扩展属性
val String.wordCount: Int
    get() = split(" ").size

"Hello world".wordCount         // 2

// 标准库大量使用
// listOf(1,2,3).first()
// "  hello  ".trim()
// 1..10.toList()
```

**核心洞察**：**扩展函数是 Kotlin 最优雅的特性之一**——既增强类，又不破坏封装。

### 3.1 扩展函数的限制

- ❌ 不能访问 private/internal 成员
- ❌ 不能 override 已有方法
- ✅ 编译期静态分发（性能 = 普通方法）
- ✅ 文件作用域，不污染全局

### 4. Lambda 与高阶函数

```kotlin
// Lambda（类似 Java arrow function）
val sum: (Int, Int) -> Int = { a, b -> a + b }
val square: (Int) -> Int = { x -> x * x }
val greet: () -> Unit = { println("Hello!") }

// 调用
println(sum(1, 2))            // 3
println(square(5))            // 25
greet()                       // Hello!

// 高阶函数（接受/返回 lambda）
fun operate(a: Int, b: Int, op: (Int, Int) -> Int) = op(a, b)
operate(1, 2, ::sum)          // 函数引用
operate(1, 2, { a, b -> a * b })  // lambda
```

### 4.1 Lambda 简化（trailing lambda + it）

```kotlin
listOf(1, 2, 3, 4, 5)
    .filter { it > 2 }                  // 单参数可省略 → 用 it
    .map { it * 2 }                    // trailing lambda
    .forEach { println(it) }           // trailing lambda

// 多参数 lambda 必须命名
listOf(1, 2, 3).fold(0) { acc, x -> acc + x }
```

## 5. 函数引用（Function Reference）

```kotlin
fun isOdd(n: Int): Boolean = n % 2 != 0

val numbers = listOf(1, 2, 3, 4, 5)

// ✅ 方法引用
val oddNumbers = numbers.filter(::isOdd)

// ✅ 构造器引用
data class User(val name: String)
val createUser: (String) -> User = ::User
val user = createUser("Mike")
```

**关键洞察**：函数引用让「方法作为值」一等公民。

## 6. 内联函数（性能关键）

```kotlin
// 内联：编译期把函数体复制到调用处
inline fun <T> measureTime(block: () -> T): T {
    val start = System.nanoTime()
    val result = block()
    val took = System.nanoTime() - start
    println("Took: ${took}ns")
    return result
}

// 使用
measureTime {
    heavyComputation()
}  // 编译期 inline：无 lambda 对象开销
```

**内联的收益**：
- ✅ 无 lambda 对象分配
- ✅ 无 invoke 虚拟调用
- ✅ 无 stack frame

**限制**：
- ❌ 不能内联 public API 的 private 字段
- ❌ 不能有 `suspend` + `inline`（除 reified）

## 7. reified 类型参数

```kotlin
// ❌ Java：instanceof + cast
public <T> T cast(Object obj) {
    return (T) obj;  // unchecked warning
}

// ✅ Kotlin：reified
inline fun <reified T> Any?.cast(): T? = this as? T

val user: Any = User("Mike", 30)
val typed: User? = user.cast()    // 类型安全

// 经典用例：JSON 反序列化
inline fun <reified T> String.fromJson(): T = Json.decodeFromString(this)
val config: HttpConfig = "...".fromJson()
```

## 8. 操作符重载

```kotlin
data class Money(val amount: Int, val currency: String) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency)
        return Money(amount + other.amount, currency)
    }
    
    operator fun compareTo(other: Money): Int = amount.compareTo(other.amount)
}

val a = Money(100, "USD")
val b = Money(50, "USD")
val c = a + b            // Money(150, "USD")
val sorted = listOf(a, b).sorted()  // 用 compareTo
```

**可重载操作符**：`+`, `-`, `*`, `/`, `%`, `+=`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `[]`, `()`, `..`, `in`, `invoke`, `get`, `set`, 等。

## 9. 中缀函数（infix）

```kotlin
infix fun Int.plus(other: Int) = this + other
val result = 1 plus 2   // = 3

// 实战：DSL 构建
infix fun <T> T.should(matcher: Matcher<T>) = ...
val result = user should beValidUser  // DSL 风格

infix fun String.shouldContain(substring: String) = substring in this
val isValid = "hello world" shouldContain "world"
```

## 10. 顶层函数（替代 Java static utility）

```kotlin
// 文件: Utils.kt
package io.zdry.utils

fun formatMoney(amount: Int): String = "$amount USD"
fun parseDate(s: String): LocalDate = LocalDate.parse(s)

// Java 等价
public class Utils {
    public static String formatMoney(int amount) { ... }
    public static LocalDate parseDate(String s) { ... }
}

// Java 调用：Utils.formatMoney(100)
// Kotlin 调用：formatMoney(100)（直接顶层）
```

**优势**：比 `object` 单例更简洁，比 `companion object` 更扁平。

## 11. 局部函数（Nested Functions）

```kotlin
fun processUser(user: User) {
    fun validate(name: String) { /* ... */ }   // 局部函数
    fun normalize(s: String) = s.trim().lowercase()
    
    validate(user.name)
    user.name = normalize(user.name)
}
```

**优势**：封装辅助逻辑，避免污染全局命名空间。

## 12. 实参命名 vs Builder 模式

```kotlin
// ❌ 传统 Builder 模式
class OkHttpClientBuilder {
    var connectTimeout: Duration = 10.seconds
    var maxRequests: Int = 64
    // ...
}

// ✅ Kotlin 函数 + 默认参数
data class HttpConfig(
    val connectTimeout: Duration = 10.seconds,
    val maxRequests: Int = 64
)

// 调用一样简洁，但实现简单 70%
val config = HttpConfig(connectTimeout = 5.seconds, maxRequests = 256)
```

**Builder 模式** 在 Kotlin 里几乎不需要（详见 [[draft-09-best-practices]]）。

## 13. 关键设计原则

1. **默认参数 > 重载**——代码少 70%
2. **扩展函数**——增强类不破坏封装
3. **Lambda + trailing**——集合操作一行
4. **inline + reified**——性能 + 类型安全
5. **顶层函数**——替代 utility 类
6. **中缀函数**——DSL 风格

## 14. 反模式

❌ **过深嵌套 lambda**——提取为具名函数
❌ **扩展函数访问 private 字段**——改用 companion object
❌ **过度用中缀函数**——只用在 DSL 场景
❌ **lambda 引用 var 闭包**——线程不安全

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **协程**: [[draft-03-coroutines]]
- **DSL**: [[draft-04-dsl-builders]]
- **集合**: [[draft-05-collections]]
- **综合入口**: [[summary]]