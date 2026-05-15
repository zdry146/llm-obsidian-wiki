---
title: TWS End-to-End Scheduling Architecture
type: concept
tags: [ibm, tivoli, tws, zos, end-to-end, scheduling, architecture]
related_references: [references/tws-best-practices.md]
---

# TWS End-to-End Scheduling Architecture

## 端到端调度概述

TWS for z/OS 支持 **End-to-End Scheduling**，实现从 distributed 环境到 mainframe 的统一作业调度。

**核心优势:**
- 跨平台作业依赖管理
- 统一的作业监控视图
- 分布式节点作为 z/OS 的作业执行代理

## 架构层次

### Level 1: z/OS Controller
- TWS for z/OS 主控节点
- 维护当前计划（Current Plan）和长期计划（Long-Term Plan）
- 管理 z/OS 内部作业调度

### Level 2: z/OS Extended Agent
- 运行在 z/OS 上的代理程序
- 负责与分布式节点通信
- 将分布式作业提交给 Controller

### Level 3: Distributed Nodes
- Windows/UNIX/Linux 服务器
- 安装 TWS Distributed Agent
- 接收来自 z/OS Controller 的作业指令

## 通信机制

Chapter 4 和 Chapter 14 详细描述了通信架构：
- TCP/IP 作为主要通信协议
- 安全连接支持 SSL/TLS
- 防火墙穿透配置

## 作业流程示例

```
[Distributed Node] 
       ↓ (submit job)
[z/OS Extended Agent]
       ↓ (MQ/bi-directional)
[z/OS Controller]
       ↓ (JCL execute)
[z/OS Initiator]
```

## 相关概念

- [[tws-overview]] - TWS 概述
- [[mainframe-high-availability-sysplex]] - Parallel Sysplex 高可用架构（可与 TWS集成）
- [[zos-batch-processing]] - z/OS 批处理机制