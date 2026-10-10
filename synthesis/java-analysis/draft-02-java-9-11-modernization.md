---
title: "Java 9-11 现代化 - 模块化 + HTTP Client"
category: synthesis
tags: [java, java-9, java-10, java-11, module-system, jigsaw, var, http-client]
sources:
  - "JEP 261 - Module System"
  - "JEP 286 - Local Variable Type Inference"
  - "JEP 321 - HTTP Client"
  - "JEP 330 - Single-File Source-Code Launch"
summary: "Java 9-11 (2017-2018) - 模块系统(JPMS) + 集合工厂方法 + var(类型推断) + HTTP Client + 单文件启动"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.89
lifecycle: draft
lifecycle_changed: 2026-10-10
created: 2026-10-03
updated: 2026-10-10
---

# §02 Java 9-11 现代化

> Java 9 (2017-09)、Java 10 (2018-03)、Java 11 (2018-09, LTS) 共同构成了 Java 现代化的第一波。Java 9 的模块系统虽然影响深远，但生态采纳缓慢；真正让开发者受益的是 Java 10 的 `var` 和 Java 11 的 HTTP Client。

## 1. 模块系统（Jigsaw / JPMS）

### 核心概念

```java
// module-info.java
module com.example.myapp {
    requires java.sql;              // 依赖 java.sql
    requires transitive com.fasterxml.jackson.core;  // 传递性依赖

    exports com.example.myapp.api;  // 公开的包
    exports com.example.myapp.spi to com.example.framework;  // 限定公开

    provides com.example.spi.Service
        with com.example.myapp.internal.MyServiceImpl;  // ServiceLoader

    uses com.example.spi.Service;  // 消费服务
}
```

**模块 vs 包**：
- 包：代码分组，访问控制用 `public` / package-private
- 模块：包的分组，**更强的访问控制**（只有 `exports` 的包才能被外部模块访问）

### 实战：从 classpath 迁移到 modulepath

```bash
# Java 8 风格
java -cp 'libs/*:target/classes' com.example.Main

# Java 11+ 模块
java -p libs -m com.example.myapp/com.example.Main
```

### 模块化的副作用

- **反射访问限制**（Java 16 默认禁止，Java 17 强封装）：
  ```java
  // Java 8 反射能访问一切
  Class<?> c = Class.forName("com.sun.misc.Unsafe");

  // Java 17+ 默认禁止
  // 必须 --add-opens java.base/sun.misc=ALL-UNNAMED
  ```

- **类路径分裂**：很多库没改造成模块，最终 `--add-modules` + `--add-opens` 满天飞

## 2. 集合工厂方法（Java 9）

```java
// Java 8 之前
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("C");
list = Collections.unmodifiableList(list);

Set<String> set = new HashSet<>();
set.add("X");
set.add("Y");
set = Collections.unmodifiableSet(set);

Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
map = Collections.unmodifiableMap(map);

// Java 9+
List<String> list = List.of("A", "B", "C");
Set<String> set = Set.of("X", "Y", "Z");
Map<String, Integer> map = Map.of("a", 1, "b", 2);

// Map.of 超过 10 个 entry 用 Map.ofEntries
Map<String, Integer> big = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
);
```

**关键点**：
- `List.of` / `Set.of` / `Map.of` 返回的是**不可变集合**（类似 Kotlin 的 `listOf` / `setOf` / `mapOf`）
- 不允许 `null` 元素
- 比 `Arrays.asList` 更紧凑、更不可变、更高效

**反模式**：
```java
// ❌ List.of 创建的 list 调用 add 会抛 UnsupportedOperationException
List<String> l = List.of("a");
l.add("b");  // 异常！

// ❌ Map.of 用 null key 或 null value
Map.of("k", null);  // NullPointerException
```

## 3. 接口私有方法（Java 9）

```java
public interface Calculator {
    default int add(int a, int b) { return compute(a, b, Long::sum); }
    default int multiply(int a, int b) { return compute(a, b, (x, y) -> (int)(x * y)); }

    // Java 9+：私有方法（default 之间共享逻辑）
    private int compute(int a, int b, BinaryOperator<Long> op) {
        validate(a);
        validate(b);
        return op.apply((long)a, (long)b).intValue();
    }

    private void validate(int v) {
        if (v < 0) throw new IllegalArgumentException("Negative: " + v);
    }
}
```

**Java 8 之前要共享逻辑**：只能用 static 方法放 default helper class 里，或者 abstract class。

## 4. Reactive Streams Flow API（Java 9）

```java
// Publisher → Subscriber 背压模型
Flow.Subscriber<String> subscriber = new Flow.Subscriber<>() {
    private Flow.Subscription subscription;

    @Override
    public void onSubscribe(Flow.Subscription s) {
        this.subscription = s;
        s.request(1);  // 第一次请求 1 条
    }

    @Override
    public void onNext(String item) {
        System.out.println("Got: " + item);
        subscription.request(1);  // 处理完再要下一条
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("Done");
    }
};

// 实际使用一般用 Reactive Streams 库（Reactor / RxJava），不直接用 Flow API
```

**意义**：Java 9 标准化了 reactive streams 接口（`Flow.Publisher` / `Flow.Subscriber` / `Flow.Subscription` / `Flow.Processor`），让不同 reactive 库能互操作。

### 4.1 Reactive Streams Flow 实战拆解

#### ① Flow API 是干嘛的

Java 9 在 `java.util.concurrent.Flow` 里定义了**响应式流（Reactive Streams）的标准接口**——让不同响应式库（Reactor / RxJava / Akka Streams）能互相操作。

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);
    void onNext(T item);
    void onError(Throwable throwable);
    void onComplete();
}

public interface Subscription {
    void request(long n); // 订阅者告诉发布者："我要 n 条"
    void cancel();             // 订阅者随时退订
}
```

#### ② 核心思想：背压（backpressure）

**订阅者说了算**，发布者不能一股脑推：

```
传统 push 模式: Publisher →[1]→[2]→[3]→[4]→ Subscriber（处理慢就爆栈）

背压 pull 模式:
  Publisher = {1,2,3,4,5,...}
  Subscriber: request(1)   ← 先要 1 条
  Publisher  → [1]
  Subscriber 处理完 → request(1)   ← 再要 1 条
  Publisher  → [2]
  ...
```

**为什么需要**：生产者和消费者速度不匹配时，消费者不会被压垮。

#### ③ 4 个回调逐个拆解

**`onSubscribe(Flow.Subscription s)`** —— 订阅成功的回调（**仅 1 次**）

```java
@Override
public void onSubscribe(Flow.Subscription s) {
    this.subscription = s;   // 把 Subscription 存下来
    s.request(1);            // 第一次主动请求 1 条
}
```

- 触发时机：`Publisher.subscribe(subscriber)` 被调用后立即触发
- 必须做的事：要么 `s.request(n)` 要数据，要么 `s.cancel()` 退订
- 不能：在 onSubscribe 里调用 `onNext` / `onComplete`（要等 `request(n)`）
- 必须存 Subscription：`onNext` / `onError` 里要用它继续 `request`

**`onNext(T item)`** —— 每条数据的回调（**0 次或多次**）

```java
@Override
public void onNext(String item) {
    System.out.println("Got: " + item);
    subscription.request(1);   // 处理完一条，再请求下一条
}
```

- 触发时机：每次 Publisher 收到 `request(n)` 后 push 最多 n 条（可能少于 n）
- 关键：处理完必须再 `request(n)`，否则 Publisher 不发 → 死锁

**`onError(Throwable t)`** —— 出错的回调（**terminal，0 或 1 次**）

```java
@Override
public void onError(Throwable throwable) {
    throwable.printStackTrace();
}
```

- 触发时机：Publisher 出任何异常
- terminal：调用后不会再有 `onNext` / `onComplete`

**`onComplete()`** —— 正常结束的回调（**terminal，0 或 1 次**）

```java
@Override
public void onComplete() {
    System.out.println("Done");
}
```

- `onError` 和 `onComplete` 互斥（只能有一个被调用一次）

#### ④ 调用序列图

```
Publisher.subscribe(subscriber)
       │
       ▼
   onSubscribe(s)
       │  s.request(1) ◄── 订阅者主动 pull
       ▼
   onNext(item1) ── 处理完 ─→ s.request(1)
       │
       ▼
   onNext(item2) ── 处理完 ─→ s.request(1)
       │
       ├──→ onComplete()       正常结束
   或  ├──→ onError(ex)        出错
```

#### ⑤ 实战中你几乎不会手写

实际项目里**手写 Flow 是教学用**，生产代码用成熟库：

| 库 | 风格 | 适用 |
|---|---|---|
| **Project Reactor** | `Flux` / `Mono` | Spring 生态首选 |
| **RxJava 3** | `Observable` / `Flowable` / `Single` | 跨平台 |
| **Akka Streams** | `Source` / `Sink` | Actor 模式 |
| **Mutiny** | `Uni` / `Multi` | Quarkus / Vert.x |

**等价 Reactor 代码**：
```java
Flux<String> flux = Flux.create(sink -> {
    sink.next("a");
    sink.next("b");
    sink.next("c");
    sink.complete();
});

flux.subscribe(
    item  -> System.out.println("Got: " + item),  // onNext
    error -> System.out.println(error),              // onError
    () -> System.out.println("Done")           // onComplete
);

// Reactor 提供的操作符（手写 Flow 完全没有）：
Flux.range(1, 1000)
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .flatMap(n -> Mono.just(n).delayElement(Duration.ofMillis(10)))
    .take(10)
    .subscribe(...);
```

#### ⑥ 底层契约（库作者必须遵守）

| 契约 | 含义 |
|---|---|
| **同步性** | 所有 on* 回调在**同一个线程**串行触发 |
| **onSubscribe 单次** | 只调一次 |
| **onNext 多次** | 0 次或多次 |
| **terminal 单次** | onError / onComplete 只调一次 |
| **request 总数 ≥ 推送数** | Publisher 不能推超过 subscriber request 的总数 |
| **cancel 立即生效** | 调用后 Publisher 必须停止推送 |

#### ⑦ 反模式

❌ **onSubscribe 里 `request(Long.MAX_VALUE)`** —— 失去背压意义，等于 push
❌ **onNext 里忘了 `request(n)`** —— Publisher 停发 → 死锁
❌ **onNext 里抛异常** —— Publisher 不知道，自动 onError，但你的逻辑可能不期望
❌ **不用库自己手搓 Flow** —— 操作符 / 调度 / 背压策略全部自己实现
❌ **onNext 里做长任务** —— 阻塞整个响应式管线，改用 `subscribeOn` 切线程

---

**核心洞察**：Flow API 是 Java 9 给"响应式流"立的**底层标准**——4 个回调（onSubscribe / onNext / onError / onComplete）+ Subscription 的 `request(n)` 实现**背压（subscriber 控制节奏）**。生产代码用 Reactor/RxJava，Flow API 是"知道有它即可"的底层契约。

## 5. `var` 局部变量类型推断（Java 10）

```java
// Java 9 之前
String message = "Hello";
ArrayList<String> list = new ArrayList<String>();

// Java 10+
var message = "Hello";       // 推断为 String
var list = new ArrayList<String>();  // 推断为 ArrayList<String>
var stream = list.stream();  // 推断为 Stream<String>
```

### `var` 的限制

- **只能用于局部变量**（含 for-each 变量、try-with-resources 变量）
- **不能用于字段、方法参数、返回类型**
- **必须有初始化器**（不能只写 `var x;`）
- **不能赋值为 null**（无法推断）

```java
// ❌ 不允许
private var name = "Mike";        // 字段不能用
public var getName() { ... }       // 返回类型不能用
void method(var x) { }            // 参数不能用
var x;                              // 没初始化器
var x = null;                       // 推断不出类型

// ✅ 允许
var x = "hello";                    // 局部变量
for (var item : list) { }           // for-each
try (var in = new FileInputStream(p)) { }  // try-with-resources
var y = switch (mode) {             // Java 21+ switch expression
    case "a" -> 1;
    case "b" -> 2;
};
```

### `var` 实战建议

```java
// ✅ 适合：右侧类型显而易见
var users = new ArrayList<User>();
var map = new ConcurrentHashMap<String, List<Order>>();

// ❌ 不适合：右侧类型不清晰（可读性优先）
var result = service.process();    // 读者不知道返回类型
var callback = handler.get();      // ???
```

## 6. HTTP Client（Java 11）

```java
// Java 11 之前：HttpURLConnection（设计古老，难用）
HttpURLConnection con = (HttpURLConnection) url.openConnection();
con.setRequestMethod("GET");
con.setRequestProperty("Accept", "application/json");
int code = con.getResponseCode();
InputStream in = con.getInputStream();
// ...

// Java 11+ HttpClient
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .version(HttpClient.Version.HTTP_2)
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users/123"))
    .header("Accept", "application/json")
    .timeout(Duration.ofSeconds(5))
    .GET()
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

System.out.println(response.statusCode());   // 200
System.out.println(response.body());         // JSON

// 异步
CompletableFuture<HttpResponse<String>> cf = client.sendAsync(request, HttpResponse.BodyHandlers.ofString());
cf.thenAccept(resp -> System.out.println(resp.body()));

// POST JSON
HttpRequest post = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("""
        {"name": "Mike", "age": 30}
        """))
    .build();

HttpResponse<String> resp = client.send(post, HttpResponse.BodyHandlers.ofString());
```

**特性**：
- 支持 HTTP/1.1 和 HTTP/2
- 同步 + 异步（返回 `CompletableFuture`）
- WebSocket 支持（Java 11+ 后续加入）
- BodyHandlers：`ofString` / `ofBytes` / `ofFile` / `ofInputStream` / `ofLines` 等

## 7. 单文件启动（Java 11, JEP 330）

```java
// 文件：Hello.java（不需要 class 定义）
import java.util.List;

public class Hello {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob");
        names.forEach(System.out::println);
    }
}
```

```bash
# Java 10 之前
javac Hello.java
java -cp . Hello

# Java 11+ 直接运行
java Hello.java
```

**意义**：让 Java 学习曲线大幅降低，"第一个 Hello World" 从 2 步变成 1 步。Java 25 的 [[draft-05-java-22-25-modern-concurrency#7-compact-source-files-final|Compact Source Files]] 是这条思路的延伸。

## 8. String 增强（Java 11）

```java
// isBlank / strip / repeat / lines / stripLeading / stripTrailing
"  hello  ".isBlank();          // false
"  hello  ".strip();            // "hello"
"hello\nworld\n".lines().count(); // 2
"=".repeat(5);                  // "====="

// 字符串模板？Java 25 才标准化 (JEP 459 之前是预览)
```

## 9. 关键设计原则

1. **模块化是基础设施**，不是日常 API — 大部分应用不需要自己定义 module
2. **`var` 是便利，不是特性** — 不要为了用 `var` 而用 `var`
3. **`List.of` 替代 `Arrays.asList`** — 不可变更安全
4. **HTTP Client 替代第三方** — OkHttp / Apache HttpClient 不再必需（虽然 OkHttp 仍更强大）
5. **单文件启动用在学习/脚本** — 不要用来替代生产打包

## 10. 反模式

❌ **`var` 滥用** — 公开 API、复杂泛型链用 `var` 反而难读
❌ **强行模块化小型应用** — Spring Boot 应用基本不用 module-info
❌ **HTTP Client 不设置 timeout** — 默认行为可能阻塞永远
❌ **`List.of` 当可变集合用** — 它是不可变的
❌ **`String.strip` vs `String.trim`** — `strip` 是 Unicode-aware，`trim` 只处理 ASCII（建议统一用 `strip`）

## 相关笔记

- **Java 8 基础**: [[draft-01-java-8-foundations]]
- **Java 14-17 数据+模式**: [[draft-03-java-14-17-data-and-patterns]]
- **OkHttp 实战对比**: [[../okhttp-analysis/summary]]
