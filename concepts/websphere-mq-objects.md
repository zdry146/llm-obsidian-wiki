---
title: WebSphere MQ Objects
category: concepts
tags: [ibm, websphere, mq, queue, channel, topic, objects]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: WebSphere MQ的核心对象类型：队列管理器（Queue Manager）、各类队列（本地/远程/别名/动态）、主题对象（Topic）、通道（Channel/SVRCONN/CLNTCONN）以及监听器（Listener）。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Objects

## 队列管理器（Queue Manager）

队列管理器是WebSphere MQ的核心资源 ^[extracted]。主要职责：
- 管理数据存储
- 故障后恢复
- 协调应用程序对队列中消息的更新
- 维护统计和状态信息

属性：可启用/禁用事件类型、控制最大同时打开队列数、在队列管理器级别启用发布/订阅功能 ^[extracted]。

## 队列类型

### 本地队列（Local Queue）

消息**物理存储**的唯一地方 ^[extracted]。所有其他队列类型都必须指向一个本地队列。

### 别名队列（Alias Queue）

指向本地队列的指针 ^[extracted]。用途：允许不同应用使用不同名称访问同一队列。也可作为指向主题的指针（用于发布/订阅）。

### 远程队列（Remote Queue）

指向**另一个队列管理器**上的队列的指针 ^[extracted]。应用程序不能从远程队列读取消息。定义包含：目标队列名、目标队列管理器名、关联的传输队列。

### 模型队列（Model Queue）

动态创建队列的模板 ^[extracted]。通常用于为回复消息动态创建唯一队列，程序结束后队列自动删除。

### 动态队列（Dynamic Queue）

通过打开模型队列而创建的队列 ^[extracted]。两个子类型：
- **TDQ（Temporary Dynamic Queue）**：创建程序结束时自动删除，只能存非持久消息 ^[extracted]
- **PDQ（Permanent Dynamic Queue）**：创建程序结束后仍存在，可被其他应用重用 ^[extracted]

## 特殊用途队列

### 传输队列（Transmission Queue）

本地队列，带有 `USAGE(XMITQ)` 属性 ^[extracted]。消息在发往远程队列管理器前先在这里暂存。如果目标不可达，消息在传输队列累积直到连接成功。

### 初始化队列（Initiation Queue）

本地队列，队列管理器在其上写入**触发消息**（trigger message）来启动应用程序或通道 ^[extracted]。

### 死信队列（Dead Letter Queue, DLQ）

用于无法成功送达的消息 ^[extracted]。场景：
- 目标队列已满
- 目标队列不存在
- 目标队列上禁止写入
- 发送方无权使用目标队列
- 消息过大

每个队列管理器最多只有一个DLQ ^[extracted]。

### 事件队列（Event Queue）

队列管理器在特定事件发生时生成事件消息 ^[extracted]。示例：
- `SYSTEM.ADMIN.QMGR.EVENT` — 队列管理器事件
- `SYSTEM.ADMIN.CHANNEL.EVENT` — 通道事件

## 主题对象（Topic Objects）

WebSphere MQ V7.0引入完整发布/订阅能力 ^[extracted]。主题对象用于控制点：
- 树形主题结构中需要不同配置的节点
- 访问控制分配（对子主题的授权）
- 管理员不需要预先创建所有可能用到的主题定义 ^[extracted]

## 通道（Channels）

通道是逻辑通信链路，分为两类 ^[extracted]：

### 消息通道（Message Channel）

连接两个队列管理器 ^[extracted]。**单向**传输消息。两个子类型：
- **SENDER**：主动发起连接
- **RECEIVER**：被动等待连接

其他变体：SERVER, REQUESTER, CLUSTER-SENDER, CLUSTER-RECEIVER ^[extracted]。

一对双向通信通常需要两对（每方向一对）消息通道定义 ^[inferred]。

### MQI通道

连接WebSphere MQ客户机到队列管理器 ^[extracted]。**双向**通信。子类型：
- **SVRCONN**：队列管理器上的定义
- **CLNTCONN**：客户机程序使用的定义

两个方向的消息传输只需要一条MQI通道（而消息通道需要一对）^[inferred]。

## 监听器（Listener）

监听程序等待入站网络连接请求，然后启动相应通道处理通信 ^[extracted]。

所有队列管理器如果需要接收消息或客户机连接，必须至少运行一个监听器。标准配置：TCP/IP端口1414 ^[extracted]。

## 对象命名规则

- 对象名最多48个字符（通道为20个字符） ^[extracted]
- **大小写敏感**：`queue.abc` ≠ `QUEUE.ABC` ^[extracted]
- runmqsc命令中的属性自动转换为大写（除非用单引号包裹）^[extracted]

## 相关概念

- [[concepts/websphere-mq-core-concepts]] — 核心概念概述
- [[concepts/websphere-mq-messages]] — 消息结构
- [[concepts/websphere-mq-configuration]] — 对象配置方法（runmqsc）
- [[concepts/websphere-mq-programming]] — MQI编程接口
- [[concepts/zos-overview]] — z/OS上的队列管理器管理特点