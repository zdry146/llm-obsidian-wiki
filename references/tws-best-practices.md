---
title: IBM Tivoli Workload Scheduler for z/OS Best Practices
source: sg247156.pdf
type: reference
project: null
tags: [ibm, tivoli, workload-scheduler, tws, zos, scheduling, redbook]
ingested_at: 2026-05-15T07:08:00Z
pages_created: []
---

# IBM Tivoli Workload Scheduler for z/OS Best Practices

**Redbook:** SG24-7156-01  
**Authors:** Vasfi Gucer, Michael A Lowry, Darren J Pfister, Cy Atkinson, Anna Dawson, Neil E Ogle, Stephen Viola, Sharon Wheeler  
**Edition:** Second Edition (May 2006)  
**Version:** Tivoli Workload Scheduler for z/OS Version 8.2  

## 内容结构

### Part 1. Mainframe Scheduling（本地调度）
- Chapter 1: Installation（安装与定制）
- Chapter 2: Installation Verification（安装验证）
- Chapter 3: The Started Tasks（启动任务）
- Chapter 4: Communication（通信机制）
- Chapter 5: Initialization Statements and Parameters（初始化语句与参数）
- Chapter 6: Exits（退出点）
- Chapter 7: Security（安全配置）
- Chapter 8: Restart and Cleanup（重启与清理）
- Chapter 9: Dataset Triggering and Event Trigger Tracking（数据集触发与事件追踪）
- Chapter 10: Variables（变量）
- Chapter 11: Audit Report Facility（审计报告）
- Chapter 12: Using TWS for z/OS Effectively（高效使用 TWS）

### Part 2. End-to-End Scheduling（端到端调度）
- Chapter 13: Introduction to End-to-End Scheduling（端到端调度概述）
- Chapter 14: End-to-End Scheduling Architecture（端到端调度架构）
- Chapter 15: End-to-End Scheduling Installation（端到端调度安装）
- Chapter 16: Job Scheduling Console（作业调度控制台）
- Chapter 17: End-to-End Scheduling Scenarios（端到端调度场景）
- Chapter 18: End-to-End Troubleshooting（端到端故障排除）

### Appendices
- Appendix A: Version 8.2 PTFs and V8.3 Preview
- Appendix B: EQQAUDNS Member Example
- Appendix C: Additional Material

## 核心概念

- **TWS for z/OS**: z/OS 上的企业级工作负载调度器，支持 batch 作业自动化
- **End-to-End Scheduling**: 跨平台统一调度，从 distributed 节点到 mainframe
- **Job Scheduling Console**: ISPF 界面管理 TWS
- **Event Triggering**: 基于数据集状态变化触发作业
- **Audit Report**: 调度作业审计跟踪

## 相关链接

- PDF: `pdf/TWS.pdf`