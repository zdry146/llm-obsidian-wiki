---
title: "Java 8 基础特性 - 函数式编程基础"
category: synthesis
tags: [java, java-8, lambda, stream, optional, completable-future, java-time]
sources:
  - "JSR 335 - Lambda Expressions for the Java Programming Language"
  - "JSR 310 - Date and Time API"
  - "Java 8 in Action (Raoul-Gabriel Urma, Mario Fusco, Alan Mycroft)"
summary: "Java 8 (2014, LTS) - Lambda / Stream / Optional / CompletableFuture / java.time / default methods"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.92
lifecycle: stable
lifecycle_changed: 2026-10-03
created: 2026-10-03
updated: 2026-10-03
---

# §01 Java 8 基础特性

> **Java 8 (2014-03)** 是 Java 史上最重要的版本之一。它让 Java 从"纯 OO"变成了"OO + 函数式"，并奠定了未来 10 年所有语言演进的基调。

## 1. Lambda 表达式

```java
// Java 7 之前
Comparator<String> byLength = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return Integer.compare(a.length(), b.length());
    }
};

// Java 8+ Lambda
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());

// 方法引用（更简洁）
Comparator<String> byLength = Comparator.comparingInt(String::length);
```

**Lambda 语法**：
- `(parameters) -> expression` — 单表达式，自动 return
- `(parameters) -> { statements; }` — 块体，需显式 return
- `() -> expression` — 无参
- `(Type x, Type y) -> ...` — 可显式标参数类型（多数情况可推断）

**函数式接口**（Functional Interface）：只有一个抽象方法的接口
```java
@FunctionalInterface
interface Transformer<T, R> {
    R apply(T t);
}

Transformer<String, Integer> toLength = s -> s.length();
toLength.apply("hello");  // 5
```

**内置函数式接口**（`java.util.function`）：

| 接口 | 签名 | 用途 |
|---|---|---|
| `Function<T, R>` | `T -> R` | 转换 |
| `Predicate<T>` | `T -> boolean` | 过滤 |
| `Consumer<T>` | `T -> void` | 消费 |
| `Supplier<T>` | `() -> T` | 提供 |
| `Runnable` | `() -> void` | 执行（来自 1.0）|
| `BiFunction<T, U, R>` | `(T, U) -> R` | 二元函数 |
| `UnaryOperator<T>` | `T -> T` | 一元运算（继承 Function）|

## 2. Stream API

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// Java 7 之前
List<String> filtered = new ArrayList<>();
for (String n : names) {
    if (n.length() > 3) {
        filtered.add(n.toUpperCase());
    }
}
Collections.sort(filtered);

// Java 8+ Stream
List<String> filtered = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .toList();
```

**Stream 操作分类**：
- **中间操作**（lazy，返回新 Stream）：`filter` / `map` / `flatMap` / `sorted` / `distinct` / `peek` / `limit` / `skip`
- **终端操作**（eager，触发计算）：`collect` / `forEach` / `reduce` / `count` / `anyMatch` / `allMatch` / `findFirst` / `findAny`

**关键陷阱**：Stream **只能用一次**
```java
Stream<String> s = names.stream();
s.filter(n -> n.length() > 3).count();  // OK
s.map(String::toUpperCase).toList();    // ❌ IllegalStateException
```

## 3. Optional

```java
// Java 7 之前：null 检查地狱
String city = null;
if (user != null) {
    Address addr = user.getAddress();
    if (addr != null) {
        city = addr.getCity();
    }
}

// Java 8+ Optional
Optional<User> optUser = Optional.ofNullable(user);
String city = optUser
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

**Optional 核心方法**：

| 方法 | 行为 |
|---|---|
| `Optional.of(x)` | 包装非空值（x 不能为 null）|
| `Optional.ofNullable(x)` | 包装可能为 null 的值 |
| `Optional.empty()` | 空 Optional |
| `.isPresent() / .isEmpty()` | 是否存在 |
| `.get()` | 取值（**不安全**，建议先检查）|
| `.orElse(default)` | 空时返回 default |
| `.orElseGet(Supplier)` | 空时调用 Supplier |
| `.orElseThrow()` | 空时抛 `NoSuchElementException` |
| `.map(Function)` | 转换内部值 |
| `.flatMap(Function<T, Optional<R>>)` | 同 map，但 Function 返回 Optional |
| `.filter(Predicate)` | 不满足时返回 empty |
| `.ifPresent(Consumer)` | 存在时执行 |

**反模式**：
```java
// ❌ 用 Optional.get() 不先检查
opt.get();

// ❌ Optional 作为字段类型
class User {
    private Optional<String> nickname;  // 不推荐（Serializable 问题 + 语义模糊）
}

// ❌ Optional 作为方法参数
void save(Optional<User> user);  // 用普通 User + null check 更清晰
```

## 4. CompletableFuture

Java 8 引入的异步编程模型，远比 `Future` 强大。

```java
// Future (Java 5) - 只能阻塞获取
ExecutorService exec = Executors.newSingleThreadExecutor();
Future<String> f = exec.submit(() -> "result");
String result = f.get();  // 阻塞

// CompletableFuture - 链式异步
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUser(123), exec)        // 异步取用户
    .thenApply(user -> user.getEmail())             // 链式转换
    .thenCompose(email -> sendEmail(email))         // 链式异步（返回 CF）
    .exceptionally(ex -> "fallback");               // 异常处理

// 多个并行 + 合并
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> fetchUser(123));
CompletableFuture<List<Post>> postsF = CompletableFuture.supplyAsync(() -> fetchPosts(123));

CompletableFuture.allOf(userF, postsF).thenRun(() -> {
    User u = userF.join();
    List<Post> p = postsF.join();
    // ...
});
```

**核心方法**：
- 静态工厂：`supplyAsync` / `runAsync`
- 链式：`thenApply` / `thenAccept` / `thenRun` / `thenCompose` / `whenComplete`
- 异常：`exceptionally` / `handle`
- 组合：`allOf` / `anyOf`
- 手动：`completedFuture` / `failedFuture`

**对比 virtual threads（Java 21）**：
- `CompletableFuture`：适合 **CPU-bound + 大量任务编排**（functional combinator）
- virtual threads：适合 **I/O-bound + 阻塞调用**（替代线程池 + Thread-per-request）

## 5. java.time API（JSR-310）

```java
// Java 7 之前：Date/Calendar 噩梦
Date d = new Date();
Calendar c = Calendar.getInstance();
c.set(2025, Calendar.DECEMBER, 25);  // 月份从 0 开始
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
String s = sdf.format(d);  // 非线程安全！

// Java 8+ java.time
LocalDate today = LocalDate.now();                       // 2025-12-25
LocalTime now = LocalTime.now();                         // 14:30:00
LocalDateTime dt = LocalDateTime.of(2025, 12, 25, 14, 30);
ZonedDateTime zoned = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
Instant instant = Instant.now();                          // UTC 时间戳

// 不可变，线程安全
String formatted = DateTimeFormatter.ISO_LOCAL_DATE.format(today);
// "2025-12-25"

// 链式计算
LocalDate nextWeek = today.plus(1, ChronoUnit.WEEKS);
LocalDate firstDayOfMonth = today.withDayOfMonth(1);

// Duration / Period
Duration d24 = Duration.ofHours(24);
Period p10y = Period.ofYears(10);
```

**核心类**：
- `LocalDate` / `LocalTime` / `LocalDateTime` — 无时区
- `ZonedDateTime` / `OffsetDateTime` — 带时区
- `Instant` — UTC 时间戳（机器友好）
- `Duration` — 时间间隔（秒/纳秒）
- `Period` — 日期间隔（年/月/日）
- `DateTimeFormatter` — 替代 SimpleDateFormat（线程安全）

## 6. default methods / 接口演化

```java
interface List<E> {
    // 抽象方法
    boolean add(E e);

    // Java 8+ default 方法（有实现）
    default void sort(Comparator<? super E> c) {
        Collections.sort(this, c);
    }
}
```

**作用**：让接口可以**添加新方法而不破坏现有实现**（接口演化）。`Collection` / `List` / `Iterable` 在 Java 8 加了一堆 default 方法（`stream()` / `forEach()` / `removeIf()` 等）。

## 7. 关键设计原则

1. **Lambda 不是替代匿名内部类** — Lambda 仅适用于函数式接口；多方法接口仍用匿名类
2. **Stream 适合集合转换，不适合控制流** — 不要在 Stream 里搞 try/catch 副作用
3. **Optional 是返回值类型，不是字段类型** — 序列化、反射等场景都不友好
4. **CompletableFuture 不强制异步** — `thenApply` 默认在调用线程执行；用 `thenApplyAsync` 才异步
5. **java.time 不可变** — 所有 plus/with 方法返回新对象，不修改原值（线程安全）

## 8. 反模式

❌ **在 Lambda 里修改外部变量** — Lambda 内部只能访问 effectively final 变量
❌ **Stream 链过长** — 超过 5-7 个操作就该拆方法
❌ **Stream 处理 I/O 副作用** — 用 `forEach` 改外部集合反而更慢
❌ **`Optional` 当 sentinel** — `Optional.empty()` 是合法值，不是错误
❌ **`CompletableFuture.get()` 阻塞主线程** — 破坏了异步模型
❌ **`SimpleDateFormat` 继续用** — 非线程安全，Java 8 已弃用

## 相关笔记

- **Java 9-11 现代化**: [[draft-02-java-9-11-modernization]]
- **Java 21 LTS**: [[draft-04-java-21-lts-virtual-threads]]
- **TypeScript 函数式**: [[../typescript-analysis/draft-02-functions]]
- **Kotlin 函数式**: [[../kotlin-analysis/draft-02-functions]]
