---
title: WebSphere MQ Messages
category: concepts
tags: [ibm, websphere, messaging, message, mqmd, mqmd]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: WebSphere MQ消息由三部分组成：MQMD消息描述符（固定结构）、消息属性（用户自定义）、消息数据（应用数据，最长100MB）。重点介绍MQMD各字段及其在路由、持久性、优先级控制中的作用。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Messages

## 消息结构

WebSphere MQ消息由三部分组成 ^[extracted]：

```
[MQMD: 消息描述符] + [Message Properties: 用户自定义属性] + [Message Data: 应用数据]
```

单个消息最大可含100MB数据 ^[extracted]。数据格式不限：文本、二进制、XML或其组合均可。

## MQMD（Message Descriptor）

MQMD是一个**固定结构**，描述消息的关键字段 ^[extracted]：

| 字段 | 用途 |
|------|------|
| **MsgType** | 消息类型：datagram, request, reply, report ^[extracted] |
| **MsgID / CorrelID** | 消息ID和关联ID（24字节），用于匹配请求与响应。发送方让MQ自动生成唯一MsgID，接收方将MsgID复制到CorrelID以标识对应的响应 ^[extracted] |
| **Persistence** | 持久性标志：持久（可恢复）/ 非持久（可丢弃） ^[extracted] |
| **Priority** | 优先级1-10，管理员可配置队列按优先级而非FIFO顺序取消息 ^[extracted] |
| **PutTime / PutDate** | 消息入队时间戳（UTC），用于跨时区一致性视图 ^[extracted] |
| **Expiry** | 过期时间，过期后消息不可返回给任何应用程序 ^[extracted] |
| **ReplyToQ / ReplyToQMgr** | 请求消息中指定回复队列和队列管理器 ^[extracted] |
| **Format** | 描述消息数据格式，如MQSTR（文本）或NONE（二进制），触发数据转换（如ASCII↔EBCDIC） ^[extracted] |
| **PutApplType / PutApplName** | 发送应用的信息（程序名、路径、运行平台） ^[extracted] |
| **Report** | 请求消息处理过程中的报告消息 ^[extracted] |
| **BackoutCount** | 消息被退回次数，应用程序可据此将消息移至其他队列处理 ^[extracted] |
| **GroupId / MsgSeqNumber** | 消息分组标识，保证消息顺序和组内消息原子处理 ^[extracted] |

## 消息属性（Message Properties）

应用程序可以设置任意属性（字符串、数字或布尔值），这些属性不属于MQMD结构体，但可以被应用程序查看和修改 ^[extracted]。

应用场景：
- 为消息设置颜色属性（red/green）
- 接收方按属性筛选消息（如：只取所有green消息）
- 在发布/订阅中结合主题提供额外筛选层（类似SQL的WHERE子句）^[extracted]

## 持久性（Persistence）与可靠性保证

持久消息在系统故障后可以恢复，通过队列管理器的**恢复日志（recovery log/journal）**实现 ^[extracted]。

非持久消息存储在系统内存中，以下情况可能丢失：网络错误、操作系统错误、硬件故障、队列管理器重启、内部软件故障 ^[extracted]。

| 持久级别 | 投递保证 | 存储位置 |
|----------|----------|----------|
| Persistent（持久） | **Exactly-once**（精确一次） | 磁盘日志 |
| Non-persistent（非持久） | **At-most-once**（最多一次） | 内存 |

## 消息过期（Expiry）

用途示例 ^[extracted]：

1. 应用程序等待响应最多10秒：请求和响应消息都设置10秒过期时间
2. 系统故障时，消息在不再有用后自动从队列中移除，不浪费磁盘或内存空间

## 相关概念

- [[concepts/websphere-mq-core-concepts]] — 消息传递和持久性概念
- [[concepts/websphere-mq-objects]] — 队列管理器对象
- [[concepts/websphere-mq-programming]] — MQPUT/MQGET代码示例
- [[concepts/websphere-mq-configuration]] — 队列的持久性默认值配置