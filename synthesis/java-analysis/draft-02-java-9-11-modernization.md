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
base_confidence: 0.87
lifecycle: draft
lifecycle_changed: 2026-10-03
created: 2026-10-03
updated: 2026-10-03
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
