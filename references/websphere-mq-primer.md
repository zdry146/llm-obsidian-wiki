---
title: WebSphere MQ Primer
category: references
tags: [ibm, websphere, messaging, redbook]
sources: [/home/openclaw/下载/ibm mq.pdf]
summary: IBM红皮书REDP-0021-01（2012年12月），Mark E. Taylor著，WebSphere MQ / MQSeries消息中间件入门读物，涵盖MOM概念、MQ核心对象、编程接口和配置实践。
provenance:
  extracted: 0.95
  inferred: 0.03
  ambiguous: 0.02
base_confidence: 0.67
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15
updated: 2026-05-15
---

# WebSphere MQ Primer

## 文档信息

- **书名：** WebSphere MQ Primer: An Introduction to Messaging and WebSphere MQ
- **作者：** Mark E. Taylor（IBM Hursley实验室，WebSphere MQ技术战略团队）
- **编号：** IBM Redbooks REDP-0021-01
- **版本：** 第二版，2012年12月
- **适用版本：** WebSphere MQ V7.1（分布式平台和z/OS）和V7.5

## 内容定位

本书面向任何希望理解**消息导向中间件（Message-Oriented Middleware, MOM）**和WebSphere MQ的读者，不需要预先具备产品或消息技术知识 ^[extracted]。

全书分三章：
1. **消息概念** — 业务案例、应用简化、典型场景
2. **WebSphere MQ介绍** — 核心概念、API、多平台集成
3. **入门配置** — 消息、对象、应用程序、触发机制

## 与本wiki其他内容的关系

本书的z/OS集成部分与以下页面交叉引用：
- [[concepts/zos-overview]] — z/OS基础
- [[concepts/cics-overview]] — CICS事务处理
- [[references/introduction-to-new-mainframe-zos-basics]] — mainframe基础知识

WebSphere MQ与CICS的桥梁机制（将MQ消息转为CICS事务输入）在[[concepts/cics-overview]]中有补充描述 ^[inferred]。

## 关键结论

- WebSphere MQ的核心价值是**灵活性 + 可靠性 + 可扩展性 + 安全性**的组合
- MQ设计理念：**应用程序不需要关心通信细节**（恢复、可靠性、平台差异都由中间件处理）
- 消息系统实现**发送方与接收方解耦**，双方不需要同时在线 ^[extracted]

## 来源

- IBM Redbooks REDP-0021-01，December 2012，Mark E. Taylor