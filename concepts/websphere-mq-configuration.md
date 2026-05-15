---
title: WebSphere MQ Configuration
category: concepts
tags: [ibm, websphere, configuration, administration, runmqsc, queue-manager]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: WebSphere MQ配置：创建队列管理器（crtmqm/strmqm）、管理对象（runmqsc命令DEFINE/ALTER/DELETE）、配置客户机/服务器连接、配置队列管理器间通信（通道/传输队列/监听器）。
provenance:
  extracted: 0.82
  inferred: 0.13
  ambiguous: 0.05
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Configuration

## 创建队列管理器

```bash
crtmqm QMA          # 创建队列管理器
strmqm QMA          # 启动队列管理器
```

队列管理器名称**大小写敏感** ^[extracted]。可同时运行多个队列管理器，无数量限制。

## 管理对象（runmqsc）

`runmqsc`是在分布式平台上管理MQ对象的命令行工具 ^[extracted]。在z/OS上也可通过SDSF面板或作业提交使用相同命令。

### 常用命令示例

```bash
# 定义本地队列
define qlocal('QUEUE1')

# 定义通道（SENDER类型，关联传输队列）
define channel('TO.QMB') chltype(sdr) xmitq('QMB') conname(hostB)

# 定义传输队列
define qlocal('QMB') usage(xmitq)

# 结束会话
end
```

注意：属性名自动大写，对象名保留原大小写 ^[extracted]。忘记大小写敏感性是常见错误，通常导致`UNKNOWN_OBJECT_NAME`错误。

### 通道子类型

- **SENDER (SDR)**：主动发起连接，发送消息
- **RECEIVER (RCVR)**：被动监听，接收消息
- **SVRCONN**：服务器连接通道（MQI通道类型）
- **CLNTCONN**：客户机连接通道（MQI通道类型） ^[extracted]

## 配置客户机/服务器连接

### 简单配置（环境变量）

```bash
# C客户端
set MQSERVER=CHAN1/TCP/9.24.104.206(1414)
```

### Java客户机

```java
MQEnvironment.hostname = "9.24.104.456";
MQEnvironment.channel = "CHAN1";
MQEnvironment.port = 1414;
```

### 队列管理器端配置

```bash
# 定义服务器连接通道
DEFINE CHANNEL('CHAN1') CHLTYPE(SVRCONN)
```

## 队列管理器间通信配置

两队列管理器之间通信需要以下对象 ^[extracted]（以QMA→QMB为例）：

**QMA端：**
- 远程队列定义：`DEFINE QREMOTE(Q1) RNAME(Q1) RQMNAME(QMB) XMITQ(QMB)`
- 传输队列：`DEFINE QLOCAL(QMB) USAGE(xmitq)`
- 发送通道：`DEFINE CHANNEL(QMA.QMB) CHLTYPE(sdr) XMITQ(QMB) TRPTYPE(tcp) CONNAME(machine2)`
- 接收通道：`DEFINE CHANNEL(QMB.QMA) CHLTYPE(rcvr) TRPTYPE(tcp)`

**QMB端：**
- 本地队列：`DEFINE QLOCAL(Q1)`
- 反向配置类似

## 启动通信

### 手动启动

```bash
strmqm QMA
start runmqlsr -t tcp -m QMA -p 1414    # Windows：后台窗口；UNIX：用 &
runmqsc QMA
start channel(QMA.QMB)
```

### 自动启动（触发机制）

在传输队列上设置触发属性 ^[extracted]：

```bash
DEFINE QLOCAL(QMB) REPLACE +
  USAGE(xmitq) +
  TRIGGER +
  TRIGTYPE(first) +
  INITQ(SYSTEM.CHANNEL.INITQ)
```

通道在有消息可用时自动启动，空闲时自动停止以释放资源 ^[extracted]。

## 安全配置（V7.1+）

从WebSphere MQ V7.1开始，默认配置阻止大多数客户机连接 ^[extracted]。启用方式：

```bash
# 禁用通道认证（不推荐用于生产）
ALTER QMGR CHLAUTH(DISABLED)

# 更好的方式：显式设置用户ID
DEFINE CHANNEL('CHAN1') CHLTYPE(SVRCONN) MCAUSER('userA')
SET CHLAUTH('CHAN1') TYPE(USERMAP) CLNTUSER('userA') USERSRC(MAP) MCAUSER('userA') ADDRESS('*') ACTION(ADD)
```

授权示例 ^[extracted]：
```bash
setmqaut -t qmgr -m QMA -p userA +connect
setmqaut -t q -n QUEUE1 -p userA +put
setmqaut -t q -n QUEUE2 -p userA +get
```

z/OS上通过外部安全管理器（如IBM RACF）进行类似配置 ^[extracted]。

## WebSphere MQ Explorer

基于Eclipse的图形化管理工具 ^[extracted]。可在单一界面管理队列管理器、队列、通道、主题等对象。

## 编程式管理（PCF）

通过向特定队列发送格式化消息来管理MQ ^[extracted]。消息格式称为**PCF（Programmable Command Format）**，许多工具和产品基于此接口构建。

## 相关概念

- [[concepts/websphere-mq-objects]] — 队列、通道、主题对象详解
- [[concepts/websphere-mq-programming]] — MQI编程接口
- [[concepts/websphere-mq-core-concepts]] — 消息传递和拓扑
- [[concepts/zos-overview]] — z/OS上的MQ管理特点
- [[concepts/cics-overview]] — CICS与MQ的桥接集成