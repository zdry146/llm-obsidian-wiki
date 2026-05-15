---
title: CICS异步编程和事件处理
category: concepts
tags: [cics, async, event-processing, parent-child, signal-event, devops]
sources: [/home/openclaw/下载/CICS.pdf]
summary: CICS TS V5.4引入异步编程API（RUN TRANSID/FETCH CHILD/FETCH ANY）简化父子任务模型；事件处理支持在不改写COBOL代码的情况下获取业务洞察。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# CICS异步编程和事件处理

## 异步编程

### 传统方式的困难

在CICS TS V5.4之前，实现异步编程需要复杂的组合使用 `EXEC CICS START`、`EXEC CICS DELAY` 加轮询，或 `EXEC CICS WAIT`/`POST` 组合 ^[extracted]。这些技术容易出错且难以调试 ^[extracted]。

### CICS异步API（CICS TS V5.4引入）

新的异步API围绕**父子编程模型**构建 ^[extracted]：

| API命令 | 用途 |
|---|---|
| `EXEC CICS RUN TRANSID` | 启动一个异步子交易 |
| `EXEC CICS FETCH CHILD` | 查询特定子任务的状态 |
| `EXEC CICS FETCH ANY` | 查询任意已完成的子任务 |
| `EXEC CICS FREE CHILD` | 丢弃未响应的子任务 |
| `PUT CONTAINER / GET CONTAINER` | 在父子任务间传递数据 |

### 父-子任务模型

**工作原理 ^[extracted]：**

1. 父任务创建子任务：`EXEC CICS RUN TRANSID(CHILD1) CHANNEL(payroll)`
2. 父任务通过 `PUT CONTAINER` 向子任务传递数据
3. 父任务可以继续处理，**不被子任务阻塞**（不像LINK会等待）
4. 子任务作为独立任务在同一Region中运行
5. 父任务准备好后调用 `FETCH CHILD` 等待结果（**不轮询**）
6. 子任务完成时用 `PUT CONTAINER` 将结果传回
7. 父任务用 `GET CONTAINER` 获取结果

```
父任务 ──RUN TRANSID──> 子任务（并行运行）
父任务 ──继续处理──>（不等子任务）
父任务 ──FETCH CHILD──>（等待子任务完成）
子任务 ──PUT CONTAINER──> 结果
父任务 ──GET CONTAINER──> 获取结果
```

### 异步编程的三原则

1. **将工作拆分为可独立运行的任务** — 可以并行处理的部分分解出来
2. **跟踪异步工作的完成状态** — 不需要轮询
3. **在主任务和异步任务间安全传递数据** — 使用Container

## 事件处理

### 背景

传统CICS应用的业务逻辑锁定在代码中，难以在不修改代码的情况下获取业务洞察 ^[extracted]。事件处理提供了一种**无代码侵入**的方式来监控业务事件。

### 事件处理架构

```
应用程序 --EXEC CICS命令--> 捕获点检查 --> 事件数据 --> 适配器 --> 外部事件消费者
                                           ↑
                                   可选: SIGNAL EVENT（显式）
                                   或: 事件绑定定义（隐式）
```

### 捕获点

捕获点是事件检测的触发点 ^[extracted]：

1. **基于现有EXEC CICS命令子集** — 由事件绑定的捕获规范定义
2. **自定义SIGNAL EVENT命令** — 在程序中显式发出事件

### 事件绑定（Event Binding）

通过IBM CICS Explorer创建事件绑定，包含两个规范 ^[extracted]：

- **事件规范（Event Specification）** — 定义业务事件名称和传递给消费者的数据
- **捕获规范（Capture Specification）** — 定义触发事件的EXEC CICS命令和过滤条件（如程序名）

### 适配器

适配器是将事件格式化和发送到外部消费者的程序 ^[extracted]。常见消费者：IBM Operational Decision Manager（ODM）等决策引擎 ^[extracted]。

### 无代码侵入的事件检测

当无法修改COBOL源代码时（如源代码已丢失），CICS可以根据事件绑定自动生成业务事件，无需修改应用程序 ^[extracted]。

### 投注公司示例

场景：投注金额超过$10,000的交易需要监控 ^[extracted]。

- **事件规范：** 事件名=BET-HIGH-VALUE，传递客户姓名和投注点位置
- **捕获规范：** 捕获点=到PLACEBET程序的EXEC CICS LINK，事件触发条件=betting程序调用PLACEBET
- **适配器：** 启动新CICS交易BET1处理事件

## Link to WebSphere Liberty

CICS TS V5.3引入Link to WebSphere Liberty特性，允许非Java CICS程序启动WebSphere Liberty JVM服务器中的Java Platform, Enterprise Edition应用 ^[extracted]。

### 工作原理

Java方法用`@CICSProgram`注解标记 ^[extracted]：

```java
@CICSProgram("CUSTGET")
public static void getCustomerDetails(String customerId) {
    // Java EE应用逻辑
}
```

注解处理器在编译时验证注解并生成所需的CICS工件 ^[extracted]。非Java程序可以用标准`EXEC CICS LINK`调用这个Java方法 ^[extracted]。

## 相关概念

- [[concepts/cics-java-modernization]] — Java在CICS中的应用
- [[concepts/cics-exec-api]] — EXEC CICS命令基础
- [[concepts/cics-devops]] — DevOps在IBM Z上的实践
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 7 "Modern IBM CICS application programming features", December 2020