---
title: z/OS JCL Basics
category: concepts
tags: [ibm, zos, jcl, batch, sdsf]
sources: [下载/zOS Basics.pdf]
summary: JCL（Job Control Language）是z/OS批处理作业的控制语言，由JOB、EXEC、DD三类语句组成，SDSF用于查看和管理作业输出，系统库（PROCLIB）存储标准流程。
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

# z/OS JCL Basics

## JCL概述

**JCL（Job Control Language）** 是用于描述批处理作业给z/OS的控制语言 ^[extracted]。

每个JCL作业由三种语句类型组成：
1. **JOB** — 标识作业，传递参数给作业管理系统
2. **EXEC** — 标识要执行的程序或过程
3. **DD（Data Definition）** — 描述输入/输出数据

## JOB语句

```jcl
//JOBNAME JOB (ACCT),'NAME',CLASS=A,MSGCLASS=X,NOTIFY=USER1
```

关键参数：
- **CLASS** — 作业类（A-Z，定义作业优先级和运行方式）
- **MSGCLASS** — 输出消息的打印类
- **NOTIFY** — 作业完成时发送消息的用户ID
- **PRTY** — 作业优先级（高于CLASS）
- **COND** — 条件执行控制

## EXEC语句

```jcl
//STEP1 EXEC PGM=PROGRAMM,PARM='PARAMETER'
//   或
//STEP1 EXEC PROC=PROCEDURE,PARM='OVERRIDE'
```

关键参数：
- **PGM** — 程序名称（编译后的load module）
- **PROC** — 过程名称（存储在PROCLIB中）
- **PARM** — 传递给程序的参数

## DD语句

```jcl
//DDNAME DD DSN=DATASET.NAME,DISP=SHR,SPACE=(TRK,(10,5)),UNIT=SYSDA
```

关键参数：
- **DSN** — 数据集名称
- **DISP** — 数据集处置状态：
  - `NEW` — 创建新数据集
  - `OLD` — 独占使用
  - `SHR` — 共享使用（多个作业可同时读取）
  - `MOD` — 追加到现有数据集
- **SPACE** — 分配空间量（TRK/CYL和主/副分配量）
- **UNIT** — 设备类型或设备组名

## 数据集DISP参数详解

DISP是三参数结构：`DISP=(status,normal-disposition,abnormal-disposition)` ^[extracted]：

| 状态 | 含义 |
|------|------|
| `CATLG` | 创建并编目数据集 |
| `KEEP` | 保留数据集（不编目） |
| `DELETE` | 删除数据集 |
| `PASS` | 传递给后续步骤 |

## JCL Procedures（过程）

Procedure是可重用的JCL代码段 ^[extracted]：

```jcl
//MYPROC PROC
//STEP1 EXEC PGM=IEFBR14
//DD1 DD DSN=USER.DATA,DISP=(NEW,CATLG),SPACE=(TRK,(1,1))
// PEND
```

调用过程：
```jcl
//JOB1 JOB
//EXEC PROC=MYPROC
```

**系统过程库（PROCLIB）** 存储标准流程，默认：`SYS1.PROCLIB` ^[extracted]。

## 符号文件（Symbolic Parameters）

JCL支持符号参数，方便过程复用 ^[extracted]：

```jcl
//MYPROC PROC DSN=USER.DATA
//STEP1 EXEC PGM=PROGRAMM
//DD1 DD DSN=&DSN
// PEND
```

## SDSF（System Display and Search Facility）

SDSF是查看和管理z/OS作业输出的主要工具 ^[extracted]：

- **DA** — 显示活动作业（Daemon）
- **I** — 显示输入队列的作业
- **O** — 显示输出队列
- **ST** — 显示执行中的作业状态

常用SDSF命令：
- `S jobname` — 提交作业
- `P jobname` — 取消作业
- `?` — 显示作业详情

## 常见JCL错误

1. **JOB NOT EXECUTED** — JCL语法错误
2. **DATA SET NOT FOUND** — 数据集不存在或未编目
3. **CONDITION CODE** — 程序返回非零退出码
4. **ABEND** — 异常结束（如S0C7数据例外）

## 相关页面

- [[zos-batch-processing]] — 批处理和JES2/JES3
- [[zos-data-sets]] — 数据集和命名
- [[zos-programming-languages]] — 编译和链接过程
- [[cics-application-development]] — CICS应用的JCL相关部分