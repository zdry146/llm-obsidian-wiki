---
title: "Kotlin 面向对象"
category: synthesis
tags: [kotlin, oop, sealed-class, data-class, object, enum, inheritance]
sources:
  - "Kotlin Docs - Classes and Inheritance"
  - "Kotlin Docs - Sealed Classes"
  - "Kotlin Docs - Objects"
summary: "Kotlin OOP：data class、sealed class、object、enum、open class 继承、扩展属性、abstract class"
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

# §06 Kotlin 面向对象

## 1. 类声明

```kotlin
// 基本类
class User(val name: String, val age: Int) {
    // init 块（构造器逻辑）
    init {
        require(age >= 0) { "Age must be positive" }
    }
    
    // 成员函数
    fun greet(): String = "Hello, $name!"
    
    // 属性
    val isAdult: Boolean
        get() = age >= 18
    
    // 计算属性（无 backing field）
    val description: String
        get() = "User($name, $age)"
}

// 使用
val user = User("Mike", 30)
println(user.greet())           // "Hello, Mike!"
println(user.isAdult)          // true
```

## 2. data class（杀手特性）

```kotlin
data class User(
    val name: String,
    val age: Int,
    val email: String? = null
)

// 自动生成：
// - equals/hashCode
// - toString() = "User(name=Mike, age=30, email=null)"
// - copy() = 复制并修改部分字段
// - componentN() = 解构支持

// 实战
val mike = User("Mike", 30, "mike@example.com")
val older = mike.copy(age = 31)
val (name, age, email) = older    // 解构

// ⚠️ 限制
// - 主构造器必须有至少 1 个参数
// - 不能是 abstract / open / sealed / inner
// - 自动生成 equals/hashCode 只看主构造器字段
```

**实战：netlib-common 的 HttpConfig 就是 data class**：

```kotlin
data class HttpConfig(
    val connectTimeout: Duration = 10.seconds,
    val readTimeout: Duration = 30.seconds,
    val maxIdleConnections: Int = 5
) {
    init {
        require(!connectTimeout.isNegative)
        require(maxIdleConnections >= 0)
    }
}

// copy() 派生配置
val prodConfig = baseConfig.copy(
    connectTimeout = 5.seconds,
    maxIdleConnections = 50
)
```

## 3. sealed class（封闭继承）

```kotlin
// 封闭类：所有子类在同一文件/包/模块内
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// ✅ 模式匹配：编译器保证穷尽
fun <T> handle(result: Result<T>) = when (result) {
    is Result.Success -> process(result.value)
    is Result.Failure -> log.error("Failed", result.error)
    is Result.Loading -> showSpinner()
    // 无需 else！编译器验证所有 case
}

// 子类必须继承 sealed
sealed class HttpResult<out T> : Result<T>() {
    data class Created(val location: String) : HttpResult<Nothing>()
    data class BadRequest(val errors: List<String>) : HttpResult<Nothing>()
}
```

**核心洞察**：**sealed class 是 Kotlin 替代 enum + 状态机的优雅方案**——比 Java enum 更灵活（每个子类可携带不同字段）。

### 3.1 sealed interface（Kotlin 1.5+）

```kotlin
// sealed interface：多个文件/模块共享
sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: List<Item>) : UiState
    data class Error(val message: String) : UiState
}
```

## 4. object（单例与伴随对象）

### 4.1 对象单例

```kotlin
// 单例：懒加载
object DatabaseManager {
    private var connection: Connection? = null
    
    fun getConnection(): Connection {
        return connection ?: synchronized(this) {
            connection ?: DriverManager.getConnection("...").also {
                connection = it
            }
        }
    }
}

// 使用：直接调用
DatabaseManager.getConnection()
```

### 4.2 companion object（伴随对象 = Java static）

```kotlin
class UserService {
    companion object {
        private const val MAX_USERS = 1000
        
        // 静态方法
        @JvmStatic                              // Java 调用更简洁
        fun create(): UserService = UserService()
        
        // 静态字段
        @JvmField                              // Java 直接访问字段
        val DEFAULT_TIMEOUT = 30L
    }
}

// Kotlin 调用：UserService.create()
// Java 调用：UserService.create() 或 UserService.Companion.create()
// @JvmStatic 后：UserService.create()（不带 Companion）
```

### 4.3 object expression（匿名对象）

```kotlin
// 类似 Java 匿名内部类
val runnable = object : Runnable {
    override fun run() {
        println("Running")
    }
}

// Java SAM（functional interface）
val listener = View.OnClickListener { view ->
    println("Clicked: $view")
}
```

## 5. 继承（open class）

```kotlin
// 默认 class 是 final（不能继承）
// 用 open 显式标记可继承
open class Animal(val name: String) {
    open fun makeSound() {
        println("???")
    }
}

class Dog(name: String, val breed: String) : Animal(name) {
    override fun makeSound() {
        println("Woof!")
    }
}

// ❌ 不需要继承时不要标记 open（设计成 final 更安全）
```

**关键原则**：**默认 final，open 需显式标记**——鼓励组合而非继承。

## 6. abstract class

```kotlin
abstract class Shape {
    abstract fun area(): Double
    abstract fun perimeter(): Double
    
    // 具体方法（默认实现）
    fun describe(): String = "Shape with area=${area()}"
}

class Circle(val r: Double) : Shape() {
    override fun area() = Math.PI * r * r
    override fun perimeter() = 2 * Math.PI * r
}

// 使用
val shape: Shape = Circle(5.0)
println(shape.area())         // 78.54
println(shape.describe())     // "Shape with area=78.54"
```

## 7. interface

```kotlin
interface Drawable {
    fun draw()                                    // 抽象方法
    fun resize(scale: Double) = defaultResize()   // 默认实现
    
    private fun defaultResize() = println("Default resize")
}

// 多实现
class Button : Drawable, Clickable {
    override fun draw() { /* ... */ }
    override fun onClick() { /* ... */ }
}

// interface 字段（Kotlin 1.0+）
interface Versioned {
    val version: String       // 抽象
}

class MyClass : Versioned {
    override val version = "1.0"
}
```

## 8. 枚举（enum）

```kotlin
enum class Status(val code: Int, val displayName: String) {
    ACTIVE(1, "Active"),
    INACTIVE(0, "Inactive"),
    DELETED(-1, "Deleted");
    
    fun isValid() = code >= 0
    
    // 枚举实现接口
    fun color(): String = when (this) {
        ACTIVE -> "green"
        INACTIVE -> "gray"
        DELETED -> "red"
    }
}

// 使用
val s = Status.ACTIVE
println(s.code)              // 1
println(s.displayName)       // "Active"
println(s.color())           // "green"

// 遍历
for (status in Status.values()) {
    println(status)
}

// 字符串转枚举
val parsed = Status.valueOf("ACTIVE")
```

## 9. 扩展属性

```kotlin
// 给已有类添加属性
val String.wordCount: Int
    get() = split(" ").size

val List<Int>.average: Double
    get() = sum().toDouble() / size

// 使用
"Hello world".wordCount       // 2
listOf(1, 2, 3).average        // 2.0

// 实战
val File.sizeMb: Double
    get() = length() / 1024.0 / 1024.0
```

## 10. 委托（by）

### 10.1 类委托（替代继承）

```kotlin
// 接口
interface Repository {
    fun findById(id: String): User?
    fun save(user: User)
}

// 委托：自动转发所有方法给 backing object
class CachingRepository(
    private val delegate: Repository
) : Repository by delegate {
    // 可以选择性 override
    override fun findById(id: String): User? {
        println("Cache miss for $id")
        return delegate.findById(id)
    }
}
```

### 10.2 属性委托

```kotlin
import kotlin.properties.Delegates

// lazy（懒加载）
val heavy: HeavyObject by lazy {
    HeavyObject()    // 只在第一次访问时初始化
}

// observable（观察变化）
var name: String by Delegates.observable("initial") { _, old, new ->
    println("Changed from $old to $new")
}

// vetoable（可拒绝变化）
var age: Int by Delegates.vetoable(0) { _, _, new ->
    new >= 0    // false 时拒绝
}

// map 委托
val map = mutableMapOf("a" to 1)
val a: Int by map                   // map["a"] 作为 a 的值
```

## 11. 嵌套类与内部类

```kotlin
// 嵌套类（= Java static nested class）
class Outer {
    class Nested {  // 不持有外部引用
        fun foo() = "Nested"
    }
}

// 内部类（= Java inner class）
class Outer {
    inner class Inner {  // 持有外部引用
        fun foo() = this@Outer.toString()
    }
}
```

## 12. 实战：HTTP 错误体系

```kotlin
// sealed class 表示 RPC 错误
sealed class ApiError(val httpStatus: Int) {
    data class BadRequest(val errors: List<String>) : ApiError(400)
    data class Unauthorized(val message: String) : ApiError(401)
    data class NotFound(val resource: String) : ApiError(404)
    data class ServerError(val code: Int) : ApiError(500)
}

// 使用：when 强制穷尽
fun handleError(error: ApiError): String = when (error) {
    is ApiError.BadRequest -> "Invalid input: ${error.errors}"
    is ApiError.Unauthorized -> "Login required: ${error.message}"
    is ApiError.NotFound -> "Not found: ${error.resource}"
    is ApiError.ServerError -> "Server error ${error.code}"
}
```

## 13. 关键设计原则

1. **默认 final**——`open` 显式标记可继承
2. **data class**——POJO 用一行
3. **sealed class**——有限状态+各自字段
4. **object**——单例和伴随对象
5. **接口优先**——继承用接口
6. **类委托**——组合优于继承

## 14. 反模式

❌ **所有类都标记 open**——破坏封装
❌ **不用 sealed class 用 enum**——enum 不能带字段
❌ **data class 继承**——data class 不能 open
❌ **object 替代 DI**——大型项目用 Hilt/Koin
❌ **多层嵌套类**——可读性差

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **协程**: [[draft-03-coroutines]]
- **Java 互操作**: [[draft-08-java-interop]]
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`
- **综合入口**: [[summary]]