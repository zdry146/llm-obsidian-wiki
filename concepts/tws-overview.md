---
title: Tivoli Workload Scheduler for z/OS - Overview
type: concept
tags: [ibm, tivoli, tws, zos, workload-scheduler, scheduling]
related_references: [references/tws-best-practices.md]
---

# Tivoli Workload Scheduler for z/OS - Overview

## 什么是 TWS for z/OS

IBM Tivoli Workload Scheduler for z/OS (TWS) 是 z/OS 平台上的企业级工作负载调度解决方案，用于自动化和管理 batch 作业。

**版本历史:**
- TWS V8.2 (2006): 本文档覆盖版本
- TWS V8.3: 预览版（见 Appendix A）

**核心功能:**
- 作业自动化调度
- 跨平台端到端调度（z/OS ↔ distributed）
- 作业监控与审计
- 事件触发（Dataset triggering）

## 架构组成

### 核心组件

| 组件 | 说明 |
|------|------|
| **Controller** | TWS for z/OS 主控程序，管理作业调度逻辑 |
| **Engine** | 调度引擎，执行作业依赖解析和触发 |
| **Agent** | z/OS Extended Agent，与分布式节点通信 |
| **EQQJOBS** | ISPF 主界面，用于作业和调度管理 |

### 关键数据集

- `EQQJOBS` - ISPF 主面板
- `EQQCPnDS` - Current Plan Data Set（当前计划）
- `EQQLTDS` - Long-Term Plan Data Set（长期计划）
- `EQQRDDS` - Special Resource Database（特殊资源数据库）

## 与其他 IBM 组件的关系

- **与 JCL 的关系**: TWS 管理 JOB 的触发时机，但不替代 JCL
- **与 WLM 的关系**: TWS 依赖 WLM 进行工作负载管理
- **与 System Automation 的关系**: TWS 专注批处理调度，SA 专注系统自动化

## 相关概念

- [[zos-batch-processing]] - z/OS 批处理背景
- [[jcl-introduction]] - JCL 作业控制基础

## Reference

- [[tws-best-practices]] - IBM Redbook SG24-7156-01（TWS 最佳实践）