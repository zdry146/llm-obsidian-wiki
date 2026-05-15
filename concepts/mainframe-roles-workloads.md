---
title: Mainframe Roles and Workloads
category: concepts
tags: [ibm, mainframe, roles, workloads]
sources: [下载/zOS Basics.pdf]
summary: mainframe有五类主要角色：系统程序员、系统管理员、应用开发人员、系统操作员和生产控制分析师。典型工作负载分为batch（批量）和OLTP（在线事务）两类，各有其特征和资源需求。
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

# Mainframe Roles and Workloads

## mainframe主要角色

mainframe系统的设计旨在供大量人员使用 ^[extracted]。五类主要角色：

### 1. 系统程序员（System Programmer）

**职责：** 安装、定制和维护操作系统，以及安装或升级在其上运行的产品 ^[extracted]。

核心任务：
- 规划硬件和软件系统升级及配置变更
- 培训系统操作员和应用程序员
- 自动化操作
- 容量规划
- 执行安装作业和脚本
- 系统级性能调优

系统程序员需要熟练使用调试工具分析dump（内存转储），并与软件供应商支持代表合作解决问题 ^[extracted]。

### 2. 系统管理员（System Administrator）

**职责：** 维护驻留在mainframe上的关键业务数据 ^[extracted]。

分类管理员：
- **DBA（Database Administrator）** — 数据库管理员，确保数据库完整性和高效访问
- **安全管理员** — 维护安全资源访问列表
- **存储管理员** — 管理存储设备和打印机
- **网络管理员** — 管理网络和连接

注意：在大型IT组织中，职责分离是合规和安全的需要 ^[extracted]。

### 3. 应用开发人员（Application Designer/Programmer）

**职责：** 设计、构建、测试和交付在mainframe上运行的应用 ^[extracted]。

工作内容：
- 基于业务需求创建设计规范
- 编写代码（COBOL、Java等）
- 迭代编译、构建和单元测试
- 与团队协作完成模块构建
- 维护和增强现有应用程序

现代mainframe程序员使用IDE（如IBM Developer for z/OS）提高生产效率 ^[extracted]。

### 4. 系统操作员（System Operator）

**职责：** 监控和控制mainframe硬件和软件的运行 ^[extracted]。

关键任务：
- 启动和停止系统任务
- 监控系统控制台，识别异常情况
- 与系统程序员和生产控制分析师合作确保系统健康
- 启动和停止子系统（CICS、DB2等）
- 使用Run Book中的操作规范管理应用程序

Run Book是指引操作员执行特定应用操作的操作手册 ^[extracted]。

### 5. 生产控制分析师（Production Control Analyst）

**职责：** 确保批处理工作负载无错误或延迟地完成执行 ^[extracted]。

关键任务：
- 管理作业调度
- 监控作业执行状态
- 协调日间在线/夜间批处理的执行模型
- 确保生产工作负载按计划完成

## 供应商角色

mainframe环境中常见的供应商角色 ^[extracted]：

| 角色 | 职责 |
|------|------|
| **硬件支持/Customer Engineer (CE)** | 硬件设备的安装和维修 |
| **软件支持** | IBM Support Center提供软件缺陷支持 |
| **FTSS/SE（现场技术销售支持）** | 售前技术支持和架构建议 |
| **客户代表** | 特定行业客户的单一联系点（SPOC） |

## 典型工作负载类型

### Batch Processing（批处理）

批处理作业特征 ^[extracted]：
- 大规模数据输入、处理和输出（TB级）
- 无即时响应时间要求，但必须在"批处理窗口"内完成
- 按预定顺序执行（数百或数千作业链）
- 典型场景：银行日结、零售库存更新、保险理赔处理

详见：[[zos-batch-processing]]

### Online Transaction Processing（OLTP）

在线事务处理特征 ^[extracted]：
- 少量输入数据、少量记录访问和处理、少量输出
- 即时响应时间（通常<1秒）
- 大量用户的大规模并发事务
- 全天候可用性要求
- 典型场景：ATM、网上银行、零售POS、航空预订

详见：[[zos-overview]]

### Specialty Engine Workloads

mainframe专用引擎优化特定工作负载 ^[extracted]：

| 引擎 | 优化工作负载 |
|------|-------------|
| **IFL** | Linux工作负载 |
| **zAAP** | Java和XML处理 |
| **zIIP** | 数据库和BI/ERP/CRM工作负载 |

## 角色与工作负载的关系

| 工作负载 | 主要角色 |
|----------|----------|
| 系统安装和升级 | 系统程序员 |
| 日常监控 | 系统操作员 |
| 批处理调度 | 生产控制分析师 |
| 新应用开发 | 应用开发人员 |
| 数据库管理 | 系统管理员（DBA） |
| 安全策略 | 系统管理员（安全） |

## 关键洞察

- **角色分离是mainframe的特色** — 大型组织中职责分离是合规要求 ^[extracted]
- **mainframe操作更规范化** — 详细的操作程序手册和严格的变更控制帮助实现高可用性 ^[inferred]
- **运维人员规模相对较小** — 尽管角色众多，但相比分布式系统，mainframe所需人员更少 ^[inferred]
- **现代工具改变角色** — Zowe、IDz等工具正在简化mainframe操作 ^[inferred]

## 相关页面

- [[zos-overview]] — z/OS系统概述
- [[zos-batch-processing]] — 批处理工作负载
- [[cics-overview]] — CICS交易处理
- [[cics-devops]] — DevOps工具链如何改变开发角色