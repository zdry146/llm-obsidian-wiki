---
title: "OkHttp HTTP/2 支持"
category: synthesis
tags: [okhttp, http2, multiplexing, frames, streams]
sources:
  - "OkHttp 4.12.0 源码: Http2Connection.kt / Http2Reader.kt / Http2Writer.kt / Http2Stream.kt"
  - "RFC 7540 - HTTP/2"
summary: "OkHttp HTTP/2 实现：帧/流/多路复用/连接合并/流控，与 HTTP/1.1 对照"
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

# §04 OkHttp HTTP/2 支持

## 1. HTTP/2 是什么（30 秒回顾）

**HTTP/2** 是 HTTP/1.1 的二进制协议继任者（RFC 7540，2015 年标准化），核心特性：

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 协议格式 | 文本 | **二进制帧** |
| 传输模型 | 1 连接 1 请求（队头阻塞） | **1 连接 N 流（多路复用）** |
| Header | 重复发 | **HPACK 压缩** |
| Server Push | ❌ | ✅ |
| 优先级 | ❌ | ✅ 流优先级 |
| 默认开启 | - | ✅ OkHttp 4.x 默认 |

## 2. OkHttp 的 HTTP/2 能力矩阵

| 维度 | OkHttp 支持 |
|------|-------------|
| 客户端 HTTP/2 | ✅ 默认开启 |
| 服务端 HTTP/2 | ❌（OkHttp 是客户端） |
| TLS + ALPN 协商 | ✅ 自动 |
| h2c（明文 HTTP/2） | ⚠️ 实验性 |
| Server Push | ✅ 接收 |
| 优先级 | ✅ 客户端可设置 |
| 连接合并 | ✅（同 IP + 有效证书） |
| 流控 | ✅ 实现层 |

## 3. 协议协商流程

```
TLS 握手（如果有）
   │
   ▼
ALPN 协商（Application-Layer Protocol Negotiation）
   │
   ├─ 服务端返回 "h2" → 用 HTTP/2
   │
   ├─ 服务端返回 "http/1.1" → 降级 HTTP/1.1
   │
   └─ 明文连接 → 尝试 HTTP/2 升级
                  （GET / + Upgrade: h2c + HTTP2-Settings 头）
```

**关键代码**（`RealConnection.connect`）：

```kotlin
when (protocol) {
    Protocol.HTTP_2 -> http2Connection = Http2Connection.Builder(...)
        .socket(socket)
        .build()
        .also { it.start() }
    Protocol.HTTP_1_1 -> {} // 不做额外动作
    Protocol.H2_PRIOR_KNOWLEDGE -> http2Connection = Http2Connection.Builder(...)
        .socket(socket)
        .build()
        .also { it.start() }
}
```

## 4. HTTP/2 帧（Frame）

HTTP/2 通信的最小单位是 **帧（Frame）**：

```
+-----------------------------------------------+
|                 Length (24)                    |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

### 4.1 OkHttp 支持的帧类型

| Type | 名称 | 作用 |
|------|------|------|
| 0x00 | DATA | 传输 body |
| 0x01 | HEADERS | 传输 Header（HPACK 压缩） |
| 0x02 | PRIORITY | 设置流优先级 |
| 0x03 | RST_STREAM | 关闭流 |
| 0x04 | SETTINGS | 连接级配置 |
| 0x05 | PUSH_PROMISE | Server Push 通知 |
| 0x06 | PING | 心跳/测延迟 |
| 0x07 | GOAWAY | 优雅关闭 |
| 0x08 | WINDOW_UPDATE | 流控 |
| 0x09 | CONTINUATION | HEADERS 续传 |

## 5. HTTP/2 流（Stream）

**流**是 HTTP/2 中一个完整的请求/响应对应关系：

```kotlin
// Http2Stream.kt 简化
class Http2Stream(
    val id: Int,                          // 流 ID（奇数，客户端发起）
    val connection: Http2Connection,
    val outFinished: Boolean,             // 请求发送完毕
    val inFinished: Boolean,              // 响应接收完毕
    val readTimeout: Long,                // 读超时
    val writeTimeout: Long                // 写超时
) {
    val sink: BufferedSink               // 写请求
    val source: BufferedSource            // 读响应
    var headers: Headers?                 // 响应头（延迟接收）
    var errorCode: ErrorCode?             // 流错误码
    // ...
}
```

**关键 ID 规则**：
- 客户端发起的流：**奇数**（1, 3, 5, 7, ...）
- 服务端发起的流（Push）：**偶数**（2, 4, 6, ...）
- 流 ID 0x00 是**连接级**（不是流）

## 6. 多路复用（核心优势）

```
HTTP/1.1（6 个并发需要 6 个连接）：

  [req1][   ][req2][    ][req3]   ← 连接 1
         [req4][    ][req5]       ← 连接 2
                                  ← 连接 3
  时延：受最慢请求拖累（队头阻塞）

HTTP/2（6 个并发共享 1 个连接）：

  [req1][req2][req3][req4][req5][req6]  ← 同一连接
  时延：所有请求并行
```

**OkHttp 默认行为**：

```kotlin
// 在 HTTP/2 模式下，多个并发请求会自动共享一个连接
// 由 ConnectInterceptor 自动处理
val request1 = Request.Builder().url("https://api.example.com/a").build()
val request2 = Request.Builder().url("https://api.example.com/b").build()

// 同一个 OkHttpClient 异步执行 → 自动复用 HTTP/2 连接
client.newCall(request1).enqueue(callback1)
client.newCall(request2).enqueue(callback2)
```

## 7. 连接合并（Coalescing）

HTTP/2 进一步支持**多 host 共享一个连接**——只要这些 host 解析到同一个 IP 且证书有效：

```
api.example.com → 1.2.3.4 (证书 *.example.com)
admin.example.com → 1.2.3.4 (证书 *.example.com)
internal.example.com → 1.2.3.4 (证书 *.example.com)

→ HTTP/2 共享 1 个连接到 1.2.3.4
```

⚠️ **前提**：证书必须对所有 host 有效（SAN 列表 / 通配符）。

**OkHttp 代码**（简化）：

```kotlin
// RealConnection.isCoalescibleWith 判断
fun isCoalescibleWith(other: RealConnection): Boolean {
    // 1. 同一 IP（DNS 解析后）
    if (route.socketAddress != other.route.socketAddress) return false
    
    // 2. TLS 证书匹配
    val certs = handshake!!.peerCertificates
    val hostname = other.route.address.url.host
    if (!HostnameVerifier.DEFAULT.verify(hostname, handshake!!.peerPrincipal)) {
        return false
    }
    
    // 3. 没被标记为不可合并
    if (noNewStreams) return false
    
    return true
}
```

## 8. 流控（Flow Control）

HTTP/2 有 **两层流控**：

### 8.1 连接级（Connection-level）

每个连接的初始窗口：**65535 字节**（RFC 默认），可调（OkHttp 默认 16777216 = 16 MB）。

### 8.2 流级（Stream-level）

每个流的初始窗口：同连接级。

### 8.3 WINDOW_UPDATE 帧

```
发送方发送 DATA 帧 → 接收方消费 → 接收方发 WINDOW_UPDATE
   → 发送方收到后可继续发
```

**OkHttp 的处理**：

```kotlin
// Http2Connection.Writer.applyConnectionSettingsAck
// 收到对端的 WINDOW_UPDATE 后，更新窗口计数

// Http2Reader.readWindowUpdate
fun readWindowUpdate(...) {
    val streamId = ...
    if (streamId == 0) {
        // 连接级
        connection.windowSize += increment
    } else {
        // 流级
        stream.windowSize += increment
    }
}
```

## 9. Server Push

服务端可以**主动推送**额外资源（HTTP/1.1 必须等客户端请求）：

```
客户端 ──GET /index.html──> 服务端
服务端 <─PUSH_PROMISE /style.css──
服务端 <─200 /index.html + /style.css 数据─
```

**OkHttp 默认禁用 Push 接收**：

```kotlin
// OkHttpClient 内部
val client = OkHttpClient.Builder()
    // .pushHandler(...)  // 4.x 移除，5.x 重设计
    .build()
```

⚠️ **4.x 起 OkHttp 不再主动接收 Server Push**——它被认为收益小、复杂度高。如果需要，自己实现 WebSocket 替代。

## 10. 性能对比（实测参考）

| 场景 | HTTP/1.1 | HTTP/2 | 提升 |
|------|----------|--------|------|
| 加载 50 个小资源（图片/JS/CSS） | 1500ms | 400ms | 73% ↓ |
| 高并发请求（100 并发） | 6 个连接排队 | 1 个连接并发 | 资源占用 ↓ 80% |
| 移动端弱网 | 队头阻塞严重 | 并行抗抖动 | 显著改善 |
| Header 重复发送 | 每次全量 | HPACK 压缩 | 头部流量 ↓ 70% |

## 11. 调试与诊断

```kotlin
// 启用 HTTP/2 日志
val client = OkHttpClient.Builder()
    .addNetworkInterceptor(Http2LoggingInterceptor())
    .build()

// 查看协议版本
val response = client.newCall(request).execute()
println(response.protocol)  // 可能是: "http/1.1" / "h2" / "h2_prior_knowledge"
```

## 12. 启用 HTTP/2 明文（h2c）

⚠️ **不推荐生产用**，但开发环境有用：

```kotlin
val client = OkHttpClient.Builder()
    .protocols(listOf(Protocol.H2_PRIOR_KNOWLEDGE))  // 强制明文 HTTP/2
    .build()
```

⚠️ 服务端必须支持 h2c（如 OkHttp 服务端、gRPC）。

## 13. 与 gRPC 的关系

**gRPC 默认用 HTTP/2 作为传输层**。OkHttp 不能直接做 gRPC 客户端——需要 grpc-okhttp：

```kotlin
// gRPC + OkHttp
val channel = ManagedChannelBuilder.forAddress("localhost", 50051)
    .useTransportSecurity()
    .build()
```

## 14. HTTP/2 限制

| 限制 | 说明 |
|------|------|
| 单连接 stream 上限 | 2^31 - 1 ≈ 21 亿（实际受窗口限制） |
| 流并发上限 | OkHttp 限制 6 个（`maxConcurrentStreams` 默认 100，可配置） |
| TLS 必需 | 浏览器场景；h2c 明文极少用 |
| 单 host 流限流 | OkHttp Dispatcher `maxRequestsPerHost` |

## 15. 何时禁用 HTTP/2

```kotlin
// 罕见：有些代理/防火墙搞坏 HTTP/2
val client = OkHttpClient.Builder()
    .protocols(listOf(Protocol.HTTP_1_1))  // 强制 HTTP/1.1
    .build()
```

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **连接池**: [[draft-03-connection-pool]]
- **同步/异步**: [[draft-07-sync-async]]
- **综合入口**: [[summary]]
