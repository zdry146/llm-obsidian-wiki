---
title: WebSphere MQ Integration
category: concepts
tags: [ibm, websphere, integration, esb, message-broker, was, zos]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: WebSphere MQ与IBM其他产品的集成：WebSphere Message Broker（ESB）以MQ为骨干提供路由和数据转换、WebSphere Application Server（Java EE）通过JMS接入MQ、z/OS上的CICS/IMS/RRS事务协调。
provenance:
  extracted: 0.78
  inferred: 0.17
  ambiguous: 0.05
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Integration

## WebSphere MQ不强制依赖其他产品

WebSphere MQ产品名称中的"WebSphere"只是IBM内部品牌标识，不强制要求使用其他WebSphere品牌产品 ^[extracted]。

## WebSphere Message Broker（ESB）

WebSphere Message Broker是IBM的**企业服务总线（ESB）**产品 ^[extracted]。

### 核心能力

企业系统包含多种逻辑端点（现成应用、服务、封装应用、Web应用、设备、自定义软件等），各自暴露不同的输入输出 ^[extracted]：
- **连接协议**：WebSphere MQ、TCP/IP、数据库、HTTP、文件、FTP、SMTP、POP3
- **数据格式**：C/COBOL结构、XML、行业特定格式（SWIFT, EDI, HL7）、用户定义格式

ESB在所有这些连接和应用之间运行，执行**路由和数据转换** ^[extracted]。

### Message Broker与MQ的关系

Message Broker要求系统上必须安装WebSphere MQ ^[extracted]。原因：
1. MQ-enabled应用是需要支持的重要连接类别
2. Message Broker使用队列管理器提供的服务：事务协调、发布/订阅、可靠消息存储

Message Broker的管理工具（WebSphere Message Broker Explorer）运行在WebSphere MQ Explorer内，为两个产品提供单一控制点 ^[extracted]。

### 零售业集成示例

典型零售企业 ^[extracted]：
- 内部和外部存在多个端点，数据格式和协议各异（TLOG、文件、JSON/HTTP）
- 新增能力（如移动应用和分析）需要灵活接入而不中断现有服务
- Message Broker使这些功能的添加变得简单

## WebSphere Application Server

WebSphere Application Server是承载和运行应用程序的JEE环境 ^[extracted]。

### JMS集成

JEE应用服务器可以使用任意厂商的JMS实现，但WebSphere Application Server：
- 内置MQ客户端运行时代码和管理面板
- 使Web应用能够可靠地向任何MQ-enabled应用发送消息并获取响应 ^[extracted]

### 连接方式

WebSphere Application Server中的Web应用通过标准JMS接口连接WebSphere MQ队列管理器，不需要为每种应用单独编写连接代码 ^[inferred]。

## z/OS平台集成

### CICS集成

MQ提供与CICS的**桥接**，将MQ消息转换为CICS事务的输入，反之亦然 ^[extracted]。既有CICS应用无需修改即可接入MQ基础设施 ^[extracted]。

在[[concepts/cics-overview]]中有更详细的CICS桥接机制说明。

### IMS集成

类似CICS，MQ也可与IMS事务集成 ^[extracted]。

### RRS（Resource Recovery Services）

在z/OS上，RRS作为全局事务协调器 ^[extracted]。应用程序通过RRS协调MQ消息操作和数据库更新等资源。

### Sysplex和共享队列

在Parallel Sysplex环境中 ^[extracted]：
- 多个z/OS LPAR上的队列管理器可以共享队列
- 一个LPAR整体故障时，消息仍可被其他LPAR上的队列管理器处理
- 配合WLM实现高可用性

## Managed File Transfer（MFT）

WebSphere MQ V7.5引入Managed File Transfer ^[extracted]：
- 面向文件-based应用的集成解决方案
- 在企业系统间可靠传输文件
- 利用现有MQ基础设施（作为传输介质）
- 提供管理、监控和安全功能

## Advanced Message Security（AMS）

V7.5的AMS组件解决消息内容安全问题 ^[extracted]：
- 即使消息在队列中或日志文件中也能保护内容不被读取或篡改
- 满足PCI（支付卡行业）等监管要求
- 数据即使在处理过程中也不以明文形式存储在磁盘上 ^[extracted]

## 相关概念

- [[references/websphere-mq-primer]] — 原始红皮书来源
- [[concepts/websphere-mq-core-concepts]] — MQ核心概念
- [[concepts/websphere-mq-programming]] — MQ编程接口
- [[concepts/zos-overview]] — z/OS平台概述
- [[concepts/mainframe-high-availability-sysplex]] — Parallel Sysplex高可用
- [[concepts/cics-overview]] — CICS事务处理与MQ桥接