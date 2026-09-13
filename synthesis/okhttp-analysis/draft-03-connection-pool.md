---
title: "OkHttp 连接池机制"
category: synthesis
tags: [okhttp, connection-pool, tcp, keep-alive, performance]
sources:
  - "OkHttp 4.12.0 源码: ConnectionPool.kt / RealConnectionPool.kt / RealConnection.kt / StreamAllocation.kt"
  - "OkHttp Wiki - ConnectionPool"
summary: "OkHttp 连接池原理：RealConnection 复用、keep-alive 5min、清理线程调度、调优参数"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §03 OkHttp 连接池机制

## 1. 为什么需要连接池

每次 HTTP 请求都要 **TCP 三次握手**（~1 RTT）+ **TLS 握手**（~2 RTT），在公网场景动辄 **100-500ms** 开销。**连接池**让多次请求**复用同一个 TCP 连接**，延迟降到 ~1ms。

```
无连接池：每次请求 100-500ms TCP/TLS 开销
有连接池：第一次 100-500ms，后续 1-10ms
```

**实测收益**：在 4G 网络下，启用 keep-alive 的请求**平均延迟降低 60-80%**。

## 2. ConnectionPool 默认参数

```kotlin
val pool = ConnectionPool(
    maxIdleConnections = 5,        // 空闲连接最多 5 条
    keepAliveDuration = 5,         // 空闲 5 分钟后清理
    timeUnit = TimeUnit.MINUTES
)
```

**默认值**（`OkHttpClient.Builder().build()`）：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `maxIdleConnections` | 5 | 最多保留 5 条空闲连接 |
| `keepAliveDuration` | 5 分钟 | 空闲超过 5 分钟的连接被清理 |
| 清理线程 | 守护线程 | 后台 `cleanup` 循环检查 |

## 3. 核心数据结构

### 3.1 RealConnection（实际连接）

```kotlin
class RealConnection(
    val connectionPool: RealConnectionPool,
    private val route: Route
) : Connection {
    val socket: Socket              // 实际 socket
    val source: BufferedSource      // Okio 输入（响应读）
    val sink: BufferedSink          // Okio 输出（请求写）
    val handshake: Handshake?       // TLS 握手信息
    val protocol: Protocol          // HTTP/1.1 / HTTP_2
    var idleAtNanos: Long = 0L      // 上次进入空闲状态时间
    
    // HTTP/2 多路复用
    val http2Connection: Http2Connection?
    var noNewStreams: Boolean = false
}
```

### 3.2 RealConnectionPool（池实现）

```kotlin
class RealConnectionPool(
    private val taskRunner: TaskRunner,
    val delegate: ConnectionPool      // 用户看到的 facade
) {
    // 核心数据结构
    private val connections: ConcurrentLinkedDeque<RealConnection> = ConcurrentLinkedDeque()
    
    // 清理任务
    private val cleanupTask = object : Task("OkHttp ConnectionPool") {
        override fun runOnce() = cleanup()  // 返回下次清理的延迟
    }
    
    init {
        taskRunner.newQueue().schedule(cleanupTask)
    }
    
    fun acquire(connection: RealConnection, ...): Boolean { ... }
    fun release(connection: RealConnection): Boolean { ... }
    fun cleanup(): Long  // 返回到下次清理的延迟（毫秒）
}
```

### 3.3 关键字段

- **`connections`**：双端队列，**LRU 风格**——新空闲连接加到队头，淘汰从队尾开始
- **`idleAtNanos`**：连接最后一次进入空闲状态的时间戳

## 4. 连接复用流程（核心时序）

```
RealCall.execute()
   │
   ▼
ConnectInterceptor.intercept(chain)
   │
   ▼
StreamAllocation.findConnection()
   │
   ├─ 1. 查 RealConnectionPool 的 connections 队列
   │     └─ 找到匹配的（host/port/TLS/HTTP/2 兼容）？
   │            ├─ 是 → 复用，acquire()
   │            └─ 否 ↓
   │
   ├─ 2. 从 RouteSelector 选新路由
   │
   ├─ 3. DNS 解析（如果需要）
   │
   ├─ 4. TCP 握手（socket.connect）
   │
   ├─ 5. TLS 握手（如果 https）
   │
   ├─ 6. HTTP/2 握手（如果协议允许）
   │
   └─ 7. 新连接入池
         │
         ▼
    chain.proceed() → CallServerInterceptor 实际写读
```

## 5. 复用判定（关键）

**OkHttp 不会复用所有空闲连接**——必须满足以下全部条件：

```kotlin
fun isEligible(address: Address, route: Route): Boolean {
    // 1. 协议匹配（HTTP/2 多路复用：所有 host 共享；HTTP/1.1：必须同 host）
    if (route.address.url.host != address.url.host) return false  // HTTP/1.1 严格
    
    // 2. 端口匹配
    if (route.address.url.port != address.url.port) return false
    
    // 3. DNS 一致（防止 DNS rebinding）
    if (route.address.dns != address.dns) return false
    
    // 4. TLS 一致（证书、cipher suite）
    if (route.address.sslSocketFactory != address.sslSocketFactory) return false
    
    // 5. HTTP/2 多路复用未禁用（noNewStreams）
    if (noNewStreams) return false
    
    // 6. 没有被标记为陈旧（stale）
    if (isStale()) return false
    
    return true
}
```

## 6. 清理算法（核心逻辑）

```kotlin
fun cleanup(): Long {
    var now = System.nanoTime()
    var longestIdleConnection: RealConnection? = null
    
    // 阶段 1：找最久空闲的连接，看是否需要清理
    for (connection in connections) {
        val idleDurationNs = now - connection.idleAtNanos
        if (idleDurationNs > keepAliveDurationNs) {
            // 超时清理
            connections.remove(connection)
            connection.socket().close()
            return 0L  // 立即调度下一次
        }
        if (longestIdleConnection == null || 
            connection.idleAtNanos < longestIdleConnection!!.idleAtNanos) {
            longestIdleConnection = connection
        }
    }
    
    // 阶段 2：达到 maxIdleConnections 吗？
    if (longestIdleConnection != null && connections.size > maxIdleConnections) {
        connections.remove(longestIdleConnection)
        longestIdleConnection.socket().close()
        return 0L
    }
    
    // 阶段 3：返回到下次清理的延迟
    return keepAliveDurationNs - (now - longestIdleConnection!!.idleAtNanos)
}
```

**关键点**：**清理任务调度延迟是动态计算的**——如果最老的连接马上要超时，下一次清理很快；如果都很新，可以 sleep 很久。

## 7. HTTP/1.1 vs HTTP/2 复用策略对比

| 维度 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 复用粒度 | 同 host + port + 路由 | 同一物理连接多 stream |
| 并发请求 | 1 个连接 1 个请求（队头阻塞） | 1 个连接 N 个 stream |
| 是否所有 host 共享 | ❌ 必须同 host | ✅ 同 host 都共享 |
| 是否需要 ConnectionPool | 强烈建议 | 同样需要 |
| 实际并发 | 5 个连接 × 1 请求 = 5 并发 | 1 个连接 × 100 stream = 100 并发 |

**HTTP/2 多路复用**把连接池的"复用"提升到**流级别**——一个连接可以并发处理多个请求/响应，彻底消除队头阻塞。

## 8. 调优实战

### 8.1 高并发场景

```kotlin
// 场景：抓取 1000 个 URL
val client = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(
        maxIdleConnections = 50,        // 提高到 50
        keepAliveDuration = 10,         // 10 分钟
        timeUnit = TimeUnit.MINUTES
    ))
    .dispatcher(Dispatcher().apply {
        maxRequests = 200               // 提高并发上限
        maxRequestsPerHost = 20         // 单 host 提防雪崩
    })
    .build()
```

### 8.2 微服务高 QPS

```kotlin
// 场景：微服务 A 调用微服务 B，每秒 5000 QPS
val client = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(
        maxIdleConnections = 100,       // 提高空闲连接
        keepAliveDuration = 10,
        timeUnit = TimeUnit.MINUTES
    ))
    .build()
```

### 8.3 短连接 / 一次性场景

```kotlin
// 场景：偶尔调一次外部 API
val client = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(0, 1, TimeUnit.SECONDS))  // 立即清理
    .build()
```

## 9. 监控与诊断

```kotlin
// 获取连接池状态（OkHttp 暴露的接口）
val pool = client.connectionPool
val idleCount = pool.idleConnectionCount()       // 空闲连接数
val totalCount = pool.connectionCount()            // 总连接数

// 调试 HTTP/2 流
println("idle=$idleCount, total=$totalCount")
```

⚠️ **OkHttp 不直接暴露每个连接的详细信息**——需要用 `addNetworkInterceptor` 自己记录。

## 10. 常见坑

### 10.1 连接泄漏

```kotlin
// ❌ 错误：Response 没 close
val response = client.newCall(request).execute()
val body = response.body?.string()  // 读取
// 忘了 close → 连接不归还池

// ✅ 正确
client.newCall(request).execute().use { response ->
    println(response.body?.string())
}
```

### 10.2 DNS 不一致

如果 DNS 解析返回不同 IP，连接**不会复用**——这在负载均衡场景需要注意。

### 10.3 协议降级

如果服务端关闭了 HTTP/2 支持，OkHttp 会**自动降级到 HTTP/1.1**，但新建连接——所以 keep-alive 失效。

### 10.4 maxRequestsPerHost 误设

```kotlin
// ❌ maxRequestsPerHost=1：每个 host 只能并发 1 个 → 串行
// ✅ 默认 5 是合理值
```

## 11. 与 HTTP 长连接的关系

**OkHttp 的连接池不是 HTTP Keep-Alive 机制本身**——它**建立在** HTTP Keep-Alive 之上：

```
HTTP Keep-Alive (协议层)
   ↓ 允许
TCP 连接复用 (Socket 层)
   ↓ 由 OkHttp 实现
ConnectionPool (应用层)
```

**OkHttp 自动发 `Connection: keep-alive` 头**（通过 BridgeInterceptor），并在 Response 完成后**不立即关闭 socket**，而是放回池里。

## 12. 何时不该用连接池

| 场景 | 建议 |
|------|------|
| 调用一次外部 API 后销毁 | `ConnectionPool(0, ...)` 立即清理 |
| 极端高并发抓取（10000+） | 自己管理连接池，OkHttp 默认不够 |
| 短连接 P2P | 不适合，用 UDP/QUIC |

## 13. 关键设计原则

1. **复用优先**：能复用就复用，不能复用才新建
2. **自动清理**：超时/超过上限自动释放
3. **协议感知**：HTTP/2 复用粒度更细
4. **DNS 一致性**：防 DNS rebinding 攻击
5. **无侵入**：用户代码完全无感知

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **HTTP/2 细节**: [[draft-04-http2]]
- **已知坑**: [[draft-10-known-issues]]（连接失败的坑）
- **综合入口**: [[summary]]
