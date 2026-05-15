---
title: Hot Cache
updated: 2026-05-15
---
## Recent Activity
- Ingested IBM Redbooks REDP-0021-01: WebSphere MQ Primer — 新增8个WebSphere MQ/MOM概念页面，涵盖消息中间件基础、MQ核心对象（队列/通道/主题）、消息结构MQMD、MQI编程、配置管理和与Message Broker/WebSphere Application Server/CICS的集成。
- Ingested IBM z/OS 3.1 MVS JCL User's Guide (SA23-1386-60) — 新增12个JCL专题概念页面，涵盖JCL语句、JOB/EXEC/DD、数据集DISP、条件执行、SPOOL/SYSOUT、调度计时、检查点重启、过程、存储资源和IBM实用程序。
- Ingested IBM Redbooks SG24-6366-02: Introduction to the New Mainframe: z/OS Basics — 新增14个z/OS和mainframe概念页面，涵盖历史、硬件架构、Parallel Sysplex、z/OS概述、工作负载管理、JCL、批处理和编程语言。
- Ingested IBM Redbooks REDP-5628-00: Modernizing Applications with IBM CICS — 新增10个CICS概念页面，覆盖概述、COBOL编程、EXEC API、Channels/Containers、Java现代化、异步编程和DevOps工具链。
- Ingested IBM Redbooks SG24-8383-00: DB2 12 for z/OS Technical Overview — 扩展至12个DB2 12 wiki页面。

## Active Threads
- MQ/MOM知识体系建立：从消息中间件基础到WebSphere MQ配置编程，补充与CICS和z/OS的集成
- z/OS知识体系建立：从硬件到软件，从batch到OLTP，覆盖mainframe完整技术栈
- JCL知识体系完善：从基础到高级特性（条件执行、检查点重启、调度、实用程序）
- CICS知识库扩充：覆盖CICS应用现代化完整技术栈，从传统COBOL开发到现代Java和DevOps工具链

## Key Takeaways
- **MOM的核心价值是解耦** — 发送方和接收方不需要同时在线，消息持久化保证可靠性
- **WebSphere MQ是市场领先的消息集成中间件** — 自1993年（MQSeries）以来支持跨平台消息传递，V7.0引入完整发布/订阅能力
- **MQ消息由三部分组成** — MQMD（描述符）+ Message Properties（用户自定义）+ Message Data（应用数据，最长100MB）
- **MQI（Message Queue Interface）**是原生编程接口，26个动词，支持C/COBOL/Java/C#等语言
- **z/OS上MQ与CICS/IMS深度集成** — MQ消息可桥接为CICS事务输入，既有问题不修改即可接入MQ基础设施
- **mainframe是share-everything架构** — 处理器、内存、I/O设备可跨LPAR共享，与分布式share-nothing架构相反
- **System/360是现代计算分水岭** — 1964年推出，统一商业和科学计算，引入微码，持续兼容是核心原则

## Flagged Contradictions