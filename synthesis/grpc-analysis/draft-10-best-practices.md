---
title: "gRPC 最佳实践"
category: synthesis
tags: [grpc, best-practices, channel, deadline, compression, observability]
sources:
  - "gRPC Performance Best Practices"
  - "gRPC Health Checking Spec"
  - "CNCF gRPC patterns"
summary: "gRPC 生产级最佳实践：Channel 复用 / Deadline / 压缩 / 健康检查 / 可观测性 / 上下文传播"
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

# §10 gRPC 最佳实践

## 1. Channel 管理

### 1.1 Channel 复用（必须）

```java
// ✅ 单例
public class GrpcClients {
    private static final ManagedChannel CHANNEL = ManagedChannelBuilder
        .forAddress("api.example.com", 443)
        .useTransportSecurity()
        .build();
    
    public static GreeterGrpc.GreeterBlockingStub newBlockingStub() {
        return GreeterGrpc.newBlockingStub(CHANNEL)
            .withDeadlineAfter(5, TimeUnit.SECONDS);
    }
}

// ❌ 每次新建
public UserDto getUser(String id) {
    ManagedChannel channel = ManagedChannelBuilder.forAddress(...).build();
    GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
    UserDto user = stub.getUser(req);
    channel.shutdown();  // 浪费连接池资源
    return user;
}
```

**理由**：
- Channel 内部有 HTTP/2 连接 + Netty EventLoopGroup + NameResolver
- 每次新建 = 新建 TCP + TLS 握手（100-300ms）

### 1.2 Channel 关闭时机

```java
// ✅ JVM shutdown 时统一关闭
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        channel.shutdown().awaitTermination(30, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        channel.shutdownNow();
    }
}, "grpc-shutdown"));

// 或用 Spring / Micronaut 等框架的 lifecycle
```

### 1.3 多服务端多 Channel

```java
// 同一 Channel 不能跨不同 endpoint（除非用 LB）
ManagedChannel orderChannel = ...;
ManagedChannel userChannel = ...;  // 不同 Channel
```

## 2. Deadline（超时）必设

### 2.1 Per-call Deadline

```java
// 短超时
stub.withDeadlineAfter(1, TimeUnit.SECONDS).sayHello(req);  // 健康检查

// 中超时
stub.withDeadlineAfter(5, TimeUnit.SECONDS).getUser(req);  // 普通 RPC

// 长超时
stub.withDeadlineAfter(60, TimeUnit.SECONDS).upload(req);  // 上传
```

### 2.2 Deadline 传播

```java
// 客户端 → 服务端自动传播 deadline
// 链路：A → B → C，A 给 B 5s，B 自动给 C 5s

// 服务端不需要自己设 deadline，让上游传过来
@Override
public void getUser(GetUserRequest req, StreamObserver<User> obs) {
    // 不需要 stub.withDeadlineAfter()，上游的 deadline 自动应用
    // Context.current().getDeadline() 可读当前 deadline
}
```

### 2.3 Context 读取 Deadline

```java
@Override
public void getUser(GetUserRequest req, StreamObserver<User> obs) {
    Deadline deadline = Context.current().getDeadline();
    if (deadline != null && deadline.timeRemaining(TimeUnit.MILLISECONDS) < 100) {
        // 留给下游的时间不够了
        obs.onError(Status.DEADLINE_EXCEEDED.asRuntimeException());
        return;
    }
    // ...
}
```

## 3. 压缩

### 3.1 全局配置

```java
// Server
ServerBuilder.forPort(50051)
    .compressorRegistry(CompressorRegistry.getDefaultInstance())  // gzip
    .decompressorRegistry(DecompressorRegistry.getDefaultInstance())
    .build();

// Client
ManagedChannelBuilder.forAddress(...)
    .compressorRegistry(CompressorRegistry.getDefaultInstance())
    .decompressorRegistry(DecompressorRegistry.getDefaultInstance())
    .build();
```

### 3.2 Per-call 覆盖

```java
stub.withCompression("gzip").sayHello(req);  // 这次用 gzip
```

### 3.3 自定义 Compressor

```java
// 1. 实现 Compressor
public class ZstdCompressor implements Compressor {
    @Override public InputStream compress(InputStream is) throws IOException {
        return new ZstdInputStream(is);
    }
    @Override public OutputStream compress(OutputStream os) throws IOException {
        return new ZstdOutputStream(os);
    }
    @Override public String getMessageEncoding() { return "zstd"; }
}

// 2. 注册
CompressorRegistry registry = CompressorRegistry.newEmptyInstance();
registry.register(new ZstdCompressor());

ManagedChannel channel = ManagedChannelBuilder.forAddress(...)
    .compressorRegistry(registry)
    .build();
```

## 4. TLS / mTLS

### 4.1 Server 端 TLS

```java
Server server = ServerBuilder.forPort(8443)
    .useTransportSecurity(
        Certs.loadCertChain("server.crt"),
        Certs.loadPrivateKey("server.key"))
    .addService(new GreeterImpl())
    .build();
```

### 4.2 Client 端 TLS

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress(...)
    .useTransportSecurity()  // 默认用 JVM trust store
    .build();

// 自定义 trust store
SslContext sslContext = ...;
ManagedChannelBuilder.forAddress(...)
    .useTransportSecurity()
    .sslContext(sslContext)
    .build();
```

### 4.3 mTLS（双向认证）

```java
// Server
SslContext serverSslContext = GrpcSslContexts
    .configure(SslContextBuilder.forServer(
        certChainFile, privateKeyFile))
    .trustManager(clientCertChainFile)  // 验客户端证书
    .clientAuth(ClientAuth.REQUIRE)
    .build();

Server server = NettyServerBuilder.forPort(8443)
    .sslContext(serverSslContext)
    .addService(...)
    .build();
```

## 5. 可观测性

### 5.1 监控（Metrics）

```java
// 用 Micrometer + grpc-java-contrib
MetricsClientInterceptor metricsClient = MetricsClientInterceptor
    .create(MeterRegistryHolder.getRegistry());

MetricsServerInterceptor metricsServer = MetricsServerInterceptor
    .create(MeterRegistryHolder.getRegistry());

Server server = ServerBuilder.forPort(50051)
    .addService(...)
    .intercept(metricsServer)
    .build();

ManagedChannel channel = ManagedChannelBuilder.forAddress(...)
    .intercept(metricsClient)
    .build();
```

**自动暴露指标**：
- `grpc.server.requests.*`（count, duration）
- `grpc.client.requests.*`（按 status / method 分桶）
- `grpc.server.active.calls`

### 5.2 日志

```java
public class LoggingInterceptor implements ClientInterceptor, ServerInterceptor {
    // 详见 [[draft-07-interceptors]]
}
```

### 5.3 链路追踪（OpenTelemetry）

```java
// grpc-java 集成 OpenTelemetry
// 自动从 metadata 提取 trace context
// 自动注入到 metadata

// 详见 [[draft-07-interceptors]]
```

## 6. 健康检查

### 6.1 标准 HealthService

```java
HealthStatusManager health = new HealthStatusManager();
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .addService(health.getHealthService())  // ← 标准服务
    .build();

// 业务方法设状态
health.setStatus("helloworld.Greeter", ServingStatus.SERVING);

// 业务不可用时
health.setStatus("helloworld.Greeter", ServingStatus.NOT_SERVING);
```

### 6.2 Kubernetes liveness/readiness

```yaml
# deployment.yaml
livenessProbe:
  grpc:
    port: 50051
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  grpc:
    port: 50051
  initialDelaySeconds: 3
  periodSeconds: 5
```

**K8s 直接支持 gRPC 健康检查**（无需额外 HTTP 端点）。

## 7. 客户端负载均衡

### 7.1 DNS Headless Service

```yaml
# K8s Service
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  clusterIP: None  # Headless
  ports:
  - port: 50051
```

```java
// 客户端
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("api-service.default.svc.cluster.local:50051")
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

### 7.2 xDS（生产推荐）

```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("xds:///api-service.default.svc.cluster.local")
    .build();
// xDS 自动从 Envoy / Istio 拉取所有配置
```

## 8. 上下文传播

### 8.1 客户端：metadata 注入

```java
// 通过 Interceptor 注入 trace ID
public class TracingInterceptor implements ClientInterceptor {
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(...) {
        return new ForwardingClientCall<ReqT, RespT>(next.newCall(...)) {
            @Override
            public void start(Listener<RespT> listener, Metadata headers) {
                headers.put(Metadata.Key.of("trace-id", Metadata.ASCII_STRING_MARSHALLER),
                            getCurrentTraceId());
                super.start(listener, headers);
            }
        };
    }
}
```

### 8.2 服务端：Context 读取

```java
// 服务端拦截器：把 metadata 注入 Context
public class TracingServerInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        
        String traceId = headers.get(TRACE_ID_KEY);
        Context ctx = Context.current().withValue(TRACE_ID_KEY, traceId);
        return Contexts.interceptCall(ctx, call, headers, next);
    }
}

// 业务方法读
@Override
public void sayHello(HelloRequest req, StreamObserver<HelloReply> obs) {
    String traceId = TRACE_ID_KEY.get(Context.current());
    // ...
}
```

## 9. 错误处理

### 9.1 客户端 switch 分支

```java
try {
    return stub.getUser(req);
} catch (StatusRuntimeException e) {
    return switch (e.getStatus().getCode()) {
        case NOT_FOUND -> null;  // 业务自行处理
        case UNAVAILABLE -> {
            metrics.recordRetryable();
            throw new RetryableException(...);
        }
        case DEADLINE_EXCEEDED -> {
            metrics.recordTimeout();
            throw new TimeoutException(...);
        }
        default -> throw e;
    };
}
```

### 9.2 不要用 HTTP 错误码转换 gRPC

```java
// ❌ 错误：把 gRPC 状态码映射为 HTTP
if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
    return ResponseEntity.status(404).body(...);
}

// ✅ 正确：业务直接处理 gRPC 错误
if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
    return null;
}
```

## 10. 性能优化

### 10.1 批处理（Batching）

```java
// ❌ 100 个 RPC 每个 1 条
for (User user : users) {
    stub.getUser(GetUserRequest.newBuilder().setId(user.getId()).build());
}

// ✅ 1 个 RPC 100 条（用 stream / batch API）
stub.getUsers(BatchRequest.newBuilder().addAllIds(ids).build());
```

### 10.2 连接复用

```java
// ✅ 一个 Channel 多个 Stub
ManagedChannel channel = ...;
GreeterGrpc.GreeterBlockingStub stub1 = GreeterGrpc.newBlockingStub(channel);
GreeterGrpc.GreeterFutureStub stub2 = GreeterGrpc.newFutureStub(channel);
// 共享同一 HTTP/2 连接
```

### 10.3 大消息压缩

```java
// 大消息（> 1MB）建议 gzip
stub.withCompression("gzip").download(req);
```

### 10.4 流式替代轮询

```java
// ❌ 客户端轮询
while (true) {
    Status status = stub.getJobStatus(req);
    if (status.isDone()) break;
    Thread.sleep(1000);
}

// ✅ Server streaming
stub.watchJob(req, new StreamObserver<JobUpdate>() {
    @Override public void onNext(JobUpdate update) {
        // 实时收到更新
    }
    // ...
});
```

## 11. 测试

### 11.1 InProcessServer（单元测试）

```java
@Test
public void testSayHello() {
    // 1. 启动 in-process server
    Server server = InProcessServerBuilder.forName("test-server")
        .addService(new GreeterImpl())
        .build();
    server.start();
    
    // 2. 用 in-process channel
    ManagedChannel channel = InProcessChannelBuilder.forName("test-server").build();
    
    // 3. 调用 stub
    GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
    HelloReply reply = stub.sayHello(HelloRequest.newBuilder().setName("test").build());
    
    // 4. 断言
    assertEquals("Hello, test", reply.getMessage());
    
    // 5. 清理
    channel.shutdown();
    server.shutdown();
}
```

### 11.2 MockStub（不需要启动 server）

```java
@Test
public void testWithMock() {
    // mock server 行为
    ServiceDescriptor service = GreeterGrpc.getServiceDescriptor();
    MockService mockService = new MockService(service);
    mockService.addMethod(
        GreeterGrpc.SayHello_METHOD,
        new HelloRequest(),  // 入参匹配
        new HelloReply()    // 返回
    );
    
    ManagedChannel channel = InProcessChannelBuilder.forName("mock")
        .directExecutor()
        .build();
    
    GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
    HelloReply reply = stub.sayHello(...);
}
```

## 12. K8s 部署最佳实践

### 12.1 Sidecar 模式

```yaml
# Pod with Istio sidecar
spec:
  containers:
  - name: myapp
    image: myorg/myapp:1.0
    ports:
    - containerPort: 50051
  - name: istio-proxy
    image: docker.io/istio/proxyv2:1.20.0
    # 自动接管进出流量
```

**好处**：
- mTLS 自动
- 重试 / 超时配置统一
- 流量镜像 / A/B 测试
- 全链路遥测

### 12.2 独立 gRPC 端口

```yaml
# Service 暴露 gRPC 端口
ports:
- name: grpc
  port: 50051
  targetPort: 50051
- name: http
  port: 8080
  targetPort: 8080
```

**gRPC port 独立**，HTTP port（如 metrics/健康检查）单独。

## 13. 14 条铁律

1. **Channel 复用**——不每次新建
2. **必设 Deadline**——避免长 RPC 占资源
3. **传压缩**——gzip / snappy / zstd
4. **状态码精确**——用对的 Status Code
5. **Deadline 传播**——让上游 deadline 自动传到下游
6. **StreamObserver 必须包 ForwardingClientCallListener**——保留下游事件
7. **客户端按状态码分支**——switch/case 处理
8. **生产用 xDS**——与 Istio 集成
9. **必加 ProtoReflectionService**——grpcurl 能用
10. **必加 HealthService**——K8s liveness/readiness
11. **shutdown hook 必加**——优雅关闭
12. **Server streaming 检测 isCancelled**——及时停止
13. **不要把 gRPC 状态码映射为 HTTP**——直接业务处理
14. **大消息限 maxInboundMessageSize**——防 DoS

## 14. 反模式

❌ **每次 RPC new channel**——资源浪费
❌ **不设 deadline**——长 RPC 永远不结束
❌ **客户端 catch (Exception)**——丢失 Status 信息
❌ **BlockingStub 在 Netty event loop 调**——死锁
❌ **重试非幂等操作**——重复副作用
❌ **错误描述泄漏内部信息**——安全风险
❌ **所有错误用 INTERNAL**——丢失错误类型

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **拦截器**: [[draft-07-interceptors]]
- **错误处理**: [[draft-09-error-handling]]
- **已知坑**: [[draft-11-known-issues]]
- **综合入口**: [[summary]]