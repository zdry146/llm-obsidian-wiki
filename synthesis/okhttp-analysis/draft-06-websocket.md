---
title: "OkHttp WebSocket 支持"
category: synthesis
tags: [okhttp, websocket, real-websocket, frames, ping-pong]
sources:
  - "OkHttp 4.12.0 源码: RealWebSocket.kt / WebSocketReader.kt / WebSocketWriter.kt"
  - "RFC 6455 - The WebSocket Protocol"
summary: "OkHttp WebSocket 客户端：握手 / 帧格式 / 消息收发 / 心跳 / 优雅关闭"
provenance:
  extracted: 0.88
  inferred: 0.10
  ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-12
created: 2026-09-12
updated: 2026-09-12
---

# §06 OkHttp WebSocket 支持

## 1. WebSocket 是什么（10 秒）

**WebSocket** 是 HTML5 起的**全双工协议**（RFC 6455），让客户端和服务端保持**长连接**，双向实时通信：

```
HTTP:     客户端 ──请求──> 服务端
          客户端 <──响应── 服务端        (短连接，每次断开)

WebSocket: 客户端 ════ 持久连接 ═════ 服务端
            ↑ 双向实时消息 ↓
```

**典型场景**：聊天、股票行情、协同编辑、实时通知、游戏。

## 2. OkHttp 的 WebSocket 定位

⚠️ **OkHttp 只支持 WebSocket 客户端**——它**不**是 WebSocket 服务端。

| 角色 | OkHttp |
|------|--------|
| WebSocket 客户端 | ✅ |
| WebSocket 服务端 | ❌（用 Netty / Jetty / [[mu-server-2.4.2-analysis/summary\|mu-server]]） |

## 3. 最简示例

```kotlin
val request = Request.Builder()
    .url("wss://echo.websocket.org")
    .build()

val listener = object : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: Response) {
        println("Connected")
        webSocket.send("Hello, WebSocket!")
    }
    
    override fun onMessage(webSocket: WebSocket, text: String) {
        println("Received: $text")
    }
    
    override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
        println("Received binary: ${bytes.hex()}")
    }
    
    override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
        webSocket.close(code, reason)
    }
    
    override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
        println("Closed: $code $reason")
    }
    
    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        println("Error: ${t.message}")
    }
}

val webSocket = client.newWebSocket(request, listener)
```

## 4. 握手协议（RFC 6455）

WebSocket 始于 **HTTP Upgrade 请求**：

```
客户端 → GET /chat HTTP/1.1
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
         Sec-WebSocket-Version: 13

服务端 ← HTTP/1.1 101 Switching Protocols
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Sec-WebSocket-Accept 算法**（防误连）：
```
Accept = Base64( SHA1( Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
```

## 5. 帧格式

WebSocket 数据传输单位是 **帧（Frame）**：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - + - - - - - - - - - - - - - - - +
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### 5.1 Opcode

| Opcode | 含义 | OkHttp 方法 |
|--------|------|-------------|
| 0x0 | Continuation | `send(...)` 自动处理 |
| 0x1 | Text | `send(String)` |
| 0x2 | Binary | `send(ByteString)` |
| 0x8 | Close | `close(code, reason)` |
| 0x9 | Ping | 自动（心跳） |
| 0xA | Pong | 自动响应 |

## 6. 关键代码：RealWebSocket

```kotlin
class RealWebSocket(...) : WebSocket {
    private val listener: WebSocketListener
    
    // 写入队列（任意线程 send）
    private val queue = LinkedBlockingQueue<Frame>()
    
    // 单个 writer 线程
    private var writerTask: Task? = null
    private var readerTask: Task? = null
    
    // 心跳
    private var pingIntervalNanos: Long = 0L  // 默认 0 = 不发心跳
    
    // 状态
    private var receivedCloseCode: Int = -1
    private var receivedCloseReason: String? = null
    private var sentCloseCode: Int = -1
    private var failed: Boolean = false
    
    override fun send(text: String): Boolean = send(text.toByteString(), Opcode.TEXT)
    override fun send(bytes: ByteString): Boolean = send(bytes, Opcode.BINARY)
    
    private fun send(payload: ByteString, opcode: Opcode): Boolean {
        // 必须 NOT_CLOSED
        if (failed || receivedCloseCode != -1) return false
        
        // 入队
        queue.put(Frame(opcode, payload))
        runWriter()  // 唤醒 writer 线程
        return true
    }
    
    override fun close(code: Int, reason: String?): Boolean {
        // ... 发送 Close 帧
    }
    
    override fun cancel() { /* 强制关闭，不发 Close 帧 */ }
}
```

## 7. 内部线程模型

OkHttp WebSocket 内部用 **2 个独立线程**：

```
应用线程 ──send(text)──> LinkedBlockingQueue<Frame>
                              ↓
                         WriterThread (单线程)
                              ↓
                         Socket OutputStream

Socket InputStream ──> ReaderThread (单线程)
                              ↓
                         WebSocketListener (回调应用线程)
```

**关键约束**：
- **Writer 是单线程**——`send()` 必须串行（OkHttp 内部用 queue 保证）
- **Reader 是单线程**——所有 callback 都在 reader 线程上
- ⚠️ **不要在 callback 里做耗时操作**——会阻塞 reader

## 8. 心跳机制

```kotlin
val client = OkHttpClient.Builder()
    .pingInterval(30, TimeUnit.SECONDS)  // 每 30 秒发 Ping
    .build()
```

**OkHttp 自动处理**：
1. 应用 pingInterval 后，writer 周期发 Ping 帧
2. 服务端回应 Pong
3. 超时未收到 Pong → `onFailure(WebSocket, IOException("Ping failed..."), null)`

⚠️ **Ping 超时**是 `okTimeout`（默认 10 秒）——非 pingInterval。

## 9. 优雅关闭 vs 强制关闭

```kotlin
// 优雅关闭（推荐）：发 Close 帧，等对端确认
webSocket.close(1000, "Normal closure")
// 1000 是 Close Code: NORMAL_CLOSURE

// 强制关闭（异常情况）：直接断开 TCP
webSocket.cancel()
// 不发 Close 帧，对端会收到 IOException
```

### 9.1 Close Code 列表（标准）

| Code | 含义 |
|------|------|
| 1000 | NORMAL_CLOSURE |
| 1001 | GOING_AWAY |
| 1002 | PROTOCOL_ERROR |
| 1003 | UNSUPPORTED_DATA |
| 1008 | POLICY_VIOLATION |
| 1009 | TOO_BIG_TO_PROCESS |
| 1011 | SERVER_ERROR |
| 4000-4999 | 应用自定义 |

⚠️ **不能使用** 1004（保留）、1005（保留）、1006（保留）、1015（保留）。

## 10. 消息分片

WebSocket 支持**大消息分片**发送：

```
[FIN=0, opcode=1, payload="hello "]
[FIN=0, opcode=0, payload="world"]
[FIN=1, opcode=0, payload="!"]  ← 最后一片 FIN=1
```

**OkHttp 4.x 简化**：**应用层不感知分片**——`send(text)` 一次性发完，内部按需分片。

⚠️ **接收端不感知分片**——`onMessage` 只在 `FIN=1` 时触发。

## 11. 性能与限制

| 维度 | 数据 |
|------|------|
| 最大 frame 大小 | 2^63 - 1（理论），实际受 socket buffer 限制 |
| 单消息限制 | OkHttp 默认 16 KB ping payload |
| 并发 send | 不安全（必须外部加锁） |
| 跨线程 send | ✅ 安全（queue + writer thread） |

## 12. 实战：Reconnect 模式

```kotlin
class ReconnectingWebSocket(
    private val client: OkHttpClient,
    private val request: Request,
    private val listener: WebSocketListener,
    private val maxRetries: Int = Int.MAX_VALUE,
    private val scheduler: Scheduler = HandlerScheduler(Handler(Looper.getMainLooper()))
) {
    private enum class State { CONNECTING, CONNECTED, RECONNECTING, CLOSED }
    
    private val lock = Any()
    private var state: State = State.CLOSED
    private var webSocket: WebSocket? = null
    private var attempt: Int = 0
    private var pendingReconnect: Cancellable? = null   // 所有权标记
    
    fun connect() {
        val currentWs = synchronized(lock) {
            when (state) {
                State.CLOSED, State.RECONNECTING -> {
                    state = State.CONNECTING
                    null  // 继续往下创建
                }
                State.CONNECTING, State.CONNECTED -> {
                    return  // 已存在连接，不重复
                }
            }
        } ?: createWebSocket()
        
        if (currentWs != null) {
            // 主路径已创建，后续赋值在 listener 里
        }
    }
    
    private fun createWebSocket() {
        val newWs = client.newWebSocket(request, object : WebSocketListener() {
            override fun onOpen(ws: WebSocket, response: Response) {
                synchronized(lock) {
                    state = State.CONNECTED
                    webSocket = ws
                    attempt = 0  // 重置重试计数
                }
                listener.onOpen(ws, response)
            }
            
            override fun onMessage(ws: WebSocket, text: String) = listener.onMessage(ws, text)
            override fun onMessage(ws: WebSocket, bytes: ByteString) = listener.onMessage(ws, bytes)
            
            override fun onClosing(ws: WebSocket, code: Int, reason: String) {
                ws.close(code, reason)  // 响应对方的 Close 帧
                listener.onClosing(ws, code, reason)
            }
            
            override fun onClosed(ws: WebSocket, code: Int, reason: String) {
                val shouldReconnect = synchronized(lock) {
                    if (state == State.CLOSED) {
                        false  // 主动 close，不重连
                    } else {
                        state = State.RECONNECTING
                        true
                    }
                }
                listener.onClosed(ws, code, reason)
                if (shouldReconnect) scheduleReconnect()
            }
            
            override fun onFailure(ws: WebSocket, t: Throwable, response: Response?) {
                val shouldReconnect = synchronized(lock) {
                    if (state == State.CLOSED) {
                        false  // 主动 close，不重连
                    } else {
                        state = State.RECONNECTING
                        webSocket = null
                        true
                    }
                }
                listener.onFailure(ws, t, response)
                if (shouldReconnect) scheduleReconnect()
            }
        })
        // 不在创建后立即赋值——要等 onOpen 才认为 CONNETED
    }
    
    private fun scheduleReconnect() {
        val currentAttempt = synchronized(lock) {
            if (state != State.RECONNECTING) return  // 已取消
            attempt++
            if (attempt > maxRetries) {
                state = State.CLOSED
                return  // 超过最大重试
            }
            attempt
        }
        
        val delay = backoff(currentAttempt)
        pendingReconnect = scheduler.schedule(delay) {
            synchronized(lock) {
                pendingReconnect = null
                if (state == State.CLOSED) return@schedule
                state = State.CONNECTING
            }
            createWebSocket()
        }
    }
    
    fun send(text: String): Boolean = synchronized(lock) {
        webSocket?.send(text) ?: false
    }
    
    fun close(code: Int = 1000, reason: String? = null) {
        val toCancel = synchronized(lock) {
            state = State.CLOSED
            webSocket.also { webSocket = null }
        }
        pendingReconnect?.cancel()
        pendingReconnect = null
        toCancel?.close(code, reason)
    }
    
    private fun backoff(attempt: Int): Long {
        // 指数退避 + 抖动：100ms, 200ms, 400ms, ... 上限 30 秒 ± 20%
        val base = (100L * (1L shl (attempt - 1))).coerceAtMost(30_000L)
        val jitter = (base * 0.2 * Random.nextDouble()).toLong()
        return base + jitter
    }
}

// 抽象调度器——避免 Handler 泄漏 Activity
interface Scheduler {
    fun schedule(delayMs: Long, block: () -> Unit): Cancellable
}

interface Cancellable {
    fun cancel()
}

class HandlerScheduler(private val handler: Handler) : Scheduler {
    override fun schedule(delayMs: Long, block: () -> Unit): Cancellable {
        val runnable = Runnable { block() }
        handler.postDelayed(runnable, delayMs)
        return object : Cancellable {
            override fun cancel() = handler.removeCallbacks(runnable)
        }
    }
}
```

## 12.1 修复的 7 个 bug

| # | 旧版 bug | 严重性 | 新版修复 |
|---|---------|---------|---------|
| 1 | **`shouldReconnect = true` 是 mutable Boolean**——多线程 `close()` + `onFailure()` 同时读写可能 race，导致「主动 close 后仍重连」 | 🔴 | 改为枚举状态机 + `synchronized(lock)` |
| 2 | **没有「主动关闭 vs 网络失败」区分**——服务端发 1000 NORMAL_CLOSURE 也会死循环重连 | 🔴 | `state == CLOSED` 时所有回调都不重连 |
| 3 | **`onFailure` + `onClosed` 都会触发重连**——某些场景下 close 也会调 onFailure，导致 **双重重连** | 🔴 | 状态机保证：同一 state 只走一次重连 |
| 4 | **`Handler(Looper.getMainLooper()).postDelayed(...)`** 持 Activity/Fragment 引用——关闭未调 close() → Handler 回调 leak → Activity 内存泄漏 | 🔴 | 抽象 `Scheduler` + `Cancellable`，生产环境传 `applicationContext` 的 Handler 或 Coroutine 调度器 |
| 5 | **`backoff()` 是注释占位符**——用户不知道具体算法 | 🟡 | 完整实现（指数 + 抖动 + 上限 30s） |
| 6 | **`onClosing` 里调 `ws.close()` → 触发 `onClosed` → 又 connect()**——死循环风险 | 🟡 | 状态机 `CONNECTED` → `RECONNECTING` 在 `onClosed` 一次性转换 |
| 7 | **`connect()` 多次调用会创建多个 WebSocket 实例**——旧的不关闭，连接/Handler 泄漏 | 🟡 | 状态检查：`CONNECTING`/`CONNECTED` 状态调 connect() 直接 return |

## 12.2 状态机图

```
                  connect()
   CLOSED ─────────────────────► CONNECTING
     ▲                              │
     │                              │ onOpen
     │                              ▼
     │                          CONNECTED
     │                              │
     │          onFailure/onClosed  │
     │              (state != CLOSED)
     │                              ▼
     │                          RECONNECTING
     │                              │
     │                              │ delay 过了
     │                              ▼
     │                          CONNECTING (重试)
     │
     │ close()
     └─────────────────────────────── (任意状态)
                                       │
                                       ▼
                                    CLOSED
```

## 12.3 主动关闭 vs 网络失败的区分

```
代码层判断：state == CLOSED → 主动关闭

实际原因：
- 服务端发 1000 NORMAL_CLOSURE → onClosed → onClosed() 查 state，发现仍 CONNECTED → 重连
- 客户端调 ws.close(1000) → 触发 onClosed → state 被设为 CLOSED → onClosed() 查 state，不重连
- 网络断开 → onFailure → state 不是 CLOSED → 重连
- 服务端 crash → onFailure → state 不是 CLOSED → 重连
```

## 12.4 避免 Handler 内存泄漏

```kotlin
// ❌ 旧版：Handler 持 Activity 引用 → 泄漏
Handler(Looper.getMainLooper()).postDelayed(...)

// ✅ 新版：注入 Scheduler，生产环境用 applicationContext
class ReconnectingWebSocket(
    ...
    private val scheduler: Scheduler = HandlerScheduler(
        Handler(context.applicationContext.mainLooper)  // ← application 而非 Activity
    )
)

// ✅✅ 协程版（推荐）
class ReconnectingWebSocket(
    private val scope: CoroutineScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
) {
    private var reconnectJob: Job? = null
    
    private fun scheduleReconnect() {
        reconnectJob?.cancel()
        reconnectJob = scope.launch {
            delay(backoff(attempt))
            if (isActive) connect()
        }
    }
    
    fun close() {
        reconnectJob?.cancel()
        // ...
    }
}
```

## 12.5 使用示例

```kotlin
// 生产用法
val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
val ws = ReconnectingWebSocket(
    client = okHttpClient,
    request = Request.Builder().url("wss://api.example.com/events").build(),
    listener = myListener,
    maxRetries = 10,                          // 最多重试 10 次
    scheduler = coroutineScheduler(scope)      // 协程调度，避免 Handler 泄漏
)

// Activity onStart
ws.connect()

// Activity onStop
ws.close(code = 1000, reason = "User navigated away")

// 监听回调
myListener = object : WebSocketListener() {
    override fun onOpen(...) { ... }
    override fun onMessage(ws, text) { ... }
    override fun onFailure(ws, t, response) {
        // 网络断开时也能收到，但 ReconnectingWebSocket 会自动重连
        // 这里通常只 log，不做 alert
    }
}
```

## 13. 与 SSE 的对比

| 维度 | WebSocket | SSE (Server-Sent Events) |
|------|-----------|--------------------------|
| 方向 | 双向 | 单向（服务端 → 客户端） |
| 协议 | WebSocket (Upgrade) | HTTP 长连接 |
| 自动重连 | ❌（需自己实现） | ✅（EventSource） |
| OkHttp 支持 | ✅ 客户端 | ✅ 客户端（4.x） |
| 适用 | 聊天、协作 | 通知、行情推送 |

## 14. 调试技巧

```kotlin
// 启用详细日志
val client = OkHttpClient.Builder()
    .addNetworkInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
    .pingInterval(10, TimeUnit.SECONDS)
    .build()

// 监控帧流量
class FrameStatsListener : WebSocketListener() {
    override fun onMessage(ws: WebSocket, text: String) {
        Metrics.counter("ws.message.received").inc()
    }
}
```

## 15. 已知坑

### 15.1 OnMessage 阻塞

```kotlin
// ❌ 错误：在 onMessage 里做耗时操作
override fun onMessage(ws: WebSocket, text: String) {
    Thread.sleep(1000)  // 阻塞 reader
    doHeavyWork(text)
}

// ✅ 正确：丢到其他线程
override fun onMessage(ws: WebSocket, text: String) {
    executor.submit { doHeavyWork(text) }
}
```

### 15.2 并发 send

```kotlin
// ❌ 不安全
thread1 { ws.send("a") }
thread2 { ws.send("b") }  // 帧交错可能有问题

// ✅ 安全
synchronized(ws) {
    ws.send("a")
    ws.send("b")
}
```

⚠️ 实际上 OkHttp 内部 queue 保证原子性——但**帧的相对顺序**在不同 send 调用之间不一定符合直觉。

### 15.3 大消息

```kotlin
// ❌ 1 GB 字符串
ws.send(hugeString)  // 内存峰值 1 GB

// ✅ 流式分块
for (chunk in chunks) {
    ws.send(chunk)
}
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器链**: [[draft-02-interceptors]]
- **综合入口**: [[summary]]
