---
title: "Kotlin vs Java vs Scala vs Groovy vs C# 对比选型"
category: synthesis
tags: [kotlin, java, scala, groovy, csharp, comparison, decision-matrix]
sources:
  - "Kotlin / Java / Scala / Groovy / C# 官方文档"
  - "GitHub 仓库数据 + 社区调查 (Stack Overflow / JetBrains Survey)"
  - "作者实战经验"
summary: "5 大 JVM 语言全方位对比：Kotlin / Java / Scala / Groovy / C# - 类型系统、生态、学习曲线、适用场景"
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

# §11 Kotlin vs Java vs Scala vs Groovy vs C# 对比

## 1. 5 大语言定位

| 语言 | 作者 | 类型系统 | 范式 | 定位 |
|------|------|----------|------|------|
| **Kotlin** | JetBrains | 静态 + 强 + null 安全 | 面向对象 + 函数式 | Java 的现代继承者 |
| **Java** | Sun/Oracle | 静态 + 强 | 面向对象 | 行业标准（生态最丰富） |
| **Scala** | EPFL（Oderski） | 静态 + 强 + 推断 | 函数式 + 面向对象 | 学术派，复杂但强大 |
| **Groovy** | Apache | 动态 + 可选静态 | 脚本 + 面向对象 | JVM 上的 Python/JavaScript |
| **C#** | Microsoft | 静态 + 强 + nullable | 面向对象 + 函数式 | .NET 平台首选 |

## 2. 6 维对比矩阵

### 2.1 类型系统

| 维度 | Kotlin | Java | Scala | Groovy | C# |
|------|--------|------|-------|--------|-----|
| **类型系统** | 静态 + 推断 | 静态 | 静态 + 强推断 | 动态 | 静态 + 推断 |
| **null 安全** | ✅ 编译期 | ❌ | Option | ✅（可选） | ✅ nullable ref |
| **泛型** | ✅ | ✅（型变弱） | ✅（高阶） | ✅（擦除） | ✅（型变强） |
| **代数数据类型** | sealed class | record (14+) | case class | @Immutable | record |

### 2.2 表达力（同样逻辑的代码量）

```java
// Java：数据类
public class User {
    private String name;
    private int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}
```

```kotlin
// Kotlin：一行
data class User(val name: String, val age: Int)
```

```scala
// Scala：一行（但语法繁）
case class User(name: String, age: Int)
```

```csharp
// C# 9+：record
record User(string Name, int Age);
```

### 2.3 函数式

| 维度 | Kotlin | Java | Scala | Groovy | C# |
|------|--------|------|-------|--------|-----|
| **Lambda** | ✅ 一等 | ✅ (8+) | ✅ 一等 | ✅ 闭包 | ✅ 一等 |
| **高阶函数** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **模式匹配** | when | switch (14+) | match | switch | switch |
| **Stream/Lazy** | Sequence | Stream | LazyList | Lazy | LINQ |
| **不可变集合** | listOf | List.of / Stream | List | ImmutableList | ImmutableArray |

### 2.4 协程/异步

| 维度 | Kotlin | Java | Scala | Groovy | C# |
|------|--------|------|-------|--------|-----|
| **协程** | ✅ 一等 | ⚠️ VirtualThread (21+) | Cats Effect | GPars | async/await |
| **结构化并发** | ✅ | ✅ (preview) | ✅ | ❌ | ❌ |
| **Channel** | ✅ | ❌ | Cats | ✅ | Channel<T> |

### 2.5 生态与生产采用

| 维度 | Kotlin | Java | Scala | Groovy | C# |
|------|--------|------|-------|--------|-----|
| **GitHub stars** | 50k+ | 100k+ | 14k+ | 5k+ | 35k+ |
| **生产采用** | Android / Server | 普遍 | 数据 / Spark | CI/CD | .NET |
| **学习曲线** | 平 | - | 陡 | 平 | 平 |
| **生态成熟度** | 高 | 极高 | 中 | 中 | 高（.NET） |
| **2026 趋势** | ↑↑ | → | → | ↓ | → |

## 3. Kotlin vs Java 详解

```java
// Java
public class UserService {
    private static final int MAX_USERS = 1000;
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public User create(String name, int age) {
        if (name == null) throw new IllegalArgumentException("name");
        return repository.save(new User(name, age));
    }
}
```

```kotlin
// Kotlin
class UserService(private val repository: UserRepository) {
    companion object {
        const val MAX_USERS = 1000
    }
    
    fun create(name: String, age: Int): User {
        require(name.isNotEmpty()) { "name" }
        return repository.save(User(name, age))
    }
}
```

**差异**：
- Kotlin：30 行 → 10 行（-66%）
- Kotlin：null 安全 + 默认参数 + data class
- Java：样板代码多，但生态兼容

## 4. Kotlin vs Scala 详解

```scala
// Scala：case class + pattern matching
sealed trait Result[+T]
case class Success[T](value: T) extends Result[T]
case class Failure(error: Throwable) extends Result[Nothing]

def handle[T](r: Result[T]): String = r match {
    case Success(v) => s"Got: $v"
    case Failure(e) => s"Error: $e"
}
```

```kotlin
// Kotlin：sealed class + when
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
}

fun <T> handle(r: Result<T>) = when (r) {
    is Result.Success -> "Got: ${r.value}"
    is Result.Failure -> "Error: ${r.error}"
    // 编译器穷尽
}
```

**差异**：
- Scala：更纯函数式（Cats、ZIO）
- Kotlin：务实（不强迫 FP 范式）
- 学习曲线：Scala 陡，Kotlin 平

## 5. Kotlin vs Groovy 详解

```groovy
// Groovy：动态 + GString
def name = "Mike"
def user = [name: name, age: 30]  // Map

user.each { k, v ->
    println "$k = $v"
}

// 编译时类型（可选）
String s = "hello" as String
```

```kotlin
// Kotlin：静态 + 类型推断
val name = "Mike"
val user = User("Mike", 30)

user.let { u ->
    println("name = ${u.name}")
}

// 类型安全
val s: String = "hello"
```

**差异**：
- Groovy：动态为主（适合脚本、DSL）
- Kotlin：静态为主（适合大型项目）
- 互操作：Groovy 调 Kotlin 复杂，Kotlin 调 Groovy 简单

## 6. Kotlin vs C# 详解

```csharp
// C# 11 + .NET 8
public class UserService {
    public async Task<User> CreateAsync(string name, int age) {
        await Task.Delay(1000);
        return new User(name, age);
    }
    
    public string? FindName(int id) => _repo.FindById(id);
}
```

```kotlin
// Kotlin 2.0
class UserService {
    suspend fun create(name: String, age: Int): User {
        delay(1000)
        return User(name, age)
    }
    
    fun findName(id: Int): String? = _repo.findById(id)
}
```

**差异**：
- C# 11 nullable reference + record + async/await（与 Kotlin 高度相似）
- C# .NET 8（性能提升） vs Kotlin/JVM
- C# Microsoft 生态 vs Kotlin JetBrains 生态
- Kotlin KMP 跨平台（C# MAUI 也跨平台）

## 7. 决策矩阵

### 7.1 新项目

| 场景 | 推荐 |
|------|------|
| **JVM/Android 全栈** | Kotlin（KMP 跨平台） |
| **企业级 JVM** | Java 或 Kotlin（保守 vs 现代） |
| **数据/Spark** | Scala（生态优势） |
| **脚本/CI/CD** | Groovy 或 Python |
| **.NET 生态** | C#（必须） |
| **跨 iOS/Android 共享** | Kotlin（KMP） |

### 7.2 维护现有

| 现有语言 | 推荐 |
|---------|------|
| **Java** | 增量迁移 Kotlin（先 data class / 文件互调） |
| **Scala** | 保留（不要迁移） |
| **Groovy** | 脚本用 Groovy，业务用 Kotlin |
| **C#** | 保留 |

### 7.3 学习路径

| 当前水平 | 建议 |
|---------|------|
| **Java 开发者** | Kotlin（1 周上手） |
| **JS/Python 开发者** | Kotlin 或 C# |
| **Scala 开发者** | Kotlin（语法相似） |
| **C# 开发者** | Kotlin（高度相似） |
| **学术派 FP** | Scala |

## 8. 性能对比（基准）

| 操作 | Kotlin | Java | Scala |
|------|--------|------|-------|
| **启动** | 中（JIT 优化） | 快 | 慢（JVM 启动慢） |
| **运行** | 中-快（接近 Java） | 快 | 中（编译慢） |
| **编译** | 中（K2 快） | 快 | 慢 |
| **内存** | 中（接近 Java） | 中 | 中-高 |
| **Lambda 调用** | 快（inline） | 中 | 中 |

⚠️ **基准因场景差异大**——具体数据仅供参考。

## 9. 一句话对比

| 语言 | 一句话 |
|------|--------|
| **Kotlin** | Java 的现代化继承者，Android 首选，KMP 跨平台 |
| **Java** | 行业标准，生态最丰富，但语法滞后 |
| **Scala** | 函数式学派，Cats/ZIO 强，复杂但强大 |
| **Groovy** | JVM 上的 Python，动态+DSL |
| **C#** | .NET 生态首选，nullable + async/await 与 Kotlin 高度相似 |

## 10. 实战建议

### 10.1 JVM 微服务（2026）

```
推荐组合：
  Kotlin + Spring Boot 3.x + Coroutines
  或
  Java 17/21 + Spring Boot 3.x + Virtual Threads
```

**Kotlin 优势**：
- 30% 更少代码
- null 安全
- coroutine（结构化并发）

**Java 优势**：
- 生态最广
- Virtual Threads (21+) 性能强
- 招聘容易

### 10.2 Android

```
唯一推荐：Kotlin
```

### 10.3 数据/Spark

```
推荐：Scala（PySpark / Scala Spark）
```

### 10.4 KMP（iOS + Android + JVM）

```
唯一推荐：Kotlin KMP + Compose Multiplatform
```

## 11. 关键洞察

1. **Kotlin 是 2026 的 JVM 主流**——不是可选项
2. **Scala 是 FP 学术派**——适合 Spark/Cats
3. **Groovy 是 CI/CD 工具语言**——不是业务首选
4. **Java 仍是行业标准**——大企业维护成本低
5. **C# 是 .NET 唯一选择**——跨平台（MAUI）

## 12. 与 OkHttp/gRPC 生态

| 框架 | 推荐语言 | 理由 |
|------|---------|------|
| **OkHttp** | Java / Kotlin | 4.x+ 用 Kotlin 重写，5.x 协程支持 |
| **gRPC** | Java / Kotlin | grpc-kotlin 提供协程 |
| **mu-server** | Java | Java only |
| **Netty** | Java / Kotlin | Kotlin ChannelHandler 扩展 |
| **Spring** | Java / Kotlin | Kotlin DSL 一等公民 |

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **协程**: [[draft-03-coroutines]]
- **应用场景**: [[draft-12-use-cases]]
- **OkHttp 实现**: [[okhttp-analysis/summary]]
- **gRPC 实现**: [[grpc-analysis/summary]]
- **综合入口**: [[summary]]