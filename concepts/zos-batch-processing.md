---
title: z/OS Batch Processing
category: concepts
tags: [ibm, zos, batch, jes, initiator]
sources: [下载/zOS Basics.pdf]
summary: 批处理是mainframe的核心工作负载之一，通过JES2/JES3作业入口子系统接收作业，Initiator管理作业执行。批处理特征：大量数据顺序读写、无需即时响应、有固定批处理窗口。
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

# z/OS Batch Processing

## 批处理定义

**Batch processing（批处理）** 是在mainframe上运行作业而无需用户交互的数据处理方式 ^[extracted]。

批处理作业特征 ^[extracted]：
- 大量输入数据被读取和处理（可能TB级）
- 产生大量输出（报表、文件更新）
- 立即响应时间不是要求，但必须在服务级别协议（SLA）规定的"批处理窗口"内完成
- 作业可按预定顺序执行（数百或数千个作业）

## 批处理的典型场景

- **银行** — 夜间日结、月末季末报表生成
- **零售** — 销售汇总、库存更新
- **保险** — 保单处理、理赔批处理
- **政府** — 税务处理、执照发放

## 作业入口子系统（JES）

JES（Job Entry Subsystem）是z/OS的作业接收和管理组件 ^[extracted]：

### JES2 vs JES3

| 特性 | JES2 | JES3 |
|------|------|------|
| 架构 | 每个系统独立 | 耦合多系统全局控制 |
| 灵活性 | 本地控制 | 全局作业调度 |
| 适用规模 | 单系统或松耦合 | 大规模sysplex环境 |

两种JES都接收作业、调度执行、管理输出 ^[extracted]。

## 作业流程

```
1. 作业提交（通过JCL或API）
      ↓
2. JES接收并记录作业（进入输入队列）
      ↓
3. 作业调度器根据优先级选择作业
      ↓
4. Initiator分配资源并启动作业
      ↓
5. 程序执行（读入数据、进行处理、写入输出）
      ↓
6. 作业完成，输出进入输出队列
      ↓
7. 用户查看输出（通过SDS F或其他工具）
```

## Initiator

**Initiator** 是z/OS管理批处理作业执行的组件 ^[extracted]：

- 每个Initiator一次运行一个作业
- 系统可配置多个Initiator（如A-I共9个）
- Initiator空闲时从作业队列取下一个作业
- 通过CLASS参数匹配作业和Initiator（作业CLASS与Initiator CLASS对应）

## 作业状态

| 状态 | 含义 |
|------|------|
| **INPUT** | 作业等待被调度 |
| **EXECUTION** | 正在执行 |
| **OUTPUT** | 执行完成，等待打印或显示 |
| **PENDING** | 等待资源（数据集、输出设备） |

## 批处理窗口

批处理窗口是业务低峰期（如夜间），用于运行所有必须在当日完成的批处理作业 ^[extracted]。随着24x7业务模式的出现，批处理窗口正在缩小 ^[inferred]。

现代做法：
- 关键批处理作业在白天与在线事务并发运行
- 使用WLM管理资源争用
- 滚动维护取代传统关机维护

## 批量作业与在线系统的关系

批处理和OLTP通常共享同一数据库 ^[extracted]：
- 白天在线系统处理事务
- 夜间批处理系统汇总交易、生成报表、更新汇总数据
- 两者通过JCL中的数据集定义协调

## 相关页面

- [[zos-jcl-basics]] — JCL语句和SDSF
- [[zos-workload-management]] — WLM管理混合工作负载
- [[zos-overview]] — z/OS作业调度系统
- [[continuous-delivery]] — 持续交付中的批处理CI/CD