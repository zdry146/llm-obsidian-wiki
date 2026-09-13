---
title: "gRPC 拦截器 (ClientInterceptor / ServerInterceptor)"
category: synthesis
tags: [grpc, interceptor, client-interceptor, server-interceptor, auth, tracing]
sources:
  - "gRPC-Java ClientInterceptor / ServerInterceptor 源码"
  - "gRPC User Guide - Interceptors"
summary: "gRPC 客户端拦截器 + 服务端拦截器详解：5 个实战模式 + 资源所有权 + 与 OkHttp interceptor 对比"
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

# §07 gRPC 拦截器（ClientInterceptor / ServerInterceptor）

## 1. 拦截器分类

| 拦截器 | 作用 |
|--------|------|
| **ClientInterceptor** | 拦截客户端发起的调用（出站） |
| **ServerInterceptor** | 拦截服务端接收的调用（入站） |

**核心特征**：
- 类似 OkHttp 的 Interceptor，但**更简单**（仅 1 层，无应用/网络之分）
- 客户端：1 个拦截器链
- 服务端：1 个拦截器链（独立于客户端）

## 2. ClientInterceptor

### 2.1 接口

```java
public interface ClientInterceptor {
    <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
        MethodDescriptor<ReqT, RespT> method,
        CallOptions callOptions,
        Channel next  // 下一个拦截器或最终 channel
    );
}
```

**返回**：`ClientCall<ReqT, RespT>`（包装的，可加 header / 监听响应）

### 2.2 ForwardingClientCall 简化

```java
public abstract class ForwardingClientCall<ReqT, RespT>
        extends ClientCall<ReqT, RespT> {
    protected abstract ClientCall<ReqT, RespT> delegate();
    
    // 默认实现转发所有方法到 delegate()
    // 子类只需 override 需要的方法
}
```

### 2.3 ForwardingClientCallListener 简化

```java
public abstract class ForwardingClientCallListener<RespT>
        extends ClientCall.Listener<RespT> {
    protected abstract ClientCall.Listener<RespT> delegate();
    
    // 默认转发所有方法
}
```

## 3. 实战模式 1：客户端鉴权

```java
public class AuthClientInterceptor implements ClientInterceptor {
    private final TokenProvider tokenProvider;
    
    public AuthClientInterceptor(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        
        return new ForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                // 1. 注入 token
                headers.put(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
                            "Bearer " + tokenProvider.getToken());
                
                // 2. 包装 listener（处理 401）
                super.start(new ForwardingClientCallListener<RespT>(responseListener) {
                    @Override
                    public void onClose(Status status, Metadata trailers) {
                        if (status.getCode() == Status.Code.UNAUTHENTICATED) {
                            // 触发 token refresh（具体逻辑见实际场景）
                            tokenProvider.refresh();
                        }
                        super.onClose(status, trailers);
                    }
                }, headers);
            }
        };
    }
}

// 注册
ManagedChannel channel = ManagedChannelBuilder.forAddress(...)
    .intercept(new AuthClientInterceptor(() -> tokenStore.getAccessToken()))
    .build();
```

**关键点**：
- **必须调 `super.start()`**——否则请求不发送
- **listener 必须包一层**——否则 onClose 等事件丢失
- **headers 在 start() 时塞入**——HTTP/2 HEADERS frame 一并发出

## 4. 实战模式 2：客户端监控

```java
public class MetricsClientInterceptor implements ClientInterceptor {
    private final MeterRegistry registry;
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        
        return new ForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                Timer.Sample sample = Timer.start(registry);
                String fullMethod = method.getFullMethodName();
                
                super.start(new ForwardingClientCallListener<RespT>(responseListener) {
                    @Override
                    public void onClose(Status status, Metadata trailers) {
                        sample.stop(registry.timer("grpc.client.duration",
                            "method", fullMethod,
                            "status", status.getCode().name()));
                        
                        if (!status.isOk()) {
                            registry.counter("grpc.client.errors",
                                "method", fullMethod,
                                "code", status.getCode().name()).increment();
                        }
                        
                        super.onClose(status, trailers);
                    }
                }, headers);
            }
        };
    }
}
```

## 5. 实战模式 3：客户端日志

```java
public class LoggingClientInterceptor implements ClientInterceptor {
    private final Logger logger = LoggerFactory.getLogger("grpc.client");
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        
        return new ForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                logger.info("→ {} {}", method.getType(), method.getFullMethodName());
                
                super.start(new ForwardingClientCallListener<RespT>(responseListener) {
                    @Override
                    public void onMessage(RespT message) {
                        logger.debug("  ← message: {}", message);
                        super.onMessage(message);
                    }
                    
                    @Override
                    public void onClose(Status status, Metadata trailers) {
                        if (status.isOk()) {
                            logger.info("← {} {} OK", 
                                method.getType(), method.getFullMethodName());
                        } else {
                            logger.warn("← {} {} {} - {}", 
                                method.getType(), method.getFullMethodName(),
                                status.getCode(), status.getDescription());
                        }
                        super.onClose(status, trailers);
                    }
                }, headers);
            }
        };
    }
}
```

## 6. 实战模式 4：客户端链路追踪（OpenTelemetry）

```java
public class TracingClientInterceptor implements ClientInterceptor {
    private final OpenTelemetry openTelemetry;
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        
        return new ForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                // 1. 启动 span
                Span span = openTelemetry.getTracer("grpc-client")
                    .spanBuilder(method.getFullMethodName())
                    .setSpanKind(SpanKind.CLIENT)
                    .startSpan();
                
                // 2. 注入 W3C Trace Context 到 gRPC metadata
                openTelemetry.getPropagators().getTextMapPropagator()
                    .inject(Context.current().with(span), headers, 
                        (carrier, key, value) -> 
                            carrier.put(Metadata.Key.of(key, Metadata.ASCII_STRING_MARSHALLER), 
                                        value));
                
                try (Scope scope = span.makeCurrent()) {
                    super.start(new ForwardingClientCallListener<RespT>(responseListener) {
                        @Override
                        public void onClose(Status status, Metadata trailers) {
                            // 3. 关闭 span
                            span.setStatus(StatusCode.error, status.getDescription());
                            if (status.isOk()) {
                                span.setStatus(StatusCode.OK);
                            }
                            span.end();
                            super.onClose(status, trailers);
                        }
                    }, headers);
                }
            }
        };
    }
}
```

## 7. ServerInterceptor

### 7.1 接口

```java
public interface ServerInterceptor {
    <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call,
        Metadata headers,
        ServerCallHandler<ReqT, RespT> next
    );
    
    // 返回包装的 Listener，调用 next.startCall() 进入下一层
}
```

**返回**：`ServerCall.Listener<ReqT>`（包装的，可加前置/后置处理）

### 7.2 ForwardingServerCallListener 简化

```java
public abstract class ForwardingServerCallListener<ReqT>
        extends ServerCall.Listener<ReqT> {
    protected abstract ServerCall.Listener<ReqT> delegate();
}
```

## 8. 实战模式 5：服务端鉴权

```java
public class AuthServerInterceptor implements ServerInterceptor {
    private static final Metadata.Key<String> AUTH_KEY = 
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    
    private final TokenValidator validator;
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // 1. 验证 token
        String auth = headers.get(AUTH_KEY);
        if (!validator.isValid(auth)) {
            // 2. 关闭调用
            call.close(Status.UNAUTHENTICATED
                .withDescription("invalid or missing token"),
                new Metadata());
            // 3. 返回空 Listener（不会再处理请求）
            return new ServerCall.Listener<ReqT>() {};
        }
        
        // 4. 放行：调用下一层
        return next.startCall(call, headers);
    }
}

// 注册
Server server = ServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .intercept(new AuthServerInterceptor())  // 第 1 个（最外层）
    .intercept(new MetricsServerInterceptor())  // 第 2 个
    .build();
```

**关键点**：
- **关闭调用用 `call.close()`，不是 `call.onMessage()`**
- **返回空 Listener**——请求不会被处理
- **必须调 `next.startCall()`**——否则业务代码不执行

## 9. 实战模式 6：服务端耗时监控

```java
public class MetricsServerInterceptor implements ServerInterceptor {
    private final MeterRegistry registry;
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        ServerCall<ReqT, RespT> wrapped = new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
            private long startTime;
            
            @Override
            public void sendMessage(RespT message) {
                super.sendMessage(message);
                if (startTime == 0) startTime = System.nanoTime();
            }
            
            @Override
            public void close(Status status, Metadata trailers) {
                if (startTime > 0) {
                    long duration = System.nanoTime() - startTime;
                    registry.timer("grpc.server.duration",
                        "method", call.getMethodDescriptor().getFullMethodName(),
                        "status", status.getCode().name())
                        .record(duration, TimeUnit.NANOSECONDS);
                }
                super.close(status, trailers);
            }
        };
        
        return next.startCall(wrapped, headers);
    }
}
```

## 10. 拦截器链执行顺序

### 10.1 客户端

```
ManagedChannel.newCall(method, options)
   │
   ▼
Interceptor1.interceptCall(...)
   │
   ▼
Interceptor2.interceptCall(...)
   │
   ▼
ChannelImpl.newCall(...)
   │
   ▼
ClientCallImpl.start(listener, headers)
   │
   ▼ (依次触发各层包装的 listener)
Interceptors → Service
```

**注册顺序 = 调用顺序**（先注册 = 先执行）。

### 10.2 服务端

```
ServerCall 被业务方法处理
   │
   ▼
ServerInterceptor1.interceptCall(...)
   │
   ▼
ServerInterceptor2.interceptCall(...)
   │
   ▼
ServerCallHandler.startCall(...)
   │
   ▼
ServerCallImpl
```

**注册顺序 = 调用顺序**。

## 11. 资源所有权（关键）

gRPC 拦截器的资源管理与 OkHttp 类似——**显式区分所有权**：

```java
@Override
public void start(Listener<RespT> responseListener, Metadata headers) {
    Span span = tracer.startSpan(...);
    
    try {
        super.start(new ForwardingClientCallListener<RespT>(responseListener) {
            @Override
            public void onClose(Status status, Metadata trailers) {
                try {
                    super.onClose(status, trailers);
                } finally {
                    span.end();  // listener 关闭时关 span
                }
            }
        }, headers);
    } catch (Exception e) {
        // 启动失败：清理 span
        span.end();
        throw e;
    }
}
```

**3 条铁律**：
1. **必须调 `super.start()`**——否则请求不发
2. **Listener 必须包一层**——否则下游事件丢失
3. **资源在 onClose 时清理**——保证任何路径都释放

## 12. 与 OkHttp Interceptor 对比

| 维度 | gRPC | OkHttp |
|------|------|--------|
| **拦截器层数** | 1 层（无应用/网络区分） | 4 层（应用/桥接/缓存/连接/网络/调用） |
| **数量限制** | 无 | 无 |
| **核心模式** | ForwardingClientCall 包装 | Chain.proceed |
| **listener 处理** | 必须包 ForwardingClientCallListener | 不需要 |
| **典型用途** | 鉴权、监控、追踪、压缩 | 同左 |
| **状态管理** | 简单（每次 RPC 1 个 Call） | 复杂（重试/连接池） |

## 13. 高级：拦截器传递 Metadata

```java
// 客户端发 metadata
public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(...) {
    return new ForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
        @Override
        public void start(Listener<RespT> listener, Metadata headers) {
            // 添加自定义 header
            headers.put(Metadata.Key.of("x-request-id", Metadata.ASCII_STRING_MARSHALLER),
                        UUID.randomUUID().toString());
            super.start(listener, headers);
        }
    };
}

// 服务端读 metadata
public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
    
    String requestId = headers.get(
        Metadata.Key.of("x-request-id", Metadata.ASCII_STRING_MARSHALLER));
    
    Context ctx = Context.current().withValue(REQUEST_ID_KEY, requestId);
    
    return Contexts.interceptCall(ctx, call, headers, next);
}
```

**`Contexts.interceptCall()`** 把 context 注入到当前线程，业务方法通过 `Context.current()` 读取。

## 14. 全局拦截器（服务发现 + 配置中心）

```java
public class ConfigClientInterceptor implements ClientInterceptor {
    private final ConfigCenter config;
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(...) {
        // 动态从配置中心读取 endpoint
        String endpoint = config.getEndpoint(method.getServiceName());
        // ...重定向逻辑
    }
}
```

## 15. 关键设计原则

1. **每个拦截器一个横切关注点**——鉴权、监控、日志、追踪分开
2. **ForwardingClientCall.SimpleForwardingServerCall**——简化包装
3. **Listener 必须包**——保证下游事件传递
4. **资源在 onClose 关**——try-finally 模式
5. **避免在 Interceptor 做业务**——业务在 Service

## 16. 反模式

❌ **忘调 super.start() / super.next.startCall()**——请求不发
❌ **Listener 不包**——onMessage/onClose 丢失
❌ **Interceptor 里做业务**——业务在 Service
❌ **资源没 finally**——异常路径泄漏
❌ **多个 Auth 拦截器**——职责重叠

## 17. 拦截器速查表

| 场景 | ClientInterceptor | ServerInterceptor |
|------|-------------------|---------------------|
| 鉴权 | ✅ Auth | ✅ Auth |
| 监控 | ✅ Metrics | ✅ Metrics |
| 日志 | ✅ Logging | ✅ Logging |
| 追踪 | ✅ OTel | ✅ OTel |
| 压缩 | ✅ Compressor | ✅ Decompressor |
| 重试 | ✅ Retry | ❌ 不常见 |
| 限流 | ✅ RateLimit | ✅ RateLimit |

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **客户端详解**: [[draft-04-client-side]]
- **服务端详解**: [[draft-05-server-side]]
- **错误处理**: [[draft-09-error-handling]]
- **OkHttp 拦截器对照**: [[okhttp-analysis/draft-02-interceptors]]
- **综合入口**: [[summary]]