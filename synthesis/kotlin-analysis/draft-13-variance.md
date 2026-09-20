---
title: "Kotlin 协变与逆变 (Variance)"
category: synthesis
tags: [kotlin, generics, variance, covariance, contravariance, type-system]
sources:
  - "Kotlin Docs - Generics"
  - "Kotlin in Action - Chapter 9: Generics"
  - "Effective Java (Joshua Bloch) - PECS 原则"
summary: "Kotlin 型变核心：协变 out T、逆变 in T、星投影 *、声明处 vs 使用处型变、PECS 原则实战"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §13 Kotlin 协变与逆变（Variance）

## 1. 核心问题

```kotlin
// ✅ String 是 AnyRef 的子类型
val s: String = "Hello"
val a: AnyRef = s                // OK

// ❌ List<String> 不是 List<AnyRef> 的子类型（默认不变）
val strs: List<String> = listOf("a", "b")
val anys: List<AnyRef> = strs     // 编译错误！
// 即使 String <: AnyRef，List<String> 也不 <: List<AnyRef>
```

**为什么？**——如果允许，会破坏类型安全：

```kotlin
// 假设 List<String> <: List<AnyRef>
val strs: List<String> = listOf("a")
val objs: List<AnyRef> = strs     // 假装成立
objs.add(123)                    // 运行时往 List<String> 加了 Int！
strs[0]                          // 类型不安全！
```

## 2. 三个核心概念

| 概念 | 关键字 | 角色 | 记忆口诀 |
|------|--------|------|----------|
| **协变（Covariance）** | `out T` | 只能**输出** T（生产者）| 「Producer Extends」 |
| **逆变（Contravariance）** | `in T` | 只能**消费** T（消费者）| 「Consumer Super」 |
| **不变（Invariance）** | 默认 | 读+写都能（普通类）| - |

## 3. 协变 `out T`（Producer 位置）

```kotlin
// ✅ 声明 List<T> 是协变的
interface List<out T> {
    operator fun get(index: Int): T          // 只能输出 T
    val size: Int                            // 无 T 参数
    // fun add(t: T)                       // ❌ 不能 add（会写）
}

// ✅ 协变后，List<String> <: List<AnyRef>
val strings: List<String> = listOf("a", "b")
val anyRefs: List<AnyRef> = strings          // ✅ OK

// ✅ 协变后，可以传给期待 AnyRef 的函数
fun printList(items: List<AnyRef>) {
    items.forEach { println(it) }
}
printList(strings)                          // ✅
```

**`out T` 约束**：
- ✅ T 只能出现在**输出位置**（返回值、val）
- ❌ T 不能出现在**输入位置**（函数参数、var setter）

## 4. 逆变 `in T`（Consumer 位置）

```kotlin
// 经典 Consumer
interface Comparator<in T> {
    fun compare(a: T, b: T): Int              // 消费 T
}

// ✅ Comparator<AnyRef> <: Comparator<String>（逆变）
val anyComparator = Comparator<AnyRef> { a, b ->
    (a as Comparable<AnyRef>).compareTo(b)
}

val strings: List<String> = listOf("a", "b", "c")
// 用 AnyRef 排序器排序 String 列表——因为 AnyRef 比较能处理 String
val sorted = strings.sortedWith(anyComparator)
```

**`in T` 约束**：
- ✅ T 只能出现在**输入位置**（函数参数）
- ❌ T 不能出现在**输出位置**（返回值）

## 5. 实操对比

### 5.1 `List<out T>` vs `MutableList<T>`

```kotlin
// ✅ List 协变（不可变）
val strs: List<String> = listOf("a")
val anys: List<AnyRef> = strs          // OK

// ❌ MutableList 不变（可变，需要写）
// interface MutableList<T> : List<T>, MutableCollection<T> {  // T 默认不变
//     override fun add(element: T): Boolean
// }
```

### 5.2 `Function<in P1, out R>`

```kotlin
// Function1 的签名是型变组合
// interface Function1<in P1, out R> {
//     operator fun invoke(p1: P1): R
// }
// 参数 in（Consumer），返回值 out（Producer）

// ✅ 协变+逆变结合：可以灵活赋值
val stringToInt: (String) -> Int = { it.length }
val anyToAny: (Any) -> Any = { it }
val anyToInt: (Any) -> Int = stringToInt       // ❌ String 输入不能给 Any 输入
val stringToAny: (String) -> Any = stringToInt  // ✅ Int 返回可以给 Any 返回
```

**核心洞察**：**参数位置 `in`，返回位置 `out`**。

### 5.3 `Result<out T>`

```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()    // T 在输出位置
    data class Failure(val error: Throwable) : Result<Nothing>()  // 失败不持有 T
}

fun handle(r: Result<String>) {
    val result: Result<AnyRef> = r        // ✅ 协变
}
```

## 6. 星投影 `*`（use-site variance 简化）

```kotlin
// T 未知时，用 * 代替
fun printList(items: List<*>) {
    items.forEach { println(it) }        // 可读
    // items.add("hello")               // ❌ 不知道 T，不能写
}

// 常用场景
val anyList: List<*> = listOf(1, "a", true)   // 混合类型
```

**关键**：星投影**只能读，不能写**。

## 7. 声明处 vs 使用处型变

### 7.1 Kotlin：声明处（更简洁）

```kotlin
// 一次声明，全局生效
interface List<out T>      // T 永远是协变

// 调用处无需再标
val strs: List<String> = listOf("a")
val anys: List<AnyRef> = strs              // 自动协变
```

### 7.2 Java：使用处（更冗）

```java
// List<String> 不是 List<AnyRef>（不变）
List<String> strs = Arrays.asList("a");
List<? extends AnyRef> anys = strs;       // 必须显式标

// 写入受限
anys.add("hello");                          // ❌（不知道 ? 是 String 还是别的）
```

**核心洞察**：**Kotlin 的 `out T` 比 Java 的 `? extends T` 更彻底**——一次声明，所有使用处受益。

## 8. PECS 原则（实战口诀）

来自 Effective Java 的金科玉律：**Producer Extends, Consumer Super**

| 场景 | 用什么 |
|------|--------|
| 容器**生产**T（List 只读、Iterable 输出） | `out T` 或 `? extends T` |
| 容器**消费**T（Comparator 输入、Consumer） | `in T` 或 `? super T` |
| 容器**读写**T（MutableList） | 不变（默认） |

**实战**：
```kotlin
// Producer：返回值是 T
fun first(list: List<T>): T = list[0]              // List<out T>（生产者）

// Consumer：参数是 T
fun sort(list: MutableList<T>, comparator: Comparator<in T>)  // Comparator 是消费者
```

## 9. 实战案例

### 9.1 netlib-common：HttpConfig 的 List 字段

```kotlin
data class HttpConfig(
    val interceptors: List<Interceptor> = emptyList()  // List<out Interceptor> 协变
)
// ✅ 接受 List<AuthInterceptor>（subtype of Interceptor）
val auths: List<AuthInterceptor> = listOf(...)
val config = HttpConfig(interceptors = auths)        // ✅ OK
```

### 9.2 OkHttp：Call 是不变

```kotlin
// OkHttp 源码（Kotlin/Java 互操作）
// Call<T> 不变——因为 execute() 会返回 Response，但 cancel() 不消费 T
// 用户拿不到协变收益

// 如果想协变收益：自定义
interface Producer<out T> {
    fun produce(): T
}

interface Consumer<in T> {
    fun consume(t: T)
}
```

### 9.3 gRPC：Status Code 用协变

```kotlin
// gRPC 错误码（见 [[grpc-analysis/draft-09-error-handling#16-个grpc-status-code|grpc 状态码]]）
sealed class GrpcError<out T>(val code: Int) {
    data class Success<T>(val value: T) : GrpcError<T>(0)
    data class NotFound(val resource: String) : GrpcError<Nothing>(5)
    // NotFound 不持有 T，但仍是 GrpcError<T> 的子类型（协变）
}

fun handle(err: GrpcError<String>) {
    val any: GrpcError<AnyRef> = err              // ✅ 协变
}
```

### 9.4 自己定义协变类型

```kotlin
// 自定义 Result<T>——协变 T
sealed class MyResult<out T> {
    data class Ok<T>(val value: T) : MyResult<T>()
    data class Err(val message: String) : MyResult<Nothing>()
}

// 自定义失败原因——逆变（为什么不能用 out？）
sealed class CompareResult<in T> {
    data class Different<T>(val a: T, val b: T) : CompareResult<T>()
    // Different(a: T, b: T) 是「消费」 T
}
```

## 10. 与 Java 泛型对比

| 维度 | Kotlin | Java |
|------|--------|------|
| **型变位置** | 声明处（`out T`） | 使用处（`? extends T`） |
| **代码量** | 1 处声明 | 每个使用处都标 |
| **协变** | `out T` | `? extends T`（只在入参/变量） |
| **逆变** | `in T` | `? super T` |
| **不变** | 默认 | 默认 |
| **星投影** | `*` | `?`（裸） |
| **PECS 原则** | ✅ 同样适用 | ✅ 来源 |

**实战互调**：
```kotlin
// Kotlin 调 Java
// Java: void process(List<? extends Number> list)
fun process(list: List<out Number>)         // Kotlin 等价：List<out Number>

// Java: void consume(List<? super Integer> list)
fun consume(list: List<in Int>)            // Kotlin 等价：List<in Int>
```

## 11. 常见错误

### 11.1 `out T` 用在错误位置

```kotlin
// ❌ 错误：T 既是输入又是输出
interface Container<out T> {
    fun get(): T                // ✅ out OK
    fun set(t: T)               // ❌ in 不能用 out（编译错）
}
```

### 11.2 `in T` 用在错误位置

```kotlin
// ❌ 错误：T 在输出位置
interface Consumer<in T> {
    fun consume(t: T)           // ✅ in OK
    fun produce(): T           // ❌ out 不能用 in（编译错）
}
```

### 11.3 期望协变却不变

```kotlin
data class Box<T>(var item: T)        // 不变（var 可写）
// val boxStr: Box<String> = Box("a")
// val boxAny: Box<AnyRef> = boxStr  // ❌ 编译错

// 解决：用 val 字段（List<out T>）或 Box<T> 在不同位置实现协变
data class Box<out T>(val item: T)   // 协变（不可变）
```

## 12. 类型擦除与协变（运行时）

```kotlin
// 编译期有泛型，运行时擦除（Type Erasure）
val list: List<String> = listOf("a")
println(list::class)                // ArrayList
// list.javaClass 看不到泛型参数

// 但协变是编译期检查——运行时不影响
```

**实战**：gRPC 反序列化从 wire 拿数据时，**编译器保证 List<String> 只装 String**——但运行时只能看到 List。

## 13. reified + 型变

```kotlin
// reified 只能在 inline 函数，且需要带泛型
inline fun <reified T> List<*>.filterIsInstance(): List<T> {
    return this.filter { it is T }
}

val mixed: List<Any> = listOf(1, "a", "b", 2.0)
val strs: List<String> = mixed.filterIsInstance()  // [a, b]

// ⚠️ reified 不能与 in/out 直接组合（编译限制）
// inline reified <reified T> 但类型参数不能用 in/out 变型
//（实战影响小——reified 主要用于类型判断）
```

## 14. 关键设计原则

1. **数据只读用 out**（List、Set、Iterable）
2. **消费者用 in**（Comparator、Consumer、Function 参数）
3. **容器可读写用不变**（MutableList、MutableMap）
4. **声明处 vs 使用处**——Kotlin 选前者
5. **PECS 原则**——Producer Extends, Consumer Super
6. **避免 `*`**——只在确实未知类型时用

## 15. 反模式

❌ **强制协变让可变集合协变**——会破坏类型安全
❌ **滥用 `in` 让所有泛型逆变**——失去灵活性
❌ **期望 List<String> = List<AnyRef>**——Kotlin 不变默认是安全的
❌ **跨语言混淆 `out`/`in`**——和 Java 的 `extends`/`super` 语义一致
❌ **用星投影代替具体类型**——失去类型检查

## 16. 实战速查表

```
是否用 List<T> 参数？  → 协变 out（只读）
是否用 MutableList<T> 参数？  → 不变（可写）
是否用 Comparator<T> 参数？  → 逆变 in（只消费）
是否用 Function<T, R>？  → T in, R out（组合）
是否用 MyResult<T>？  → 协变 out（值容器）
是否用 Consumer<T>？  → 逆变 in（消费者）
```

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **OOP**: [[draft-06-oop]]（sealed class 常用协变）
- **netlib-common**: [[netlib-common HttpConfig]]（List<Interceptor> 用 out）
- **gRPC 错误码**: [[grpc-analysis/draft-09-error-handling]]
- **综合入口**: [[summary]]