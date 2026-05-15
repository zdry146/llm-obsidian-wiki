---
title: CICS Java现代化
category: concepts
tags: [cics, java, jcics, jcicsx, modernization, maven, gradle]
sources: [/home/openclaw/下载/CICS.pdf]
summary: 在CICS中引入Java：JCICS API调用CICS资源，JCICSX支持本地开发和Mock测试，Java程序可调用COBOL程序实现互操作。
provenance:
  extracted: 0.78
  inferred: 0.17
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# CICS Java现代化

## 为什么在CICS中使用Java

### 人才优势

COBOL程序员稀缺，Java程序员池更广 ^[extracted]。

### 协议和框架支持

Java拥有比其他语言（如C、COBOL）更丰富的协议和框架支持 ^[extracted]：
- JSON处理
- HTTP请求/响应
- RESTful服务

### zIIP引擎利用

Java代码主要可以在zIIP（IBM Z Integrated Information Processor）上运行，不消耗通用处理器容量，从而降低月度许可费用 ^[extracted]。

## CICS对Java的支持

CICS内置：
- **JVM服务器** — 在CICS Region内运行Java虚拟机
- **WebSphere Liberty应用处理器** — 运行Java EE或Spring Boot应用

CICS保证Java应用与C语言编写的COBOL应用享有同等的交易、安全和性能保障 ^[extracted]。

## JCICS API

JCICS是CICS为Java提供的原生API ^[extracted]。它允许Java程序：

- 访问CICS资源（文件、程序、终端等）
- 发起CICS交易
- 与COBOL程序互操作

**示例：** Java中的Hello World：
```java
public class App {
    public static void main(String[] args) {
        Task.getTask().out.println("Hello World");
    }
}
```

`Task.getTask()`获取当前CICS任务的引用，`.out`是CICS终端输出流 ^[extracted]。

## JCICSX — 增强的API

JCICSX是JCICS的现代扩展 ^[extracted]，在Maven Central提供。相比旧版JCICS，JCICSX提供：

- **Remoting** — 允许开发者在本地IDE中开发和测试Java CICS应用
- **Mocking** — 支持Mock对象，无需真实CICS环境即可运行单元测试

## Java调用COBOL程序

典型的用例：编写一个Java Servlet暴露COBOL业务逻辑作为REST API ^[extracted]：

```java
// 创建Channel和Container
String channelName = task.createChannel("comm");
ByteContainer input = task.createByteContainer(channelName, "old-commarea");
input.putString(inputData);

// 调用COBOL程序PAYBUS
ProgramLinker linker = task.createProgramLinkerWithChannel("PAYBUS", channelName);
linker.setInputCharContainer("old-commarea", inputData);
String result = linker.link().getOutputCharContainer("old-commarea").get();
```

关键方法：`createProgramLinkerWithChannel`创建链路器，设置输入Container，调用LINK，从输出Container获取结果 ^[extracted]。

## 审计日志示例

Java还可以直接调用CICS的瞬时数据队列（TDQ）来写审计日志 ^[extracted]：

```java
TDQ queue = new TDQ();
queue.setName("AUDIT");
queue.writeString(message);
```

无需调用单独的COBOL程序，CICS API直接处理TDQ写入 ^[extracted]。

## 本地开发和测试

### JCICSX Remoting

开发时，Java应用运行在本地笔记本上的应用服务器中。JCICSX API将CICS调用透明地转换为HTTP请求，发送到CICS Region中的JCICSX服务器端点 ^[extracted]。

### JCICSX Mocking

单元测试中，使用Mockito Mock掉所有JCICSX API对象 ^[extracted]：

```java
@Before
public void setUp() throws Exception {
    String containerData = "DISP 12345" + ...;
    task = Mockito.mock(CICSContext.class);
    cpl = Mockito.mock(ChannelProgramLinker.class);
    cplr = Mockito.mock(ChannelProgramLinkerResponse.class);
    rcc = Mockito.mock(ReadableCHARContainer.class);

    Mockito.when(task.createProgramLinkerWithChannel("PAYBUS1", "PAYROLL"))
        .thenReturn(cpl);
    Mockito.when(cpl.setStringInput("old-commarea", "DISP 12345"))
        .thenReturn(cpl);
    Mockito.when(cpl.link()).thenReturn(cplr);
    Mockito.when(cplr.getOutputCHARContainer("old-commarea")).thenReturn(rcc);
    Mockito.when(rcc.get()).thenReturn(containerData);
}
```

这样可以在没有CICS连接的情况下运行单元测试 ^[extracted]。

## 部署

开发完成后，Java WAR文件部署到CICS中的WebSphere Liberty JVM服务器 ^[extracted]。此时JCICSX API检测到运行在CICS内部，自动切换到标准内存调用而非HTTP remoting ^[extracted]。

## 构建工具

CICS TS V5.6引入了对Maven和Gradle的原生支持，Java库已发布到Maven Central ^[extracted]。

## 相关概念

- [[concepts/cics-channels-containers]] — COBOL程序从COMMAREA迁移到Channels/Containers
- [[concepts/cics-overview]] — CICS对混合语言的支持
- [[concepts/cics-async-event-processing]] — CICS TS V5.4引入的Java异步API
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 6 "Modernizing applications with Java", December 2020