---
title: "gRPC 核心架构"
category: synthesis
tags: [grpc, architecture, channel, stub, service, server, call]
sources:
  - "gRPC-Java 1.66+ 源码: ManagedChannel.java / ClientCall.java / Server.java / ServerCall.java / ServiceDescriptor.java"
  - "gRPC Core Concepts"
summary: "gRPC 四件套 ManagedChannel / ClientCall / Server / ServerCall + 完整调用时序图"
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

# §01 gRPC 核心架构

## 1. 四件套全景

gRPC 的所有能力都通过 4 个核心抽象暴露：

| 抽象 | 角色 | 谁持有 |
|------|------|--------|
| **`ManagedChannel`** | 客户端连接抽象 | 客户端（应用） |
| **`ClientCall`** | 单次客户端调用 | gRPC 内部 |
| **`Server`** | 服务端抽象 | 服务端（应用） |
| **`ServerCall`** | 单次服务端调用 | gRPC 内部 |

**关键洞察**：**Channel/Server 由应用持有，Call 由 gRPC 内部管理**——应用不直接操作 Call，而是通过 Stub 间接使用。

## 2. 客户端四件套（再细化）

```
应用
 ├─ ManagedChannel（连接，可复用）
 │   ├─ ManagedChannelBuilder 创建
 │   ├─ 持有 1..N 条 HTTP/2 连接
 │   ├─ 内部线程池（schedulers）
 │   └─ NameResolver（服务发现）
 │
 ├─ Stub（代理，由 .proto 生成）
 │   ├─ BlockingStub（同步，返回 Response）
 │   ├─ FutureStub（异步，返回 ListenableFuture）
 │   ├─ Stub（异步流式，返回 StreamObserver）
 │   └─ 内部持有一个 Channel
 │
 ├─ ClientCall（单次调用，gRPC 内部）
 │   ├─ ClientCallImpl 实现
 │   ├─ sendMessage() / halfClose() / request()
 │   └─ listener.onMessage() / onClose()
 │
 └─ Metadata / Request / Response（消息）
```

## 3. 服务端四件套

```
应用
 ├─ Server（监听 TCP 端口）
 │   ├─ ServerBuilder 创建
 │   ├─ 绑定多个 Service
 │   ├─ Netty ServerBootstrap（实际底层）
 │   └─ 内部线程池（schedulers）
 │
 ├─ BindableService（业务实现，由 .proto 生成基类）
 │   ├─ 继承自 .proto 生成的 MyServiceGrpc.MyServiceImplBase
 │   ├─ 实现 serviceDescriptor()（方法列表）
 │   └─ 实现各方法（sayHello 等）
 │
 ├─ ServerCall（单次服务端调用，gRPC 内部）
 │   ├─ ServerCallImpl 实现
 │   ├─ sendMessage() / close()
 │   └─ listener.onMessage() / onHalfClose() / onComplete()
 │
 └─ Metadata / Request / Response（消息）
```

## 4. ManagedChannel — 客户端连接抽象

```java
// 1. 创建 Channel
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("localhost", 50051)
    .usePlaintext()                              // 明文（开发）
    // .useTransportSecurity()                   // TLS（生产）
    .build();

// 2. Stub 持有 Channel
GreeterGrpc.GreeterBlockingStub stub = 
    GreeterGrpc.newBlockingStub(channel);

// 3. 调用
HelloRequest req = HelloRequest.newBuilder()
    .setName("mike")
    .build();
HelloReply resp = stub.sayHello(req);  // 阻塞

// 4. 关闭（重要！释放 HTTP/2 连接）
channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
```

### 4.1 关键特性

- **长生命周期**：一个 Channel 跨多个 RPC 调用复用
- **HTTP/2 连接复用**：Channel 内部维护 HTTP/2 streams（每个 RPC 一个 stream）
- **自动重连**：连接断开后自动重新建立
- **NameResolver 集成**：可基于 DNS / 服务发现动态选择连接目标
- **负载均衡**：内置多种 LB 策略（pick_first、round_robin、xDS）

### 4.2 ManagedChannel 内部结构

```java
abstract class ManagedChannel extends Channel {
    public ManagedChannel shutdown() { /* 标记关闭 */ }
    public ListenableFuture<Void> shutdownNow() { /* 强制关 */ }
    public boolean isShutdown() { /* 是否已 shutdown */ }
    public boolean awaitTermination(long timeout, TimeUnit unit) { /* 阻塞等待 */ }
    
    // 创建 stub
    public abstract <Req, Resp> ClientCall<Req, Resp> newCall(
        MethodDescriptor<Req, Resp> method,
        CallOptions callOptions
    );
}
```

## 5. ClientCall — 单次调用（应用不直接持有）

```java
// 高级 Stub 内部会创建 ClientCall
ClientCall<HelloRequest, HelloReply> call = channel.newCall(
    MethodDescriptor.<HelloRequest, HelloReply>newBuilder()
        .setType(MethodType.UNARY)
        .setFullMethodName("helloworld.Greeter/SayHello")
        .setRequestMarshaller(...).setResponseMarshaller(...)
        .build(),
    CallOptions.DEFAULT
);

// 启动调用
call.start(new ClientCall.Listener<HelloReply>() {
    @Override public void onMessage(HelloReply reply) { /* 收到响应 */ }
    @Override public void onClose(Status status, Metadata trailers) { /* 结束 */ }
}, new Metadata());

// 发送请求
call.sendMessage(req);
call.halfClose();  // 客户端发完
call.request(1);   // 请求 1 个响应
```

**应用代码通常不直接用 ClientCall**——用 Stub 包装。

## 6. Stub — 类型安全的代理

```java
// .proto 自动生成 3 个 Stub
GreeterGrpc.GreeterBlockingStub blocking = 
    GreeterGrpc.newBlockingStub(channel);     // 同步

GreeterGrpc.GreeterFutureStub future = 
    GreeterGrpc.newFutureStub(channel);       // Future

GreeterGrpc.GreeterStub async = 
    GreeterGrpc.newStub(channel);             // 异步 StreamObserver
```

**使用示例**（4 种流式）：

```java
// 1. Unary: 同步
HelloReply resp = blocking.sayHello(HelloRequest.newBuilder().setName("mike").build());

// 2. Server streaming
Iterator<HelloReply> iter = blocking.greetManyTimes(req);  // 阻塞 iter
while (iter.hasNext()) {
    HelloReply r = iter.next();
    // 处理
}

// 3. Client streaming: 阻塞版
ListenableFuture<HelloReply> future = future.collectMessages(requests);
HelloReply resp = future.get();

// 4. Bidirectional
StreamObserver<HelloRequest> req = async.sayHelloBidirectional(new StreamObserver<HelloReply>() {
    public void onNext(HelloReply r) { /* 收到响应 */ }
    public void onError(Throwable t) { /* 错误 */ }
    public void onCompleted() { /* 服务端完成 */ }
});
// 客户端发送
req.onNext(req1);
req.onNext(req2);
req.onCompleted();
```

## 7. Server — 服务端抽象

```java
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())  // BindableService
    .build();

server.start();  // 启动监听
System.out.println("Server started on " + server.getPort());
server.awaitTermination();  // 阻塞直到 shutdown
```

### 7.1 服务端内部结构

```java
abstract class Server {
    public abstract Server start() throws IOException;
    public abstract Server shutdown();              // 优雅关闭
    public abstract Server shutdownNow();           // 强制
    public abstract int getPort();                   // 监听端口
    public abstract void awaitTermination();         // 阻塞
    public abstract void awaitTermination(long timeout, TimeUnit unit);
}
```

**关键方法**：
- `shutdown()`：优雅关闭，等待 in-flight 调用完成
- `shutdownNow()`：立即关闭，丢弃 in-flight
- `awaitTermination(timeout, unit)`：阻塞直到关闭完成或超时

## 8. ServerCall — 单次服务端调用

```java
// 服务端实现示例（继承 .proto 生成的基类）
class GreeterImpl extends GreeterGrpc.SServiceImplBase {
    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
        HelloReply reply = HelloReply.newBuilder()
            .setMessage("Hello, " + req.getName())
            .build();
        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
}
```

**关键点**：
- 业务方法返回 `void`（不是返回 Response）
- 通过 `StreamObserver.onNext(reply)` 发响应
- 通过 `StreamObserver.onCompleted()` 标记完成
- 通过 `StreamObserver.onError(t)` 发错误

## 9. 完整调用时序图（Unary）

```
Client                                    Server
  │                                          │
  ▼                                          │
stub.sayHello(req)                           │
  │                                          │
  ▼                                          │
BlockingStubCall.sendMessage(req)            │
  │                                          │
  ▼                                          │
ClientCallImpl.start(listener)               │
  │                                          │
  ▼                                          │
ManagedChannelImpl.newStream (HTTP/2 stream) │
  │                                          │
  ▼ HTTP/2 DATA frame + Headers              │
  ├──────────────────────────────────────►   │
  │                                          ▼
  │                                  NettyServerHandler
  │                                          │
  │                                  ServerCallImpl
  │                                          │
  │                                  ServerCallsHandler
  │                                          │
  │                                  GreeterImpl.sayHello
  │                                          │
  │                                  responseObserver.onNext(reply)
  │                                          │
  │                                  responseObserver.onCompleted()
  │                                          │
  │ ◄────────────────────────────────────── │
  ▼ HTTP/2 DATA frame + Trailers             │
ClientCallImpl.onMessage(reply)              │
  │                                          │
  ▼                                          │
BlockingStubCall → return reply               │
  │                                          │
  ▼                                          │
stub 调用返回                                  │
```

## 10. Metadata vs Headers vs Trailers

| 概念 | 角色 | 时机 | 类比 |
|------|------|------|------|
| **Metadata（Headers）** | 客户端发送的"headers" | 发起调用时 | HTTP/2 HEADERS frame |
| **Metadata（Trailers）** | 服务端发送的"状态" | 响应结束 | HTTP/2 HEADERS frame (END_STREAM) |

```java
// 客户端：发 metadata
Metadata headers = new Metadata();
headers.put(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER), "Bearer xxx");
call.start(listener, headers);

// 服务端：读 metadata
@Override
public void sayHello(HelloRequest req, StreamObserver<HelloReply> obs) {
    Metadata headers = ...;  // 通过 Context 获取
    String auth = headers.get(AUTH_KEY);
    // ...
}

// 服务端：发 trailers（带状态码）
@Override
public void onCompleted() {
    Metadata trailers = new Metadata();
    trailers.put(Status.CODE_KEY, Status.UNAUTHENTICATED);
    // ...
}
```

## 11. Status 与 StatusException

**gRPC 有自己的状态码系统**（详见 [[draft-09-error-handling]]）：

```java
public final class Status {
    public static final Status OK = ...;
    public static final Status CANCELLED = ...;
    public static final Status UNKNOWN = ...;
    public static final Status INVALID_ARGUMENT = ...;
    public static final Status DEADLINE_EXCEEDED = ...;
    // ... 共 16 个
    
    public Status.Code getCode();
    public String getDescription();
}
```

## 12. Stub 与 Channel 的关系（一图）

```
                    ┌──────────────────┐
                    │   ManagedChannel  │  ← 应用持有（可复用）
                    │  （1..N HTTP/2）  │
                    └────────┬─────────┘
                             │ newCall()
                             ▼
┌──────────────────────────────────────────┐
│  ClientCall (单次调用，gRPC 内部)         │  ← gRPC 内部
└──────────────────────────────────────────┘
                             ▲
                             │ Stub.method()
                             │
                    ┌────────┴─────────┐
                    │      Stub         │  ← 应用持有
                    │  (proto 生成)      │     - BlockingStub
                    │  持有一个 Channel  │     - FutureStub
                    └──────────────────┘     - Stub (异步)
```

**关键原则**：
- **Channel 复用**：1 个 Channel 对 1 个服务端（可创建 N 个 Stub）
- **Stub 复用**：1 个 Stub 可调用 N 次
- **ClientCall 内部**：每次 RPC 创建 1 个（应用无感知）

## 13. 与 [[okhttp-analysis/summary\|OkHttp]] 的架构对比

| 维度 | gRPC | [[okhttp-analysis/summary\|OkHttp]] |
|------|------|--------|
| 客户端核心抽象 | Channel | OkHttpClient |
| 单次调用 | ClientCall | Call |
| 类型安全 | ✅（Stub） | ❌ |
| 流式 | ✅ 4 种 | ❌（仅 WebSocket） |
| 序列化 | Protobuf（强约束） | 任意 |
| 协议 | HTTP/2 强制 | HTTP/1.1+2 |
| 服务端对应 | Server | 无（OkHttp 只是客户端） |

**核心差异**：
- OkHttp 是 HTTP 客户端库，gRPC 是 RPC 框架
- gRPC 类型安全靠 Stub + Protobuf
- gRPC 流式是协议级别（HTTP/2 stream），OkHttp WebSocket 是单独的协议

## 14. 设计原则

1. **强类型优先**：.proto 生成 stub，避免运行时错误
2. **Channel 复用**：避免每次调用新建连接
3. **Stub 复用**：避免每次调用 newClient
4. **应用不持有 ClientCall**：gRPC 内部管理，应用通过 Stub 调用
5. **状态码统一**：跨语言一致（16 个 Status Code）

## 15. 反模式

❌ **每次 RPC new ManagedChannel()**——浪费连接池、线程池
❌ **忘调 `channel.shutdown()`**——HTTP/2 连接泄漏
❌ **直接用 ClientCall 而不是 Stub**——绕过类型安全
❌ **Block stub 用在异步上下文**——阻塞事件循环
❌ **BlockingStub 在 Netty event loop 上调**——Netty 线程死锁

## 相关笔记

- **线协议**: [[draft-02-wire-protocol]]
- **服务定义**: [[draft-03-service-definition]]
- **客户端详解**: [[draft-04-client-side]]
- **服务端详解**: [[draft-05-server-side]]
- **综合入口**: [[summary]]