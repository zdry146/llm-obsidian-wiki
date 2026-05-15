---
title: JCL SDSF Output Management
category: concepts
tags: [ibm, zos, jcl, sdsf, output, job, jes]
sources: [下载/jcl user guide.pdf]
summary: SDSF（System Display and Search Facility）是查看和管理z/OS作业输出的主要工具，支持DA/I/O/ST/H等面板查看作业状态和控制输出。
provenance:
  extracted: 0.88
  inferred: 0.08
  ambiguous: 0.04
base_confidence: 0.72
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL SDSF Output Management

## 概述

**SDSF（System Display and Search Facility）** 是查看和管理z/OS作业输出的主要工具 ^[extracted]。

SDSF替代了早期的RJE、ROTS和HASP等工具，提供统一的界面来查看和控制作业输出。

## SDSF面板

SDSF提供多个面板用于不同目的 ^[extracted]：

| 面板 | 命令 | 用途 |
|---|---|---|
| **DA** | `DA` | 显示活动作业（Daemon） |
| **Input Queue** | `I` | 显示输入队列的作业 |
| **Output Queue** | `O` | 显示输出队列 |
| **Held Output** | `H` | 显示保持的输出 |
| **Status** | `ST` | 显示作业状态 |

## 查看作业输出

### 步骤1：显示作业列表

在SDSF Primary Option Menu中选择 `ST` 查看作业状态：

```
HQX7760 –-------------COMMAND INPUT ===> ST
DA   I   O   H   ST         SDSF PRIMARY OPTION MENU
                              - Active users
                              - Input queue
                              - Output queue
                              - Held output queue
                              - Status of jobs
```

### 步骤2：查看作业数据集

在作业名称旁边输入 `?` 查看作业的数据集列表：

```
SDSF STATUS DISPLAY ALL CLASSES
COMMAND INPUT ===>
PREFIX=* DEST=(ALL) OWNER=userid*
NP JOBNAME JOBID    OWNER    PRTY C QUEUE  STATUS   POS  ASYS MAX-RC CC
?     jobname JOB20482 userid   7 H Print   77            0000
```

### 步骤3：选择数据集显示

输入 `S` 选择要显示的数据集：

```
SDSF JOB DATA SET DISPLAY - JOB userid (JOB20482)
COMMAND INPUT ===>
NP DDNAME    STEPNAME PROCSTEP DSID OWNER  C DEST   REC-CNT
S  JESMSGLG  JES2              2 userid  H LOCAL
S  JESJCL    JES2              3 userid  H LOCAL
S  JESYSMSG  JES2              4 userid  H LOCAL
S  SYSOUT    SORT    103 userid  H LOCAL
S  SORTOUT   SORT    104 userid  H LOCAL
```

## SDSF显示的数据集类型

| 数据集 | 内容 |
|---|---|
| **JESMSGLG** | JES消息，作业控制语句处理消息 |
| **JESJCL** | 展开过程、应用覆盖和解析符号后的JCL |
| **JESYSMSG** | MVS系统消息 |
| **SYSOUT** | 程序产生的消息（如编译器输出、SORT统计信息） |
| **SORTOUT** | 程序的实际输出（如排序后的数据） |

## SDSF命令

### 作业控制命令

| 命令 | 用途 |
|---|---|
| `S jobname` | 提交作业 |
| `P jobname` | 取消（purge）作业 |
| `H jobname` | 保持作业输出 |
| `?` | 显示作业详情和数据集列表 |

### 输出控制命令

| 命令 | 用途 |
|---|---|
| `/` | 显示作业的JCL |
| `K` | 显示作业的关键统计信息 |
| `A` | 显示作业的ACB信息 |
| `O` | 显示作业的输出选项 |

### 过滤器命令

| 命令 | 用途 |
|---|---|
| `PREFIX=pattern` | 按作业名前缀过滤 |
| `DEST=dest` | 按目的地过滤 |
| `OWNER=userid` | 按用户过滤 |
| `CLASS=c` | 按作业类过滤 |

## 作业日志内容解读

### 典型的作业成功输出结构

```
15.21.28 JOB17653 $HASP373 SORT STARTED - INIT 9 - CLASS 5 - SYS AQTS
15.21.28 JOB17653 IEF403I SORT - STARTED - TIME=15.21.28
15.21.28 JOB17653 IEF404I SORT - ENDED - TIME=15.21.28
15.21.28 JOB17653 $HASP395 SORT ENDED
```

### 条件码（Condition Code）

条件码（CC）表示程序执行的结果 ^[extracted]：

| 条件码 | 含义 |
|---|---|
| `0000` | 程序成功执行 |
| `0004` | 产生警告，但有输出 |
| `0008` | 严重错误，输出可能不可用 |
| `0012` | 致命错误 |
| `S0C7` | 数据例外（abend） |

### EXCP计数

EXCP（Execute Channel Program）计数表示I/O操作的数量 ^[inferred]：

```
EXCP
SERV PAGE SWAP VIO SWAPS
1    211    0     0     0
```

## SDSF与JES2/JES3

SDSF在JES2和JES3环境中均可使用，但某些显示和功能可能不同 ^[extracted]：

- JES2显示系统消息和作业控制信息
- JES3提供更详细的调度和处理器分配信息

## 使用ISPF查看作业输出

除了SDSF，还可以使用ISPF查看作业输出：

1. 在ISPF Primary Option Menu中选择 `3. Utilities`
2. 选择 `6. Job Output` 或类似选项
3. 输入作业名和输出类

## 常见作业输出问题

### 作业未显示

可能原因：
- 作业已完成并从队列中清除
- 作业名过滤条件不正确
- 没有查看正确队列的权限

### 输出被保持

检查：
- 是否指定了HELD输出类
- 是否使用了 `/*OUTPUT HOLD` 语句
- 操作员是否手动保持了输出

### 作业卡住

可能原因：
- 等待资源（数据集、设备）
- 步骤时间过长
- 系统繁忙

## 相关页面

- [[concepts/zos-jcl-basics]] — JCL基础和SDSF介绍
- [[concepts/zos-batch-processing]] — 批处理和作业管理
- [[concepts/jcl-spooling-sysout]] — SYSOUT和输出管理

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 2 and related sections