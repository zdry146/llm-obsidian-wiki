---
title: "gRPC 错误处理与状态码"
category: synthesis
tags: [grpc, error-handling, status-code, statusexception, trailers, deadline]
sources:
  - "gRPC Status Code Spec"
  - "Status / StatusException / StatusRuntimeException 源码"
  - "gRPC Error Model (google.rpc.Status)"
summary: "gRPC 16 个 Status Code + StatusException + StatusRuntimeException + Deadline / Cancel / 重试策略"
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

# §09 gRPC 错误处理与状态码

## 1. 16 个 gRPC Status Code

| Code | 数值 | HTTP 映射 | 含义 |
|------|------|-----------|------|
| **OK** | 0 | 200 | 成功 |
| **CANCELLED** | 1 | 499 | 操作被取消（客户端主动取消） |
| **UNKNOWN** | 2 | 500 | 未知错误（保留，不要用） |
| **INVALID_ARGUMENT** | 3 | 400 | 客户端参数无效 |
| **DEADLINE_EXCEEDED** | 4 | 504 | 超时 |
| **NOT_FOUND** | 5 | 404 | 资源不存在 |
| **ALREADY_EXISTS** | 6 | 409 | 资源已存在 |
| **PERMISSION_DENIED** | 7 | 403 | 权限不足 |
| **RESOURCE_EXHAUSTED** | 8 | 429 | 资源耗尽（限流、配额） |
| **FAILED_PRECONDITION** | 9 | 400 | 前置条件不满足 |
| **ABORTED** | 10 | 409 | 操作中止（并发冲突） |
| **OUT_OF_RANGE** | 11 | 400 | 参数超出范围 |
| **UNIMPLEMENTED** | 12 | 501 | 方法未实现 |
| **INTERNAL** | 13 | 500 | 内部错误（服务端 bug） |
| **UNAVAILABLE** | 14 | 503 | 服务不可用（重试安全） |
| **DATA_LOSS** | 15 | 500 | 数据丢失 |
| **UNAUTHENTICATED** | 16 | 401 | 未认证 |

**关键洞察**：
- gRPC 有自己的状态码系统（**不是 HTTP 状态码**）
- 16 个足够覆盖 99% 业务场景
- **HTTP status 永远 200**——错误在 Trailers 里

## 2. Status / StatusException / StatusRuntimeException

### 2.1 Status（不可变）

```java
public final class Status {
    public static final Status OK = Status.CODE_OK;  // Code 0
    public static final Status CANCELLED = ...;
    // ... 16 个常量
    
    public Code getCode();
    public String getDescription();
    public Throwable getCause();
    
    // Builder
    public Status withDescription(String description);
    public Status withCause(Throwable cause);
    
    // 转换
    public RuntimeException asRuntimeException(Metadata trailers);
    public StatusException asException(Metadata trailers);
    
    // 解析
    public static Status fromCodeValue(int codeValue);
    public static Status fromThrowable(Throwable t);
}
```

### 2.2 StatusException（Checked）

```java
Status status = Status.NOT_FOUND.withDescription("user not found");
throw status.asException(trailers);  // 受检异常
```

### 2.3 StatusRuntimeException（Unchecked）

```java
Status status = Status.NOT_FOUND.withDescription("user not found");
throw status.asRuntimeException(trailers);  // 运行时异常
```

## 3. 服务端发错误

### 3.1 Unary 发错误

```java
@Override
public void getUser(GetUserRequest req, StreamObserver<User> responseObserver) {
    User user = userRepository.findById(req.getId());
    if (user == null) {
        // 1. 主动关闭，状态码 NOT_FOUND
        responseObserver.onError(
            Status.NOT_FOUND
                .withDescription("user not found: " + req.getId())
                .asRuntimeException()
        );
        return;
    }
    responseObserver.onNext(user);
    responseObserver.onCompleted();
}
```

### 3.2 流式发错误

```java
@Override
public StreamObserver<Request> bidi(streamObserver) {
    return new StreamObserver<Request>() {
        @Override
        public void onError(Throwable t) {
            // 客户端发了非法数据
            // 关闭服务端流
            streamObserver.onError(
                Status.INVALID_ARGUMENT.withDescription("bad data").asRuntimeException()
            );
        }
        // ...
    };
}
```

### 3.3 发错误 metadata

```java
Metadata trailers = new Metadata();
trailers.put(Metadata.Key.of("retry-after", Metadata.ASCII_STRING_MARSHALLER), "30");

Status status = Status.RESOURCE_EXHAUSTED
    .withDescription("rate limit exceeded");
call.close(status, trailers);  // 服务端关闭
```

## 4. 客户端收错误

### 4.1 BlockingStub 错误捕获

```java
try {
    User user = stub.getUser(req);
} catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    status.getCode();              // Status.Code.NOT_FOUND
    status.getDescription();       // "user not found: 123"
    e.getTrailers();               // Metadata
    e.getCause();; // 原始异常
}
```

### 4.2 异步 Stub 错误处理

```java
async.getUser(req, new StreamObserver<User>() {
    @Override public void onNext(User user) { /* ... */ }
    @Override public void onError(Throwable t) {
        Status status = Status.fromThrowable(t);
        status.getCode();  // Status.Code.NOT_FOUND
    }
    @Override public void onCompleted() { /* ... */ }
});
```

### 4.3 FutureStub 错误处理

```java
ListenableFuture<User> future = futureStub.getUser(req);
try {
    User user = future.get(5, TimeUnit.SECONDS);
} catch (ExecutionException e) {
    Status status = Status.fromThrowable(e.getCause());
    // ...
} catch (TimeoutException e) {
    // Future.get 超时（不是 RPC 超时）
}
```

## 5. Deadline（超时）

### 5.1 客户端设 Deadline

```java
// 相对时间
stub.withDeadlineAfter(5, TimeUnit.SECONDS).getUser(req);

// 绝对时间
long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(5);
stub.withDeadline(deadline).getUser(req);
```

### 5.2 Deadline 传播（链式）

```java
// 客户端 A 调用服务端 B，B 又调 C
// A 给 B 设 deadline，B → C 时自动用 A 的 deadline
// 这是 gRPC 的 **Deadline Propagation** 机制

// 客户端 → metadata: grpc-timeout: 5S
// 服务端拦截器读 grpc-timeout，自动应用到下游调用
```

### 5.3 Deadline 超时处理

```java
try {
    stub.getUser(req);
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
        // 客户端 deadline 到了
        // 注意：服务端可能还在处理（但客户端已放弃）
    }
}
```

## 6. 取消（Cancellation）

### 6.1 客户端取消

```java
// 1. 直接取消
ClientCallStreamObserver<Request> obs = (ClientCallStreamObserver<Request>) observer;
obs.cancel("user navigated away", new CancellationException());

// 2. 通过 Context（推荐）
Context.CancellableContext withCancel = Context.current().withCancellation();
withCancel.run(() -> {
    // 业务调用
});
withCancel.cancel(new CancellationException("..."));  // 取消整个上下文
```

### 6.2 服务端检测取消

```java
ServerCallStreamObserver<Reply> obs = (ServerCallStreamObserver<Reply>) responseObserver;
if (obs.isCancelled()) {
    return;  // 客户端已经取消
}

// 注册取消回调
obs.setOnCancelHandler(() -> {
    // 清理资源
});
```

### 6.3 取消后状态

客户端发起取消后，**客户端侧的流状态**：
- onError(Status.CANCELLED) 被触发
- 服务端收到**CANCELLED** 状态码
- 不会发剩余响应

## 7. 错误重试策略

### 7.1 可重试的错误

| Status Code | 可重试？ | 理由 |
|-------------|---------|------|
| UNAVAILABLE (14) | ✅ | 服务暂时不可用，重试可能成功 |
| DEADLINE_EXCEEDED (4) | ⚠️ | 视场景（业务可能允许） |
| RESOURCE_EXHAUSTED (8) | ⚠️ | 需 backoff |
| ABORTED (10) | ✅ | 并发冲突，重试可能成功 |
| CANCELLED (1) | ❌ | 通常不应重试（重复工作） |

### 7.2 不可重试的错误

| Status Code | 不可重试 |
|-------------|---------|
| INVALID_ARGUMENT | 参数错，重试没用 |
| NOT_FOUND | 资源不存在 |
| PERMISSION_DENIED | 权限错 |
| UNAUTHENTICATED | 需重新认证 |
| INTERNAL | 服务端 bug，需先修复 |
| DATA_LOSS | 数据真丢失，重试没用 |

### 7.3 gRPC 内置重试策略

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress(...)
    .defaultServiceConfig(
        MethodDescriptor.newBuilder()
            .setRetryPolicy(
                RetryPolicy.newBuilder()
                    .setMaxAttempts(5)
                    .setInitialBackoff(Duration.ofMillis(100))
                    .setMaxBackoff(Duration.ofSeconds(5))
                    .setBackoffMultiplier(2.0)
                    .setRetryableCodes(
                        Status.Code.UNAVAILABLE,
                        Status.Code.RESOURCE_EXHAUSTED
                    )
                    .build()
            )
    )
    .build();
```

**gRPC 内置重试**：
- ✅ 服务端要支持（`grpc.service_config` 配置 + 服务端 idempotent）
- ⚠️ 仅适合幂等 RPC（GET、PUT、DELETE），不适合 POST

## 8. 错误处理最佳实践

### 8.1 服务端

```java
// ✅ 准确的状态码
responseObserver.onError(
    Status.NOT_FOUND.withDescription("user not found").asRuntimeException()
);

// ❌ 不要用 INTERNAL 掩盖一切
responseObserver.onError(
    Status.INTERNAL.withDescription("user not found").asRuntimeException()
);

// ❌ 不要抛 RuntimeException（不带状态码）
throw new RuntimeException("user not found");
```

### 8.2 客户端

```java
// ✅ 按状态码分支处理
try {
    User user = stub.getUser(req);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND:
            return null;  // 返回 null，业务自行决定
        case UNAVAILABLE:
            throw new RetryableException(...);  // 触发上游重试
        case DEADLINE_EXCEEDED:
            metrics.recordTimeout();
            throw new TimeoutException(...);
        default:
            throw e;
    }
}

// ❌ 不要 catch (Exception e)（丢失状态码）
catch (Exception e) { /* 不知道具体错误 */ }
```

### 8.3 自定义错误结构（google.rpc.Status）

```protobuf
import "google/rpc/status.proto";

message ErrorResponse {
  google.rpc.Status status = 1;
  // 自定义字段
  string user_message = 2;
}
```

```java
// 服务端
Status status = Status.NOT_FOUND
    .withDescription("user not found")
    .augmentDescription("user_id: 123, attempted_at: 2026-09-13");

responseObserver.onError(status.asRuntimeException(trailers));
```

## 9. 与 HTTP 错误码对比

| HTTP | gRPC | 含义 |
|------|------|------|
| 200 | OK | 成功 |
| 400 | INVALID_ARGUMENT | 参数错 |
| 401 | UNAUTHENTICATED | 未认证 |
| 403 | PERMISSION_DENIED | 权限不足 |
| 404 | NOT_FOUND | 资源不存在 |
| 409 | ALREADY_EXISTS / ABORTED | 冲突 |
| 429 | RESOURCE_EXHAUSTED | 限流 |
| 500 | INTERNAL / DATA_LOSS | 服务端错 |
| 503 | UNAVAILABLE | 服务不可用 |
| 504 | DEADLINE_EXCEEDED | 超时 |

**注意**：gRPC 状态码比 HTTP 状态码**更精确**——例如 ABORTED 和 ALREADY_EXISTS 都映射到 409，但语义不同。

## 10. 错误传播链

```
客户端 A
  │
  ▼ A 设 deadline = 5s
gRPC call (metadata: grpc-timeout: 5S)
  │
  ▼
服务端 B (ServerInterceptor 读 grpc-timeout)
  │
  ▼ B 用 A 的 deadline 调 C
gRPC call (metadata: grpc-timeout: 4.8S)  ← 减去 B 的开销
  │
  ▼
服务端 C 处理
  │
  ▼ 如果 4.8s 超时
DEADLINE_EXCEEDED 回传 B → 回传 A
```

**Deadline 传播** 是 gRPC 的关键优势——避免在链路上反复设置超时。

## 11. Metadata 携带错误细节

```java
// 服务端：错误时附详细 metadata
Metadata trailers = new Metadata();
trailers.put(Metadata.Key.of("error-code", Metadata.ASCII_STRING_MARSHALLER), "USER_LOCKED");
trailers.put(Metadata.Key.of("retry-after", Metadata.ASCII_STRING_MARSHALLER), "60");

call.close(Status.FAILED_PRECONDITION, trailers);

// 客户端：从 trailers 读
catch (StatusRuntimeException e) {
    Metadata trailers = e.getTrailers();
    String errorCode = trailers.get(Metadata.Key.of("error-code", Metadata.ASCII_STRING_MARSHALLER));
    // "USER_LOCKED"
}
```

## 12. 错误处理陷阱

### 12.1 错误的 Status 码

```java
// ❌ 用 INVALID_ARGUMENT 表示 "未找到"
Status.INVALID_ARGUMENT.withDescription("user not found");
// 应该用 NOT_FOUND
```

### 12.2 客户端忘 catch

```java
// ❌ 顶层忘 catch StatusRuntimeException
public UserDto getUser(String id) {
    return stub.getUser(req);  // 抛 StatusRuntimeException 直接到上层
}

// ✅ 业务层 catch 转换为业务异常
public UserDto getUser(String id) {
    try {
        return stub.getUser(req);
    } catch (StatusRuntimeException e) {
        switch (e.getStatus().getCode()) {
            case NOT_FOUND: throw new UserNotFoundException(id);
            case UNAVAILABLE: throw new ServiceUnavailableException();
            default: throw new ServiceException(e);
        }
    }
}
```

### 12.3 描述信息泄漏

```java
// ❌ 把内部错误暴露给客户端
Status.INTERNAL.withDescription(
    "SQL exception at UserRepository.findById(): column user_id not found in table 'users'"
);
// 应该 log 到服务端，只返回通用消息

// ✅
Status.INTERNAL.withDescription("Internal error").asRuntimeException();
log.error("Internal error", e);  // 详细 log 在服务端
```

## 13. 关键设计原则

1. **状态码精确**——用对的 Status Code，不滥用 INTERNAL
2. **描述简洁**——不泄漏内部信息
3. **客户端按状态码分支**——switch/case 处理
4. **Deadline 必设**——避免长 RPC 占资源
5. **重试仅幂等操作**——POST/INSERT 不重试

## 14. 反模式

❌ **所有错误都用 INTERNAL**——丢失错误类型
❌ **客户端 catch (Exception)**——丢失 Status 信息
❌ **不设 deadline**——长 RPC 占资源
❌ **重试所有错误**——非幂等操作会重复
❌ **描述信息泄漏服务端实现**——安全风险

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器**: [[draft-07-interceptors]]
- **最佳实践**: [[draft-10-best-practices]]
- **已知坑**: [[draft-11-known-issues]]
- **综合入口**: [[summary]]