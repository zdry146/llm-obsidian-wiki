---
title: "Kotlin ↔ Java 互操作"
category: synthesis
tags: [kotlin, java, interop, jvm, jvm-field, jvm-static, jvm-overloads]
sources:
  - "Kotlin Docs - Java Interop"
  - "Kotlin Docs - Calling Java from Kotlin"
  - "Kotlin Docs - Calling Kotlin from Java"
summary: "Kotlin ↔ Java 互操作：平台类型、@JvmStatic/@JvmField/@JvmOverloads、nullability 映射、synthetic method"
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

# §08 Kotlin ↔ Java 互操作

## 1. 互操作的核心：100% 兼容

Kotlin **100% 兼容 Java**：
- ✅ Kotlin 可以调 Java 任何类
- ✅ Java 可以调 Kotlin 类（带注解调整 API 风格）
- ✅ 同一个 project 混用 Kotlin + Java
- ✅ 同一个 .class 文件可被 Java/Kotlin 互读

**核心洞察**：**Kotlin 不替代 Java，是 Java 的超集**。

## 2. Kotlin 调 Java

### 2.1 平台类型（Platform Type）

```java
// Java
public class UserService {
    public String findName(int id) { return "Mike"; }  // 可能为 null
}
```

```kotlin
// Kotlin
val service = UserService()
// 平台类型：String!（编译器不知道是否可空）
val name = service.findName(1)  // type: String!
// 可以当非空用
println(name.length)            // ✅ 编译通过（运行时可能 NPE）

// 显式处理
val nullable: String? = service.findName(1)  // ✅ 接受 null
val safe: String = service.findName(1) ?: "default"
```

**核心洞察**：**Java 代码的返回类型，Kotlin 视为「平台类型」——既可当可空也可当非空**。

### 2.2 常见 Java 类型映射

| Java | Kotlin |
|------|--------|
| `int` | `Int`（非空） |
| `Integer` | `Int!`（平台类型） |
| `@Nullable String` | `String?` |
| `String[]` | `Array<String!>` |
| `List<String>` | `List<String!>` |
| `Map<K, V>` | `Map<K!, V!>` |
| `T` (泛型) | `T!`（平台类型） |
| `void` | `Unit` |

### 2.3 Java 集合操作

```kotlin
// Java 返回 List
val list = javaService.getUsers()   // List<User!>

// Kotlin 操作：调用 Java 的方法
list.add(User())                    // ✅ Java 集合可变
list[0]                              // ✅
// Kotlin 操作：转换成 Kotlin 风格
list.toList()                        // 转 Kotlin List<User>
list.filterNotNull()                 // 过滤 null
```

## 3. Java 调 Kotlin

### 3.1 默认映射

```kotlin
// Kotlin
class UserService {
    fun create(name: String, age: Int): User { ... }
    companion object {
        fun default(): UserService = UserService()
    }
}
```

```java
// Java 调用
UserService service = new UserService();
User user = service.create("Mike", 30);                 // 直接调
UserService.Companion.default();                          // companion 调
// 不用 Companion 需 @JvmStatic
```

### 3.2 @JvmStatic（让 Java 调 companion 不用 .Companion）

```kotlin
companion object {
    @JvmStatic
    fun default(): UserService = UserService()
}
```

```java
// Java：不需要 .Companion
UserService.default();    // ✅ @JvmStatic 后直接调
```

### 3.3 @JvmField（暴露字段给 Java）

```kotlin
class Config {
    companion object {
        @JvmField
        val MAX_USERS = 1000   // 暴露为 Java static field
    }
}
```

```java
int max = Config.MAX_USERS;    // ✅ 直接字段访问（不用 getter）
```

### 3.4 @JvmOverloads（生成 Java 风格重载）

```kotlin
class HttpClient @JvmOverloads constructor(
    val host: String = "localhost",
    val port: Int = 50051,
    val useTls: Boolean = true
)
```

```java
// Java：自动生成 4 个重载
new HttpClient();                          // 全默认
new HttpClient("api.example.com");         // 1 个参数
new HttpClient("api.example.com", 443);    // 2 个参数
new HttpClient("api.example.com", 443, true);  // 3 个参数
```

### 3.5 @JvmName（修改 Java 可见的方法名）

```kotlin
// Kotlin 关键字在 Java 里也是合法
fun @JvmName("isValid") isValid(): Boolean = ...

// 字段名冲突
val @JvmName("value") _value: Int = 0
```

### 3.6 @Throws（声明检查异常）

```kotlin
@Throws(IOException::class)
fun readFile(path: String): String = File(path).readText()
```

```java
// Java：必须 catch IOException（Kotlin 编译器声明）
try {
    String text = userService.readFile("file.txt");
} catch (IOException e) {
    // 必须处理
}
```

## 4. null 注解互操作

### 4.1 Kotlin 看 Java 注解

```java
// Java：@Nullable / @NotNull
import org.jetbrains.annotations.Nullable;

public class UserService {
    @Nullable
    public String findName(int id) { return null; }
}
```

```kotlin
// Kotlin：识别为可空
val name: String? = userService.findName(1)    // 正确推断
```

**支持的注解**：
- `org.jetbrains.annotations.Nullable`
- `org.jetbrains.annotations.NotNull`
- `androidx.annotation.Nullable` (Android)
- `javax.annotation.Nullable` (JSR-305)

### 4.2 Java 看 Kotlin 类型

```kotlin
// Kotlin：String? 暴露为带 @Nullable 的 Java 类型
fun findName(): String? = null
```

```java
// Java 看到的字节码：@Nullable String findName()
// 但 IDE 不强制处理（除非用 Checker Framework / Error Prone）
```

## 5. 类型映射速查

| Kotlin | Java |
|--------|------|
| `String` (非空) | `@NotNull String` |
| `String?` | `@Nullable String` |
| `List<String>` | `List<String>` (元素无注解) |
| `List<String?>` | `List<@Nullable String>` |
| `MutableList<String>` | `List<String>` (Java 仍可调 add) |
| `Array<String>` | `String[]` |
| `IntArray` | `int[]` |
| `Map<String, Int>` | `Map<String, Integer>` |
| `suspend fun foo()` | `Object foo(Continuation cont)` |

## 6. suspend 函数 ↔ Java

```kotlin
// Kotlin：suspend
suspend fun fetchUser(id: String): User { ... }
```

```java
// Java 字节码：自动添加 Continuation 参数
Object fetchUser(String id, Continuation<? super User> cont);
```

**Java 调用**：
```java
// 复杂（不推荐）
// 直接调 Future 更友好
CompletableFuture<User> future = CompletableFuture.supplyAsync(() -> ...);

// 推荐：用 grpc-kotlin / ktor 包装
```

## 7. Lambda ↔ SAM

```kotlin
// Kotlin 调 Java SAM（functional interface）
val runnable = Runnable { println("Hi") }
val listener = View.OnClickListener { view -> ... }

// Kotlin 调 Java 普通接口（需要 object expression）
val listener = object : MyInterface {
    override fun foo() { ... }
}
```

## 8. 实战：netlib-common Java 互操作

```kotlin
// Kotlin：HttpClientFactory
class HttpClientFactory {
    fun create(config: HttpConfig): OkHttpClient { ... }
}

// Java 调用
HttpClientFactory factory = new HttpClientFactory();
HttpConfig config = new HttpConfig(...);  // Java 看到的（带 @JvmOverloads）
OkHttpClient client = factory.create(config);
```

**关键**：每个文件都设计成 Kotlin/Java 都能调——OK。

## 9. 编译产物差异

```kotlin
// Kotlin：class User(val name: String)
// Java 字节码：
public class User {
    private final String name;
    public User(String name) { this.name = name; }
    public String getName() { return name; }   // getter
    public String component1() { return name; }  // 解构
    public User copy(String name) { ... }
}
```

**核心洞察**：**Kotlin property 自动生成 getter + setter**——Java 看到的就是 POJO。

## 10. sealed class ↔ Java

```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
}
```

**Java 看到的**：
- `Result` 是 abstract class
- 子类是 `Result.Success`、`Result.Failure`
- Java 没法用 when 穷尽（只能 if/else）
- ⚠️ **Java 端用 sealed 价值有限**

## 11. 协程 ↔ Java 互操作

```kotlin
// Kotlin：暴露 CompletableFuture 给 Java
suspend fun fetchUser(id: String): User = api.fetch(id)

// Java 友好的包装
fun fetchUserAsync(id: String): CompletableFuture<User> =
    GlobalScope.future { fetchUser(id) }
```

```java
// Java 调用
CompletableFuture<User> future = service.fetchUserAsync("123");
future.thenAccept(user -> System.out.println(user.getName()));
```

## 12. KMP ↔ Java

KMP **不**是 K2 时代才出现——但 v2.0 真正稳定了：

```kotlin
// KMP 共享代码
expect fun platformName(): String

actual fun platformName(): String = "JVM ${System.getProperty("java.version")}"
```

Java 调用 KMP 代码：和调用普通 Kotlin 一样。

## 13. 实战建议

### 13.1 Kotlin 调用 Java

```kotlin
// ✅ 用 @Nullable/@NotNull 注解（让 Kotlin 知道）
// ✅ 显式处理 null（不假设非空）
// ✅ Java 集合转 Kotlin 集合（用 .toList()/.toMap()）
```

### 13.2 Java 调用 Kotlin

```kotlin
// ✅ @JvmStatic（让 static 更简洁）
// ✅ @JvmField（暴露字段）
// ✅ @JvmOverloads（生成 Java 风格重载）
// ✅ @Throws（声明检查异常）
// ✅ 文档注释（KDoc → JavaDoc）
```

## 14. 关键设计原则

1. **100% 兼容**——Java ↔ Kotlin 无缝
2. **平台类型**——Java 类型视为平台类型
3. **@Nullable 注解**——Java 端声明可空
4. **@JvmStatic/Field/Overloads**——优化 Java 体验
5. **suspend ↔ Future**——异步互调

## 15. 反模式

❌ **assumeNotNull（Java 类型）**——运行 NPE
❌ **不写 @JvmStatic**——Java 端调用繁琐
❌ **Java 用 when sealed**——用 if/else 不安全
❌ **不标 @Throws**——Java 编译错误

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **KMP**: [[draft-07-kmp]]
- **最佳实践**: [[draft-09-best-practices]]
- **坑**: [[draft-10-pitfalls]]
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`
- **综合入口**: [[summary]]