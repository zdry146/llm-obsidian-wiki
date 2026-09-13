---
title: "gRPC 4 种流式模式"
category: synthesis
tags: [grpc, streaming, unary, server-streaming, client-streaming, bidirectional]
sources:
  - "gRPC Core Spec - Streaming"
  - "gRPC-Java StreamObserver 源码"
  - "gRPC User Guide - Streaming RPCs"
summary: "gRPC 4 种流式模式详解：Unary / Server streaming / Client streaming / Bidirectional + 完整代码示例"
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

# §06 gRPC 4 种流式模式

## 1. 4 种模式全景

| 模式 | 客户端发 | 服务端回 | 适合 |
|------|---------|---------|------|
| **Unary** | 1 条 | 1 条 | 普通请求-响应（90% 场景） |
| **Server streaming** | 1 条 | N 条 | 订阅、推送、实时通知 |
| **Client streaming** | N 条 | 1 条 | 上传、聚合、批量写 |
| **Bidirectional** | N 条 | M 条 | 聊天、协作、实时双向 |

## 2. Unary（最常用）

**定义**：
```protobuf
rpc SayHello(HelloRequest) returns (HelloReply);
```

**服务端实现**：
```java
@Override
public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    HelloReply reply = HelloReply.newBuilder()
        .setMessage("Hello, " + req.getName())
        .build();
    
    responseObserver.onNext(reply);  // 发 1 条响应
    responseObserver.onCompleted();  // 标记完成
}
```

**客户端调用（3 种 Stub）**：
```java
// 1. BlockingStub - 同步
HelloReply resp = GreeterGrpc.newBlockingStub(channel)
    .sayHello(HelloRequest.newBuilder().setName("mike").build());

// 2. FutureStub - 异步
ListenableFuture<HelloReply> future = GreeterGrpc.newFutureStub(channel)
    .sayHello(HelloRequest.newBuilder().setName("mike").build());

// 3. Stub (StreamObserver) - 异步回调
GreeterGrpc.newStub(channel).sayHello(
    HelloRequest.newBuilder().setName("mike").build(),
    new StreamObserver<HelloReply>() {
        @Override public void onNext(HelloReply reply) { /* 1 条响应 */ }
        @Override public void onError(Throwable t) { /* 错误 */ }
        @Override public void onCompleted() { /* 完成 */ }
    }
);
```

**Wire 流程**：
```
Client HEADERS ─────►
Server HEADERS ◄─────
Client DATA (request) ─────►
Server DATA (response) ◄─────
Server HEADERS (Trailers) ◄───── grpc-status: 0
```

## 3. Server Streaming

**定义**：
```protobuf
rpc LotsOfReplies(HelloRequest) returns (stream HelloReply);
```

> 返回类型前加 `stream` 关键字 = Server streaming

**服务端实现**：
```java
@Override
public void lotsOfReplies(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    // 可以分批响应
    for (int i = 0; i < 10; i++) {
        if (responseObserver instanceof ServerCallStreamObserver) {
            ServerCallStreamObserver<HelloReply> serverObs = 
                (ServerCallStreamObserver<HelloReply>) responseObserver;
            
            // 流控：客户端还没准备好就阻塞
            while (!serverObs.isReady()) {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
        
        HelloReply reply = HelloReply.newBuilder()
            .setMessage("Reply " + i)
            .build();
        responseObserver.onNext(reply);
    }
    responseObserver.onCompleted();
}
```

**客户端调用**：
```java
// BlockingStub - 同步 Iterator
Iterator<HelloReply> iterator = blockingStub.lotsOfReplies(req);
while (iterator.hasNext()) {
    HelloReply reply = iterator.next();
    // 处理
}

// Stub (异步)
async.lotsOfReplies(req, new StreamObserver<HelloReply>() {
    @Override public void onNext(HelloReply reply) {
        // 每收到一条响应就调用一次（多次）
    }
    @Override public void onError(Throwable t) { /* ... */ }
    @Override public void onCompleted() { /* 服务端发完了 */ }
});
```

**Wire 流程**：
```
Client HEADERS ─────►
Server HEADERS ◄─────
Client DATA (request) ─────►
Server DATA (response 1) ◄─────
Server DATA (response 2) ◄─────
Server DATA (response N) ◄─────
Server HEADERS (Trailers) ◄───── grpc-status: 0
```

**适用场景**：
- 股票行情推送
- 日志实时显示
- 订阅通知
- 进度条推送
- SSE 替代

## 4. Client Streaming

**定义**：
```protobuf
rpc LotsOfGreetings(stream HelloRequest) returns (HelloReply);
```

> 请求参数前加 `stream` 关键字 = Client streaming

**服务端实现**（返回 `StreamObserver<Request>`）：
```java
@Override
public StreamObserver<HelloRequest> lotsOfGreetings(
        StreamObserver<HelloReply> responseObserver) {
    
    return new StreamObserver<HelloRequest>() {
        private final StringBuilder allNames = new StringBuilder();
        
        @Override
        public void onNext(HelloRequest req) {
            // 客户端每发一条就调一次（多次）
            allNames.append(req.getName()).append(", ");
        }
        
        @Override
        public void onError(Throwable t) {
            System.err.println("Client streaming error: " + t.getMessage());
        }
        
        @Override
        public void onCompleted() {
            // 客户端 onCompleted() 后调用一次
            HelloReply reply = HelloReply.newBuilder()
                .setMessage("Hello, " + allNames.toString())
                .build();
            responseObserver.onNext(reply);
            responseObserver.onCompleted();
        }
    };
}
```

**客户端调用**：
```java
// 用 Stub (异步)
StreamObserver<HelloRequest> requestObserver = 
    async.lotsOfGreetings(new StreamObserver<HelloReply>() {
        @Override public void onNext(HelloReply reply) { /* 最终 1 条响应 */ }
        @Override public void onError(Throwable t) { /* ... */ }
        @Override public void onCompleted() { /* ... */ }
    });

// 持续发送
requestObserver.onNext(HelloRequest.newBuilder().setName("Alice").build());
requestObserver.onNext(HelloRequest.newBuilder().setName("Bob").build());
requestObserver.onNext(HelloRequest.newBuilder().setName("Charlie").build());

requestObserver.onCompleted();  // 告诉服务端：客户端发完了
```

**Wire 流程**：
```
Client HEADERS ─────►
Server HEADERS ◄─────
Client DATA (request 1) ─────►
Client DATA (request 2) ─────►
Client DATA (request N) ─────►
Client HEADERS (Trailers, END_STREAM) ─────► grpc-status: 0
Server DATA (response) ◄─────
Server HEADERS (Trailers) ◄───── grpc-status: 0
```

**适用场景**：
- 文件上传（大文件分块）
- 批量数据收集（IoT 设备）
- 实时数据聚合

## 5. Bidirectional（双向流式）

**定义**：
```protobuf
rpc BidiHello(stream HelloRequest) returns (stream HelloReply);
```

> 请求和返回都加 `stream` = Bidirectional

**服务端实现**（返回 `StreamObserver<Request>`）：
```java
@Override
public StreamObserver<HelloRequest> bidiHello(
        StreamObserver<HelloReply> responseObserver) {
    
    return new StreamObserver<HelloRequest>() {
        @Override
        public void onNext(HelloRequest req) {
            // 每收到客户端一条，立刻响应
            HelloReply reply = HelloReply.newBuilder()
                .setMessage("Hello, " + req.getName())
                .build();
            responseObserver.onNext(reply);
            // 注意：不要在这里 onCompleted()——客户端还要发
        }
        
        @Override
        public void onError(Throwable t) { /* ... */ }
        
        @Override
        public void onCompleted() {
            // 客户端 onCompleted() 后关闭服务端流
            responseObserver.onCompleted();
        }
    };
}
```

**客户端调用**：
```java
StreamObserver<HelloRequest> req = async.bidiHello(
    new StreamObserver<HelloReply>() {
        @Override public void onNext(HelloReply reply) {
            // 收到服务端响应（多次）
        }
        @Override public void onError(Throwable t) { /* ... */ }
        @Override public void onCompleted() { /* 服务端完成 */ }
    });

// 持续发送 + 接收
new Thread(() -> {
    while (running) {
        req.onNext(HelloRequest.newBuilder()
            .setName("user-" + i)
            .build());
        sleep(1000);
        i++;
    }
    req.onCompleted();
}).start();
```

**Wire 流程**：
```
Client HEADERS ─────►
Server HEADERS ◄─────
Client DATA 1 ─────►  Server DATA 1 ◄─────
Client DATA 2 ─────►  Server DATA 2 ◄─────
Client DATA N ─────►  Server DATA M ◄─────
Client HEADERS (Trailers) ─────►  grpc-status: 0
Server HEADERS (Trailers) ◄─────  grpc-status: 0
```

**适用场景**：
- 实时聊天
- 多人协作（文档编辑）
- 双向实时通知
- 游戏状态同步

## 6. StreamObserver 关键 API

### 6.1 客户端 StreamObserver

```java
public interface StreamObserver<V> {
    void onNext(V value);          // 收到一条响应
    void onError(Throwable t);     // 出错（流结束）
    void onCompleted();             // 正常完成（流结束）
}
```

### 6.2 客户端发送侧

```java
StreamObserver<Request> requestObserver = stub.someMethod(
    new StreamObserver<Response>() { /* 接收响应 */ }
);

requestObserver.onNext(req1);     // 发请求 1
requestObserver.onNext(req2);     // 发请求 2（流式必须连续调用）
// ...
requestObserver.onCompleted();    // 发完成信号（服务端会收到 onCompleted）
```

⚠️ **onNext() 不能并行调用**——必须串行！否则会抛 `IllegalStateException`。

### 6.3 服务端 ServerCallStreamObserver

服务端可以转 `StreamObserver` 为 `ServerCallStreamObserver`：

```java
@Override
public void lotsOfReplies(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    ServerCallStreamObserver<HelloReply> serverObs = 
        (ServerCallStreamObserver<HelloReply>) responseObserver;
    
    // 流控
    while (!serverObs.isReady()) {
        // 客户端还没处理完上一批，等待
        Thread.sleep(10);
    }
    
    // 设置上限
    serverObs.setOnReadyThreshold(1);  // buffer ≤ 1 就触发 isReady
    
    // 检查是否被客户端取消
    if (serverObs.isCancelled()) {
        return;  // 别再 onNext 了
    }
    
    serverObs.onNext(reply);
}
```

## 7. 流控（Flow Control）

### 7.1 HTTP/2 层流控

HTTP/2 每个 stream 有 **window size**（默认 65535 字节）。超过则发方暂停。

### 7.2 应用层流控

服务端用 `ServerCallStreamObserver.isReady()` 检查：

```java
ServerCallStreamObserver<Reply> obs = (ServerCallStreamObserver<Reply>) responseObserver;

// 设置 buffer 阈值
obs.setOnReadyThreshold(1);  // buffer 中 ≤ 1 条未处理时触发 isReady=true

while (hasMore()) {
    while (!obs.isReady()) {
        Thread.sleep(10);  // 等客户端处理
    }
    obs.onNext(reply);
}
```

**客户端**也可以通过 `ClientCallStreamObserver.request(n)` 触发客户端请求服务端发更多：

```java
ClientCallStreamObserver<Request> req = (ClientCallStreamObserver<Request>) observer;
req.request(10);  // 请求服务端发 10 条
```

## 8. 取消（Cancellation）

### 8.1 客户端取消

```java
ClientCallStreamObserver<Request> observer = ...;

// 1. 直接取消
observer.cancel("user navigated away", new CancellationException());

// 2. 通过 Context
Context.CancellableContext withCancel = Context.current().withCancellation();
observer.cancel(...);
```

### 8.2 服务端检测取消

```java
ServerCallStreamObserver<Reply> obs = (ServerCallStreamObserver<Reply>) responseObserver;
if (obs.isCancelled()) {
    // 客户端已经取消，停止 onNext
    return;
}
```

**onError(Throwable) 在客户端和服务端都会触发**——如果取消是因客户端发起的。

## 9. 错误处理

### 9.1 Unary 错误

```java
// 服务端
@Override
public void sayHello(HelloRequest req, StreamObserver<HelloReply> obs) {
    if (req.getName().isEmpty()) {
        obs.onError(Status.INVALID_ARGUMENT
            .withDescription("name is required")
            .asRuntimeException());
        return;
    }
    // ...
}

// 客户端
try {
    HelloReply resp = stub.sayHello(req);
} catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    status.getCode();       // Status.Code.INVALID_ARGUMENT
    status.getDescription(); // "name is required"
}
```

### 9.2 流式错误

```java
// 服务端发到一半出错
try {
    for (Item item : items) {
        obs.onNext(reply(item));
    }
    obs.onCompleted();
} catch (Exception e) {
    obs.onError(Status.INTERNAL.withDescription("...").asRuntimeException());
    // 不要再 onNext/onCompleted
}

// 客户端收到错误
new StreamObserver<Reply>() {
    @Override public void onError(Throwable t) {
        Status status = Status.fromThrowable(t);
        // status.getCode() = INTERNAL
    }
};
```

详见 [[draft-09-error-handling]]。

## 10. 实战：聊天应用（Bidi 示例）

### 10.1 .proto

```protobuf
service Chat {
    rpc Connect(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
    string user = 1;
    string text = 2;
}
```

### 10.2 服务端（广播模式）

```java
class ChatServiceImpl extends ChatGrpc.ChatImplBase {
    private final Set<StreamObserver<ChatMessage>> observers = 
        Collections.synchronizedSet(new HashSet<>());
    
    @Override
    public StreamObserver<ChatMessage> connect(
            StreamObserver<ChatMessage> responseObserver) {
        
        // 1. 新用户加入
        ServerCallStreamObserver<ChatMessage> serverObs = 
            (ServerCallStreamObserver<ChatMessage>) responseObserver;
        observers.add(serverObserver);
        
        // 2. 清理逻辑（用户断开）
        serverObserver.setOnCancelHandler(() -> 
            observers.remove(serverObserver)
        );
        
        // 3. 接收客户端消息
        return new StreamObserver<ChatMessage>() {
            @Override
            public void onNext(ChatMessage msg) {
                // 广播给所有用户
                for (StreamObserver<ChatMessage> obs : observers) {
                    try {
                        obs.onNext(msg);
                    } catch (Exception e) {
                        // 用户断开
                        observers.remove(obs);
                    }
                }
            }
            @Override public void onError(Throwable t) { /* ... */ }
            @Override public void onCompleted() { /* ... */ }
        };
    }
}
```

### 10.3 客户端

```java
class ChatClient {
    private StreamObserver<ChatMessage> outgoing;
    
    public void start(String user) {
        outgoing = ChatGrpc.newStub(channel).connect(
            new StreamObserver<ChatMessage>() {
                @Override public void onNext(ChatMessage msg) {
                    // 显示收到的消息
                    System.out.println(msg.getUser() + ": " + msg.getText());
                }
                @Override public void onError(Throwable t) { /* ... */ }
                @Override public void onCompleted() { /* ... */ }
            }
        );
    }
    
    public void sendMessage(String text) {
        outgoing.onNext(ChatMessage.newBuilder()
            .setUser(user).setText(text).build());
    }
    
    public void close() {
        outgoing.onCompleted();
    }
}
```

## 11. 性能对比

| 模式 | 客户端请求延迟 | 内存占用 | 网络流量 |
|------|--------------|---------|---------|
| Unary | 200ms (1 RTT) | 低 | 低 |
| Server streaming | 200ms (首批) | 中 | 中 |
| Client streaming | 200ms (最后) | 中 | 中 |
| Bidirectional | 100ms (首批) | 中-高 | 中-高 |

**Bidi 比 Unary 快**——因为可以双向并行（少 1 个 RTT）。

## 12. 与 WebSocket / SSE 对比

| 维度 | gRPC Bidi | WebSocket | SSE |
|------|-----------|-----------|-----|
| 协议 | HTTP/2 | HTTP/1.1 升级 | HTTP/1.1 长连接 |
| 双向 | ✅ | ✅ | ❌ 单向 |
| 类型安全 | ✅ Protobuf | ❌ | ❌ |
| 浏览器 | 需 gRPC-Web | 原生 | 原生 |
| 性能 | 高（HTTP/2） | 中 | 低 |
| 跨语言 | ✅ | ❌ | ❌ |

## 13. 关键设计原则

1. **90% 用 Unary**——其他 3 种仅在必要时用
2. **流式必须处理 onError**——半路断流是常态
3. **服务端检测 isCancelled()**——及时停止 onNext
4. **客户端按需 request(n)**——流控
5. **Bidi 的关闭顺序**——先收齐所有响应再 onCompleted

## 14. 反模式

❌ **Client streaming 后忘了 onCompleted()**——服务端永远不会响应
❌ **服务端在 Bidi 中过早 onCompleted()**——客户端还没发完
❌ **onNext() 并发调用**——抛 IllegalStateException
❌ **不处理 onError**——资源泄漏
❌ **流式不带 deadline**——长连接永远活着

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **线协议**: [[draft-02-wire-protocol]]
- **拦截器**: [[draft-07-interceptors]]
- **错误处理**: [[draft-09-error-handling]]
- **综合入口**: [[summary]]