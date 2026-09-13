---
title: "gRPC 已知坑与实战"
category: synthesis
tags: [grpc, known-issues, debugging, troubleshooting, performance, production]
sources:
  - "gRPC-Java GitHub Issues"
  - "gRPC Production Best Practices (Google SRE)"
  - "作者实战经验 + 社区案例"
summary: "gRPC 实战踩坑：连接泄漏 / 流式 OOM / 序列化兼容 / Deadline 传播 / 浏览器兼容 / gRPC-Web 限制"
provenance:
  extracted: 0.85
  inferred: 0.12
  ambiguous: 0.03
  base_confidence: 0.85
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §11 gRPC 已知坑与实战

## 1. 客户端坑

### 1.1 Channel 连接泄漏

**现象**：
- 应用启动正常
- 跑一段时间后所有 RPC 卡住
- 连接数飙升至上限

**根因**：
```java
// ❌ 每次调用新建 channel 不关闭
public UserDto getUser(String id) {
    ManagedChannel channel = ManagedChannelBuilder.forAddress(...).build();
    GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
    UserDto user = stub.getUser(req);
    channel.shutdown();  // 忘调？
    return user;
}
```

**修复**：
```java
// ✅ 单例 channel + JVM shutdown hook
public class GrpcClients {
    private static final ManagedChannel CHANNEL = ...;
    public static GreeterGrpc.GreeterBlockingStub newStub() {
        return GreeterGrpc.newBlockingStub(CHANNEL);
    }
}

// JVM 关闭时清理
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        GrpcClients.shutdownAll();
    } catch (Exception e) {
        log.error("Shutdown error", e);
    }
}));
```

### 1.2 BlockingStub 在 Netty event loop 阻塞

**现象**：
- 启动 Tomcat / Netty 等服务器，调用 gRPC 客户端
- 服务卡死，无响应

**根因**：
```java
// ❌ BlockingStub 阻塞 Netty event loop
@Override
protected void channelRead0(ChannelHandlerContext ctx, HttpRequest req) {
    UserDto user = stub.getUser(req);  // 阻塞！event loop 被占用
    // ...
}
```

**修复**：
```java
// ✅ 用 FutureStub 或 Stub (异步)
@Override
protected void channelRead0(ChannelHandlerContext ctx, HttpRequest req) {
    ListenableFuture<UserDto> future = futureStub.getUser(req);
    future.addListener(() -> {
        try {
            UserDto user = future.get();
            ctx.writeAndFlush(user);
        } catch (Exception e) {
            ctx.writeAndFlush(new ErrorResponse(e));
        }
    }, businessExecutor);  // 用业务线程池
}
```

### 1.3 Deadline 未设置 → 长 RPC 占资源

**现象**：
- 服务调用卡住，资源占用飙高
- 客户端永远等

**根因**：
```java
// ❌ 不设 deadline
stub.sayHello(req);
// RPC 永远不超时（除非服务端出错）
```

**修复**：
```java
// ✅ 每调用设 deadline
stub.withDeadlineAfter(5, TimeUnit.SECONDS).sayHello(req);

// 或全局默认
GreeterGrpc.newBlockingStub(channel).withDeadlineAfter(5, SECONDS);
```

### 1.4 StreamObserver 忘 onCompleted

**现象**：
- 服务端永远不响应
- 客户端 onNext 一直不来

**根因**：
```java
// ❌ Client streaming，客户端忘 onCompleted
StreamObserver<HelloRequest> observer = async.lotsOfGreetings(new StreamObserver<HelloReply>() {
    @Override public void onNext(HelloReply r) { /* ... */ }
    @Override public void onError(Throwable t) { /* ... */ }
    @Override public void onCompleted() { /* ... */ }
});

observer.onNext(req1);
observer.onNext(req2);
// 忘了 observer.onCompleted()
// 服务端一直等 → 永远不会响应
```

**修复**：
```java
// ✅ 显式关闭
observer.onNext(req1);
observer.onNext(req2);
observer.onCompleted();  // 必调
```

## 2. 服务端坑

### 2.1 maxInboundMessageSize 默认 4MB

**现象**：
- 客户端上传大文件失败
- Status.Code.RESOURCE_EXHAUSTED 或 INTERNAL

**根因**：
```java
// 默认 4MB，超出断流
Server server = ServerBuilder.forPort(50051).addService(...).build();
```

**修复**：
```java
// 调大
Server server = ServerBuilder.forPort(50051)
    .maxInboundMessageSize(64 * 1024 * 1024)  // 64MB
    .addService(...)
    .build();
```

### 2.2 Server streaming 不响应 onError

**现象**：
- 流中断后客户端一直在等
- 服务端资源占用飙高

**根因**：
```java
@Override
public void lotsOfReplies(HelloRequest req, StreamObserver<HelloReply> obs) {
    try {
        for (int i = 0; i < 1000; i++) {
            obs.onNext(reply(i));  // 如果某次抛异常，整个流挂掉
        }
    } catch (Exception e) {
        // ❌ 忘了 onError
    }
    obs.onCompleted();
}
```

**修复**：
```java
@Override
public void lotsOfReplies(HelloRequest req, StreamObserver<HelloReply> obs) {
    try {
        for (int i = 0; i < 1000; i++) {
            obs.onNext(reply(i));
        }
        obs.onCompleted();
    } catch (Exception e) {
        log.error("Stream error", e);
        obs.onError(Status.INTERNAL.withDescription("...").asRuntimeException());
        // 不要继续 onNext/onCompleted
    }
}
```

### 2.3 keepAlive 未设 → 防火墙 idle 断开

**现象**：
- 内网环境（NAT / 防火墙）部署
- 跑几天后 RPC 全失败

**根因**：
```java
// 默认 keepAlive 禁用
// NAT / 防火墙 idle 60s 后断开 TCP
```

**修复**：
```java
Server server = ServerBuilder.forPort(50051)
    .keepAliveTime(30, TimeUnit.SECONDS)            // 服务端主动 PING
    .keepAliveTimeout(10, TimeUnit.SECONDS)
    .permitKeepAliveWithoutCalls(true)               // 无 RPC 时也 PING
    .build();
```

### 2.4 Bidi 不处理 onError → 客户端卡死

**现象**：
- Bidi 流中途出错
- 客户端不响应

**根因**：
```java
@Override
public StreamObserver<HelloRequest> bidiHello(StreamObserver<HelloReply> obs) {
    return new StreamObserver<HelloRequest>() {
        @Override public void onNext(HelloRequest req) {
            try {
                obs.onNext(reply(req));
            } catch (Exception e) {
                // ❌ 忘了关闭服务端流
            }
        }
        // ...
    };
}
```

**修复**：
```java
@Override
public StreamObserver<HelloRequest> bidiHello(StreamObserver<HelloReply> obs) {
    return new StreamObserver<HelloRequest>() {
        @Override public void onNext(HelloRequest req) {
            try {
                obs.onNext(reply(req));
            } catch (Exception e) {
                log.error("Process error", e);
                obs.onError(Status.INTERNAL.asRuntimeException());
                // 关键：服务端流也关闭
            }
        }
        @Override public void onError(Throwable t) {
            log.error("Client error", t);
            obs.onError(Status.INVALID_ARGUMENT.asRuntimeException());
        }
        @Override public void onCompleted() {
            obs.onCompleted();
        }
    };
}
```

## 3. 序列化坑

### 3.1 Protobuf 向后兼容被破坏

**现象**：
- 新 schema 上线后旧客户端崩溃
- 反序列化失败

**根因**：
```protobuf
// ❌ 错误：修改字段编号
message User {
  string name = 1;
  int32 age = 2;
}

// v2 修改了字段编号
message User {
  string name = 1;      // 改成了 5
  int32 age = 5;        // ❌
}
```

**修复**：
```protobuf
// ✅ 用 reserved 防修改
message User {
  string name = 1;
  int32 age = 2;
  reserved 3, 4;        // 保留已删字段编号
  reserved "old_name";  // 保留已删字段名
}
```

### 3.2 proto2 vs proto3 混淆

**现象**：
- 客户端用 proto2，服务端用 proto3（或反之）
- 序列化不兼容

**根因**：
- proto2 和 proto3 wire format **部分不兼容**（如 default 值处理）

**修复**：
- ✅ 统一 `syntax = "proto3"`
- ✅ 用 Buf 工具检查兼容性

### 3.3 数值类型溢出

**现象**：
- int32 存超过 21亿 → 溢出
- int64 序列化错误

**修复**：
```protobuf
// ❌ int32 存 32 位 ID
message Item {
  int32 user_id = 1;
}
// user_id = 3000000000 溢出

// ✅ int64
message Item {
  int64 user_id = 1;
}
```

## 4. 性能坑

### 4.1 默认连接限制

**现象**：
- 单连接 RPC 上限受 maxConcurrentCallsPerConnection 限制
- 高 QPS 时延迟飙高

**修复**：
```java
Server server = ServerBuilder.forPort(50051)
    .maxConcurrentCallsPerConnection(500)  // 单连接并发
    .build();
```

### 4.2 客户端单 Channel 性能

**现象**：
- 单 Channel 1000+ QPS 时延迟高
- 瓶颈在 HTTP/2 多路复用

**修复**：
- 多个 Channel 共享 LoadBalancer（gRPC 不支持）
- 或者拆分服务（不同 endpoint）

### 4.3 大消息未压缩

**现象**：
- 1MB JSON-like 消息 → Protobuf 仍 1MB+
- 网络带宽瓶颈

**修复**：
```java
stub.withCompression("gzip").upload(req);
// 1MB → 200KB（典型比例）
```

## 5. 浏览器兼容

### 5.1 浏览器不能直接用 gRPC

**根因**：
- 浏览器只能发 HTTP/1.1 请求
- 不能直接控制 HTTP/2 帧

**解决方案 1：gRPC-Web**

```javascript
import {GreeterClient} from './greeter_grpc_web_pb.js';

const client = new GreeterClient('https://api.example.com');
const request = new proto.helloworld.HelloRequest();
request.setName('World');

client.sayHello(request, {}, (err, response) => {
    if (err) console.error(err);
    else console.log(response.getMessage());
});
```

**gRPC-Web 限制**：
- ❌ **不支持流式**（Unary only）
- ✅ 仅 1.0 版本以上

**解决方案 2：Connect（Buf 推荐）**

```typescript
// Connect 支持 Unary + Stream + 浏览器 + Node.js
import { createPromiseClient } from "@bufbuild/connect";
import { createConnectTransport } from "@bufbuild/connect-web";

const transport = createConnectTransport({ baseUrl: "https://api.example.com" });
const client = createPromiseClient(GreeterService, transport);

const response = await client.sayHello({ name: "World" });
console.log(response.message);
```

**Connect 优势**：
- ✅ 浏览器 + Node.js
- ✅ 支持流式（用 fetch + ReadableStream）
- ✅ 兼容 gRPC server（也支持 Connect protocol）

## 6. 监控 / 调试

### 6.1 grpcurl 缺失

**现象**：
- 第三方团队想调试 gRPC 服务但不知道方法列表

**修复**：
```java
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(ProtoReflectionService.newInstance())  // ←
    .build();

// grpcurl 可用
// $ grpcurl -plaintext localhost:50051 list
// helloworld.Greeter
// grpc.health.v1.Health
// grpc.reflection.v1alpha.ServerReflection
```

### 6.2 链路追踪断链

**现象**：
- 服务端看不到客户端 trace ID
- 链路追踪断掉

**根因**：
- 客户端 interceptor 没注入 trace context

**修复**：
```java
// 客户端 interceptor 注入 trace context（详见 [[draft-07-interceptors]]）
// 服务端 interceptor 读 trace context
```

## 7. K8s 部署坑

### 7.1 Service 不是 headless

**现象**：
- DNS 只解析到 ClusterIP
- 客户端只连 1 个 Pod

**根因**：
```yaml
# 默认 Service 有 ClusterIP
spec:
  clusterIP: <auto>  # 不是 None
```

**修复**：
```yaml
spec:
  clusterIP: None  # headless
```

### 7.2 Pod 启动慢导致 DNS 失败

**现象**：
- Pod 启动慢（> K8s service ready 时间）
- 客户端启动时 DNS 解析失败

**修复**：
- ✅ readinessProbe 用 gRPC 健康检查
- ✅ 客户端 retry / fail-fast 合理配置

## 8. 实战：微服务 RPC 卡死排查

### 8.1 现象
- 服务 A 调服务 B 的 gRPC
- B 处理慢，A 卡死

### 8.2 排查清单

1. **客户端有无设 deadline？**——`stub.withDeadlineAfter(...)`
2. **服务端是阻塞调用？**——业务是否在 Netty event loop 阻塞
3. **连接是否 alive？**——keepAlive 配置
4. **服务端负载？**——CPU / DB / 锁
5. **DNS / 路由？**——endpoint 列表是否准确
6. **gRPC 版本兼容？**——client 和 server 版本差异

### 8.3 调试工具

```bash
# grpcurl 探活
grpcurl -plaintext -d '{"name":"mike"}' \
    localhost:50051 helloworld.Greeter/SayHello

# 启用详细日志
-Dio.grpc.netty.shaded.io.netty.handler.logging.LoggingHandler.level=DEBUG

# JFR (Java Flight Recorder) 录制
java -XX:+UnlockCommercialFeatures -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -jar myapp.jar
```

## 9. 总结：8 个高频坑

| # | 坑 | 频率 | 严重性 |
|---|----|------|--------|
| 1 | Channel 泄漏（忘 shutdown） | 🔴 高 | 🔴 严重 |
| 2 | BlockingStub 阻塞 event loop | 🔴 高 | 🔴 严重 |
| 3 | 不设 deadline | 🟡 中 | 🔴 严重 |
| 4 | StreamObserver 忘 onCompleted | 🟡 中 | 🟡 中 |
| 5 | maxInboundMessageSize 默认 4MB | 🟡 中 | 🟡 中 |
| 6 | Protobuf 兼容破坏 | 🟢 低 | 🔴 严重 |
| 7 | keepAlive 未配置 | 🟡 中 | 🟡 中 |
| 8 | 流式忘处理 onError | 🟡 中 | 🟡 中 |

## 10. 关键经验

1. **Channel 单例 + JVM shutdown**——永远不忘记
2. **每调用设 deadline**——避免长 RPC 占资源
3. **大消息调 maxInboundMessageSize**——防 DoS
4. **流式必处理 onError**——避免资源泄漏
5. **Proto3 + Buf 检查兼容性**——避免破坏

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器**: [[draft-07-interceptors]]
- **错误处理**: [[draft-09-error-handling]]
- **最佳实践**: [[draft-10-best-practices]]
- **综合入口**: [[summary]]