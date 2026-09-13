---
title: "Kotlin 协程 (Structured Concurrency)"
category: synthesis
tags: [kotlin, coroutines, suspend, dispatcher, flow, structured-concurrency]
sources:
  - "Kotlin Coroutines Guide (kotlinlang.org/docs/coroutines-guide.html)"
  - "Structured Concurrency (developer.android.com)"
  - "Roman Elizarov - Coroutines Series"
summary: "Kotlin 协程核心：suspend、CoroutineScope、Dispatcher、structured concurrency、Flow、Channel"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §03 Kotlin 协程（Structured Concurrency）

## 1. 为什么需要协程？

**线程 vs 协程**：

| 维度 | 线程 (Thread) | 协程 (Coroutine) |
|------|---------------|-----------------|
| **抽象层级** | OS 内核 | 用户态 |
| **创建成本** | 高（~1MB stack） | 低（~100 bytes） |
| **切换成本** | 高（~100ns） | 低（~10ns） |
| **并发数** | 几百 | 几万 |
| **编程模型** | 回调 / Future | suspend / await |
| **结构化并发** | ❌ 需手动 join | ✅ 自动 |

**核心洞察**：协程 = **轻量级线程 + 编译期挂起**——结构化并发让异步代码像同步代码一样写。

## 2. 最简示例

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  // 阻塞主线程直到协程完成
    launch {                 // 启动协程
        delay(1000L)         // 非阻塞挂起（不是 Thread.sleep！）
        println("World!")
    }
    println("Hello,")
}
// 输出：
// Hello,
// World!   （1 秒后）
```

**对比 Thread**：
```kotlin
// 同样效果，但 1000 个 thread 会 OOM
Thread {
    Thread.sleep(1000L)       // 阻塞线程（不释放 CPU）
    println("World!")
}.start()
println("Hello,")
```

## 3. suspend 函数（核心）

```kotlin
suspend fun fetchUser(id: String): User {
    delay(1000L)               // 挂起（不阻塞线程）
    return api.getUser(id)
}

// suspend 函数只能在协程或另一个 suspend 函数中调用
fun main() = runBlocking {
    val user = fetchUser("123")   // ✅
}

// ❌ 不能在普通函数中调用
fun notSuspend() {
    fetchUser("123")              // ❌ 编译错误
}
```

**关键点**：
- `suspend` 是编译器关键字
- `delay()` 不阻塞线程，而是挂起协程（释放 CPU）
- 线程可以被其他协程复用

## 4. 协程作用域（CoroutineScope）

```kotlin
// 三种作用域
GlobalScope.launch {          // 整个应用生命周期（慎用）
    // ...
}

runBlocking {                  // 阻塞当前线程（main/test 用）
    // ...
}

CoroutineScope(Dispatchers.IO).launch {  // 自定义作用域
    // ...
}

// Android 专用
lifecycleScope.launch {       // Activity/Fragment 生命周期
    // 自动取消
}

viewModelScope.launch {       // ViewModel 生命周期
    // 自动取消
}
```

## 5. Dispatcher（线程调度）

```kotlin
// 4 种内置 Dispatcher
Dispatchers.Main              // UI 线程（Android/Swing）
Dispatchers.IO                // IO 线程（网络、文件、DB）
Dispatchers.Default           // CPU 密集（默认）
Dispatchers.Unconfined         // 不切换（当前线程）

// 切换线程
suspend fun loadUser(): User = withContext(Dispatchers.IO) {
    val data = httpClient.fetchData()      // IO 线程
    // 自动切回原线程（Main）
    parseUser(data)                        // 主线程
}
```

**实战**：
```kotlin
// 网络请求（IO 线程）
suspend fun fetchFromApi(): Response = withContext(Dispatchers.IO) {
    client.newCall(request).execute()
}

// 数据库（IO 线程）
suspend fun queryDb(): List<User> = withContext(Dispatchers.IO) {
    db.query("SELECT * FROM users")
}

// 解析（Default 线程）
suspend fun parseResponse(data: String): User = withContext(Dispatchers.Default) {
    Json.decodeFromString<User>(data)
}
```

## 6. Structured Concurrency（结构化并发，核心）

```kotlin
// 父协程取消，所有子协程自动取消
fun main() = runBlocking {
    val parent = launch {
        launch {
            while (true) {
                delay(1000)
                println("child 1")
            }
        }
        launch {
            while (true) {
                delay(1000)
                println("child 2")
            }
        }
    }
    delay(3000)
    parent.cancel()              // 取消父 → 所有子自动取消
}
```

**vs Java Future / ExecutorService**：
```java
// Java：忘调用 .get() → 资源泄漏
ExecutorService.submit(task);
// task 可能在后台一直跑

// Kotlin：作用域结束 → 所有子自动取消
coroutineScope {
    launch { task1() }
    launch { task2() }
}  // 所有 task 自动 join 或 cancel
```

**核心洞察**：**结构化并发让协程生命周期可预测**——不会泄漏，不会孤儿。

## 7. Job 与生命周期

```kotlin
val job = launch {
    try {
        doWork()
    } catch (e: CancellationException) {
        println("Cancelled")
        throw e
    } finally {
        // 清理资源
        closeResource()
    }
}

delay(500)
job.cancel("User navigated away")  // 取消协程
job.join()                         // 等待完成
```

**取消传播**：
- 父协程取消 → 子协程自动取消
- 子协程异常 → 父协程取消（默认 SupervisorJob 除外）

## 8. 并发执行（async/await）

```kotlin
suspend fun fetchDashboard(): Dashboard = coroutineScope {
    val userDeferred = async { fetchUser() }
    val ordersDeferred = async { fetchOrders() }
    
    // await 不会阻塞线程，挂起协程
    Dashboard(
        user = userDeferred.await(),
        orders = ordersDeferred.await()
    )
}

// 并行：fetchUser 和 fetchOrders 同时执行
// 串行时间：user + orders
// 并发时间：max(user, orders)
```

**核心洞察**：`async/await` 是协程版的 `CompletableFuture`——但**结构化并发**让它更安全。

## 9. Flow（冷流，异步数据流）

```kotlin
// 生产者
fun fetchItems(): Flow<Item> = flow {
    for (i in 1..100) {
        delay(100)                  // 模拟异步数据
        emit(Item(i))               // 发射数据
    }
}

// 消费者
fun main() = runBlocking {
    fetchItems()
        .filter { it.isActive }
        .map { it.toDto() }
        .take(10)
        .collect { println(it) }
}
```

**Flow vs List/Channel**：
| 维度 | Flow | List | Channel |
|------|------|------|---------|
| 异步 | ✅ | ❌ | ✅ |
| 冷/热 | 冷（按需） | - | 热 |
| 背压 | ❌ | - | ❌ |
| 取消 | ✅ | - | ✅ |
| 生命周期 | 协程 | - | 手动 |

## 10. Channel（热流，协程通信）

```kotlin
// 生产者-消费者模式
val channel = Channel<String>()

// 生产者
launch {
    for (i in 1..100) {
        channel.send("item $i")
    }
    channel.close()
}

// 消费者
launch {
    for (item in channel) {
        println("Received: $item")
    }
}
```

**Channel 类型**：
- `Channel()` = `RendezvousChannel`（无缓冲）
- `Channel(capacity)` = `ArrayChannel`（缓冲）
- `Channel.UNLIMITED` = 无限制
- `Channel.CONFLATED` = 只保留最新

## 11. SharedFlow / StateFlow（热共享）

```kotlin
// StateFlow：当前状态
val counter = MutableStateFlow(0)
launch {
    counter.value = 1   // 更新
}
launch {
    counter.collect { println(it) }   // 收到最新值
}

// SharedFlow：事件流
val events = MutableSharedFlow<Event>()
launch {
    events.emit(Event.Click)
}
launch {
    events.collect { println(it) }   // 收到每次 emit
}
```

**StateFlow vs SharedFlow**：
| 维度 | StateFlow | SharedFlow |
|------|-----------|------------|
| 初值 | 必需 | 无 |
| 去重 | ✅（相同值不重发） | ❌ |
| 适用 | UI 状态 | 事件流 |

## 12. 协程异常处理

```kotlin
// try/catch
suspend fun fetch() {
    try {
        api.call()
    } catch (e: IOException) {
        log.error("Network error", e)
        throw e
    }
}

// CoroutineExceptionHandler
val handler = CoroutineExceptionHandler { _, e ->
    log.error("Coroutine failed", e)
}

val scope = CoroutineScope(Dispatchers.IO + handler)
scope.launch {
    // 异常会被 handler 捕获
}

// SupervisorJob（隔离失败）
val supervisor = SupervisorJob()
val scope = CoroutineScope(Dispatchers.IO + supervisor)

scope.launch { throw RuntimeException("oops") }   // 失败，但不影响其他子
scope.launch { /* 继续跑 */ }                   // 不受影响
```

## 13. 与 OkHttp / gRPC 的整合

### 13.1 OkHttp + Kotlin 协程

```kotlin
// OkHttp 5.x 原生支持
val response: Response = client.newCall(request).await()  // suspend

// 4.x 需要 wrap
suspend fun Call.await(): Response = suspendCancellableCoroutine { cont ->
    enqueue(object : Callback {
        override fun onResponse(call: Call, response: Response) {
            cont.resume(response)
        }
        override fun onFailure(call: Call, e: IOException) {
            cont.resumeWithException(e)
        }
    })
    cont.invokeOnCancellation {
        cancel()  // 协程取消 → 自动 cancel call
    }
}
```

### 13.2 gRPC + Kotlin 协程

```kotlin
// grpc-kotlin
implementation("io.grpc:grpc-kotlin-stub:1.4.1")

val stub = GreeterCoroutineGrpc.newStub(channel)
val reply = stub.sayHello(HelloRequest.newBuilder().setName("mike").build())
// 自动 suspend

// Server streaming
stub.lotsOfReplies(request).collect { reply ->
    println(reply.message)
}
```

## 14. 实战模式

### 14.1 带超时的网络请求

```kotlin
suspend fun fetchWithTimeout(url: String): String =
    withTimeout(5000) {        // 5 秒超时
        client.fetch(url)
    }

// 超时后抛 TimeoutCancellationException
// 通过 try/catch 处理
```

### 14.2 重试模式

```kotlin
suspend fun <T> retryWithBackoff(
    maxAttempts: Int = 3,
    block: suspend () -> T
): T {
    repeat(maxAttempts - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            if (attempt == maxAttempts - 1) throw e
            delay(2.0.pow(attempt).toLong() * 1000)
        }
    }
    return block()
}
```

### 14.3 并行请求

```kotlin
suspend fun fetchAllData(): DashboardData = coroutineScope {
    val userDeferred = async(Dispatchers.IO) { fetchUser() }
    val ordersDeferred = async(Dispatchers.IO) { fetchOrders() }
    val productsDeferred = async(Dispatchers.IO) { fetchProducts() }
    
    DashboardData(
        user = userDeferred.await(),
        orders = ordersDeferred.await(),
        products = productsDeferred.await()
    )
    // 3 个请求并行，总时间 = max(单个时间)
}

### 14.4 长轮询（指数退避）

```kotlin
suspend fun longPoll(
    initialDelay: Long = 1_000,
    maxDelay: Long = 30_000
): Update = coroutineScope {
    var delayMs = initialDelay
    while (isActive) {                          // 协程取消时自动跳出
        try {
            val update = api.fetchUpdate()       // 阻塞型调用
            delayMs = initialDelay                // 成功后重置退避
            emit(update)                          // 消费者收集
        } catch (e: CancellationException) {
            throw e                               // 保留取消语义
        } catch (e: Exception) {
            delay(delayMs)                        // 出错后等
            delayMs = (delayMs * 2).coerceAtMost(maxDelay)  // 指数退避
        }
    }
}
```

### 14.5 防抖/节流（搜索框自动补全）

```kotlin
// 200ms 防抖：用户连续输入只触发最后一次
fun <T> Flow<T>.debounce(delayMs: Long): Flow<T> = flow {
    var last: T? = null
    coroutineScope {
        launch {
            collect { last = it }
        }
        delay(delayMs)
        last?.let { emit(it) }
    }
}

// 使用：搜索框
searchQuery
    .debounce(200)
    .map { query -> fetchSuggestions(query) }
    .collect { suggestions -> updateUI(suggestions) }
```

### 14.6 生产者-消费者模式（Channel）

```kotlin
// 数据采集场景：后台生产 + UI 线程消费
val channel = Channel<Int>(Channel.BUFFERED)

// 生产者协程
val producer = launch(Dispatchers.IO) {
    for (i in 1..1000) {
        val data = fetchSensorData(i)
        channel.send(data)
        delay(100)                               // 模拟周期
    }
    channel.close()                               // 发完关闭
}

// 消费者协程
val consumer = launch(Dispatchers.Main) {
    for (data in channel) {
        updateUI(data)                            // 逐条更新
    }
}

// 两个独立生命周期，consumer 会在 channel.close() 后自动结束
```

### 14.7 Data Streaming（Flow + WebSocket/SSE）

```kotlin
// 从 WebSocket 读股票行情
fun stockStream(symbol: String): Flow<StockTick> = channelFlow {
    val ws = OkHttpClient.newWebSocket(
        Request.Builder().url("wss://stream.example.com/$symbol").build(),
        object : WebSocketListener() {
            override fun onMessage(ws: WebSocket, text: String) {
                val tick = Json.decodeFromString<StockTick>(text)
                trySend(tick)                       // 送入 Flow
            }
            override fun onClosed(ws: WebSocket, code: Int, reason: String) {
                close()                              // 关闭 Flow
            }
            override fun onFailure(ws: WebSocket, t: Throwable, r: Response?) {
                close(t)                             // 异常关闭
            }
        }
    )
    awaitClose { ws.cancel(1000, "Consumer gone") }  // 资源清理
}

// 消费者：取最近 100 条计算平均价
stockStream("AAPL")
    .filter { it.price > 100 }
    .map { it.price }
    .take(100)
    .average()
    .collect { avg -> displayAvg(avg) }
```

### 14.8 gRPC + Coroutines 实战

```kotlin
// 依赖
implementation("io.grpc:grpc-kotlin-stub:1.4.1")

class GreeterClient(channel: ManagedChannel) {
    private val stub = GreeterCoroutineGrpc.newStub(channel)

    // Unary
    suspend fun sayHello(name: String): String =
        stub.sayHello(HelloRequest.newBuilder().setName(name).build()).message

    // Server streaming
    fun streamReplies(name: String): Flow<String> = flow {
        stub.lotsOfReplies(
            HelloRequest.newBuilder().setName(name).build()
        ).collect { reply ->
            emit(reply.message)
        }
    }
}

// 使用
scope.launch {
    val msg = client.sayHello("Mike")
    println(msg)

    client.streamReplies("Alice").collect { msg ->
        println("Got: $msg")
    }
}
```

### 14.9 协程取消与清理（资源管理）

```kotlin
class ConnectionManager {
    private val scope = CoroutineScope(
        SupervisorJob() + Dispatchers.IO
    )
    private var connection: Connection? = null

    fun start() {
        scope.launch {
            connection = openConnection()
            try {
                // 业务逻辑...
                while (isActive) {
                    keepAlive()
                    delay(30_000)
                }
            } finally {
                // ✅ 协程取消/异常时都会执行
                connection?.close()
                connection = null
            }
        }
    }

    fun stop() {
        scope.cancel()              // 取消所有子协程
        scope.coroutineContext[Job]?.join()   // 等清理完成
    }
}

// Android 上：onDestroy 调 stop()
// JVM 后台服务：JVM shutdown hook 调 stop()
```

### 14.10 协程异常处理（完整模式）

```kotlin
// 模式 1：try/catch 在业务层
suspend fun fetchUser() {
    try {
        api.getUser()
    } catch (e: IOException) {
        log.error("网络错误", e)
        throw UserServiceException("获取用户失败", e)
    } catch (e: HttpException) {
        when (e.code) {
            404 -> throw UserNotFoundException()
            401 -> throw AuthRequiredException()
            else -> throw UserServiceException("HTTP ${e.code}", e)
        }
    }
}

// 模式 2：CoroutineExceptionHandler（全局兜底）
val handler = CoroutineExceptionHandler { _, e ->
    log.error("Coroutine failed", e)
    metrics.recordFailure(e)
}

val scope = CoroutineScope(
    SupervisorJob() + Dispatchers.IO + handler
)

// 模式 3：supervisorScope（隔离失败）
suspend fun loadDashboard() = supervisorScope {
    val userDeferred = async { fetchUser() }
    val ordersDeferred = async { fetchOrders() }
    // 一个失败不影响另一个
    Dashboard(
        user = userDeferred.await(),
        orders = ordersDeferred.await()
    )
}
```

### 14.11 测试（kotlinx-coroutines-test）

```kotlin
import kotlinx.coroutines.test.*

@Test
fun `fetchUser 返回数据`() = runTest {
    // virtual time：delay() 不真等
    val user = fetchUser()
    assertEquals("Mike", user.name)
    // 测试耗时 ~10ms（不需真等 5s）
}

@Test
fun `timeout 后抛异常`() = runTest {
    assertFailsWith<TimeoutCancellationException> {
        withTimeout(100) {                        // 虚拟 100ms
            delay(5000)                           // 虚拟 5s（会超时）
        }
    }
}

@Test
fun `测试 Flow`() = runTest {
    val flow = flow {
        emit(1)
        delay(100)
        emit(2)
    }

    val results = flow.toList()
    assertEquals(listOf(1, 2), results)
}

@Test
fun `测试 ViewModel`() = runTest {
    val viewModel = MyViewModel()
    viewModel.uiState.test {
        // 收集 Flow，验证每个状态
        assertEquals(Loading, awaitItem())
        assertEquals(Success(data), awaitItem())
    }
}
```

## 15. 与 Netty 整合

```kotlin
// Netty + Kotlin
class NettyHandler : ChannelInboundHandlerAdapter() {
    override fun channelRead(ctx: ChannelHandlerContext, msg: Any) {
        // 在 event loop 中，不能阻塞
        // 用 coroutine 调度
        GlobalScope.launch(Dispatchers.IO) {
            val result = process(msg)
            ctx.writeAndFlush(result)
        }
    }
}

// 实际推荐：ChannelHandlerAdapter + CoroutineScope 注入
```

## 16. 关键设计原则

1. **suspend 优于回调**——代码像同步
2. **structured concurrency**——生命周期可预测
3. **withContext 切换线程**——不阻塞
4. **Flow 异步数据流**——替代 callback stream
5. **Channel/SharedFlow**——协程间通信
6. **SupervisorJob**——隔离失败

## 17. 反模式

❌ **GlobalScope.launch**——生命周期泄漏
❌ **runBlocking 在主线程**——ANR (Android)
❌ **CoroutineScope(Dispatchers.IO).launch** 在业务代码——scoping 混乱
❌ **async { } 不 await**——异常吞没
❌ **delay(1000) 替代 sleep**——runBlocking 内 OK，独立线程不行

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **DSL**: [[draft-04-dsl-builders]]
- **Java 互操作**: [[draft-08-java-interop]]
- **最佳实践**: [[draft-09-best-practices]]
- **OkHttp 整合**: [[okhttp-analysis/summary]]
- **gRPC 整合**: [[grpc-analysis/summary]]
- **综合入口**: [[summary]]