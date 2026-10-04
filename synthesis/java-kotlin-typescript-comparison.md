---
title: "Java vs Kotlin vs TypeScript 三方对照"
category: synthesis
tags: [java, kotlin, typescript, comparison, jvm, jvm-vs-js, decision-matrix, cross-language]
sources:
  - "Java 8 → 25 [[java-analysis/summary]]"
  - "Kotlin 2.0+ [[kotlin-analysis/summary]]"
  - "TypeScript 5.x [[typescript-analysis/summary]]"
  - "JEP / KEEP / TC39 各语言官方文档"
summary: "Java vs Kotlin vs TypeScript 全方位对照 - 类型系统/函数/异步/泛型/类/工具链/生态/决策矩阵"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-10-04
created: 2026-10-04
updated: 2026-10-04
---

# Java vs Kotlin vs TypeScript 三方对照

> 三语言统一对照：**Java**（企业级、1995）、**Kotlin**（现代 JVM、2016）、**TypeScript**（现代 JS、2012）。
> 三者分别代表三个时代：Java 是工业级权威、Kotlin 是 JVM 现代继承者、TypeScript 是 JS 生态工业标准。

---

## 1. 一句话定位

| 语言 | 一句话 |
|------|--------|
| **Java** | JVM 工业标准、生态最丰富、1995 至今的企业级首选 |
| **Kotlin** | Java 的现代继承者，null 安全 + 协程 + KMP 跨平台 |
| **TypeScript** | JavaScript 的现代化层，编译到 JS 在任何环境跑 |

**核心洞察**：三者形成「**JVM 内部演进 + JS 生态进化**」的完整对比。

---

## 2. 6 维定位对比

| 维度 | Java | Kotlin | TypeScript |
|------|------|--------|------------|
| **作者** | Sun → Oracle | JetBrains | Microsoft |
| **首个版本** | 1995 | 2016 | 2012 |
| **运行时** | JVM | JVM | V8 / Browser / Deno / Bun |
| **类型系统** | 静态 + 名义 | 静态 + 名义 | 静态 + 结构化 |
| **健全性** | ✅ 健全 | ✅ 健全 | ⚠️ 不健全 |
| **null 安全** | ⚠️ Optional（8+）| ✅ 强制（编译期） | ✅ 严格模式开启 |
| **生态成熟度** | 极高 | 高（Android 70%） | 极高（GitHub 第4）|
| **跨平台** | JVM 跨 OS | JVM + Android + iOS（KMP）| Web + Node.js + 任意 JS 引擎 |
| **2026 趋势** | → 稳定 | ↑↑ 增长 | ↑↑ 增长 |

---

## 3. 类型系统 side-by-side

### 3.1 数据类对比

```java
// Java: 30+ 行模板代码
public final class User {
    private final String name;
    private final int age;
    private final String email;

    public User(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public String getEmail() { return email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return age == user.age
            && Objects.equals(name, user.name)
            && Objects.equals(email, user.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age, email);
    }

    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + ", email='" + email + "'}";
    }
}

// Java 14+: record 简化
public record User(String name, int age, String email) {}
```

```kotlin
// Kotlin: 1 行
data class User(
    val name: String,
    val age: Int,
    val email: String? = null  // 可选参数 + null 安全
)
// 自动生成 equals/hashCode/toString/copy/componentN
```

```typescript
// TypeScript: 5 行
interface User {
    readonly name: string;
    readonly age: number;
    readonly email?: string;  // 可选
}
// 类型擦除：编译后 = 普通 JS 对象
// 运行时无 User 类型
```

**对比**：
- Kotlin `data class` = Java `record`（14+）+ Lombok + Builder
- TypeScript `interface` = 编译期类型，运行时是普通对象
- **Kotlin 介于两者之间**：编译期 + 运行时都有（data class 是真实类）

### 3.2 Null 安全对比

```java
// Java 8-16: 注解驱动（JSR-305）
@Nullable
public String findUserName() {
    return null;  // 编译器不强制
}
// 调用方忘了 null check → NPE
String name = service.findUserName();
System.out.println(name.length());  // 💥 NPE

// Java 17+: 类型注解 + checker framework
// 但需引入额外依赖，生态不完整
```

```kotlin
// Kotlin: 编译期强制 null 安全
fun findUserName(): String? = null

// 编译错：必须先 null check
val name = service.findUserName()
println(name.length)            // ❌ String? 没有 length
println(name?.length)            // ✅ 安全调用，返回 Int?
println(name?.length ?: 0)       // ✅ Elvis 操作符
println(name!!.length)           // ⚠️ 强制非空（NPE 风险）

// sealed class + when：编译器强制穷尽
when (val result = service.findUserName()) {
    null -> println("User not found")
    else -> println("Found: ${result.length}")
}
```

```typescript
// TypeScript: 严格模式开启
function findUserName(): string | null {
    return null;
}

// strictNullChecks: true
const name = findUserName();
console.log(name.length);  // ❌ 编译错：'name' is possibly 'null'
console.log(name?.length);  // ✅ 1 行解决
console.log(name?.length ?? 0);  // ✅ nullish coalescing

// 类型守卫
if (name !== null) {
    console.log(name.length);  // ✅ 类型缩窄
}
```

**对比**：
| | Java | Kotlin | TypeScript |
|---|---|---|---|
| **null 标记** | `@Nullable`（注解） | `String?`（类型系统） | `string \| null`（联合） |
| **强制检查** | ❌ 注解不强制 | ✅ 编译器 | ✅ strict mode |
| **强制非空** | ❌ | `!!`（危险） | `!`（危险） |
| **安全调用** | Optional.ofNullable | `?.` | `?.` |
| **默认值** | ❌ | `?:` (Elvis) | `??` |

### 3.3 类型推断对比

```java
// Java: 局部变量类型推断（10+）
var list = new ArrayList<String>();  // 推断 ArrayList<String>
var name = "Mike";                    // 推断 String

// Java 限制：
// - 只能用于局部变量
// - 不能用于字段、方法参数、返回类型
// - lambda 参数不能用 var
```

```kotlin
// Kotlin: 全场景推断
val name = "Mike"                      // String
val list = listOf("a", "b")             // List<String>
val map = mapOf(1 to "one", 2 to "two") // Map<Int, String>
val fn: (Int) -> Int = { it * 2 }       // 必须标注 lambda 参数类型

class Service {
    private val name: String = "..."  // 字段必须有类型或初始化
    fun add(a: Int, b: Int) = a + b   // 参数必须有类型
}
```

```typescript
// TypeScript: 几乎全场景推断
const name = "Mike";                   // string
const list = ["a", "b"];              // string[]
const map = { 1: "one", 2: "two" };   // { [k: number]: string }
const fn: (n: number) => number = (n) => n * 2;  // 必须显式参数类型

// 函数返回类型：推荐显式标注
function getUser(id: string): User { /* ... */ }
```

**对比**：
- Kotlin：必填字段/参数必须显式类型（其他可推断）
- TypeScript：几乎都可不写
- Java：限制最多（10+）

---

## 4. 函数对比

### 4.1 基本函数

```java
// Java
public int add(int a, int b) {
    return a + b;
}

public String greet(String name) {
    return "Hello, " + name + "!";
}

// 默认参数 + 命名参数（Java 不支持！）
public User createUser(String name, int age) { /* ... */ }
createUser("Mike", 30);  // 必须位置参数
```

```kotlin
// Kotlin
fun add(a: Int, b: Int): Int = a + b
fun greet(name: String) = "Hello, $name!"

// 默认参数 + 命名参数（杀手特性！）
fun createUser(
    name: String = "Anonymous",
    age: Int = 0,
    email: String? = null
) = User(name, age, email)

createUser()                                        // 全默认
createUser("Mike")                                  // 只 name
createUser("Mike", age = 30)                        // 命名参数
createUser(age = 30, name = "Mike")                // 全命名（任意顺序）
```

```typescript
// TypeScript
function add(a: number, b: number): number {
    return a + b;
}
const greet = (name: string) => `Hello, ${name}!`;

// 默认参数（位置）
function createUser(
    name = "Anonymous",
    age = 0,
    email: string | null = null
): User {
    return { name, age, email };
}

createUser();                     // 全默认
createUser("Mike");               // 只 name
createUser("Mike", 30);           // 位置参数
// 命名参数：必须传对象
createUser({ name: "Mike", age: 30 });
```

**对比**：
- Kotlin **杀手特性**：默认参数 + 命名参数 → 构造器重载的简化
- TypeScript：默认参数支持，命名需用对象
- Java：完全不支持默认参数（必须重载）

### 4.2 Lambda

```java
// Java 8+
list.forEach(s -> System.out.println(s));
list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)  // 方法引用
    .collect(Collectors.toList());

// 函数式接口
Runnable r = () -> System.out.println("Hi");
Comparator<String> cmp = (a, b) -> a.compareTo(b);
```

```kotlin
// Kotlin：lambda 是头等公民
list.forEach { println(it) }                    // 隐式 it
list.filter { it.length > 3 }
    .map { it.uppercase() }
    .sortedBy { it }                             // 简洁

val sum: (Int, Int) -> Int = { a, b -> a + b }
val square = { x: Int -> x * x }                 // 类型推断

// 函数引用
val names = users.map(User::name)                // 方法引用
```

```typescript
// TypeScript: arrow function
list.forEach(s => console.log(s));
list
    .filter(s => s.length > 3)
    .map(s => s.toUpperCase());

const sum = (a: number, b: number) => a + b;
const square = (x: number) => x * x;

// 函数引用
const names = users.map(u => u.name);            // lambda
const sortByName = users.sort((a, b) => /* */);  // 比较器
```

**对比**：
| | Java | Kotlin | TypeScript |
|---|---|---|---|
| **Lambda** | 8+ | ✅ 一等 | ✅ arrow |
| **隐式 it** | ❌ | ✅ `{ it }` | ❌ 必须命名 |
| **方法引用** | `String::toUpperCase` | `String::uppercase` | ❌ 必须 lambda |
| **Trailing lambda** | ✅ | ✅ | ✅ |
| **inline** | ❌ | ✅（性能关键）| ❌ |

---

## 5. 异步编程对比

### 5.1 Java: CompletableFuture + Virtual Threads

```java
// Java 8-20: CompletableFuture
CompletableFuture<User> future = client.getUser(id)
    .thenApply(this::enrich)
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return null;
    });

// Java 21+: Virtual Threads（杀手特性！）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        User user = client.getUser(id);  // 阻塞调用
        log.info("Got: {}", user);
    });
}
// 异步代码，HTTP 提升 100x
```

### 5.2 Kotlin: 协程（structured concurrency）

```kotlin
// 协程杀手特性：结构化并发
suspend fun loadUser(id: String): User = withContext(Dispatchers.IO) {
    client.fetchUser(id)
}

// 自动取消、自动异常传播、父子作用域
suspend fun loadDashboard() = coroutineScope {
    val userDeferred = async { loadUser() }
    val postsDeferred = async { loadPosts() }
    Dashboard(
        user = userDeferred.await(),
        posts = postsDeferred.await()
    )
}
// 任何子协程失败 → 兄弟也取消

// Flow 异步数据流
fun stockStream(symbol: String): Flow<StockTick> = channelFlow {
    val ws = client.newWebSocket(/* ... */)
    awaitClose { ws.cancel() }
}
```

### 5.3 TypeScript: Promise + async/await

```typescript
// async/await（语法糖糖）

// JavaScript/TypeScript
async function loadUser(id: string): Promise<User> {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
}

async function loadDashboard(): Promise<Dashboard> {
    const [user, posts] = await Promise.all([
        loadUser("1"),
        loadPosts("1")
    ]);
    return { user, posts };
}

// 错误处理
try {
    const user = await loadUser("1");
} catch (e) {
    console.error(e);
}
```

### 5.4 异步模型对比

| | Java | Kotlin | TypeScript |
|---|---|---|---|
| **基础** | Thread + Future | 协程（用户态） | Promise（microtask）|
| **结构化并发** | ⚠️（CompletableFuture） | ✅（结构化）| ⚠️（需手写）|
| **轻量级** | 1MB / Thread | 100B / Coroutine | 几乎零开销 / Promise |
| **取消传播** | 需手写 | ✅ 自动 | 需手写 |
| **流式数据** | ❌ | ✅ Flow | ⚠️ AsyncIterator |
| **async 语法** | ❌ | ✅ `suspend` | ✅ `async`/`await` |
| **生态成熟** | ✅ 极高 | ✅ 高 | ✅ 极高 |

**核心洞察**：
- **Java 21 Virtual Threads**：thread-per-task 模型，代码简单
- **Kotlin Coroutines**：用户态，比 Virtual Threads 更轻量
- **TypeScript Promises**：JS 原生，生态最广

---

## 6. 泛型对比

### 6.1 基本泛型

```java
// Java: 类型擦除 + bounded wildcards
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

public <K, V> V getOrDefault(Map<K, V> map, K key, V defaultValue) {
    V value = map.get(key);
    return value != null ? value : defaultValue;
}

// 使用 wildcards（use-site variance）
public void processNumbers(List<? extends Number> list) {
    // PECS: Producer Extends, Consumer Super
}
```

```kotlin
// Kotlin: declaration-site variance + reified
// 声明处 out / in
fun <T : Comparable<T>> max(a: T, b: T): T {
    return if (a > b) a else b  // operator
}

fun <K, V> getOrDefault(map: Map<K, V>, key: K, default: V): V {
    return map[key] ?: default
}

// 协变 + 逆变
class Producer<out T> {
    fun produce(): T = /* ... */
}
class Consumer<in T> {
    fun consume(t: T) { /* ... */ }
}

// reified（运行时类型信息）
inline fun <reified T : Any> Any?.cast(): T? = this as? T
```

```typescript
// TypeScript: 结构化 + 条件类型
function max<T extends Comparable<T>>(a: T, b: T): T {
    return a > b ? a : b;
}

// 内置工具类型
type Partial<T> = { [K in keyof T]?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };

// 条件类型 + infer
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### 6.2 泛型 vs 反射

| 维度 | Java | Kotlin | TypeScript |
|---|---|---|---|
| **类型擦除** | ✅ 编译期泛型，运行时擦除 | 同 Java | ✅ 完全擦除 |
| **运行时类型信息** | `Class<T>` + reflection | `reified` + `KClass` | ❌ 无（运行时 = JS） |
| **协变** | wildcards `? extends` | `out T`（声明处）| `T extends U`（约束） |
| **逆变** | wildcards `? super` | `in T`（声明处）| ❌（结构化弱化） |
| **约束** | `T extends X` | `T : X` | `T extends X` |
| **条件类型** | ❌ | ❌ | ✅ `T extends U ? X : Y` |

---

## 7. 类与 OOP 对比

### 7.1 基本类

```java
// Java
public final class User {
    private final String name;
    private final int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
}

// Java 17+: sealed（封闭继承）
public sealed class Shape
    permits Circle, Square {
    public abstract double area();
}
public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    @Override public double area() { return Math.PI * radius * radius; }
}
```

```kotlin
// Kotlin
class User(val name: String, val age: Int)  // 1 行 = Java 30+ 行

// sealed class（设计首选）
sealed class Shape {
    abstract fun area(): Double
}
data class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}
```

```typescript
// TypeScript: interface + class
interface User {
    readonly name: string;
    readonly age: number;
}

class UserImpl implements User {
    constructor(public readonly name: string, public readonly age: number) {}
}

// 运行时类型 = 编译期类型
// （运行时是普通 JS 对象）
```

### 7.2 抽象类 vs 接口

```java
// Java 17-20: 接口默认方法
interface Drawable {
    void draw();
    default void clear() { /* 默认实现 */ }
}
abstract class Shape implements Drawable {
    abstract double area();
}
```

```kotlin
// Kotlin: interface + abstract class
interface Drawable {
    fun draw()
    fun clear() = println("默认")  // 默认实现
}
abstract class Shape : Drawable {
    abstract fun area(): Double
}
```

```typescript
// TypeScript: interface only
interface Drawable {
    draw(): void;
    clear(): void;  // 必须实现
}
// 没有抽象类的概念
```

---

## 8. 工具链对比

| 维度 | Java | Kotlin | TypeScript |
|------|------|--------|------------|
| **编译器** | `javac` | `kotlinc` | `tsc` |
| **构建工具** | Maven / Gradle | Maven / Gradle | npm / yarn / pnpm |
| **打包** | fat JAR / shade | fat JAR / shade | esbuild / Vite / Rollup |
| **运行时** | JVM | JVM | V8 / Browser / Deno / Bun |
| **启动时间** | 100ms - 1s | 200ms - 2s | 50ms - 500ms |
| **包大小** | 中（10-100MB JAR） | 同 Java | 小（100-500KB JS） |
| **增量编译** | ✅ | ✅ | ✅（tsc `--incremental`）|
| **IDE** | IntelliJ | IntelliJ | VS Code |
| **Linter** | Checkstyle / PMD | detekt / ktlint | ESLint |
| **Formatter** | google-java-format | - | Prettier |

### 编译速度对比（1000 文件）

```
javac:        ~5s
kotlinc:      ~10s
tsc:          ~10s
tsc (--incremental): ~2s
esbuild:      ~0.5s   (~20x tsc)
swc:          ~0.3s   (~33x tsc)
tsx (dev):    ~0.05s  (~200x tsc)
```

**核心洞察**：TS dev 用 esbuild/swc（20-200x），CI 必跑 tsc（完整类型检查）。

---

## 9. 生态对比

| 维度 | Java | Kotlin | TypeScript |
|------|------|--------|------------|
| **GitHub Octoverse 2024** | #6 | 增长中 | **#4** |
| **包管理** | Maven Central | Maven Central + npm（如 KMP）| npm |
| **生态规模** | 极大（数百万包）| 大（复用 Java）| 极大（数百万包）|
| **生产采用** | 企业 / 银行 / Android | Android / Spring / KMP | React / Vue / Node.js |
| **2026 趋势** | → 稳定 | ↑↑ 增长 | ↑↑ 增长 |

### 9.1 框架对照

| 角色 | Java | Kotlin | TypeScript |
|------|------|--------|------------|
| **Web 后端** | Spring Boot | Spring Boot / Ktor | Express / Fastify / NestJS |
| **前端** | - | - | React / Vue / Angular |
| **Android** | Java legacy | Kotlin 首选 | ❌ |
| **跨端** | - | KMP（iOS/Android/JVM）| - |
| **构建工具** | Maven/Gradle | Maven/Gradle | Vite/esbuild/webpack |
| **测试** | JUnit | JUnit + Kotest | Vitest/Jest |
| **依赖注入** | Spring DI / Guice | Koin / Hilt | NestJS DI / tsyringe |

### 9.2 异步库对比

| | Java | Kotlin | TypeScript |
|---|---|---|---|
| **基础** | CompletableFuture | 协程（stdlib）| Promise（内置）|
| **Web** | WebClient / RestTemplate | Ktor / OkHttp + 协程 | fetch / axios |
| **RPC** | gRPC-Java | gRPC-Kotlin | gRPC-Web / tRPC |
| **流式** | RxJava | Flow / Channel | AsyncIterator |

---

## 11. 决策矩阵

### 11.1 新项目

| 场景 | 推荐 |
|------|------|
| **大型企业级 Java 项目** | Java（保守）/ Kotlin（现代化）|
| **新 Spring Boot 后端** | Kotlin + Spring（推荐）/ Java |
| **Android 应用** | Kotlin（官方首选）|
| **Node.js 后端** | TypeScript（首选）|
| **React/Vue/Angular 前端** | TypeScript（必备）|
| **跨端 iOS + Android + JVM** | Kotlin KMP |
| **跨端 Web + Mobile** | TypeScript + React Native / Flutter |
| **小型 CLI 工具** | TypeScript（tsx）或 Go / Python |
| **AI/ML 脚本** | Python / TypeScript |

### 11.2 维护现有

| 现有 | 推荐 |
|------|------|
| **Java 项目** | 增量 Kotlin 迁移（Spring + Kotlin）|
| **JavaScript 项目** | 增量 TypeScript（allowJs）|
| **TypeScript 项目** | 保持 |
| **Kotlin 项目** | 保持 |

### 11.3 学习路径

| 当前 | 建议 |
|------|------|
| **Java 开发者** | Kotlin（1 周上手，语法相似）|
| **JS 开发者** | TypeScript（1 周上手）|
| **新开发者** | TypeScript（最平）或 Java（生态最广）|

---

## 12. 实战组合（推荐栈）

### 12.1 全栈 Web（企业级）

```
前端：TypeScript + React + Vite
后端：Kotlin + Spring Boot 3.x + JPA
通信：REST / gRPC
数据库：PostgreSQL
部署：Docker + Kubernetes
```

### 12.2 跨端应用

```
UI：Kotlin Multiplatform + Compose Multiplatform（iOS/Android/Desktop）
业务逻辑：Kotlin shared module
后端：Kotlin + Spring Boot / Ktor
```

### 12.3 微服务 / Serverless

```
Node.js 后端：TypeScript + tRPC + Lambda
数据库：DynamoDB / Postgres
通信：gRPC-Web（与 Kotlin 服务通信）
```

### 12.4 Android + 后端

```
Android：Kotlin + Compose + Coroutines
后端：Kotlin + Spring Boot
通信：gRPC + Protobuf
```

---

## 13. 互操作

### 13.1 Kotlin 调 Java（无缝）

```kotlin
// Kotlin 直接调 Java 库
val list = ArrayList<String>()  // Java 类
list.add("Mike")
val stream = list.stream()      // Java Stream API
stream.filter { it.length > 3 }
```

### 13.2 TypeScript 调 Java/Kotlin（gRPC）

```typescript
// 通过 gRPC-Web 调 Kotlin 服务
const client = new GreeterClient('https://api.example.com');
const reply = await client.sayHello({ name: 'mike' });
```

### 13.3 Java/Kotlin 调 JS（不常见）

```kotlin
// 通过 GraalVM polyglot（实验性）
val context = Context.newBuilder().allowAllAccess(true).build()
val jsFunction = context.eval("js", "function(x) { return x * 2 }")
val result = jsFunction.execute(5)  // 10
```

---

## 14. 三语言协作实战（grpc-netty + OkHttp + grpc-web）

```
┌─────────────────────────┐      ┌─────────────────────────┐
│   前端（浏览器）           │      │   后端（JVM）              │
│   TypeScript + React      │      │   Kotlin / Java + Spring  │
│   grpc-web                  │◄────►│   OkHttp（出站）          │
│   (gRPC-Web client)        │ HTTP │   gRPC-Java（入站 RPC）   │
│   tRPC（可选）             │      │   mu-server（HTTP）       │
└─────────────────────────┘      └─────────────────────────┘
       Node.js / Browser                JVM
```

详见 [[okhttp-analysis/summary]] + [[grpc-analysis/summary]] + [[mu-server-2.4.2-analysis/summary]]。

---

## 15. 关键洞察

1. **Java = JVM 工业基础**，生态最丰富，2026 年仍企业首选
2. **Kotlin = Java 的现代继承者**，Android 首选、JVM 现代、null 安全
3. **TypeScript = JS 的工业标准**，前端/Node.js 必备
4. **三者形成完整生态**：JVM（JVM 内部演进 + JS 生态）
5. **没有「最优」**——只有最适合场景
6. **互操作友好**：Kotlin ↔ Java 无缝、TS ↔ JS 无缝、跨生态通过 gRPC

---

## 16. 一句话推荐

| 场景 | 一句话 |
|------|--------|
| **企业级 Java 项目** | Java（保守）或 Kotlin（现代化） |
| **新 Spring Boot 后端** | **Kotlin** + Spring |
| **Android 应用** | **Kotlin** |
| **Node.js 后端** | **TypeScript** + NestJS |
| **React 前端** | **TypeScript** |
| **跨端 KMP** | **Kotlin** Multiplatform |
| **CLI 工具** | **TypeScript** (tsx) |

---

## 相关笔记

- **Java 分析**: [[java-analysis/summary]]
- **Kotlin 分析**: [[kotlin-analysis/summary]]
- **TypeScript 分析**: [[typescript-analysis/summary]]
- **OkHttp**: [[okhttp-analysis/summary]]（Kotlin 实现）
- **gRPC**: [[grpc-analysis/summary]]（Java/Kotlin/TS 三实现）
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`（Kotlin 实战）