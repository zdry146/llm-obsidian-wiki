---
title: JCL JOB, EXEC, and DD Statements
category: concepts
tags: [ibm, zos, jcl, job, exec, dd, statement]
sources: [下载/jcl user guide.pdf]
summary: JCL的三类核心语句：JOB（标识作业）、EXEC（标识程序/过程执行）、DD（描述数据）。每类语句有多个关键参数控制作业行为。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.72
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL JOB, EXEC, and DD Statements

## 概述

JCL作业由三类核心语句控制：JOB、EXEC和DD ^[extracted]。

## JOB语句

`JOB`语句标记作业开始，分配作业名，提供安全、会计和标识信息 ^[extracted]。

```jcl
//JOBNAME JOB (ACCT),'NAME',CLASS=A,MSGCLASS=X,NOTIFY=USER1
```

### 关键参数

| 参数 | 用途 |
|---|---|
| `CLASS` | 作业类（A-Z），定义作业优先级和运行方式 |
| `MSGCLASS` | 输出消息的打印类 |
| `MSGLEVEL` | 控制JCL语句和分配消息的打印级别 |
| `NOTIFY` | 作业完成时发送消息的用户ID |
| `PRTY` | 作业优先级（高于CLASS） |
| `COND` | 条件执行控制 |
| `SCHENV` | WLM调度环境名称 |
| `ADDRSPC` | 虚拟或实（central）存储类型 |
| `REGION` | 作业可用的存储量 |
| `TIME` | 作业或步骤的CPU时间限制 |

### 作业标识

每个作业必须在JOB语句的作业名字段中标识 ^[extracted]：
```jcl
//MYJOB JOB
```

下一个JOB语句或输入流结束标记作业的结束。

## EXEC语句

`EXEC`语句标记作业步开始，分配步名，标识要执行的程序或过程 ^[extracted]。

```jcl
//STEP1 EXEC PGM=PROGRAMM,PARM='PARAMETER'
//   或
//STEP1 EXEC PROC=PROCEDURE,PARM='OVERRIDE'
```

### 关键参数

| 参数 | 用途 |
|---|---|
| `PGM` | 程序名（编译后的load module） |
| `PROC` | 过程名（存储在PROCLIB中） |
| `PARM` | 传递给程序的参数 |
| `COND` | 条件执行控制 |
| `REGION` | 该步骤可用的存储量（覆盖JOB语句的REGION） |
| `TIME` | 该步骤的CPU时间限制 |

### 步骤命名

所有步骤都应命名 ^[extracted]。系统使用步骤名在消息中标识。如果省略步骤名，该字段在消息中留空，导致难以确定哪个步骤产生了问题。

## DD语句

`DD（Data Definition）`语句标识并描述作业步中使用的输入和输出数据 ^[extracted]。

```jcl
//DDNAME DD DSN=DATASET.NAME,DISP=SHR,SPACE=(TRK,(10,5)),UNIT=SYSDA
```

### 关键参数

| 参数 | 用途 |
|---|---|
| `DSN` | 数据集名称 |
| `DISP` | 数据集处置状态：`NEW`, `OLD`, `SHR`, `MOD` |
| `SPACE` | 分配空间量（TRK/CYL/BLKS）和主/副分配量 |
| `UNIT` | 设备类型、设备号或设备组名 |
| `VOLUME` | 卷序列号、卷引用 |
| `DCB` | 数据控制块参数 |
| `SYSOUT` | 系统输出数据集 |
| `DUMMY` | 标识空设备 |

### DISP参数详解

DISP是三参数结构：`DISP=(status,normal-disposition,abnormal-disposition)` ^[extracted]：

| 状态 | 含义 |
|------|------|
| `NEW` | 创建新数据集 |
| `OLD` | 独占使用 |
| `SHR` | 共享使用（多个作业可同时读取） |
| `MOD` | 追加到现有数据集 |

| 处置 | 含义 |
|------|------|
| `CATLG` | 创建并编目数据集 |
| `KEEP` | 保留数据集（不编目） |
| `DELETE` | 删除数据集 |
| `PASS` | 传递给后续步骤 |

### 数据集名称规则

- 永久数据集：指定完整名称如 `DSN=MY.DATA.SET`
- 临时数据集：`DSN=&&TEMP`（系统生成qualified name）
- 成员引用：`DSN=PDS(MEMBER)` ^[extracted]

## 典型作业示例

```jcl
//SORT JOB 'ACCT01','USER NAME',CLASS=A,MSGCLASS=H
//STEP1 EXEC PGM=SORT
//SYSIN DD *
  SORT FIELDS=(1,75,CH,A)
/*
//SYSOUT DD SYSOUT=*
//SORTIN DD DSN=INPUT.DATA,DISP=SHR
//SORTOUT DD DSN=&&SORTED.OUTPUT,
//    DISP=(NEW,CATLG,DELETE),
//    SPACE=(TRK,(10,5)),UNIT=SYSDA
```

## 相关页面

- [[concepts/zos-jcl-basics]] — JCL基础概述
- [[concepts/jcl-data-set-disposition]] — 数据集DISP和编目机制详解
- [[concepts/jcl-conditional-execution]] — COND参数和IF/THEN/ELSE/ENDIF条件执行
- [[concepts/jcl-storage-resources]] — REGION和ADDRSPC存储请求

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapters 4-5 and related sections