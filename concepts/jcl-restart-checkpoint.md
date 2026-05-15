---
title: JCL Restart and Checkpoint
category: concepts
tags: [ibm, zos, jcl, restart, checkpoint, recovery]
sources: [下载/jcl user guide.pdf]
summary: 检查点/重启机制允许作业在异常终止后从检查点重新启动，包括SYSCHK DD、RD参数、以及GDG重启注意事项。
provenance:
  extracted: 0.82
  inferred: 0.12
  ambiguous: 0.06
base_confidence: 0.71
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL Restart and Checkpoint

## 概述

检查点/重启机制允许作业在异常终止后从检查点重新启动，而不是从头开始重新执行 ^[extracted]。

## 检查点（Checkpoint）

程序可以在执行过程中定期写入检查点记录，包含程序状态和数据集位置信息。

### 检查点数据集

检查点数据集通常存储在磁盘上，包含：
- 作业标识
- 步骤标识
- 程序状态
- 数据集位置（DSN和卷信息）

### 在JCL中指定检查点数据集

```jcl
//STEP1 EXEC PGM=MYPROG,CHECKPT=YES
//SYSCHK DD DSN=MY.CHKPT,DISP=(NEW,CATLG,DELETE)
```

`CHECKPT=YES` 启用检查点（如果程序支持）。

## 重启（Restart）

### 作业重启类型

| 类型 | 触发条件 | 重启点 |
|---|---|---|
| **Step restart** | 步骤异常终止 | 步骤开始处 |
| **Checkpoint restart** | 作业/步骤异常终止 | 最后检查点 |
| **Step restart after system failure** | 系统故障后作业恢复 | 步骤开始处 |

### 指定重启方式

```jcl
//JOB1 JOB ,'NAME',RESTART=(STEP2,CHKP1)
```

`RESTART` 参数指定重启点：
- `RESTART=stepname` — 从指定步骤开始重启
- `RESTART=(stepname,chkpt-id)` — 从指定步骤的检查点重启

### RD参数控制重启行为

`RD` 参数控制何时允许自动重启 ^[extracted]：

| 值 | 含义 |
|---|---|
| `R` | 允许从检查点重启，也允许step restart |
| `RNC` | 不允许从检查点重启，但允许step restart |
| `RNO` | 不允许自动重启 |
| `S` | 仅从系统检查点重启（用于测试） |

## 系统故障后的重启

### JES2系统

当JES2系统发生故障时，系统可以自动重新启动受影响的作业 ^[extracted]。

### JES3系统

当JES3系统发生故障时，作业从系统重新初始化时继续执行。

## 远程节点执行的重启

在远程节点执行时，重启行为取决于作业是使用APPC还是非APPC调度环境 ^[extracted]。

## GDG（Generation Data Group）重启注意事项

当作业使用GDG并包含检查点时，重启可能需要特殊考虑 ^[extracted]：

- 某些世代可能已过期
- 需要正确处理GDG基础条目
- 检查点记录中包含的相对世代号需要相对于当前日期重新计算

### GDG重启示例

```jcl
//JOB1 JOB ACCT,'NAME',RESTART=(STEP1,GDGCHK)
//STEP1 EXEC PGM=GDGPROG
//GDG DD DSN=MY.GDG(+1),DISP=(NEW,CATLG,DELETE)
//...
```

## 作业日志中的重启信息

重启信息记录在作业日志中 ^[extracted]：

- `IEF285I` — 数据集分配信息
- `IEF404I` — 步骤结束信息
- 检查点记录号

## 相关页面

- [[concepts/jcl-job-proc-step]] — EXEC语句参数
- [[concepts/jcl-conditional-execution]] — 条件执行和步骤状态
- [[concepts/zos-batch-processing]] — 批处理和作业管理

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 5 "Entering jobs - execution"