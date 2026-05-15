---
title: JCL Introduction
category: concepts
tags: [ibm, zos, jcl, statements, job-control]
sources: [下载/jcl user guide.pdf]
summary: JCL（作业控制语言）是z/OS批处理作业的描述语言，包含JCL语句（JOB/EXEC/DD等）和JECL语句（JES2/JES3控制语句）两类。
provenance:
  extracted: 0.88
  inferred: 0.08
  ambiguous: 0.04
base_confidence: 0.73
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# JCL Introduction

## JCL概述

**JCL（Job Control Language，作业控制语言）** 是用于向MVS系统描述工作请求和控制作业执行的领域特定语言 ^[extracted]。

可以将JCL比作餐厅的菜单：用户从菜单中选择作业项（程序和数据），通过 waiter（JES）提交到厨房（z/OS），厨师（系统）分配资源完成工作，结果再通过JES返回给用户 ^[extracted]。

## JCL语句类别

MVS JCL语句分为两大类：

### JCL语句（JCL statements）

| 语句 | 名称 | 用途 |
|---|---|---|
| `// JOB` | Job | 标记作业开始，分配作业名，提供安全/会计/标识信息 |
| `// EXEC` | Execute | 标记作业步开始，标识要执行的程序或过程 |
| `// DD` | Data Definition | 标识并描述输入/输出数据集 |
| `//*` | Comment | 包含注释，用于文档说明 |
| `// INCLUDE` | Include | 标识PDS/PDSE中包含JCL语句的成员 |
| `// JCLLIB` | JCL Library | 指定搜索INCLUDE组和过程的库 |
| `// PROC` | Procedure | 标记in-stream过程开始，分配符号参数默认值 |
| `// PEND` | Procedure End | 标记in-stream或cataloged过程的结束 |
| `// SET` | Set | 定义并赋值符号参数 |
| `// IF/THEN/ELSE/ENDIF` | Conditional | 指定作业步的条件执行 |
| `// OUTPUT` | Output JCL | 指定sysout数据集的打印处理选项 |
| `// CNTL/ENDCNTL` | Control | 标记程序控制语句块的开始/结束 |
| `// DELIMITER` | `/*` | 标记输入流中嵌入数据的结束 |
| `// NULL` | `//` | 标记作业结束 |
| `// XMIT` | Transmit | 将输入流记录从一个节点传送到另一个节点 |
| `// COMMAND` | Command | 指定系统转换JCL时发出的MVS或JES命令 |

### JECL语句（JES2/JES3 control statements）

**JES2控制语句：**
- `/*$command` — 通过输入流输入JES2操作员命令
- `/*JOBPARM` — 在输入时指定作业相关参数
- `/*MESSAGE` — 通过操作员控制台向操作员发送消息
- `/*NETACCT` — 指定网络作业的账户号
- `/*NOTIFY` — 指定通知消息的目的地
- `/*OUTPUT` — 为sysout数据集指定处理选项
- `/*PRIORITY` — 分配作业队列选择优先级
- `/*ROUTE` — 指定作业的输出目的地或执行节点
- `/*SETUP` — 请求挂载作业所需的卷
- `/*SIGNOFF / *SIGNON` — 结束/开始远程作业流处理会话
- `/*XEQ` — 指定作业的执行节点
- `/*XMIT` — 将作业或数据流传送到另一个JES2节点

**JES3控制语句：**
- `//**command` — 通过输入流输入JES3操作员命令
- `//*DATASET / *ENDDATASET` — 在输入流中开始/结束输入数据集
- `//*ENDPROCESS / *PROCESS` — 结束/开始一系列//*PROCESS语句
- `//*FORMAT` — 为sysout或JES3管理的打印/穿孔数据集指定处理选项
- `//*MAIN` — 为作业定义选定的处理参数
- `//*NET` — 标识前驱和后继作业在依赖作业控制网中的关系
- `//*NETACCT` — 指定网络作业的账户号
- `//*OPERATOR` — 向操作员发送消息
- `//*PAUSE` — 暂停输入读取器
- `//*ROUTE` — 指定作业的执行节点

## 作业与作业步

每个作业由以下最小语句集组成 ^[extracted]：

1. **JOB语句** — 标记作业开始，每个作业有且只有一个JOB语句
2. **EXEC语句** — 标记作业步开始，标识要执行的程序或过程，每个作业至少有一个EXEC语句
3. **DD语句** — 标识和描述作业步中使用的输入/输出数据集，大多数作业包含多个DD语句

一个作业可以包含多个作业步，每个作业步运行一个程序 ^[extracted]。

## 作业提交流程

```
1. 用户确定作业 (User action)
      ↓
2. 创建JCL (User action)
      ↓
3. 提交作业 (User action)
      ↓
4. JES解释JCL并传递给MVS (System action)
      ↓
5. MVS执行工作 (System action)
      ↓
6. 系统消息从系统流向用户 (System action)
      ↓
7. JES收集输出和作业信息 (System action)
      ↓
8. 用户查看和解释输出 (User action)
```

## 过程（Procedures）

JCL支持两种类型的过程复用 ^[extracted]：

- **In-stream过程** — 嵌入在作业中、可在同一作业内重复执行的命名JCL语句集
- **Cataloged过程** — 存储在分区数据集（PDS）或PDSE（过程库，如SYS1.PROCLIB）中的命名JCL语句集，可被任何作业调用

## 相关页面

- [[concepts/zos-jcl-basics]] — JCL基础：JOB/EXEC/DD语句和SDSF
- [[concepts/zos-batch-processing]] — 批处理和JES2/JES3
- [[concepts/jcl-job-proc-step]] — JOB、EXEC、DD语句的详细参数
- [[references/zos-jcl-user-guide]] — IBM官方JCL用户指南

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Chapter 1 "Introduction - job control statements"