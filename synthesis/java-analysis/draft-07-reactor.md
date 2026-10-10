---
title: "Project Reactor - 响应式编程实战"
category: synthesis
tags: [java, reactor, reactive, mono, flux, spring-webflux, r2dbc, reactive-streams]
sources:
  - "Project Reactor Reference Guide (https://projectreactor.io/docs)"
  - "Reactive Streams Specification (https://www.reactive-streams.org/)"
  - "Spring WebFlux Documentation"
  - "Reactor 3.7+ Release Notes"
summary: "Project Reactor - Mono / Flux / 操作符 / 调度 / 背压 / Context / StepVerifier / 与 Java Flow + virtual threads 集成 / Spring 生态实战"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-10-10
created: 2026-10-10
updated: 2026-10-10
---

# §07 Project Reactor — 响应式编程实战

> Project Reactor 是 Spring 团队（Pivotal → VMware → Broadcom）开发的**响应式库**，实现了 Reactive Streams 规范，是 Spring WebFlux / R2DBC / Spring Cloud Gateway 的底层。Reactor 3.7+ (2024) 适配 Java 21 虚拟线程。

## 1. Reactor 是什么

### 定位

- **实现** Reactive Streams 规范（基于 [[draft-02-java-9-11-modernization#4-reactive-streams-flow-api-的底层是-jdk-9-的-flow-api\|Java 9 Flow API]]）
- **构建** Spring 生态响应式栈（WebFlux / R2DBC / Cloud Gateway / Security Reactive）
- **对手**：RxJava 3 / Akka Streams / Mutiny / Vert.x

### 历史

| 版本 | 时间 | 关键变化 |
|---|---|---|
| Reactor 1.x | 2014-2016 | 初版（基于 RxJava 思路）|
| Reactor 2.x | 2016 | 第一次响应 Spring 5 / WebFlux |
| Reactor 3.x | 2017-至今 | 当前主分支，3.7+ 适配 Java 21+ |
| Reactor 3.7 | 2024-12 | [[draft-04-java-21-lts-virtual-threads\|virtual threads]] 适配、`Schedulers.boundedElastic()` 重写 |

### 生态

```
Spring WebFlux (HTTP server/client)
  └─ Reactor Netty (底层 NIO)
R2DBC (响应式数据库连接)
  └─ 各数据库驱动 (PostgreSQL / MySQL / H2 / Oracle)
Spring Cloud Gateway
  └─ Reactor Netty
Spring Security Reactive
RSocket
  └─ Reactor
```

## 2. Mono 与 Flux（两个核心类型）

```java
// Mono - 0 或 1 个元素（单值/单事件）
Mono<String> mono = Mono.just("hello");
Mono<String> empty = Mono.empty();
Mono<String> never = Mono.never();        // 永不发射
Mono<User> fromCallable = Mono.fromCallable(() -> fetchUser(123));

// Flux - 0 到 N 个元素（流）
Flux<Integer> range = Flux.range(1, 100);
Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));  // 每秒发射
Flux<String> merge = Flux.merge(mono, fromList);              // 合并多个流

// 两种类型都是 Publisher<T>（实现 Reactive Streams）
Publisher<String> pub = mono;  // ✅ 多态
```

**对照表**：

| 维度 | Mono | Flux |
|---|---|---|
| 元素数 | 0 或 1 | 0 到 N |
| 完成信号 | 1 次（onComplete）| 1 次（onComplete）|
| 类比 | `Optional<T>` / `Future<T>` | `Stream<T>` |
| 适用 | 单值查询、HTTP 请求、单条事件 | 列表查询、事件流、批量处理 |

## 3. 创建流（6 种基本方式）

```java
// ① just - 直接发射值
Mono.just("hello");
Flux.just("a", "b", "c");

// ② empty / never / error
Mono.empty();              // 立即完成，0 个元素
Mono.never();              // 永不发射（用于测试 / 占位）
Mono.error(new RuntimeException("oops"));

// ③ fromIterable / fromArray / fromStream
Flux.fromIterable(List.of(1, 2, 3));
Flux.fromArray(new String[]{"a", "b"});
Flux.fromStream(Stream.of(1, 2, 3));

// ④ fromCallable / fromRunnable / fromFuture
Mono.fromCallable(() -> heavyCompute());           // 同步阻塞 → 异步发射
Mono.fromRunnable(() -> doCleanup());             // 同上，但 runnable
Mono.fromFuture(executor.submit(() -> "result"));  // 包装 Future

// ⑤ create / push（编程式发射，适合复杂源）
Flux.create(sink -> {
    // 异步源（如 WebSocket）每次有数据
    websocket.onMessage(msg -> sink.next(msg));
    websocket.onClose(() -> sink.complete());
    sink.onCancel(() -> websocket.close());
});

// ⑥ defer - 延迟创建（每次订阅都重新执行）
Flux<Integer> defer = Flux.defer(() -> Flux.range(1, random.nextInt(10)));
// 每次 .subscribe() 都生成不同长度的流
```

## 4. 核心操作符（12 个常用）

### 转换类

```java
// map - 一对一转换
Flux.just("a", "b").map(String::toUpperCase).subscribe();   // A, B

// flatMap - 一对多（异步映射）
Flux.just(1, 2, 3)
    .flatMap(n -> Mono.just(n * n))         // 1→1, 2→4, 3→9
    .subscribe();

// flatMapSequential - 保持顺序的 flatMap
Flux.just(1, 2, 3)
    .flatMapSequential(n -> Mono.just(n).delayElement(Duration.ofMillis(n * 100)))
    .subscribe();
// 顺序发射（flatMap 是并发无序的）

// switchMap - 只保留最新
Flux.just(1, 2, 3)
    .switchMap(n -> searchAPI(n))          // 后一个取消前一个
    .subscribe();
```

### 过滤类

```java
// filter / take / skip / distinct
Flux.range(1, 100)
    .filter(n -> n % 2 == 0)              // 过滤
    .take(5)                              // 取前 5
    .skip(2)                              // 跳前 2
    .distinct()                           // 去重
    .subscribe();
```

### 组合类

```java
// merge / concat / zip / combineLatest
Flux.merge(flux1, flux2, flux3);                    // 并发合并（任意顺序）
Flux.concat(flux1, flux2, flux3);                   // 顺序合并（前一个完成流）
Flux.zip(flux1, flux2, (a, b) -> a + b);            // 配对合并（一对一）
Flux.combineLatest(flux1, flux2, (a, b) -> a + b);  // 最新值合并（任意）
```

### 聚合类

```java
// collectList / collectMap / reduce / count
Flux.range(1, 100)
    .collectList()                                  // → Mono<List<Integer>>
    .subscribe();

Flux.just("a", "b", "c")
    .reduce("", (acc, val) -> acc + val)            // → Mono<String> ("abc")

Flux.range(1, 100)
    .count()                                        // → Mono<Long> (100)
    .subscribe();
```

## 5. 错误处理（4 种核心方法）

```java
// ① onErrorReturn - 出错时返回默认值
Mono.just("hello")
    .map(this::mightFail)
    .onErrorReturn("fallback")                       // 失败 → 返回 "fallback"
    .subscribe();

// ② onErrorResume - 出错时切换到另一个 publisher
Mono.just("hello")
    .map(this::mightFail)
    .onErrorResume(ex -> {
        log.warn("Failed, using cache", ex);
        return cacheMono;                            // 失败 → 切到缓存
    })
    .subscribe();

// ③ retry - 失败时重试（可指定次数 / 条件）
Mono.just("hello")
    .map(this::mightFail)
    .retry(3)                                       // 失败重试 3 次
    .subscribe();

Mono.just("hello")
    .map(this::mightFail)
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))  // 指数退避
    .subscribe();

// ④ onErrorContinue - 跳过错误继续处理（Flux only）
Flux.range(1, 10)
    .map(n -> {
        if (n == 5) throw new RuntimeException("skip me");
        return n * 2;
    })
    .onErrorContinue((ex, n) -> log.warn("skip {}", n))  // 跳过 n=5 继续
    .subscribe();                                      // 2,4,6,8,10,12,14,16,18,20（跳过 10）
```

**选择决策**：

| 场景 | 用法 |
|---|---|
| 简单 fallback | `onErrorReturn(default)` |
| 切到备份源 | `onErrorResume(backup)` |
| 重试网络请求 | `retryWhen(Retry.backoff(...))` |
| 跳过坏数据继续处理 | `onErrorContinue(...)` |

## 6. 调度与线程（3 种 Scheduler）

```java
// Schedulers 4 种内置实现
Schedulers.parallel();         // CPU-bound（适合计算）
Schedulers.boundedElastic();   // 阻塞 I/O（替代 thread pool，自动扩容）
Schedulers.single();           // 单线程顺序执行
Schedulers.immediate();        // 当前线程（默认）

// subscribeOn - 控制订阅线程
Mono.just("hello")
    .subscribeOn(Schedulers.boundedElastic())      // 订阅在 elastic
    .subscribe();

// publishOn - 控制下游线程切换
Mono.just("hello")
    .map(this::blockingCall)                       // 跑在订阅线程
    .publishOn(Schedulers.parallel())              // 下游切到 parallel
    .map(this::cpuBound)                           // 跑在 parallel
    .subscribe();

// 实战：HTTP 客户端 + 数据库（混用 parallel + elastic）
webClient.get().uri("/api/users/123")
    .retrieve().bodyToMono(User.class)              // Netty 线程
    .publishOn(Schedulers.boundedElastic())         // 切到弹性线程池
    .flatMap(user -> r2dbc.insert(user))            // DB I/O
    .subscribe();
```

**Reactor 3.7+ 重要变化**：虚拟线程适配后 `boundedElastic()` 重写，更智能处理阻塞任务。

## 7. 背压策略（5 种 onBackpressureXxx）

```java
// ① buffer - 缓冲（默认，OOM 风险）
Flux.interval(Duration.ofMillis(1))
    .onBackpressureBuffer()        // 缓冲所有元素（可能 OOM）
    .subscribe();

// ② drop - 丢弃新元素（适合监控）
Flux.interval(Duration.ofMillis(1))
    .onBackpressureDrop()          // 订阅者慢就丢新元素
    .subscribe();

// ③ latest - 保留最新
Flux.interval(Duration.ofMillis(1))
    .onBackpressureLatest()        // 只保留最新元素
    .subscribe();

// ④ error - 报错
Flux.interval(Duration.ofMillis(1))
    .onBackpressureError()         // 队列满时抛异常
    .subscribe();

// ⑤ limitRate - 批量请求（控制节奏）
Flux.range(1, 1_000_000)
    .limitRate(100)                // 每次只请求 100 个
    .subscribe();
```

**实战选择**：

| 场景 | 策略 | 原因 |
|---|---|---|
| 用户输入流 | `onBackpressureBuffer` | 不能丢 |
| 监控指标 | `onBackpressureDrop` | 老数据不重要 |
| 实时价格 | `onBackpressureLatest` | 只看最新 |
| 关键事件 | `onBackpressureError` | 不能丢也不能慢 |

## 8. 上下文传递（Reactor Context vs ScopedValue）

Reactor Context 是**不可变的键值对**，沿反应链向下传递，**子 publisher 可见，父 publisher 不可见**。

```java
// 写入
Mono.just("hello")
    .contextWrite(ctx -> ctx.put("userId", "123"))
    .flatMap(value -> 
        Mono.deferContextual(ctx -> {
            String userId = ctx.get("userId");        // "123"
            return process(value, userId);
        })
    )
    .subscribe();

// 嵌套订阅也可见
Mono.just("hello")
    .contextWrite(ctx -> ctx.put("traceId", "abc"))
    .flatMap(value -> Mono.just(value)
        .flatMap(v -> Mono.deferContextual(ctx -> {
            String tid = ctx.get("traceId");          // "abc"
            return Mono.just(v + ":" + tid);
        }))
    )
    .subscribe();                                      // "hello:abc"
```

**vs Scoped Value**（Java 25）：

| 维度 | Reactor Context | ScopedValue |
|---|---|---|
| 引入版本 | Reactor 1.0 (2014) | Java 25 (2025) |
| 类型 | `Context`（不可变 Map）| `ScopedValue<T>`（类型安全单值）|
| 多值 | ✅ Context 可存 N 个键值 | 单值（多个 ScopedValue）|
| 传递方向 | 下游 | 下游（与 fork 自动绑定）|
| 跨线程 | ✅（自动传递）| ❌（需显式 where）|
| 性能 | 每次 write 创建新 Context | O(1) 单实例 |

**迁移建议**：虚拟线程环境用 [[draft-05-java-22-25-modern-concurrency#11-scoped-values-final\|Scoped Value]]，响应式环境继续用 Reactor Context。

## 9. 测试（StepVerifier）

```java
@Test
void testFlux() {
    Flux<Integer> flux = Flux.just(1, 2, 3)
        .map(n -> n * 2);
    
    StepVerifier.create(flux)
        .expectNext(2)
        .expectNext(4)
        .expectNext(6)
        .expectComplete()
        .verify();
}

@Test
void testWithError() {
    Mono<String> mono = Mono.error(new RuntimeException("oops"));
    
    StepVerifier.create(mono)
        .expectErrorMatches(ex -> ex.getMessage().equals("oops"))
        .verify();
}

@Test
void testVirtualTime() {
    Flux<Long> interval = Flux.interval(Duration.ofSeconds(1))
        .take(3);
    
    StepVerifier.withVirtualTime(() -> interval)
        .expectSubscription()
        .expectNoEvent(Duration.ofSeconds(1))
        .expectNext(0L)
        .thenAwait(Duration.ofSeconds(2))
        .expectNextCount(2)
        .expectComplete()
        .verify();
}

@Test
void testWithTestPublisher() {
    TestPublisher<Integer> publisher = TestPublisher.create();
    
    StepVerifier.create(publisher.flux())
        .then(() -> publisher.emit(1, 2, 3))
        .expectNext(1, 2, 3)
        .expectComplete()
        .verify();
}
```

## 10. 实战：HTTP 客户端 + WebFlux

### WebClient（非阻塞 HTTP 客户端）

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.USER_AGENT, "my-app/1.0")
    .build();

Mono<User> user = client.get()
    .uri("/users/{id}", 123)
    .retrieve()
    .bodyToMono(User.class);

// 链式组合
Mono<UserPosts> userPosts = user.flatMap(u ->
    client.get().uri("/users/{id}/posts", u.id())
        .retrieve()
        .bodyToFlux(Post.class)
        .collectList()
        .map(p -> new UserPosts(u, p))
);

// 错误处理
userPosts
    .timeout(Duration.ofSeconds(5))
    .onErrorResume(WebClientResponseException.class, ex -> {
        log.warn("HTTP {} for user", ex.getStatusCode());
        return Mono.just(UserPosts.empty());
    })
    .subscribe();
```

### Spring WebFlux Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserRepository repo;
    
    @GetMapping("/{id}")
    public Mono<User> getById(@PathVariable Long id) {
        return repo.findById(id);
    }
    
    @GetMapping
    public Flux<User> list() {
        return repo.findAll();
    }
    
    @PostMapping
    public Mono<User> create(@RequestBody Mono<User> user) {
        return repo.save(user);
    }
}
```

### R2DBC（响应式数据库）

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByName(String name);
    
    @Query("SELECT * FROM users WHERE age > :minAge")
    Flux<User> findByMinAge(int minAge);
}

@Service
public class UserService {
    private final UserRepository repo;
    
    public Mono<User> createUser(User user) {
        return repo.save(user)
            .timeout(Duration.ofSeconds(2))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));
    }
}
```

## 11. 与 Java Flow / virtual threads 集成

### Reactor ↔ Flow API 互转

```java
// Reactor → Flow.Publisher（自然就是）
Flow.Publisher<String> flowPub = monoOrFlux;

// Flow.Publisher → Reactor
Flux.from(flowPublisher);
Mono.from(flowPublisher);
```

### Reactor 在虚拟线程环境（Reactor 3.7+）

```java
// Reactor 3.7+ 在 JDK 21+ 会自动用虚拟线程
// boundedElastic() 重写为基于虚拟线程

// 显式桥接
Mono.fromCallable(() -> blockingIO())
    .subscribeOn(Schedulers.boundedElastic())      // 内部用虚拟线程
    .publishOn(Schedulers.parallel())              // 切换回并行线程池
    .subscribe();
```

### 不要混用的模式

```java
// ❌ 在 Reactor 链中阻塞
Flux.range(1, 100)
    .map(n -> {
        Thread.sleep(1000);                    // 阻塞整个 Reactor 链
        return n;
    })
    .subscribe();

// ✅ 改用 Mono.fromCallable + Schedulers.boundedElastic
Flux.range(1, 100)
    .flatMap(n -> Mono.fromCallable(() -> {
            Thread.sleep(1000);
            return n;
        })
        .subscribeOn(Schedulers.boundedElastic()))
    .subscribe();
```

## 12. 关键设计原则

1. **Mono vs Flux 选择** — 一个值用 Mono（HTTP 请求/查询），多个值用 Flux（列表/流）
2. **背压必须考虑** — 默认 `BUFFER` 可能 OOM，热数据用 `DROP` / `LATEST`
3. **调度切换要明确** — `subscribeOn` 决定订阅源线程，`publishOn` 切换下游线程
4. **错误处理不要 silent** — `onErrorContinue` 慎用，会丢数据
5. **测试用 StepVerifier** — 不要 mock subscribe，verify 触发链
6. **不要阻塞 Reactor 链** — 阻塞操作放 `boundedElastic`（Java 21+ 虚拟线程）

## 13. 反模式

❌ **`.subscribe()` 在业务代码** —— 应该 return Mono/Flux，让框架订阅（WebFlux/R2DBC）
❌ **`.block()` 阻塞等待** —— 虚拟线程场景下可以用，普通线程场景丢失异步优势
❌ **背压默认 `BUFFER`** —— 热数据流必须显式指定（`DROP`/`LATEST`/`ERROR`）
❌ **onErrorContinue 静默吞错** —— 关键数据处理跳过错误会导致数据丢失
❌ **共享可变状态 in operators** —— `Flux.just(sharedList)` 多个订阅者会互相干扰
❌ **混用 Schedulers 不记录** —— 复杂链里切了 N 次线程，stack trace 看不出在哪里切
❌ **不用 `Mono.error`/`Flux.error` 而抛 RuntimeException** —— 异常链中断，无法 retry/onErrorResume
❌ **`.log()` 留在生产代码** —— 只用于开发/调试

## 14. 与其他响应式库对比

| 维度 | Reactor | RxJava 3 | Mutiny | Akka Streams |
|---|---|---|---|---|
| Reactive Streams 实现 | ✅ | ✅ | ❌（自有规范）| ✅ |
| 类型 | Mono / Flux | Single / Maybe / Flowable / Observable | Uni / Multi | Source / Sink / Flow |
| 上下文传递 | Context | 不内置 | Context | 不内置 |
| Java 21 虚拟线程 | ✅ 3.7+ | 部分 | ✅ | 部分 |
| Spring 生态 | ✅ WebFlux/R2DBC | ❌ | Quarkus | Akka |
| 学习曲线 | 中 | 中 | 低 | 高（Actor）|
| 适合 | Spring 服务 | 跨平台 | Quarkus | Actor 系统 |

## 15. 相关笔记

- **Flow API 底层**: [[draft-02-java-9-11-modernization#4-reactive-streams-flow-api]]
- **Flow API 实战拆解**: [[draft-02-java-9-11-modernization#41-reactive-streams-flow-实战拆解]]
- **Virtual Threads (Reactor 3.7+ 适配)**: [[draft-04-java-21-lts-virtual-threads]]
- **Structured Concurrency (Reactor 对比)**: [[draft-05-java-22-25-modern-concurrency#9-structured-concurrency-jep-505-java-25-preview-5]]
- **Scoped Values (Reactor Context 替代)**: [[draft-05-java-22-25-modern-concurrency#11-scoped-values-jep-506-java-25-final]]
- **OkHttp (非 Reactor HTTP 客户端)**: [[../okhttp-analysis/summary]]
- **gRPC (响应式 server)**: [[../grpc-analysis/summary]]
- **TypeScript Promise/async (类似模型)**: [[../typescript-analysis/draft-02-functions]]
- **Kotlin Coroutines (类似模型)**: [[../kotlin-analysis/draft-03-coroutines]]