---
title: "Java 21 LTS - 虚拟线程 + 模式匹配"
category: synthesis
tags: [java, java-21, lts, virtual-threads, sequenced-collections, record-patterns, switch-pattern, string-templates, unnamed]
sources:
  - "JEP 444 - Virtual Threads"
  - "JEP 431 - Sequenced Collections"
  - "JEP 420 - Pattern Matching for switch"
  - "JEP 440 - Record Patterns"
  - "JEP 430 - Pattern Matching for switch (Preview)"
  - "JEP 459 - String Templates (Second Preview)"
  - "JEP 456 - Unnamed Variables & Patterns"
summary: "Java 21 (2023-09, LTS) - 虚拟线程 + sequenced collections + pattern switch 标准 + record patterns + string templates"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.89
lifecycle: draft
lifecycle_changed: 2026-10-03
created: 2026-10-03
updated: 2026-10-03
---

# §04 Java 21 LTS - 虚拟线程 + 模式匹配

> **Java 21 (2023-09)** 是 Java 17 之后的第二个新节奏 LTS（2 年周期）。它是过去 10 年最重要的版本，因为**虚拟线程（virtual threads）**让 Java 第一次有了"百万级并发"的原生支持。加上 **pattern matching switch** 和 **record patterns** 的标准化，Java 21 是真正的"现代 Java"分水岭。

## 1. virtual threads（JEP 444, final）

### 核心概念

```java
// Java 21 之前：平台线程（OS 线程 1:1）
Thread thread = new Thread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
});
thread.start();
thread.join();

// Java 21+ 虚拟线程（JVM 调度，轻量级）
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
    // 实际跑在 ForkJoinPool 公共池
});
vt.join();

// 命名虚拟线程
Thread.builder().virtual().name("worker-1").task(() -> {
    // ...
}).start();

// 使用 ExecutorService
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> {
    // 每个任务一个虚拟线程
});
executor.close();
```

### 1.1 Virtual threads 实战拆解

#### ① `Thread.startVirtualThread` —— 最直接

```java
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("Hello from " + Thread.currentThread());
});
vt.join();  // 等待完成
```

- 一次性启动，无需持有 Thread 引用
- 自动使用 `ForkJoinPool.commonPool()` 作为 carrier
- 适用：脚本 / 一次性任务 / 测试代码

#### ② `Executors.newVirtualThreadPerTaskExecutor` —— 服务推荐

```java
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1000; i++) {
        exec.submit(() -> handleRequest());
    }
}  // 退出 try 时自动 await termination
```

- 每个任务一个虚拟线程（**不要池化**）
- try-with-resources 自动关闭 + 等待
- 适用：HTTP 服务器 / 批处理 / 替代 `newFixedThreadPool`

#### ③ `Thread.ofVirtual()` —— 细粒度配置

```java
Thread vt = Thread.ofVirtual()
    .name("worker-1")
    .daemon(false)
    .uncaughtExceptionHandler((t, e) -> log.error("VT crashed", e))
    .start(() -> process());
```

- 配置：`name` / `daemon` / `priority` / `uncaughtExceptionHandler` / `contextClassLoader`
- 适用：库代码 / 自定义线程管理

#### ④ `Thread.Builder` —— 工厂模式（共享配置）

```java
Thread.Builder factory = Thread.ofVirtual()
    .name("worker-")
    .daemon(true);

Thread t1 = factory.start(() -> work(1));  // name=worker-0
Thread t2 = factory.start(() -> work(2));  // name=worker-1
```

- 工厂复用配置（不用每次写 name/daemon）
- 适合：批量创建同类虚拟线程

#### ⑤ Pinning 机制详解

```java
// ❌ synchronized + I/O → pinning（Java 21）
synchronized (lock) {
    blockingIO();  // 虚拟线程被 pin 在 carrier 上，占着 OS 线程
}

// ✅ 改用 ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingIO();
} finally {
    lock.unlock();
}

// 检测 pinning（启动参数）
// java -Djdk.tracePinnedThreads=full MyApp
// 输出详细 pinning 栈，定位问题
```

**Pinning 演进时间线**：
- **Java 21**：synchronized + I/O → pinning ⚠️（常见问题）
- **Java 24**（JEP 491）：synchronized 基本不 pinning（除 JNI）✅
- 解决方案：迁移到 `ReentrantLock`，或等升级到 24+

#### ⑥ vs Platform Thread 速查

| 维度 | Platform Thread | Virtual Thread |
|---|---|---|
| 调度 | OS 内核 1:1 | JVM M:N（虚拟→ForkJoinPool）|
| 默认栈 | 1 MB | ~几 KB（按需扩容）|
| 创建成本 | 高（系统调用）| 极低（Java 对象）|
| 数量上限 | 几千 | 百万级 |
| 阻塞 I/O | 占 OS 线程 | **释放 carrier** |
| `synchronized` | 安全 | ⚠️ pinning（21）/ 安全（24+）|
| `Thread.sleep` | 占 OS 线程 | 占虚拟线程但不占 OS 线程 |
| `ThreadLocal` | 正常 | ⚠️ 内存放大（百万虚拟线程 × TL = OOM）→ 改用 [[draft-05-java-22-25-modern-concurrency#11-scoped-values-final\|Scoped Value]] |
| 适用 | CPU-bound | **I/O-bound** |

#### ⑦ 4 种创建方式对照

| 方式 | 一行写法 | 适用场景 |
|---|---|---|
| `startVirtualThread` | `Thread.startVirtualThread(task)` | 一次性脚本 |
| `newVirtualThreadPerTaskExecutor` | `Executors.newVirtualThreadPerTaskExecutor()` | 服务/批处理 |
| `Thread.ofVirtual()` | `Thread.ofVirtual().name(...).start(task)` | 单条 + 配置 |
| `Thread.Builder` | `factory.start(task)` | 批量复用配置 |

---

**核心洞察**：虚拟线程的"心智模型"和平台线程**根本不同**——
- ❌ **错误思维**：把虚拟线程当"轻量级线程"，用线程池 + 队列
- ✅ **正确思维**：把虚拟线程当"自动让出的同步代码"——按需创建，I/O 阻塞时让出 carrier，CPU 运行时才占用

实战铁律：
1. **不要池化虚拟线程**（`newFixedThreadPool(100)` + 虚拟线程 = 错的）
2. **不要在虚拟线程里用 synchronized + 阻塞 I/O**（pinning 浪费 OS 线程，Java 24+ 才解决）
3. **不要用 ThreadLocal 存大对象**（百万虚拟线程 × TL = OOM，改用 [[draft-05-java-22-25-modern-concurrency#11-scoped-values-final\|Scoped Value]]）
4. **同步 API 直接用**（`RestTemplate` / `JdbcTemplate` / `HttpClient`）——虚拟线程让它们"自动异步"

### 虚拟线程 vs 平台线程

| 维度 | Platform Thread | Virtual Thread |
|---|---|---|
| 1:1 OS 线程 | ✅ | ❌（M:N 到 OS 线程）|
| 默认栈大小 | 1 MB | ~几 KB（按需扩容）|
| 创建成本 | 高（系统调用）| 极低（Java 对象）|
| 上限 | 几千（受 OS / 内存限制）| 百万级 |
| 阻塞成本 | 高（占着 OS 线程）| 低（自动释放 OS 线程）|
| synchronized 块 | 安全 | ⚠️ pinning（Java 24 解决）|
| 适用场景 | CPU-bound | **I/O-bound** |

### 实战：百万级并发

```java
// ❌ Java 8 - 100 万线程直接 OOM
for (int i = 0; i < 1_000_000; i++) {
    new Thread(() -> blockingIO()).start();  // 1MB × 100万 = 1TB
}

// ✅ Java 21+ - 100 万虚拟线程轻松
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> blockingIO());  // ~几 KB × 100万 = 几 GB（还可接受）
    }
}  // 自动 await termination
```

### 关键陷阱：pinning 问题

```java
// ❌ synchronized 块会导致虚拟线程 pinned 到 carrier 线程
synchronized (lock) {
    blockingIO();  // 虚拟线程不能释放，浪费 OS 线程
}

// ✅ 改用 java.util.concurrent.locks.ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingIO();
} finally {
    lock.unlock();
}
```

> Java 24（JEP 491）通过改进 synchronized 实现基本解决了 pinning。

### 适用 vs 不适用

```java
// ✅ 适合：I/O-bound（HTTP / DB / 文件）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (URL url : urls) {
        executor.submit(() -> fetchUrl(url));
    }
}

// ❌ 不适合：CPU-bound（计算密集）
// 虚拟线程帮不了忙，用 parallel stream 或 ForkJoinPool
IntStream.range(0, 1_000_000)
    .parallel()
    .mapToObj(i -> heavyCompute(i))
    .toList();
```

## 2. sequenced collections（JEP 431, final）

```java
// Java 21 之前：没有"第一个/最后一个/反向"的统一 API
list.get(0);                  // 第一个
list.get(list.size() - 1);    // 最后一个
for (int i = list.size() - 1; i >= 0; i--) list.get(i);  // 反向

// Java 21+ SequencedCollection
SequencedCollection<String> seq = new ArrayList<>(List.of("A", "B", "C"));

seq.getFirst();      // "A"
seq.getLast();       // "C"

SequencedCollection<String> reversed = seq.reversed();
// 反向迭代器
for (var s : reversed) { /* "C", "B", "A" */ }

// 头部 / 尾部操作
seq.addFirst("X");
seq.addLast("Z");

// SequencedSet / SequencedMap
SequencedSet<Integer> seqSet = new LinkedHashSet<>();
SequencedMap<String, Integer> seqMap = new LinkedHashMap<>();
seqMap.firstEntry();    // 第一个 entry
seqMap.lastEntry();     // 最后一个 entry
seqMap.reversed();      // 反向视图
```

**意义**：标准化了"有序集合"的首/末/反向操作，让 List / Deque / LinkedHashSet / LinkedHashMap 有了统一接口。

## 3. pattern matching switch standard（JEP 420, final）

```java
// Java 21 前还是 preview，Java 21 final
static String describe(Shape shape) {
    return switch (shape) {
        case Circle c    -> "Circle radius=" + c.radius();
        case Rectangle r -> "Rectangle " + r.width() + "x" + r.height();
        case Square s    -> "Square side=" + s.side();
        case null        -> "Null shape";
    };
}

// sealed class 时 switch 自动 exhaustive（无 default）
static double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Square s    -> s.side() * s.side();
        // 不需要 default - sealed permits 已穷举
    };
}
```

**when 子句**（模式守卫）：
```java
static String classify(Number n) {
    return switch (n) {
        case Integer i when i > 0  -> "Positive integer: " + i;
        case Integer i             -> "Non-positive integer: " + i;
        case Double d when d > 0.0 -> "Positive double";
        default                    -> "Other";
    };
}
```

## 4. record patterns（JEP 440, final）

```java
// Java 21+ 解构 record
static double area(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
        case Square(double s) -> s * s;
    };
}

// instanceof record pattern
if (shape instanceof Circle(double r)) {
    System.out.println("Radius: " + r);
}

// 嵌套 record pattern
record Container(List<Item> items) {}

if (obj instanceof Container(List<Item> items)) {
    for (Item item : items) {
        // ...
    }
}

// 解构 + 类型守卫
if (obj instanceof Container(List<Item> items) && items.size() > 0) {
    // ...
}
```

**AST 处理**（配合 sealed + record）：
```java
sealed interface JsonValue permits JsonNull, JsonBool, JsonNumber, JsonString, JsonArray, JsonObject {}
record JsonNull() implements JsonValue {}
record JsonBool(boolean value) implements JsonValue {}
record JsonNumber(double value) implements JsonValue {}
record JsonString(String value) implements JsonValue {}
record JsonArray(List<JsonValue> values) implements JsonValue {}
record JsonObject(Map<String, JsonValue> entries) implements JsonValue {}

static String render(JsonValue v) {
    return switch (v) {
        case JsonNull()           -> "null";
        case JsonBool(boolean b)  -> b ? "true" : "false";
        case JsonNumber(double n) -> String.valueOf(n);
        case JsonString(String s) -> "\"" + s + "\"";
        case JsonArray(List<JsonValue> vs)
                                  -> "[" + vs.stream().map(JsonUtil::render)
                                              .collect(Collectors.joining(",")) + "]";
        case JsonObject(Map<String, JsonValue> e)
                                  -> "{" + e.entrySet().stream()
                                              .map(en -> "\"" + en.getKey() + "\":" + render(en.getValue()))
                                              .collect(Collectors.joining(",")) + "}";
    };
}
```

## 5. string templates（JEP 459, second preview）

```java
// Java 21 之前 - String.format 或字符串拼接
String msg = String.format("Hello, %s! You are %d years old.", name, age);
String html = "<html><body>" + "<p>" + name + "</p></body></html>";

// Java 21+ String Templates (preview 2)
String msg = STR."Hello, \{name}! You are \{age} years old.";
// STR = 简单插值模板处理器

// FMT - 格式化
String formatted = FMT."%05d\{n}";  // 00042

// RAW - 不转义
String raw = RAW."hello\nworld";   // 字面 \n
```

> Java 22 取消预览，Java 23 重新进入 preview（具体标准化时间待定）

## 6. unnamed patterns & variables（JEP 456, preview）

```java
// Java 21 前
try (var in = new FileInputStream("a");
     var out = new FileOutputStream("b")) {
    // in / out 都用了
}

// Java 21+ 用 _ 表示"不关心"
try (var _ = new FileInputStream("a");
     var out = new FileOutputStream("b")) {
    // _ 表明只关心 close，不关心变量
}

// Lambda 参数
list.forEach(_ -> System.out.println("processing"));

// catch 不需要的异常
try {
    // ...
} catch (InterruptedException _) {  // 不需要 e
    Thread.currentThread().interrupt();
}

// unnamed pattern (record 解构时)
if (obj instanceof Container(List<Item> _)) {  // 不关心 list 内容
    // ...
}
```

## 7. 实战：并发模型升级

```java
// Spring Boot 3.2+ 自动用虚拟线程
// application.yml
spring:
  threads:
    virtual:
      enabled: true

// Tomcat 自动用虚拟线程处理 HTTP 请求
// 无需改业务代码

// 配合 StructuredTaskScope (Java 25 final 之前用 incubator)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> fetchUser(123));
    var posts = scope.fork(() -> fetchPosts(123));

    scope.join();
    scope.throwIfFailed();

    return new UserPosts(user.get(), posts.get());
}
```

## 8. 关键设计原则

1. **虚拟线程是 I/O 的解药** — 但不是 CPU 的解药
2. **避免 synchronized 块** — 改用 `ReentrantLock`（Java 24+ 已基本解决 pinning）
3. **pattern switch + sealed = exhaustive** — 编译器帮你验证分支完整
4. **record patterns 替代 instanceof 链** — 数据驱动的代码更清晰
5. **string templates 别预览特性** — 还在演进，标准 API 可能改

## 9. 反模式

❌ **synchronized 块内做 I/O** — pinning 浪费 OS 线程
❌ **虚拟线程用 thread pool** — `newFixedThreadPool(100)` + 虚拟线程毫无意义
❌ **pattern switch 不穷尽 sealed** — 加 default 反而破坏 exhaustive 检查
❌ **record pattern 用 _ 占位符时还要引用** — `_` 表示不用
❌ **虚拟线程配 CPU-bound** — 不会变快
❌ **string templates 写死模板** — 仍在 preview，API 可能变

## 相关笔记

- **Java 14-17**: [[draft-03-java-14-17-data-and-patterns]]
- **Java 22-25 并发**: [[draft-05-java-22-25-modern-concurrency]]
- **mu-server 实战**: [[../mu-server-2.4.2-analysis/summary]]
- **Kotlin coroutines 对比**: [[../kotlin-analysis/draft-03-coroutines]]
