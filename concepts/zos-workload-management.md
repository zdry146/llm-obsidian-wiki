---
title: z/OS Workload Management
category: concepts
tags: [ibm, zos, wlm, workload-management]
sources: [下载/zOS Basics.pdf]
summary: z/OS Workload Manager (WLM)根据服务级别目标自动管理处理器、内存和I/O资源，支持在单一系统或跨Sysplex实现工作负载平衡，同时管理batch和OLTP两类工作负载。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.57
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:21:00Z
updated: 2026-05-15T01:21:00Z
---

# z/OS Workload Management

## 工作负载管理的必要性

在大型机上，工作负载分为两类 ^[extracted]：
- **Batch（批处理）** — 大量数据后台处理，无需用户交互，如月末报表、曰结处理
- **OLTP（在线事务处理）** — 实时交互式处理，如ATM、网上银行、零售POS

传统方式下，操作员手动分配资源。这种方式低效且易出错。WLM通过自动化资源分配解决此问题 ^[inferred]。

## WLM的核心概念

**Service Class（服务类别）** — 具有相同服务级别目标的工作负载分组 ^[extracted]。

**Service Level Agreement（SLA）** — 定义响应时间目标，WLM据此分配资源 ^[extracted]。

**Goals-based WLM** — 管理员定义业务目标（如"95%的交易在0.5秒内完成"），WLM自动调整资源以达成目标 ^[extracted]。

## WLM的四个功能领域

1. **Resource Assignment（资源分配）** — 动态分配处理器、内存和通道资源给各服务类别
2. **Performance Measurement（性能测量）** — 收集各服务类别的性能数据
3. **Reporting（报告）** — 生成工作负载性能报告
4. **Scheduling（调度）** — 决定作业和事务的执行顺序

## WLM的工作方式

1. 系统程序员定义服务类别及其性能目标
2. WLM持续监控系统性能
3. 当某服务类别未达到目标时，WLM自动提升其优先级或分配更多资源
4. 在LPAR环境下，WLM可跨系统分配工作负载（通过Sysplex）

WLM支持两种模式 ^[extracted]：
- **Goal mode** — 根据业务目标动态调整
- **Statement mode** — 根据固定的优先级规则调度

## 与Parallel Sysplex的集成

在Parallel Sysplex环境下，WLM可实现 **[sysplex级工作负载管理](mainframe-high-availability-sysplex)** ^[extracted]：

- 作业可从一个系统动态迁移到另一个系统以平衡负载
- 系统故障时，工作负载自动重定向到可用系统
- 支持滚动维护（一次升级一个系统，其他系统继续处理工作负载）

## 处理器分配与优先级

WLM将工作分配到不同类型的处理器 ^[extracted]：
- **通用CP** — 执行标准z/OS工作负载
- **zAAP** — 执行Java和XML工作负载（降低许可证成本）
- **zIIP** — 执行数据库和BI工作负载（降低许可证成本）

这允许不同类型的工作负载运行在成本优化的处理器上 ^[inferred]。

## 相关页面

- [[zos-overview]] — z/OS操作系统概述
- [[zos-batch-processing]] — 批处理和作业调度
- [[mainframe-high-availability-sysplex]] — Sysplex和WLM的集成
- [[cics-overview]] — CICS事务处理系统