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
base_confidence: 0.88
lifecycle: draft
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
