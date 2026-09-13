---
title: "gRPC 线协议 (HTTP/2 + Protobuf)"
category: synthesis
tags: [grpc, http2, protobuf, wire-protocol, length-prefixed-message, lpm]
sources:
  - "gRPC Core Spec PROTOCOL-HTTP2.md"
  - "gRPC-Java NettyClientHandler 源码"
  - "Protobuf Encoding Spec"
summary: "gRPC 线协议详解：HTTP/2 HEADERS frame + Length-Prefixed Message + Protobuf wire format"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
  base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §02 gRPC 线协议（HTTP/2 + Protobuf）

## 1. 协议栈全景

```
┌─────────────────────────────────────────────┐
│  应用层：MyService.sayHello(req, observer)   │
├─────────────────────────────────────────────┤
│  RPC 语义层：gRPC stub（类型安全）             │
├─────────────────────────────────────────────┤
│  序列化层：Protobuf wire format              │
├─────────────────────────────────────────────┤
│  帧层：Length-Prefixed Message (LPM)         │
├─────────────────────────────────────────────┤
│  传输层：HTTP/2 (HEADERS + DATA frames)      │
├─────────────────────────────────────────────┤
│  TCP / TLS                                   │
└─────────────────────────────────────────────┘
```

## 2. HTTP/2 在 gRPC 中的角色

**gRPC 强制使用 HTTP/2**（不是 HTTP/1.1，不是裸 TCP）。HTTP/2 提供：

| 能力 | gRPC 用法 |
|------|----------|
| 多路复用（Multiplexing） | 一个 TCP 连接承载多个 RPC（每 RPC 一个 stream） |
| 二进制分帧（Binary Framing） | 替代 HTTP/1.1 文本帧，解析更快 |
| 流控制（Flow Control） | 防止快速发送压垮慢速接收方 |
| 头压缩（HPACK） | 减少 metadata 字节数 |
| 服务器推送（Server Push） | gRPC 不使用 |
| 优先级（Prioritization） | gRPC 内部不强制 |

## 3. gRPC 必须的 HTTP/2 headers

每个 gRPC 调用必带这些 HTTP/2 headers（gRPC Core Spec § "Protocol"）：

### 3.1 请求端（Request Headers）

```
:method: POST                    ← 固定 POST
:scheme: http | https            ← 由 usePlaintext / useTransportSecurity 决定
:path: /<service>/<method>      ← 例如 /helloworld.Greeter/SayHello
:authority: <host>:<port>        ← 来自 forAddress()
content-type: application/grpc   ← 固定（gRPC over HTTP/2）
te: trailers                     ← 告诉服务端用 trailers
user-agent: grpc-java/1.66.0     ← 由实现决定
```

**关键点**：
- `:path` 的格式：`/<package.ServiceName>/<MethodName>`
- `content-type: application/grpc` 是 gRPC 专属，不能用其他 MIME type
- `te: trailers` 是 **HTTP/2 必传**——表明客户端期望通过 trailers（HEADERS frame）接收最终状态

### 3.2 响应端（Response Headers）

```
:status: 200                              ← HTTP 状态码（gRPC 不依赖它）
content-type: application/grpc           ← 固定
```

**gRPC 的 HTTP status 始终是 200**——业务错误通过 **gRPC status**（在 Trailers 里）表达。

### 3.3 响应端（Trailers）

```
grpc-status: 0                            ← gRPC 状态码（0=OK）
grpc-message: <error message>             ← 错误描述
grpc-encoding: gzip                       ← 响应压缩方式
grpc-accept-encoding: gzip, snappy        ← 客户端接受的压缩
```

**grpc-status** 在 **trailers** 而非 **headers** 里——这是关键设计。

## 4. Length-Prefixed Message (LPM)

gRPC 在 HTTP/2 DATA frame 内传输 **Length-Prefixed Message**（5 字节 header）：

```
┌────────────────────────────────────────┐
│  Compressed-Flag (1 byte)               │
│   0 = 未压缩，1 = gzip/snappy/zstd 等   │
├────────────────────────────────────────┤
│  Message-Length (4 bytes, big-endian)   │
│   消息字节数（不含这 5 字节 header）     │
├────────────────────────────────────────┤
│  Message Payload (N bytes)              │
│   Protobuf 序列化数据                   │
└────────────────────────────────────────┘
```

**Wire 格式**：

```
+--------+--------+--------+--------+--------+
|   C    |    LENGTH (4 bytes, big-endian)     |
+--------+--------+--------+--------+--------+
|                  PAYLOAD                       |
+-----------------------------------------------+
```

### 4.1 为什么用 LPM？

- **自描述**：每个消息有边界，TCP 帧不会破坏消息
- **压缩协商**：每个消息独立标记是否压缩
- **流式友好**：客户端流式发送 N 条消息，服务器可逐条解析

## 5. Protobuf Wire Format（序列化层）

gRPC 的消息体是 **Protobuf v3** 序列化的二进制。

### 5.1 Varint 编码（核心）

每个字段 ID 和 wire type 用 **varint**（变长整数）编码：

```
field_id << 3 | wire_type   →  varint 编码
```

```
MSB 规则：每个字节最高位（MSB）是 continuation bit
  1 = 后面还有字节
  0 = 最后一个字节
低 7 位：数据位
```

**示例**：field_id=3, wire_type=2 (length-delimited) → `(3 << 3) | 2 = 26` → varint = `0x1A`

### 5.2 Wire Types

| Type | 值 | 含义 | 示例 |
|------|---|------|------|
| 0 | VARINT | int32/64/uint/... | field_id << 3 \| 0 |
| 1 | FIXED64 | double, fixed64 | field_id << 3 \| 1 |
| 2 | LENGTH_DELIMITED | string, bytes, embedded message | field_id << 3 \| 2 |
| 3 | START_GROUP | 已弃用（Proto 1） | - |
| 4 | END_GROUP | 已弃用（Proto 1） | - |
| 5 | FIXED32 | float, fixed32 | field_id << 3 \| 5 |

### 5.3 序列化示例

```protobuf
// hello.proto
message HelloRequest {
  string name = 1;
}
```

发送 `HelloRequest{name: "mike"}`：

```
# Field 1 (string, wire_type=2) → (1 << 3) | 2 = 0x0A
0x0A        # field tag
0x04        # length = 4 bytes
'm' 'i' 'k' 'e'   # ASCII bytes
```

最终字节：`0A 04 6D 69 6B 65`（6 字节）。

## 6. 完整 wire 流程（Unary）

```
Client                                    Server
  │                                          │
  │ HEADERS frame (END_HEADERS)              │
  ├──────────────────────────────────────►   │
  │ :method: POST                             │
  │ :path: /helloworld.Greeter/SayHello       │
  │ content-type: application/grpc            │
  │ te: trailers                              │
  │ grpc-encoding: gzip                       │
  │                                          ▼
  │                                  Netty Server 处理 HEADERS
  │                                          │
  │ DATA frame (END_STREAM 不置位)            │
  ├──────────────────────────────────────►   │
  │ 0x00 + length(4) + protobuf bytes         │
  │                                          ▼
  │                                  Protobuf 反序列化
  │                                  业务方法调用
  │                                          │
  │ HEADERS frame (END_STREAM 不置位)         │
  │ ◄──────────────────────────────────────  │
  │ grpc-encoding: gzip                       │
  │                                          │
  │ DATA frame                                │
  │ ◄──────────────────────────────────────  │
  │ 0x00 + length(4) + protobuf bytes         │
  │                                          │
  │ HEADERS frame (END_STREAM 置位) = Trailers│
  │ ◄──────────────────────────────────────  │
  │ grpc-status: 0                            │
  │ grpc-message: OK                          │
  │                                          ▼
  │                                  调用结束
```

## 7. 压缩协商

```
客户端 HEADERS:        grpc-encoding: gzip, snappy
服务端 HEADERS:        grpc-encoding: gzip    ← 服务端选定算法
所有 DATA frame:       Compressed-Flag = 1   ← 该算法压缩
TRAILERS:              grpc-encoding: gzip
```

**支持的压缩算法**：
- `gzip`（默认，gRPC-Java 自带）
- `snappy`（需 snappy-java 依赖）
- `zstd`（需 zstd-jni 依赖）
- `deflate`（基本不用）
- `identity`（不压缩）

## 8. 流式协议的 wire 特征

| 流式模式 | 客户端发 | 服务端发 | DATA frame 数 |
|---------|---------|---------|---------------|
| **Unary** | 1 条 | 1 条 | 客户端 1 + 服务端 1 |
| **Server streaming** | 1 条 | N 条 | 客户端 1 + 服务端 N |
| **Client streaming** | N 条 | 1 条 | 客户端 N + 服务端 1 |
| **Bidirectional** | N 条 | M 条 | 客户端 N + 服务端 M |

**每个 DATA frame 包含一条 LPM 消息**——流式就是 N 个连续的 DATA frame。

## 9. Trailer-only response（特殊）

某些错误场景，**服务端不发 DATA frame，只发 Trailers HEADERS frame**：

```
客户端 HEADERS ─────►
服务端的 HEADERS (END_STREAM 置位)
        ◄─────────────
grpc-status: 14  ← UNAVAILABLE
grpc-message: "upstream unavailable"
```

**典型场景**：
- 客户端发送前就取消
- 服务端启动失败
- 流量控制拒绝

## 10. 与 REST/JSON 的协议对比

| 维度 | gRPC | REST/JSON |
|------|------|-----------|
| 传输 | HTTP/2（强制） | HTTP/1.1+（兼容） |
| 序列化 | Protobuf（二进制） | JSON（文本） |
| 消息边界 | LPM 5 字节 header | Content-Length header |
| 状态码 | gRPC 16 个 code | HTTP 30+ code |
| 压缩 | per-message（gzip flag） | per-request（Accept-Encoding） |
| 内容协商 | grpc-encoding header | Accept header |
| 类型定义 | .proto 文件 | OpenAPI / RAML |
| 可读性 | ❌（二进制） | ✅（文本） |
| 浏览器 | 需 gRPC-Web | 原生 |

## 11. 调试工具

```bash
# Wireshark：解析 HTTP/2 + gRPC（需安装 grpc-dissector）
# 或 tcpdump + 自己解析

# curl 模拟 gRPC 请求（理论上可行但很麻烦）
curl -v \
  --http2 \
  -H "Content-Type: application/grpc" \
  -H "TE: trailers" \
  --data-binary $'\x00\x00\x00\x00\x04\x08\x01\x12' \
  http://localhost:50051/helloworld.Greeter/SayHello

# grpcurl（推荐）
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext -d '{"name":"mike"}' localhost:50051 helloworld.Greeter/SayHello
```

## 12. 关键设计原则

1. **HTTP/2 强制**——gRPC 不支持 HTTP/1.1
2. **LPM 自描述**——每个消息有边界，TCP 帧不会破坏
3. **Trailers 表达状态**——HTTP status 始终 200，错误在 Trailers
4. **每消息独立压缩**——支持多种算法灵活切换
5. **Protobuf schema-first**——.proto 文件是契约

## 13. 与 [[okhttp-analysis/summary\|OkHttp]] 对比

| 维度 | gRPC | OkHttp |
|------|------|--------|
| 线格式 | HTTP/2 + LPM + Protobuf | HTTP/1.1+2 + 任意 body |
| 消息边界 | 5 字节 LPM header | Content-Length header |
| 状态表达 | grpc-status (Trailers) | HTTP status code |
| 序列化 | Protobuf（强制） | 用户自定 |
| 流式 | 协议级 4 种 | 仅 WebSocket |
| 浏览器 | 需 gRPC-Web | 原生 |

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **服务定义**: [[draft-03-service-definition]]
- **4 流式模式**: [[draft-06-streaming]]
- **综合入口**: [[summary]]