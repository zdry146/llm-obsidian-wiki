---
title: Message-Oriented Middleware
category: concepts
tags: [messaging, middleware, mom, integration, architecture]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: 消息导向中间件（MOM）通过异步消息传递实现应用程序解耦，简化开发、保证可靠性和跨平台集成，是企业应用集成（EAI）和SOA的基础设施。
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

# Message-Oriented Middleware

## 什么是MOM

消息导向中间件（Message-Oriented Middleware, MOM）是一种基础设施软件，通过**异步消息传递**实现应用程序之间的通信 ^[extracted]。

核心价值：**解耦（decoupling）** — 发送方应用不需要知道接收方应用在哪里、不需要对方同时在线、不需要了解对方的内部细节 ^[extracted]。

## 业务驱动因素

企业IT环境不断变化 ^[extracted]：
- 20年前的应用程序需要与上周刚写好的应用程序交换数据
- 公司合并后不同IT系统需要集成
- 原本仅供内部使用的应用现在要暴露给客户
- 新增移动端渠道（智能手机应用）需要接入同一数据库

传统点对点集成的痛点：
- 每新增一个连接就要写一对接口 → O(n²)复杂度
- 连接双方必须同时在线
- 平台差异（字符编码、操作系统、网络协议）全由应用层处理

## 消息系统的核心特性

根据IBM红皮书，一个消息中间件系统需要提供以下能力 ^[extracted]：

| 特性 | 说明 |
|------|------|
| **Once-only处理** | 业务交易必须精确执行一次，不能丢失也不能重复 |
| **Ubiquity（普适性）** | 跨操作系统、硬件平台无缝集成 |
| **Easy to change** | 使用行业标准，快速编写和部署新应用 |
| **Easy to extend** | 无停机扩展处理能力 |
| **Compliance** | 支持审计和合规目标 |
| **Performance & availability** | 充分利用系统资源，提供高可用性 |
| **Security** | 数据保护，防丢失、防篡改、防未授权读取 |

## 消息传递模式

### 点对点（Point-to-Point）

消息发送到一个**队列**，由一个消费者取走。同一条消息不会同时被多个消费者处理 ^[extracted]。

典型场景：
- 零售前台系统向后台库存系统下订单
- Linux应用通过CICS事务查询z/OS数据库
- 银行支付处理 ^[extracted]

### 发布/订阅（Publish/Subscribe）

信息提供者（发布者）将消息标记为特定**主题（topic）**，多个订阅者各自收到一份副本 ^[extracted]。

典型场景：
- 股票价格分发（交易员随时加入/离开，发布者不需要知道有多少订阅者）
- 零售商中央目录变更向所有分店广播 ^[extracted]

## 典型应用场景

### 零售 kiosk（自助服务终端）

某零售商在25,000+自助服务终端上展示一致的数据。通过MQ将变更从中央数据库推送到所有终端 ^[extracted]。

> 实际数字：每秒处理超过14,000笔交易 ^[extracted]

### 银行快速支付

监管要求支付清算时间从几天缩短到2小时以内 ^[extracted]。

某高并发支付方案使用MQ路由系统间的消息流量：
- 大量既有组件通过适配器与新的消息层对接，保护既有投资
- 实际处理能力达到秒级完成（远超监管要求的2小时）
- 每天处理数百万笔交易 ^[extracted]

### 机场信息系统

机场需要灵活的基础设施应对快速变化的政府法规和不断增长的客流量 ^[extracted]。通过消息导向集成方案：
- 多个系统的航班变更信息可以无缝整合
- 新系统可以灵活接入，无需改造已有系统

## MOM在SOA中的角色

SOA（Service-Oriented Architecture）要求应用之间能够互相交互 ^[extracted]。MOM是连接基础设施的7个关键属性之一：

> "A flexible and robust messaging backbone is a key component of SOA and provides the transport foundation for an ESB." ^[extracted]

ESB（Enterprise Service Bus）的消息总线通常基于MOM构建 ^[inferred]。

## 相关概念

- [[references/websphere-mq-primer]] — 原始红皮书来源
- [[concepts/websphere-mq-core-concepts]] — WebSphere MQ核心概念
- [[concepts/websphere-mq-messages]] — 消息结构和MQMD
- [[concepts/zos-overview]] — mainframe上的z/OS消息处理背景
- [[concepts/cics-overview]] — CICS与MQ的消息桥接机制