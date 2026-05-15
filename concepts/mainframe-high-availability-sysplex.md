---
title: Mainframe High Availability and Sysplex
category: concepts
tags: [ibm, mainframe, sysplex, high-availability, parallel-sysplex]
sources: [下载/zOS Basics.pdf]
summary: Parallel Sysplex通过Coupling Facility实现32台z/OS镜像共享数据和负载均衡，达到99.999%可用性。配合IRD动态工作负载管理和无单点故障设计，实现近连续运行。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.57
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:21:00Z
updated: 2026-05-15T01:21:00Z
---

# Mainframe High Availability and Sysplex

## RAS：可靠性的核心原则

RAS（Reliability, Availability, Serviceability）是mainframe设计的核心原则 ^[extracted]：

- **Reliability（可靠性）** — 硬件组件具有广泛的自检和自恢复能力；操作系统具有Health Checker，在问题影响可用性之前识别潜在问题
- **Availability（可用性）** — 系统可从组件故障中恢复而不影响运行中的系统；最高级别可用性通过DB2和Parallel Sysplex在System z架构上实现
- **Serviceability（可服务性）** — 系统可确定故障原因，便于硬件/软件元素的更换

MTBF（Mean Time Between Failures）在新型mainframe上以数十年为单位测量 ^[extracted]。System z设计目标是配合Parallel Sysplex达到**99.999%**可用性 ^[extracted]。

## 集群技术的演进

mainframe集群经历三个发展阶段 ^[extracted]：

### 1. Basic Shared DASD（基础共享DASD）

多个z/OS图像共享磁盘，通过RESERVE/RELEASE命令控制并发访问。操作员需人工协调避免数据冲突。适合测试、恢复和谨慎的负载均衡 ^[extracted]。

### 2. CTC Rings（通道到通道环）

在共享DASD基础上增加CTC（Channel-to-Channel）连接，实现系统间控制信息传递 ^[extracted]：
- 数据集使用锁定信息自动防止重复访问
- 作业队列共享（单一输入队列供所有系统使用）
- 统一的安全控制
- 统一的磁盘元数据控制

这是GRS（Global Resource Serialization）环的前身 ^[extracted]。

### 3. Parallel Sysplex

## Parallel Sysplex详解

**Parallel Sysplex** 是使用 multisystem data-sharing 技术的对称sysplex ^[extracted]。关键特征：

- **最多32台服务器** 通过高速耦合设施互连
- **近线性可扩展性** — 聚合多服务器容量处理商业工作负载
- **共享数据集群** — 区别于分布式环境的"shared nothing"架构，采用"shared everything"设计 ^[inferred]
- **动态工作负载平衡** — 关键业务应用可利用多服务器聚合容量

### Coupling Facility（CF）

Coupling Facility是Parallel Sysplex的核心组件 ^[extracted]：

- 专用逻辑分区（使用ICF处理器）或独立系统
- 充当高速暂存存储（scratch pad）
- 三种用途：
  1. **锁定** — 跨系统共享资源的锁管理
  2. **缓存** — 数据库等共享数据的缓存管理
  3. **数据列表** — 跨系统共享的列表数据

CF中信息驻留在内存中，典型配置有大型内存 ^[extracted]。

### 时间同步：Sysplex Timer → STP

**历史：** 早期Parallel Sysplex需要独立的Sysplex Timer设备来同步多服务器的TOD时钟 ^[extracted]。

**现代实现：** Server Time Protocol（STP）取代了Sysplex Timer ^[extracted]：
- 在Licensed Internal Code中实现
- 提供单一时间视图给PR/SM
- 分层（strata）结构：Stratum 1向下分发到Stratum 2，依此类推
- 允许服务器动态加入/离开时间同步网络

### 无单点故障（No SPOF）

Parallel Sysplex设计通过以下方式消除单点故障 ^[extracted]：

- 每个组件都有备用或冗余
- 系统或应用故障时，工作负载可动态重定向到可用服务器
- 软件和硬件升级可一次一个系统地滚动进行（滚动维护）

## Intelligent Resource Director（IRD）

IRD可视为Parallel Sysplex的Stage 2 ^[extracted]。Stage 1允许数据和工作负载跨多系统图像共享，但并非所有应用都支持数据共享。IRD提供能力将资源移到工作负载所在位置 ^[extracted]。

## 共享存储架构的优势

mainframe是**share-everything**架构 ^[extracted]：
- 处理器、内存、I/O设备可跨LPAR共享
- 这与分布式计算的share-nothing架构形成鲜明对比 ^[inferred]
- 通过共享数据架构，多系统协同看起来像单一大型系统，有单一操作员界面 ^[extracted]

## 与DB2和CICS的关系

Parallel Sysplex技术广泛用于：
- **DB2数据共享** — 多个DB2子系统并发读写同一数据库，无性能或数据完整性损失 ^[extracted]
- **CICS交易处理** — 支持CICS集群和高可用交易处理 ^[inferred]

参见：[[db2-12-data-sharing]]（DB2 12数据共享增强）

## 相关页面

- [[mainframe-hardware-architecture]] — 硬件架构基础
- [[mainframe-history-s360]] — 架构演进历史
- [[zos-overview]] — z/OS系统视图
- [[db2-12-data-sharing]] — DB2数据共享与Sysplex集成