---
title: "Java 22-25 现代并发 + 教学体验"
category: synthesis
tags: [java, java-22, java-23, java-24, java-25, lts, structured-concurrency, scoped-values, ffm-api, primitive-patterns, module-imports, compact-source-files, stable-values]
sources:
  - "JEP 447 - Statements before super"
  - "JEP 454 - Foreign Function & Memory API"
  - "JEP 456 - Unnamed Variables & Patterns"
  - "JEP 459 - String Templates (Second Preview)"
  - "JEP 461 - Stream Gatherers"
  - "JEP 462 - Structured Concurrency (Second Preview)"
  - "JEP 463 - Implicitly Declared Classes and Instance Main Methods"
  - "JEP 476 - Module Import Declarations"
  - "JEP 477 - Primitive Patterns in switch"
  - "JEP 480 - Structured Concurrency (Third Preview)"
  - "JEP 481 - Scoped Values (Preview)"
  - "JEP 485 - Stream Gatherers"
  - "JEP 502 - Stable Values (Preview)"
  - "JEP 505 - Structured Concurrency (Fifth Preview)"
  - "JEP 506 - Scoped Values"
  - "JEP 507 - Primitive Patterns (Third Preview)"
  - "JEP 511 - Module Import Declarations"
  - "JEP 512 - Compact Source Files and Instance Main Methods"
  - "JEP 513 - Flexible Constructor Bodies"
summary: "Java 22-25 - 现代并发(structured/scoped) + FFM API + 模块导入 + 紧凑源文件 + 灵活构造器 + 原始类型 pattern"
provenance:
  extracted: 0.85
  inferred: 0.12
  ambiguous: 0.03
base_confidence: 0.93
lifecycle: stable
lifecycle_changed: 2026-10-10
created: 2026-10-03
updated: 2026-10-10
---

# §05 Java 22-25 现代并发 + 教学体验

> **Java 25 (2025-09-16, LTS)** 是当前最新的 LTS，Java 22-25 这 4 个版本共同完成了 Java 的"现代并发 + 学习曲线降低"双重目标：**Scoped Values**（替代 ThreadLocal）、**Module Import Declarations**（减少 import 噪音）、**Compact Source Files**（学生学习 Hello World 从 8 行变 4 行）、**Flexible Constructor Bodies**（构造器前置校验）。

## 1. Unnamed Variables & Patterns（JEP 456, Java 22 final）

```java
// Java 21 之前
try (var in = new FileInputStream("a");
     var out = new FileOutputStream("b")) {
    // in 不使用也得命名
}

// Java 22+
try (var _ = new FileInputStream("a");
     var out = new FileOutputStream("b")) {
    // _ 表示"不关心"
}

// catch 异常不需要
try { /* ... */ }
catch (NumberFormatException _) { /* 静默忽略 */ }

// Lambda 忽略参数
list.forEach(_ -> System.out.println("processing"));

// record pattern 忽略字段
if (obj instanceof Point(int x, _)) {  // 只关心 x
    System.out.println(x);
}
```

**核心点**：`_` 是合法变量名，编译器识别为"未使用"，**不报编译警告**。

## 2. Foreign Function & Memory API（JEP 454, Java 22 final）

Java 终于可以**安全调用 C 库**和**直接操作堆外内存**，无需 JNI。

```java
// 调用 C 标准库 strlen
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;

try (Arena arena = Arena.ofConfined()) {
    MemorySegment str = arena.allocateUtf8String("Hello, World!");

    MethodHandle strlen = Linker.nativeLinker().downcallHandle(
        FunctionDescriptor.of(JAVA_LONG, JAVA_LONG)
    );

    long len = (long) strlen.invoke(str.address());
    System.out.println("Length: " + len);  // 13
}
```

**对比 JNI**：

| 维度 | JNI（Java 1.1+） | FFM API（Java 22+） |
|---|---|---|
| 写 C 代码 | 必须 | **不用** |
| 编译 | javah + C 编译 | **没有** |
| 安全 | 容易 JVM crash | 受 Arena 管理 |
| 调用开销 | 高 | 接近 C |
| 适用 | 老的 native 代码 | 新项目首选 |

### 2.2 FFM API 实战拆解

#### ① Arena 生命周期（3 种）

```java
// Confined - 单线程，最快，自动 close
try (Arena arena = Arena.ofConfined()) {
    MemorySegment seg = arena.allocate(100);
    // 仅当前线程访问
}

// Shared - 多线程共享访问
try (Arena arena = Arena.ofShared()) {
    MemorySegment seg = arena.allocate(100);
    // 多线程可同时访问（需要同步）
}

// Auto - GC 回收（不推荐生产用）
Arena auto = Arena.ofAuto();
MemorySegment seg = auto.allocate(100);
// GC 时自动释放（时机不可控）
```

**关键**：**try-with-resources 是推荐写法**，Arena.close() 会立即释放底层 native 内存（不依赖 GC）。

#### ② 内存分配与读写

```java
try (Arena arena = Arena.ofConfined()) {
    // 分配 100 字节
    MemorySegment seg = arena.allocate(100);
    
    // 写（offset 0 处写 int 42）
    seg.set(ValueLayout.JAVA_INT, 0, 42);
    seg.set(ValueLayout.JAVA_INT, 4, 100);  // offset 4 处
    
    // 读
    int v1 = seg.get(ValueLayout.JAVA_INT, 0);   // 42
    int v2 = seg.get(ValueLayout.JAVA_INT, 4);   // 100
    
    // 字符串（自动 UTF-8 编码）
    MemorySegment str = arena.allocateUtf8String("Hello, FFM!");
    String s = str.getUtf8String(0);  // "Hello, FFM!"
    
    // 数组
    MemorySegment intArr = arena.allocateFrom(ValueLayout.JAVA_INT, 1, 2, 3, 4, 5);
}
```

#### ③ FunctionDescriptor + 调用 C 函数

```java
Linker linker = Linker.nativeLinker();

// 方法 1：lookup by name
MethodHandle strlen = linker.downcallHandle(
    linker.defaultLookup().find("strlen").orElseThrow(),
    FunctionDescriptor.of(JAVA_LONG, JAVA_LONG)  // (返回 long, 参数 long)
);

try (Arena arena = Arena.ofConfined()) {
    MemorySegment str = arena.allocateUtf8String("Hello");
    long len = (long) strlen.invoke(str.address());
    System.out.println(len);  // 5
}

// 方法 2：直接指定 C 库
MethodHandle getpid = linker.downcallHandle(
    Linker.nativeLinker().defaultLookup().find("getpid").orElseThrow(),
    FunctionDescriptor.of(JAVA_INT)
);
int pid = (int) getpid.invoke();
```

#### ④ UpcallStub（C 调用 Java）

```java
// 定义 Java 回调
FunctionDescriptor callbackDesc = FunctionDescriptor.ofVoid(JAVA_INT);

try (Arena arena = Arena.ofConfined()) {
    MethodHandle callback = linker.upcallStub(
        (Consumer<Integer>) value -> System.out.println("C called back: " + value),
        callbackDesc,
        arena
    );
    
    // 把 callback.address() 传给 C 函数，让 C 内部回调
    cFunction.invoke(callback.address());
}
```

#### ⑤ vs JNI 完整对比

| 维度 | JNI（Java 1.1+） | FFM API（Java 22+）|
|---|---|---|
| 写 C 代码 | 必须 | **不用** |
| 编译步骤 | javah + C 编译 + native loader | **没有** |
| 内存管理 | 手动（容易泄漏）| Arena 自动 |
| 性能 | 接近 C | 接近 C |
| 安全 | 容易 JVM crash | 受 Arena 管理 |
| 学习曲线 | 陡（C 知识必需）| 平（Java API）|
| 调用开销 | 较高 | 接近 C |
| 回调（C → Java）| 复杂（C 实现 Java 方法）| UpcallStub（一行）|
| 适用 | 老的 native 代码 | **新项目首选** |

#### ⑥ 6 条反模式

❌ **不用 try-with-resources 包 Arena** —— 内存永不释放
❌ **跨 Arena 传递 address** —— use-after-free
❌ **共享 Arena 不 close** —— 整个进程累积 native 内存
❌ **用 `Arena.ofAuto()` 当 fallback** —— GC 时机不可控
❌ **不检查 C 函数返回值** —— NPE / 段错误
❌ **在 lambda 内捕获 native 引用** —— native 生命周期比 lambda 短

---

**核心洞察**：FFM API 是 Java 22 给"调 C 库 + 操作 native 内存"的**现代化方案**。它把 JNI 的"写 C 代码 + 手动管理内存 + 学习曲线陡"三大痛点全部解决。**适用场景**：调系统库（OpenSSL / zstd）、高性能计算（SIMD）、跨语言互操作。**不适合**：纯 Java 应用、与 native 无关的逻辑。

## 3. Statements before super（JEP 447, Java 22 preview）

```java
// Java 21 之前 - 构造器首行必须是 super(...) 或 this(...)
public class PositiveBigInteger extends BigInteger {
    public PositiveBigInteger(long value) {
        if (value <= 0) throw new IllegalArgumentException("non-positive: " + value);
        super(value);  // 只能在这之后
    }
}

// Java 22+ preview - super 前可以先做点事
public class PositiveBigInteger extends BigInteger {
    public PositiveBigInteger(long value) {
        super(validate(value));   // 在 super 调用前运算
    }

    private static long validate(long value) {
        if (value <= 0) throw new IllegalArgumentException("non-positive: " + value);
        return value;
    }
}

// ✅ 更直接的写法（Java 22 preview 2）
public PositiveBigInteger(long value) {
    if (value <= 0) throw new IllegalArgumentException("non-positive: " + value);
    super(value);  // 允许在 super 前执行语句
}
```

> Java 25 final（JEP 513：Flexible Constructor Bodies）

## 4. Stream Gatherers（JEP 461 / 485, Java 23 preview, Java 24 final）

Stream API 的扩展，让你可以**自定义中间操作**。

```java
// 内置 gatherer
List<List<Integer>> batched = Stream.of(1, 2, 3, 4, 5, 6, 7)
    .gather(Gatherers.windowFixed(3))
    .toList();
// [[1, 2, 3], [4, 5, 6], [7]]

List<List<Integer>> sliding = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowSliding(3))
    .toList();
// [[1, 2, 3], [2, 3, 4], [3, 4, 5]]

// fold-like 聚合
String result = Stream.of("a", "b", "c")
    .gather(Gatherers.scan(() -> "", (acc, s) -> acc + s))
    .toList().get(2);
// "abc"

// 自定义 Gatherer
Gatherer<String, ?, String> distinctByFirstChar = Gatherer.of(
    () -> new HashMap<Character, String>(),
    (map, element, downstream) -> {
        char first = element.charAt(0);
        if (!map.containsKey(first)) {
            map.put(first, element);
            return downstream.push(element);
        }
        return true;
    },
    (m1, m2) -> { m1.putAll(m2); return m1; },
    map -> {}
);

List<String> result = Stream.of("apple", "ant", "banana", "berry")
    .gather(distinctByFirstChar)
    .toList();
// ["apple", "banana"]
```

## 5. Primitive Patterns（JEP 507, Java 25 preview 3）

```java
// Java 24 之前 - 模式只能匹配引用类型
switch (obj) {
    case Integer i -> ...;
    case Long l -> ...;
    // case int i -> ...; ❌ 不行
}

// Java 25 preview - 原始类型 pattern
static String classify(Object obj) {
    return switch (obj) {
        case int i     -> "int: " + i;
        case long l    -> "long: " + l;
        case double d  -> "double: " + d;
        case Boolean b -> "boolean: " + b;
        default        -> "unknown";
    };
}

// 直接用 switch 处理 int
static String dayOfWeek(int day) {
    return switch (day) {
        case 1 -> "Monday";
        case 2 -> "Tuesday";
        case 3 -> "Wednesday";
        case 4 -> "Thursday";
        case 5 -> "Friday";
        case 6, 7 -> "Weekend";
        default -> "Invalid";
    };
}
```

## 6. Module Import Declarations（JEP 511, Java 25 final）

```java
// Java 24 之前 - 每个类都要 import
import java.util.List;
import java.util.Map;
import java.util.HashMap;
import java.util.ArrayList;
import java.util.stream.Collectors;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

class OldStyle {
    void main() throws IOException {
        List<String> l = new ArrayList<>();
        // ...
    }
}

// Java 25+ - 一次性 import 整个模块
import module java.base;        // java.util.* + java.io.* + java.nio.* + ...
import module java.sql;

class ModernStyle {
    void main() throws IOException {
        List<String> l = new ArrayList<>();  // 不需要单独 import
        Path p = Path.of("test.txt");
        Files.readString(p);
    }
}
```

**支持的模块**：
- `java.base` — 核心（默认就用得到的所有包）
- `java.sql` — JDBC
- `java.desktop` — AWT / Swing
- `java.logging` — java.util.logging
- `java.xml` — JAXP
- 第三方模块需要 JPMS exports 才能被 import

**实战**：教学/脚本/快速 prototype 时大量减少 import 行。

## 7. Compact Source Files（JEP 512, Java 25 final）

```java
// Java 24 之前 - 教学 Hello World 8 行
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// Java 25+ - 4 行（甚至可以不要 class 定义）
// 文件名不必是 Hello.java
import module java.base;

void main() {
    IO.println("Hello, Java 25!");   // IO 是新工具类
}
```

**JEP 463 / 512 允许**：
- 文件名不必与 public class 一致
- 不必显式 class 声明（隐式 class）
- main 方法不必 `public static`
- main 方法不必有 `String[] args`
- `IO.println` 比 `System.out.println` 短

**意义**：让学生/初学者的第一个程序从 8 行变 4 行。Python / Kotlin / JavaScript 用户的体验差距进一步缩小。

**反模式**：
```java
// ❌ 不要在生产代码用 compact source files
// 教学 / 脚本 / single-file launch 才用
// JAR 部署仍要正规 class + main 签名
```

## 8. Flexible Constructor Bodies（JEP 513, Java 25 final）

```java
// Java 24 之前 - super 前不能写语句（除了 this(...) / super(...) 链）
public PositiveBigInteger(long value) {
    if (value <= 0) throw new IllegalArgumentException("non-positive: " + value);
    // 报错：构造器首行必须是 super
    super(value);
}

// Java 25+ - 允许 super 前执行语句（但不能读 this）
public PositiveBigInteger(long value) {
    if (value <= 0) throw new IllegalArgumentException("non-positive: " + value);
    log("Creating positive BI: " + value);  // 可以写
    super(value);
}

// 注意：super 前不能读 this.field（防止对象未初始化时被使用）
public PositiveBigInteger(long value) {
    super(value);
    // 这里才能用 this.field
}
```

## 9. Structured Concurrency（JEP 505, Java 25 preview 5）

把多个并发任务当作**一个单元**管理，简化并行代码。

```java
// Java 24 之前 - CompletableFuture 链式
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> fetchUser(123));
CompletableFuture<List<Post>> postsF = CompletableFuture.supplyAsync(() -> fetchPosts(123));

CompletableFuture.allOf(userF, postsF)
    .thenRun(() -> {
        // 一个失败另一个不取消
    });

// Java 25+ StructuredTaskScope
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userF = scope.fork(() -> fetchUser(123));
    var postsF = scope.fork(() -> fetchPosts(123));

    scope.join();           // 等待所有
    scope.throwIfFailed();  // 任一失败 → 抛异常 + 取消其他

    return new UserPosts(userF.get(), postsF.get());
}
// 退出 try 块时自动清理所有子任务（一个失败全部取消）
```

**核心优势**：
- **生命周期绑定** — 子任务不能超过父任务的 scope
- **失败传播** — 一个失败 → 全部取消
- **错误透明** — 不需要手动检查每个 Future

**三种 shutdown 语义**：
```java
// 1. ShutdownOnFailure - 任一失败 → 全部取消
new StructuredTaskScope.ShutdownOnFailure()

// 2. ShutdownOnSuccess - 任一成功 → 全部取消（拿最快结果）
new StructuredTaskScope.ShutdownOnSuccess()

// 3. Custom - 自己实现 shutdown 逻辑
new StructuredTaskScope<>() {
    @Override
    protected void handleComplete(...) {
        // ...
    }
}
```

### 9.1 Structured Concurrency 实战拆解

#### ① 3 种 shutdown 策略

```java
// 策略 1：ShutdownOnFailure - 任一失败 → 全部取消
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var a = scope.fork(() -> fetchA());
    var b = scope.fork(() -> fetchB());
    scope.join();
    scope.throwIfFailed();        // 任一失败抛 ExecutionException + 取消其他
    return combine(a.get(), b.get());
}

// 策略 2：ShutdownOnSuccess - 任一成功 → 全部取消（拿最快）
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> fetchFromCDN("us"));
    scope.fork(() -> fetchFromCDN("eu"));
    scope.fork(() -> fetchFromCDN("asia"));
    scope.join();
    return scope.result();  // 第一个成功的
}

// 策略 3：自定义（继承 StructuredTaskScope）
class MyScope<T> extends StructuredTaskScope<T> {
    @Override
    protected void handleComplete(...) {
        // 自定义 shutdown 逻辑
    }
}
```

#### ② fork + join 标准模式

```java
T result = ScopedValue.call(...)
    .call(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var a = scope.fork(task1);
            var b = scope.fork(task2);
            
            scope.join();           // 阻塞等待所有
            scope.throwIfFailed();   // 任一失败抛异常 + 取消其他
            
            return combine(a.get(), b.get());
        }
    });
```

#### ③ 异常传播机制

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var a = scope.fork(() -> {
        throw new RuntimeException("a 失败");
    });
    var b = scope.fork(() -> {
        Thread.sleep(5000);   // 也会被取消
        return "b";
    });
    
    scope.join();             // a 抛异常 → 立即触发 shutdown
    scope.throwIfFailed();    // 抛 ExecutionException，cause = RuntimeException
    // b 已被 interrupt 取消
}
```

**传播规则**：
- 任一子任务抛异常 → `shutdown()` 被调用 → 其他子任务 interrupt
- `throwIfFailed()` 抛 `ExecutionException`（cause 是首个失败的子任务异常）
- 所有未完成的子任务被取消，资源释放

#### ④ 子任务生命周期

```java
// 父 scope 退出 → 自动取消所有子任务
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var task = scope.fork(() -> {
        while (true) {
            doWork();
            Thread.sleep(1000);
        }
    });
    scope.join();              // 阻塞
}  // 退出 try 时 task 自动被取消（structured binding）

// 子任务不能"逃逸"
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var task = scope.fork(() -> longRunningWork());
    scope.join();
    // ❌ 不能在这里把 task.get() 抛到 scope 外
    // Future<Object> outsideRef = task;  // ❌ 子任务不能超过 scope 生命周期
}
```

#### ⑤ vs CompletableFuture

| 维度 | CompletableFuture | Structured Concurrency |
|---|---|---|
| 生命周期 | 不绑定父 | 绑定父 scope |
| 失败处理 | `exceptionally()` / `handle()` | `throwIfFailed()` |
| 取消传播 | 需要 `.whenComplete()` 手动 | 自动 |
| 嵌套 | CF 可以返回 CF | 子任务不能 spawn 独立 task |
| 错误传播链 | `CompletionException` | `ExecutionException` |
| 适合 | 长生命周期 pipeline | **短期并行任务** |
| 监控 | 复杂 | 自动 structured |

#### ⑥ 实战：并行 HTTP fetch

```java
record Dashboard(User user, List<Post> posts, Stats stats) {}

Dashboard loadDashboard(int userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var userF  = scope.fork(() -> http.get("/users/" + userId, User.class));
        var postsF = scope.fork(() -> http.get("/users/" + userId + "/posts", List.class));
        var statsF = scope.fork(() -> http.get("/users/" + userId + "/stats", Stats.class));

        scope.join();
        scope.throwIfFailed();  // 任一 HTTP 失败 → 全部取消

        return new Dashboard(userF.get(), postsF.get(), statsF.get());
    }
}

// Scoped Values 集成
private static final ScopedValue<User> CURRENT = ScopedValue.newInstance();

Dashboard loadDashboard(int userId) {
    User user = fetchUser(userId);
    return ScopedValue.where(CURRENT, user).call(() -> {
        // CURRENT 在子任务里可见（与 fork 自动绑定）
        return loadDashboardInternal();
    });
}
```

#### ⑦ 6 条反模式

❌ **在 fork 里做无限循环** —— 阻塞 join()，永不返回
❌ **fork 返回 CompletableFuture 的 lambda** —— 失去 structured 语义
❌ **fork 出去的子任务访问父 scope 外资源** —— 生命周期错乱
❌ **不用 try-with-resources** —— 子任务永不取消
❌ **多个 scope 嵌套 + 异常** —— 异常传播复杂
❌ **在 fork 里做慢同步 I/O** —— 阻塞其他子任务调度（应配合虚拟线程）

---

**核心洞察**：Structured Concurrency 是 Java 25 给"短期并行任务"立的**结构化并发**标准——把 fork-join 模式从"手动 Future 链"压缩成"一个 try-with-resources + fork/join"。核心收益是**生命周期绑定**：父任务结束 → 子任务自动取消，**没有泄漏的孤儿线程**。**所有"并行 N 个调用 + 合并结果"的场景都应该用这个替代 CompletableFuture.allOf**。

## 10. Stable Values（JEP 502, Java 25 preview 1）

让 JVM 能把**延迟初始化的不可变对象**当作常量优化。

```java
// Java 25 之前 - lazy 初始化要么每次检查、要么 final field
class Config {
    private static final Map<String, String> DATA = loadData();  // 启动就加载（可能很重）

    public static String get(String key) { return DATA.get(key); }
}

// Java 25+ - 延迟 + JVM 优化
class Config {
    private static final StableValue<Map<String, String>> DATA =
        StableValue.of(Config::loadData);  // 第一次访问时加载

    public static String get(String key) { return DATA.get().get(key); }
}
```

**优势**：启动快 + 第一次访问开销低 + JVM 可以做激进优化（因为不可变）。

## 11. Scoped Values（JEP 506, Java 25 final）

**ThreadLocal 的现代化替代品**，专为虚拟线程 + 结构化并发设计。

```java
// Java 24 之前 - ThreadLocal
private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
public static void setCurrentUser(User user) { CURRENT_USER.set(user); }
public static User getCurrentUser() { return CURRENT_USER.get(); }

// 问题：
// - 内存泄漏（线程池复用，忘记 remove()）
// - 虚拟线程数百万个时 ThreadLocal map 占内存
// - 子线程继承父线程的值（Mutable 状态蔓延）

// Java 25+ ScopedValue
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

public void handleRequest(User user, Runnable work) {
    // try-with-resources 自动绑定
    try (var binding = CURRENT_USER.bind(user)) {
        work.run();
    }  // 自动解绑
}

// 嵌套绑定
ScopedValue.where(CURRENT_USER, user)
    .where(REQUEST_ID, "req-123")
    .run(() -> processRequest());

// 读取
ScopedValue.get(CURRENT_USER);  // 当前 scope 内的值
```

**对比 ThreadLocal**：

| 维度 | ThreadLocal | ScopedValue |
|---|---|---|
| 不可变 | ❌ | ✅ |
| 内存泄漏 | 易 | 无（自动绑定/解绑）|
| 百万虚拟线程 | O(n) map 内存 | O(1) 单实例 |
| 子线程继承 | 隐式 | 显式（`where().run()`）|
| 嵌套 | 需手动栈 | 天然嵌套（栈式绑定）|

### 11.1 Scoped Values 实战拆解

#### ① 基础用法：bind + get

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// 调用方
public void handleRequest(User user, Runnable work) {
    ScopedValue.where(CURRENT_USER, user)
        .run(work);
    // 退出 run 时自动解绑
}

// 接收方（任何嵌套调用深度）
static void process() {
    User u = ScopedValue.get(CURRENT_USER);  // 拿到当前 scope 的 User
    System.out.println("Processing for: " + u);
}
```

#### ② try-with-resources 写法

```java
public void handleRequest(User user, Runnable work) {
    try (var binding = CURRENT_USER.bind(user)) {
        work.run();
    }  // 自动 close（解绑）
}
```

**两种写法对比**：

| 写法 | 形式 | 灵活度 |
|---|---|---|
| `where(...).run(work)` | 闭包式（lambda）| 紧凑 |
| `try (var binding = bind(...))` | 块体式（try/catch）| 灵活（可加异常处理）|

#### ③ 嵌套绑定（栈式）

```java
private static final ScopedValue<String> USER = ScopedValue.newInstance();
private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

ScopedValue.where(USER, "mike")
    .where(REQUEST_ID, "req-123")  // 嵌套：第二个 where 嵌套第一个
    .run(() -> {
        System.out.println(ScopedValue.get(USER));        // "mike"
        System.out.println(ScopedValue.get(REQUEST_ID));   // "req-123"

        // 内层 where 临时覆盖
        ScopedValue.where(USER, "alice")
            .run(() -> {
                System.out.println(ScopedValue.get(USER));        // "alice"（内层覆盖）
                System.out.println(ScopedValue.get(REQUEST_ID));   // "req-123"（未覆盖）
            });

        // 退出内层后恢复
        System.out.println(ScopedValue.get(USER));  // "mike"
    });
```

**关键**：栈式语义——内层 where 临时覆盖外层，退出后恢复。

#### ④ 与 ThreadLocal 完整对比

| 维度 | ThreadLocal | ScopedValue |
|---|---|---|
| 可变性 | ❌（但 TL 本身可变）| ✅ 不可变 |
| 内存泄漏 | ⚠️（线程池复用忘 remove）| ✅ 自动解绑 |
| 百万虚拟线程 | ⚠️ 每 VT 一个 TL entry → OOM | ✅ 单一实例 |
| 子线程继承 | 隐式（`InheritableThreadLocal`）| 显式（`where().run()`）|
| 嵌套 | 需手动栈管理 | 天然嵌套（栈式绑定）|
| 类型安全 | `Object`（要转型）| 泛型 `ScopedValue<T>` |
| API 风格 | `set()` / `get()` / `remove()` | `where().run()` / `get()` |
| 是否能中途修改值 | ✅（`tl.set(new)`）| ❌（不可变）|

#### ⑤ 不自动继承子线程（vs InheritableThreadLocal）

```java
// ThreadLocal - 子线程继承父线程的值（隐式）
InheritableThreadLocal<String> tl = new InheritableThreadLocal<>();
tl.set("parent-value");
new Thread(() -> {
    System.out.println(tl.get());  // "parent-value"（隐式继承）
}).start();

// ScopedValue - 子任务不继承
ScopedValue<String> sv = ScopedValue.newInstance();
ScopedValue.where(sv, "parent-value")
    .run(() -> {
        // 新建虚拟线程
        Thread.startVirtualThread(() -> {
            System.out.println(ScopedValue.get(sv));  // ❌ NoSuchElementException
        });

        // 子线程要拿到，必须显式 where() 重新绑定
        Thread.startVirtualThread(() -> {
            ScopedValue.where(sv, ScopedValue.get(sv))  // 手动传
                .run(() -> {
                    System.out.println(ScopedValue.get(sv));  // ✅
                });
        });
    });
```

**关键差异**：ScopedValue **不自动继承**——子任务必须显式 `where()`。

#### ⑥ 在结构化并发中使用

```java
private static final ScopedValue<User> CURRENT = ScopedValue.newInstance();

public UserPosts loadDashboard(int userId) {
    User user = fetchUser(userId);

    return ScopedValue.where(CURRENT, user)
        .call(() -> {
            try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                var postsF = scope.fork(() -> {
                    User u = CURRENT.get();        // 子任务里能拿到
                    return fetchPosts(u.id());
                });

                var statsF = scope.fork(() -> {
                    User u = CURRENT.get();        // 同样能拿到
                    return fetchStats(u.id());
                });

                scope.join();
                scope.throwIfFailed();
                return new UserPosts(postsF.get(), statsF.get());
            }
        });
}
```

**关键**：ScopedValue 在 structured concurrency 里直接被子任务可见（因为 fork 的任务继承父 scope 的绑定）。

#### ⑦ 实战反模式

❌ **当 ThreadLocal 用 set/remove** —— ScopedValue 没有 set()，只能 bind
❌ **子线程直接 `get()` 不 where()** —— 抛 `NoSuchElementException`
❌ **绑大对象** —— 不可变快照，存太多数据 → 内存压力
❌ **混用 ThreadLocal 和 ScopedValue** —— 同一份上下文存两份
❌ **多层嵌套 where 不退出** —— 嵌套越深栈越深
❌ **当 cache 用** —— ScopedValue 是上下文传递，不是缓存

---

**核心洞察**：ScopedValue 是 Java 25 给"虚拟线程时代的上下文传递"立的**新标准**。它把 ThreadLocal 的"可变 + 隐式继承 + 内存泄漏"三大问题全部解决，代价是**必须显式 `where()` + 子任务不自动继承**。所有 ThreadLocal 在虚拟线程环境都应该评估改写——**特别是 request ID / user context / trace ID 这类"只读上下文"**。

## 12. 实战：现代并发全栈

```java
// Java 25 - 虚拟线程 + 结构化并发 + Scoped Values
class UserService {
    private static final ScopedValue<User> CURRENT = ScopedValue.newInstance();

    public UserPosts loadDashboard(int userId) {
        return ScopedValue.where(CURRENT, fetchUser(userId))
            .call(() -> {
                try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                    var postsF = scope.fork(() -> fetchPosts(userId));
                    var statsF = scope.fork(() -> fetchStats(userId));

                    scope.join();
                    scope.throwIfFailed();

                    return new UserPosts(CURRENT.get(), postsF.get(), statsF.get());
                }
            });
    }
}

// 运行
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> userService.loadDashboard(123));
}
```

## 13. 关键设计原则

1. **`_` 用在不关心的命名** — 不再写 `unusedException` 等 hack
2. **FFM 替代 JNI** — 新项目首选 FFM；JNI 仅维护老代码
3. **Stream Gatherer 自定义中间操作** — 不再写 `for` 循环包装
4. **primitive pattern 让 switch 更通用** — 等 final
5. **`import module java.base`** — 减少 import 噪音
6. **compact source file 仅用于学习/脚本** — 生产代码用正规 class
7. **flexible constructor body 不能读 this** — 防半初始化
8. **structured concurrency 替代手动 Future 编排**
9. **stable values 是 final field 的延迟版** — JVM 优化更好
10. **scoped values 替代 ThreadLocal** — 在虚拟线程环境下

## 14. 反模式

❌ **预览特性写进生产** — `preview` 特性每次升级可能破坏
❌ **FFM + JNI 混用** — 容易引起双重管理内存
❌ **`import module java.base` 滥用** — 公开 API 不要用 module import（IDE 跳转困难）
❌ **compact source file 写生产** — JAR 部署必须用正规 class
❌ **super 前读 this** — 编译错误（JEP 513 明确禁止）
❌ **structured concurrency 不在 try-with-resources** — 资源泄漏
❌ **scoped value 改 ThreadLocal 已有的代码** — 迁移前评估
❌ **stable value 当 lazy** — 设计意图不同

## 相关笔记

- **Java 21 LTS**: [[draft-04-java-21-lts-virtual-threads]]
- **Kotlin coroutines 对比**: [[../kotlin-analysis/draft-03-coroutines]]
- **Java 速查**: [[draft-06-version-timeline-cheatsheet]]
