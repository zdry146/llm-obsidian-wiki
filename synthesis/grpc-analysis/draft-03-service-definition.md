---
title: "gRPC 服务定义与代码生成"
category: synthesis
tags: [grpc, proto, protoc, codegen, service-definition, idl]
sources:
  - "Protobuf Language Guide (https://protobuf.dev/programming-guides/proto3/)"
  - "gRPC-Java Codegen Plugin"
  - "protoc-gen-grpc-java 源码"
summary: "gRPC 服务定义：.proto 语法 + protoc 编译 + Maven/Gradle 集成 + 生成代码结构"
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

# §03 gRPC 服务定义与代码生成

## 1. .proto 文件结构

```protobuf
// helloworld.proto
syntax = "proto3";                        // 强制 proto3

option java_multiple_files = true;        // 每个 message 独立 .java
option java_package = "io.grpc.helloworld";
option java_outer_classname = "HelloWorldProto";

package helloworld;                        // 命名空间

// 服务定义
service Greeter {
  // 4 种方法
  rpc SayHello (HelloRequest) returns (HelloReply);          // Unary
  rpc LotsOfReplies (HelloRequest) returns (stream HelloReply);     // Server streaming
  rpc LotsOfGreetings (stream HelloRequest) returns (HelloReply);   // Client streaming
  rpc BidiHello (stream HelloRequest) returns (stream HelloReply);  // Bidirectional
}

// 消息定义
message HelloRequest {
  string name = 1;                        // field number 1
}

message HelloReply {
  string message = 1;
}
```

## 2. 关键 Proto3 语法规则

### 2.1 Field Numbers（字段编号）

```protobuf
message User {
  string name = 1;      // 字段编号 1-15 用 1 字节（高频字段）
  int32 age = 2;
  string email = 16;    // 字段编号 16+ 用 2 字节
  // 删除字段用 reserved（防被复用）
  reserved 3, 4;
  reserved "old_field";
}
```

**设计原则**：
- **1-15**：1 字节编码，**高频字段**放这里
- **16-2047**：2 字节编码
- ⚠️ **不要删除字段后重用编号**——用 `reserved`

### 2.2 默认值（Defaults）

```protobuf
message Request {
  string name = 1;      // 默认 ""
  int32 count = 2;      // 默认 0
  bool enabled = 3;     // 默认 false
  repeated string tags = 4;  // 默认 []
}
```

⚠️ **proto3 区分不了"未设置"和"默认值"**——除非用 `optional`：

```protobuf
message Request {
  optional string name = 1;       // 区分未设置 vs ""
  string default_name = 1;        // 默认
}
```

### 2.3 Repeated 字段

```protobuf
message Order {
  repeated Item items = 1;        // 数组，0..N
}
```

### 2.4 Map 字段

```protobuf
message Config {
  map<string, string> labels = 1; // 等价于 entry + key/value
}
```

### 2.5 Any（任意类型）

```protobuf
import "google/protobuf/any.proto";

message Wrapper {
  google.protobuf.Any payload = 1;  // 装任意 message
}

// 设置
Wrapper w = Wrapper.newBuilder()
    .setPayload(Any.pack(someMessage))
    .build();

// 解包
Any a = w.getPayload();
if (a.is(HelloRequest.class)) {
    HelloRequest hr = a.unpack(HelloRequest.class);
}
```

### 2.6 Enum

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;        // 必须有 0 值（默认值）
  ACTIVE = 1;
  DELETED = 2;
}
```

⚠️ **枚举第一个值必须 0**——这是 Protobuf 序列化要求。

## 3. protoc 命令行编译

```bash
# 安装 protoc
# macOS: brew install protobuf
# Linux: apt-get install protobuf-compiler
# Windows: 下载 https://github.com/protocolbuffers/protobuf/releases

# 编译
protoc \
  --proto_path=src/main/proto \   # .proto 搜索路径
  --java_out=build/generated \    # Protobuf Java 输出
  --plugin=protoc-gen-grpc-java \ # gRPC Java 插件
  --grpc-java_out=build/generated \ # gRPC stub 输出
  src/main/proto/helloworld.proto
```

**生成产物**：
- `build/generated/.../HelloRequest.java`（Protobuf message）
- `build/generated/.../GreeterGrpc.java`（gRPC stub + service 基类）

## 4. Maven 集成

### 4.1 pom.xml

```xml
<dependencies>
    <!-- gRPC -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.66.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.66.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.66.0</version>
    </dependency>
    <!-- Protobuf -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.5</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
        </extension>
    </extensions>
</build>
```

### 4.2 完整 plugin 配置

```xml
<plugin>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.66.0:exe:${os.detected.classifier}</pluginArtifact>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>compile-custom</goal>  <!-- 触发 gRPC 插件 -->
            </goals>
        </execution>
    </executions>
</plugin>
```

**编译命令**：`mvn compile`（自动生成代码 + 编译）

## 5. Gradle 集成

```groovy
// build.gradle
plugins {
    id 'java'
    id 'com.google.protobuf' version '0.9.4'
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.25.5"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:1.66.0"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}

dependencies {
    implementation 'io.grpc:grpc-netty-shaded:1.66.0'
    implementation 'io.grpc:grpc-protobuf:1.66.0'
    implementation 'io.grpc:grpc-stub:1.66.0'
    compileOnly 'org.apache.tomcat:annotations-api:6.0.53'  // @Generated 注解
}
```

## 6. 生成代码结构

`protoc --java_out + --grpc-java_out` 生成两个文件：

### 6.1 Protobuf Message 生成（HelloRequest.java）

```java
package io.grpc.helloworld;

public final class HelloRequest extends com.google.protobuf.GeneratedMessageV3 {
    // 字段访问器
    public java.lang.String getName();
    public com.google.protobuf.ByteString getNameBytes();
    
    // Builder 模式
    public static Builder newBuilder();
    public Builder toBuilder();
    
    public static final class Builder extends com.google.protobuf.GeneratedMessageV3.Builder<Builder> {
        public Builder setName(java.lang.String value);
        public Builder clearName();
        // ...
    }
    
    // 解析（反序列化）
    public static HelloRequest parseFrom(byte[] data);
    public static HelloRequest parseFrom(InputStream input);
    
    // 序列化
    public byte[] toByteArray();
    public void writeTo(OutputStream output);
}
```

**特点**：纯 Builder 模式，无 setter。

### 6.2 gRPC Stub + Service 基类生成（GreeterGrpc.java）

```java
package io.grpc.helloworld;

public final class GreeterGrpc {
    public static final String SERVICE_NAME = "helloworld.Greeter";
    
    // ============ Service 基类（服务端继承）============
    public static abstract class GreeterImplBase implements io.grpc.BindableService {
        // 子类必须实现的业务方法
        public abstract void sayHello(HelloRequest request,
                                       io.grpc.stub.StreamObserver<HelloReply> responseObserver);
        public abstract void lotsOfReplies(HelloRequest request,
                                            io.grpc.stub.StreamObserver<HelloReply> responseObserver);
        // ... 其他方法
        
        @Override
        public final io.grpc.ServerServiceDefinition bindService() {
            return io.grpc.ServerServiceDefinition.builder(SERVICE_NAME)
                .addMethod(SayHello_METHOD, this::sayHello)
                .addMethod(LotsOfReplies_METHOD, this::lotsOfReplies)
                .build();
        }
    }
    
    // ============ Method Descriptors ============
    public static final io.grpc.MethodDescriptor<HelloRequest, HelloReply> SayHello_METHOD;
    // ... 其他 3 个 method descriptors
    
    // ============ BlockingStub（同步）============
    public static final class BlockingStub extends io.grpc.stub.AbstractStub<BlockingStub> {
        public HelloReply sayHello(HelloRequest request);
        public java.util.Iterator<HelloReply> lotsOfReplies(HelloRequest request);
        // ...
    }
    
    // ============ FutureStub（异步 Future）============
    public static final class FutureStub extends io.grpc.stub.AbstractStub<FutureStub> {
        public com.google.common.util.concurrent.ListenableFuture<HelloReply> sayHello(HelloRequest request);
        // ...
    }
    
    // ============ Stub（异步 StreamObserver）============
    public static final class Stub extends io.grpc.stub.AbstractStub<Stub> {
        public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver);
        public void lotsOfReplies(HelloRequest request, StreamObserver<HelloReply> responseObserver);
        // ...
    }
    
    // ============ Factory 方法 ============
    public static BlockingStub newBlockingStub(io.grpc.Channel channel);
    public static FutureStub newFutureStub(io.grpc.Channel channel);
    public static Stub newStub(io.grpc.Channel channel);
}
```

## 7. 服务端实现模板

```java
public class GreeterServer {
    // 1. 继承 .proto 生成的 GreeterImplBase
    static class GreeterImpl extends GreeterGrpc.GreeterImplBase {
        @Override
        public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
            HelloReply reply = HelloReply.newBuilder()
                .setMessage("Hello, " + req.getName())
                .build();
            responseObserver.onNext(reply);
            responseObserver.onCompleted();
        }
    }
    
    public static void main(String[] args) throws IOException {
        // 2. 创建 server，注册 service
        Server server = ServerBuilder.forPort(50051)
            .addService(new GreeterImpl())
            .build();
        
        server.start();
        System.out.println("Server started on " + server.getPort());
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutting down...");
            server.shutdown();
        }));
        server.awaitTermination();
    }
}
```

## 8. 客户端使用模板

```java
public class GreeterClient {
    public static void main(String[] args) {
        // 1. 创建 channel（生命周期 = 应用生命周期）
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("localhost", 50051)
            .usePlaintext()
            .build();
        
        // 2. 创建 stub
        GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
        
        // 3. 调用
        HelloRequest req = HelloRequest.newBuilder().setName("Mike").build();
        HelloReply resp = stub.sayHello(req);
        System.out.println(resp.getMessage());
        
        // 4. 关闭
        channel.shutdown();
    }
}
```

## 9. 跨语言契约（gRPC 杀手锏）

**同一份 .proto 在 11+ 语言生成完全相同的 stub**：

```
helloworld.proto（单一真相源）
   ├─→ Java:  GreeterGrpc.java
   ├─→ Go:    greeter_grpc.pb.go
   ├─→ Python: greeter_pb2.py + greeter_pb2_grpc.py
   ├─→ TypeScript: greeter_pb.ts + GreeterClient.ts
   └─→ ... 其他 7+ 语言
```

**实测**：服务端 Java、客户端 Go、服务端 C++、客户端 Dart，全部**类型安全、消息可互认**。

## 10. Schema 演进（向后兼容）

```protobuf
// v1
message User {
  string name = 1;
  int32 age = 2;
}

// v2：增加字段
message User {
  string name = 1;
  int32 age = 2;
  string email = 3;          // ✅ 新字段，老客户端忽略
  optional string phone = 4; // ✅ optional 字段
  
  reserved 5, 6;            // 保留字段编号（防复用）
  reserved "old_name";      // 保留字段名
}

// v3：删除字段
message User {
  string name = 1;
  int32 age = 2;
  string email = 3;
  // age 已删除，但保留编号 2（不删字段本身，只删使用）
}
```

**兼容性原则**：
- ✅ **可改**：添加字段、加 `optional`、改 enum 值
- ❌ **不能改**：删除字段、修改字段编号、改字段类型、改 wire type
- ⚠️ **reserved**：保留已删除的编号/名字，防新字段复用

## 11. 高级特性

### 11.1 Deadline（截止时间）

```protobuf
// 不在 .proto 里，在 Metadata 里
google.rpc.Deadline deadline = ...;
```

实际用法：

```java
stub.withDeadlineAfter(5, TimeUnit.SECONDS)  // 客户端设置
    .sayHello(req);
```

### 11.2 错误细节（google.rpc.Status）

```protobuf
import "google/rpc/status.proto";

message ErrorResponse {
  google.rpc.Status status = 1;
  // ... 自定义错误字段
}
```

### 11.3 OpenAPI 互转

```bash
# gRPC → OpenAPI（grpc-gateway）
protoc -I . \
  --grpc-gateway_out=... \
  --openapiv2_out=... \
  helloworld.proto
```

## 12. 关键工具

| 工具 | 用途 |
|------|------|
| **protoc** | Protobuf 编译器 |
| **protoc-gen-grpc-java** | Java gRPC 插件 |
| **protoc-gen-grpc-kotlin** | Kotlin gRPC 插件（推荐） |
| **buf** | 现代 Protobuf 工具链（schema registry, lint, breaking change detection） |
| **grpcurl** | 类 curl 工具，支持 server reflection |
| **grpc-cli** | Google 官方 CLI（功能较 grpcurl 弱） |

## 13. 推荐：使用 Buf

```bash
# buf.yaml
version: v1
modules:
  - path: proto
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT
```

```bash
# lint + breaking change check
buf lint
buf breaking --against '.git#branch=main'
```

**buf 的优势**：
- ✅ 更快的 protoc（并行）
- ✅ 内置 lint 规则
- ✅ Breaking change 检测
- ✅ 集中式 schema registry

## 14. 设计原则

1. **Schema-first**：先定义 .proto，再写代码
2. **向后兼容**：永远不破坏 wire format
3. **类型安全**：避免运行时错误
4. **跨语言**：一份 schema 11+ 语言
5. **生成的代码不修改**：用户代码通过继承基类实现

## 相关笔记

- **核心架构**: [[draft-01-architecture]]
- **客户端详解**: [[draft-04-client-side]]
- **服务端详解**: [[draft-05-server-side]]
- **最佳实践**: [[draft-10-best-practices]]
- **综合入口**: [[summary]]