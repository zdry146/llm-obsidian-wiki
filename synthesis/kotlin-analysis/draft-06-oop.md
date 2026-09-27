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
base_confidence: 0.92
lifecycle: stable
lifecycle_changed: 2026-09-27
created: 2026-09-13
updated: 2026-09-27
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

### 1.1 User 类逐项拆解

#### ① 主构造函数（primary constructor）

```kotlin
class User(val name: String, val age: Int)
```

- 写在类头上的就是**主构造函数**
- 关键点：`val name` / `val age` 里的 `val` 让这俩参数**自动变成类的属性**（生成私有字段 + getter）
- 如果只写 `class User(name: String, age: Int)`（不带 `val/var`），那只是构造参数，构造完就丢，**不能** `user.name` 这样访问

对应 Java：
```java
class User {
    private final String name;
    private final int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
}
```

#### ② `init` 块

```kotlin
init {
    require(age >= 0) { "Age must be positive" }
}
```

- `init { ... }` 是**主构造函数的一部分**，构造对象时跟着一块跑
- 一个类可以有**多个** `init` 块，按声明顺序从上到下执行
- `require(condition) { "msg" }` = Kotlin 标准库断言：
  - `true` → 通过
  - `false` → 抛 `IllegalArgumentException(msg)`
- `{ "..." }` 是 lazy lambda — **只有真出错时才拼字符串**

#### ③ 成员函数

```kotlin
fun greet(): String = "Hello, $name!"
```

- `fun` 声明函数
- `= "..."` 是**表达式体**（expression body），单表达式时更简洁
- 等价写法：
```kotlin
fun greet(): String { return "Hello, $name!" }
```
- 字符串模板：`$name` 直接插变量；`${expr}` 插任意表达式（`${name.uppercase()}`）

#### ④ 自定义 getter 的属性

```kotlin
val isAdult: Boolean
    get() = age >= 18
```

- 声明格式：`val/var 属性名: 类型 get() = ...`
- 这种写法 = **没有 backing field**（每次访问都重算）
- 编译器看到只读了 `age`、没存自己，所以不生成字段
- 调用时跟普通属性一样：`user.isAdult`（**不加括号**）

#### ⑤ 另一个 computed property

```kotlin
val description: String
    get() = "User($name, $age)"
```

- 跟 `isAdult` **完全同款**，都是 computed property，无 backing field
- 注释里分开写只是为了演示两种典型用法 — 一个返回 `Boolean`、一个返回 `String`，机制相同

---

**对照速查**：

| 写法 | 含义 |
|---|---|
| `class Foo(val x: Int)` | 参数 + 自动属性（带 backing field） |
| `class Foo(x: Int)` | 纯构造参数（构造完就丢，访问不到） |
| `val y: Int = 10` | 普通属性，有 backing field |
| `val y: Int get() = ...` | 计算属性，**无 backing field** |
| `var y: Int get()=...; set(v){...}` | 自定义 getter + setter，可加校验 |
| `init { ... }` | 构造期执行，可多个 |
| `fun foo() = expr` | 表达式体函数（单表达式） |
| `fun foo() { ... }` | 块体函数（多行用这个） |

**核心洞察**：整段 User 类**没有任何 Java 风格的 getter/setter/构造器样板**，全靠 `val`、expression body、computed property 把样板消掉。这就是 Kotlin OOP 的核心收益 —— **声明即规格，计算即访问**。

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

## 8.1 实战枚举设计模式（完整示例）

### 模式 1：多属性 + 分类助手（HTTP Status 风格）

```kotlin
// 类似 gRPC/OkHttp 的 status code 设计
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "OK"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    SERVER_ERROR(500, "Internal Server Error");

    fun isSuccess() = code in 200..299
    fun isClientError() = code in 400..499
    fun isServerError() = code in 500..599
    fun isError() = !isSuccess()

    companion object {
        private val byCode = values().associateBy { it.code }
        fun fromCode(code: Int): HttpStatus = byCode[code]
            ?: throw IllegalArgumentException("Unknown HTTP code: $code")
    }
}

val status = HttpStatus.OK
status.isSuccess()           // true
HttpStatus.fromCode(404)     // NOT_FOUND
HttpStatus.fromCode(999)     // 抛 IllegalArgumentException
```

**实战应用**：gRPC 的 16 个 Status Code 是同模式（见 [[grpc-analysis/draft-09-error-handling]]）。

### 模式 2：实现接口（每个枚举值重写方法）

```kotlin
interface Drawable {
    fun draw(): String
}

enum class Shape : Drawable {
    CIRCLE {
        override fun draw() = "○"
        override fun area(r: Double) = Math.PI * r * r
    },
    SQUARE {
        override fun draw() = "□"
        override fun area(r: Double) = r * r
    },
    TRIANGLE {
        override fun draw() = "△"
        override fun area(r: Double) = Math.sqrt(3.0) / 4 * r * r
    };

    abstract fun area(r: Double): Double   // 每个枚举重写
}

Shape.CIRCLE.draw()           // "○"
Shape.SQUARE.area(2.0)        // 4.0
```

**核心洞察**：每个枚举值是**匿名子类**，可重写方法。

### 模式 3：伴生对象 + 工厂方法（性能优化）

```kotlin
enum class LogLevel(val level: Int, val tag: String) {
    DEBUG(0, "DEBUG"),
    INFO(1, "INFO"),
    WARN(2, "WARN"),
    ERROR(3, "ERROR");

    companion object {
        // 反向索引（O(1) 查找）
        private val byTag = values().associateBy { it.tag }

        fun fromTag(tag: String): LogLevel = byTag[tag.uppercase()]
            ?: throw IllegalArgumentException("Unknown level: $tag")

        fun isValid(tag: String) = tag.uppercase() in byTag
    }
}

val level = LogLevel.fromTag("warn")    // WARN
LogLevel.isValid("INFO")                // true
LogLevel.isValid("trace")               // false
```

### 模式 4：Kotlin 1.9+ `entries`（替代 `values()`）

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }

// ✅ Kotlin 1.9+：推荐使用 entries
val allDirections: List<Direction> = Direction.entries
// - 不可变 List（更安全）
// - 不每次创建新数组（更高效）

// ❌ values()（仍可用但不推荐）
val array: Array<Direction> = Direction.values()
// - 每次调用都 new Array
// - 可变（可被外部修改）
```

**实战**：用 `entries` 做映射、过滤：

```kotlin
val activeDirections = Direction.entries.filter { it != Direction.WEST }

// 用 entries 做反索引（init 块中构建）
enum class HttpMethod(val method: String) {
    GET("GET"), POST("POST"), PUT("PUT"), DELETE("DELETE");

    companion object {
        val byMethod = entries.associateBy { it.method }
    }
}

HttpMethod.byMethod["POST"]   // POST
```

### 模式 5：枚举的扩展函数

```kotlin
fun Status.next(): Status = when (this) {
    Status.ACTIVE -> Status.INACTIVE
    Status.INACTIVE -> Status.DELETED
    Status.DELETED -> Status.ACTIVE
}

fun Status.shortName(): String = name.take(3)

Status.ACTIVE.next()       // INACTIVE
Status.ACTIVE.shortName()  // "ACT"
```

**实战**：在 `Extensions.kt` 集中放扩展函数，不污染枚举定义文件。

### 模式 6：与 sealed class 决策树

```kotlin
sealed class Event {
    object Idle : Event()
    data class Loading(val progress: Int) : Event()
    data class Success(val data: List<Item>) : Event()
    data class Failure(val error: Throwable) : Event()
}

fun render(event: Event) = when (event) {
    is Event.Idle -> "空闲"
    is Event.Loading -> "加载中: ${event.progress}%"
    is Event.Success -> "成功: ${event.data.size} 条"
    is Event.Failure -> "失败: ${event.error.message}"
    // 编译器穷尽检查
}
```

**何时用 enum** vs **sealed class**：
- ✅ enum：所有实例同构（相同字段）
- ✅ sealed class：每个子类字段不同

### 模式 7：枚举 in gRPC 16 个状态码（真实实战）

```kotlin
// 与 grpc-analysis/draft-09-error-handling 对照
enum class GrpcStatus(val code: Int, val isOk: Boolean) {
    OK(0, true),
    CANCELLED(1, false),
    UNKNOWN(2, false),
    INVALID_ARGUMENT(3, false),
    DEADLINE_EXCEEDED(4, false),
    NOT_FOUND(5, false),
    ALREADY_EXISTS(6, false),
    PERMISSION_DENIED(7, false),
    RESOURCE_EXHAUSTED(8, false),
    FAILED_PRECONDITION(9, false),
    ABORTED(10, false),
    OUT_OF_RANGE(11, false),
    UNIMPLEMENTED(12, false),
    INTERNAL(13, false),
    UNAVAILABLE(14, false),
    DATA_LOSS(15, false),
    UNAUTHENTICATED(16, false);

    companion object {
        private val byCode = entries.associateBy { it.code }

        fun fromCode(code: Int): GrpcStatus = byCode[code] ?: UNKNOWN

        fun isRetryable(code: Int): Boolean {
            val s = fromCode(code)
            return s in setOf(UNAVAILABLE, RESOURCE_EXHAUSTED, ABORTED)
        }
    }
}

GrpcStatus.fromCode(14)          // UNAVAILABLE
GrpcStatus.isRetryable(14)       // true（可重试）
GrpcStatus.isRetryable(3)        // false（参数错，不重试）
GrpcStatus.isRetryable(-1)       // false（未知 code 返回 UNKNOWN）
```

**实战**：`netlib-common` 的 `SmartRetryInterceptor` 重试逻辑可以直接用这个枚举。

### 模式 8：自定义序列化（与 Protobuf 互转）

```kotlin
enum class Priority(val protoValue: Int) {
    LOW(1),
    MEDIUM(2),
    HIGH(3);

    fun toProto(): com.example.Priority = when (this) {
        LOW -> com.example.Priority.LOW
        MEDIUM -> com.example.Priority.MEDIUM
        HIGH -> com.example.Priority.HIGH
    }

    companion object {
        fun fromProto(p: com.example.Priority): Priority = when (p) {
            com.example.Priority.LOW -> LOW
            com.example.Priority.MEDIUM -> MEDIUM
            com.example.Priority.HIGH -> HIGH
            else -> MEDIUM
        }
    }
}
```

**实战**：在 gRPC service 中转换 enum ↔ Protobuf enum。

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

### 10.2.1 四个属性委托逐项拆解

#### ① `lazy` 委托

```kotlin
val heavy: HeavyObject by lazy {
    HeavyObject()
}
```

- `lazy { ... }` 是 Kotlin 标准库的**委托工厂**
- 接收一个 lambda，**第一次**访问 `heavy` 时才执行 lambda，之后缓存返回值
- 默认线程安全（`LazyThreadSafetyMode.SYNCHRONIZED`）
- 可选模式：
  ```kotlin
  val heavy by lazy(LazyThreadSafetyMode.NONE) { HeavyObject() }       // 单线程，更快
  val heavy by lazy(LazyThreadSafetyMode.PUBLICATION) { HeavyObject() } // 多读单写
  ```
- 适用：初始化开销大、不一定用到的对象

Java 对照（最接近的等价写法）：
```java
class HeavyHolder {
    private volatile HeavyObject heavy;
    public HeavyObject getHeavy() {
        if (heavy == null) {
            synchronized (this) {
                if (heavy == null) heavy = new HeavyObject();
            }
        }
        return heavy;
    }
}
```

#### ② `Delegates.observable` 委托

```kotlin
var name: String by Delegates.observable("initial") { _, old, new ->
    println("Changed from $old to $new")
}
```

- 第一个参数：初始值
- 第二个参数：lambda `(property, oldValue, newValue) -> Unit`
- **每次赋值后**触发 callback（after the assignment）
- 适用：UI 状态变化通知、数据变更日志、Property Change 监听
- ⚠️ observable **不能拒绝赋值**，只能观察

#### ③ `Delegates.vetoable` 委托

```kotlin
var age: Int by Delegates.vetoable(0) { _, _, new ->
    new >= 0    // false 时拒绝
}
```

- lambda 签名 `(property, oldValue, newValue) -> Boolean`
- 返回 `true` → 接受新值；返回 `false` → **拒绝，旧值保留**
- 适用：表单校验、范围限制、不变量保护

**observable vs vetoable**：

| 维度 | observable | vetoable |
|---|---|---|
| Lambda 返回类型 | `Unit` | `Boolean` |
| 触发时机 | 赋值后（after） | 赋值前（before） |
| 能拒绝赋值？ | ❌ | ✅ |
| 拒绝时 | — | 旧值保留 |

#### ④ `map` 委托

```kotlin
val map = mutableMapOf("a" to 1)
val a: Int by map                   // map["a"] 作为 a 的值
```

- 把 **Map 的 key** 当成属性的"存储后端"
- 读 `a` ≡ `map["a"]`；写 `a = 2` ≡ `map["a"] = 2`
- **Map 必须可变**（`mutableMapOf` / `HashMap`），不可变 Map 只能读不能写
- 适用：从配置 / JSON / DB 读出的 Map 直接绑成属性

实战：动态配置
```kotlin
val config = mutableMapOf<String, Any>(
    "timeout" to 30,
    "retries" to 3
)
val timeout: Int by config
val retries: Int by config

println(timeout)            // 30
config["timeout"] = 60
println(timeout)            // 60 — 自动反映
```

---

**核心洞察**：所有 `by xxx` 委托的本质都是**把属性的 getter/setter 转发给某个 backing object**。Kotlin 编译器在编译期自动生成对应的 `getValue()` / `setValue()` 扩展函数调用 — 所以你写的 `val heavy by lazy { ... }` 实际上是 Kotlin 帮你写了访问 lazy 委托的样板代码。这也是为什么你可以给任意类自定义委托：只要实现 `operator fun getValue(...)` 和 `operator fun setValue(...)`，就能 `by` 它。

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