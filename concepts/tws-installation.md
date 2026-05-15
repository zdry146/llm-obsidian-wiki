---
title: TWS for z/OS Installation and Verification
type: concept
tags: [ibm, tivoli, tws, zos, installation, verification]
related_references: [references/tws-best-practices.md]
---

# TWS for z/OS Installation and Verification

## 安装流程概览

### 关键步骤

1. **分配数据集** - 创建 TWS 所需的数据集（HLQ.SKELETON）
2. **配置 Controller** - 通过 ISPF 界面 `=0.1` 配置主控节点
3. **创建 Workstation** - 使用 `=1.1.2` 创建 z/OS workstation
4. **创建 Calendar** - 定义作业调度历法
5. **创建 Application** - 定义作业和依赖关系

### 安装验证

Chapter 2 描述了安装验证流程，包括：
- 运行 `EQQJOBS` 主面板检查状态
- 验证 Controller 和 Engine 通信
- 检查数据集触发是否正常工作

## ISPF 操作命令

| 命令 | 功能 |
|------|------|
| `=0.1` | 配置 Controller |
| `=1.1.2` | 创建 Workstation |
| `=1.2.2` | 创建 Calendar |
| `RUN` on Command line | 构建运行周期 |

## 常见安装问题

- 数据集权限不足
- Controller 配置不正确
- 与分布式 Agent 的网络通信问题

→ 参考 Chapter 18 (Troubleshooting)

## 相关概念

- [[tws-overview]] - TWS 概述
- [[zos-tso-e-ispf-unix]] - ISPF 使用基础