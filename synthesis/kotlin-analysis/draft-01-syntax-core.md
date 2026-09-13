---
title: "Kotlin 核心语法与类型系统"
category: synthesis
tags: [kotlin, syntax, type-system, null-safety, data-class, sealed-class]
sources:
  - "Kotlin Docs - Basic Syntax"
  - "Kotlin Docs - Null Safety"
  - "Kotlin Docs - Classes"
summary: "Kotlin 核心语法：val/var、type inference、null 安全、smart cast、data class、sealed class、object、enum"
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

# §01 Kotlin 核心语法与类型系统

## 1. val vs var

```kotlin
val name = "Mike"          // 不可变（final）
var age = 30               // 可变

val name: String = "Mike"  // 显式类型
val age = 30               // 类型推断：Int
```

**核心规则**：
- ✅ **优先用 val**（不可变 = 函数式编程 + 线程安全）
- ❌ **避免 var**（除非必要）

## 2. 基本类型

```kotlin
val int: Int = 100
val long: Long = 100L
val double: Double = 3.14
val float: Float = 3.14f
val bool: Boolean = true
val char: Char = 'a'
val str: String = "Hello"

// 字符串模板
val name = "Mike"
println("Hello, $name!")               // 单变量
println("Sum: ${1 + 2}")                 // 表达式
println("""Multi
line
string""")                              // 三引号（无转义）
```

**与 Java 对比**：
| 维度 | Kotlin | Java |
|------|--------|------|
| 数字 | `Int` (32-bit) | `int` |
| 长 | | `long` |
| 字符 | | `char` (16-bit) |
| 字符串模板 | ✅ 内置 | ❌ 需 `String.format()` |

## 3. Null 安全（核心创新）

```kotlin
// 非空类型：永远不会是 null
var name: String = "Mike"   // ✅ 编译期保证
name = null                 // ❌ 编译错误

// 可空类型：必须显式标记
var name: String? = "Mike"  // String? = 可空
name = null                 // ✅

// 安全调用
val length = name?.length   // name 为 null → 返回 null

// Elvis 操作符
val length = name?.length ?: 0   // name 为 null → 返回 0

// 强制非空（危险，慎用）
val length = name!!.length       // name 为 null → NPE
```

**与 Java 对比**：

```java
// Java
String name = null;            // 编译通过
int length = name.length();    // NPE
```

```kotlin
// Kotlin
var name: String? = null
val length = name?.length      // null.length 返回 null（无 NPE）
```

**核心洞察**：**Kotlin 把 NPE 从「运行时错误」变为「编译期错误」**。

### 3.1 安全转换（!!）的使用场景

```kotlin
// 场景 1：依赖前置检查
requireNotNull(user) { "User must be set" }
val name = user.name  // 现在 user 是非空

// 场景 2：lateinit
class MyService {
    lateinit var db: Database
    
    fun init() {
        db = Database.connect(...)
        db.query()  // ✅ OK
    }
}

// 场景 3：Java 互操作（平台类型不可控）
val user = javaMethod()       // 可能是 null
val name = user?.name ?: ""    // 显式处理
```

⚠️ **避免 `!!`**：除非你 100% 确定非空。

## 4. 类型推断

```kotlin
val a = 5                    // Int
val b = "hello"              // String
val c = listOf(1, 2, 3)     // List<Int>
val d = mapOf("a" to 1)      // Map<String, Int>

// 显式声明（避免歧义）
val x: Int = 5
val y: List<String> = emptyList()

// 函数返回类型推断
fun add(a: Int, b: Int) = a + b  // 返回 Int
```

**规则**：编译器自动推断类型，但你可以显式声明。

## 5. 智能转换（Smart Cast）

```kotlin
fun describe(obj: Any): String = when (obj) {
    1 -> "One"                // Int 自动转换
    "Hello" -> "Greeting"     // String 自动转换
    is Long -> "Long: $obj"   // is 检查后自动转换
    !is String -> "Not String"
    is Array<*> -> "Array"
    else -> "Unknown"
}

// if 也是表达式
val x: Any = 5
val description = when (x) {
    is Int -> "Number $x"
    is String -> "String of length ${x.length}"
    else -> "Other"
}
```

**vs Java**：
```java
// Java 必须显式 cast
if (obj instanceof String) {
    String s = (String) obj;
    s.length();
}
```

## 6. data class（杀手特性）

```kotlin
// 一行 = 完整 POJO
data class User(
    val name: String,
    val age: Int,
    val email: String? = null  // 默认参数
)

// 自动生成：
// - equals() / hashCode()
// - toString() = "User(name=Mike, age=30, email=null)"
// - copy() / componentN() (解构)
val user = User("Mike", 30)
val older = user.copy(age = 31)
val (name, age, email) = user  // 解构

// ✅ 不可变（val 字段）
// ✅ 简洁（1 行 = 30 行 Java POJO）
```

**实际应用**：
```kotlin
data class HttpConfig(
    val connectTimeout: Duration = 10.seconds,
    val readTimeout: Duration = 30.seconds,
    val interceptors: List<Interceptor> = emptyList()
)

// copy() 用于派生配置
val prodConfig = baseConfig.copy(
    connectTimeout = 5.seconds,
    maxRequests = 256
)
```

## 7. sealed class（受控继承）

```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// 使用：when 表达式强制穷尽
fun <T> handle(result: Result<T>) = when (result) {
    is Result.Success -> println("Got: ${result.value}")
    is Result.Failure -> println("Error: ${result.error}")
    is Result.Loading -> println("Loading...")
    // 无需 else！编译器保证所有 case 处理
}
```

**与 enum 的区别**：
| 维度 | sealed class | enum |
|------|--------------|------|
| 实例化 | 子类（可携带不同状态） | 单例值 |
| 状态 | 每个子类独立字段 | 所有实例同字段 |
| when 表达式 | 编译器强制穷尽 | 强制穷尽 |
| 适用 | 「有限的几种状态 + 各自字段」 | 「固定值」 |

## 9. object（单例）

```kotlin
// 单例
object Database {
    fun connect() { /* ... */ }
}
Database.connect()  // 直接用

// companion object（伴随对象 = Java static）
class UserService {
    companion object {
        private const val MAX_USERS = 1000
        
        @JvmStatic
        fun create() = UserService()
    }
}

// Java 调用：UserService.create() 或 UserService.Companion.create()
// @JvmStatic 标注后：UserService.create()（不需要 .Companion）
```

## 10. enum

```kotlin
enum class Status(val code: Int) {
    ACTIVE(1),
    INACTIVE(0),
    DELETED(-1);
    
    fun isValid() = code >= 0
}

// 使用
val s = Status.ACTIVE
println(s.code)        // 1
println(s.isValid())   // true
```

## 11. 范围（Range）

```kotlin
val range1 = 1..10          // [1, 10] 闭区间
val range2 = 1 until 10    // [1, 10) 半开区间
val range3 = 10 downTo 1   // [10, 1] 倒序
val range4 = 1..10 step 2  // 1, 3, 5, 7, 9

// 包含检查
5 in 1..10            // true
15 !in 1..10          // true

// 遍历
for (i in 1..5) {
    println(i)
}
```

## 12. 类型系统总览

| 类型 | 示例 | 说明 |
|------|------|------|
| `Int` | `42` | 32-bit |
| `Long` | `42L` | 64-bit |
| `Double` | `3.14` | 64-bit 浮点 |
| `String` | `"hello"` | 不可变字符串 |
| `Boolean` | `true`/`false` | |
| `Any` | — | 所有类型的父类（= Java Object）|
| `Nothing` | — | 无值（永不返回的函数） |
| `Unit` | — | 无有意义返回（= Java void） |
| `Array<T>` | `arrayOf(1, 2)` | |
| `List<T>` | `listOf(1, 2)` | 不可变 |
| `MutableList<T>` | `mutableListOf(1, 2)` | 可变 |
| `Map<K, V>` | `mapOf("a" to 1)` | |
| `Set<T>` | `setOf(1, 2, 3)` | |

## 13. 与 Java 类型对比

```kotlin
val int: Int     // Java: int (primitive)
val long: Long   // Java: long (primitive)
val boxed: Int?  // Java: Integer (wrapper)
```

**关键点**：
- Kotlin 不区分 primitive vs wrapper（编译期优化）
- `Int?` 自动对应 `Integer`
- `List<Int>` 是 `List<Integer>`（Java 看到）

## 14. 类型别名

```kotlin
typealias UserMap = Map<String, User>
typealias Handler = (Request) -> Response

val users: UserMap = mapOf("mike" to User("Mike", 30))
fun register(h: Handler) { /* ... */ }
```

## 15. 关键设计原则

1. **类型安全**——null 用 `?` 显式标记
2. **类型推断**——不需要冗余类型声明
3. **不可变优先**——val > var
4. **data class**——POJO 用一行
5. **sealed class**——封闭的继承层级
6. **object**——单例和伴随对象

## 16. 反模式

❌ **var 滥用**——优先 val
❌ **!! 滥用**——除非要 100% 确定
❌ **data class 字段过多**——考虑普通类
❌ **object 替代 DI**——大型项目用 Hilt/Koin
❌ **nullable Boolean**——很难用，考虑 sealed class

## 相关笔记

- **函数与扩展**: [[draft-02-functions]]
- **协程**: [[draft-03-coroutines]]
- **DSL**: [[draft-04-dsl-builders]]
- **集合**: [[draft-05-collections]]
- **OOP**: [[draft-06-oop]]
- **综合入口**: [[summary]]