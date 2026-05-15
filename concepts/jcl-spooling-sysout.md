---
title: JCL SPOOLing and SYSOUT
category: concepts
tags: [ibm, zos, jcl, spool, sysout, jes, output]
sources: [下载/jcl user guide.pdf]
summary: SPOOLing（假脱机）机制和SYSOUT类管理作业输出，包括输出类、目的地控制、格式化、输出限制。
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

# JCL SPOOLing and SYSOUT

## 概述

**SPOOLing（Simultaneous Peripheral Operations On-Line，假脱机）** 是JES使用的一种技术，用于将作业输出存储在磁盘上（而非直接发送到打印机），以便稍后打印或显示 ^[inferred]。

**SYSOUT** 是系统管理的输出数据集，用于处理作业产生的打印、穿孔等输出 ^[extracted]。

## SYSOUT类（Output Class）

SYSOUT参数指定输出数据集的处理方式：

```jcl
//OUTPUT DD SYSOUT=A
```

### 关键参数

| 参数 | 用途 |
|---|---|
| `SYSOUT=class` | 指定输出类（如A、B等） |
| `SYSOUT=*` | 使用JOB语句MSGCLASS参数指定的类 |
| `DEST=destination` | 指定输出目的地（如LOCAL、REMOTE） |
| `FORMS=formname` | 指定打印表格类型 |
| `COPIES=n` | 指定打印份数 |

### 输出类的工作方式

JES维护多个输出队列，每个输出类对应一个队列 ^[extracted]：

- 操作员可以保持（hold）、释放（release）、打印或删除队列中的输出
- 作业可以在JOB语句的MSGCLASS参数中指定默认输出类
- 不同输出类可以有不同的处理优先级和打印设备

## 作业输出限制

可以限制作业输出的数量 ^[extracted]：

```jcl
//JOB1 JOB ACCT01,'NAME',BYTES=(50,WARNING),CARDS=(120,CANCEL)
```

| 参数 | 用途 |
|---|---|
| `BYTES` | 限制spooled的字节数 |
| `CARDS` | 限制穿孔的卡片数 |
| `LINES` | 限制打印的行数 |
| `PAGES` | 限制打印的页数 |
| `WARNING` | 超过限制时发送警告消息 |
| `CANCEL` | 超过限制时取消作业 |

## OUTPUT JCL语句

OUTPUT JCL语句为sysout数据集指定处理选项 ^[extracted]：

```jcl
//OUT1 OUTPUT JESDS=ALL,CLASS=D,COPIES=2,BURST=YES
```

### 关键参数

| 参数 | 用途 |
|---|---|
| `JESDS` | 控制打印哪些系统管理的数据集 |
| `CLASS` | 输出类 |
| `COPIES` | 打印份数 |
| `BURST` | 批量输出模式（用于支持burst的打印机） |
| `NOTIFY` | 打印完成时通知用户 |
| `DEST` | 输出目的地 |

### JESDS参数

`JESDS=ALL` 请求打印所有系统管理的数据集 ^[extracted]：

- 作业日志（JCL语句、过程语句、相关消息）
- 硬拷贝日志（与操作员控制台的所有消息流量）
- 系统消息

## 目的地控制

### JES2 /*ROUTE语句

```jcl
/*ROUTE PRINT destination
/*ROUTE XEQ node
```

### JES3 //*ROUTE语句

```jcl
//*ROUTE XEQ=(main-name,main-name,...)
```

## 输出格式化

### 3800 Printing Subsystem的格式化选项

| 参数 | 用途 |
|---|---|
| `CHARS` | 字符排列集 |
| `FLASH` | 闪光（表单覆盖） |
| `MODIFY` | 修改字符位置 |
| `FCB` | 格式控制缓冲器 |
| `UCS` | 通用字符集 |

### 格式化示例

```jcl
//DD1 DD SYSOUT=(A,,FCB=STD3,CHARS=DUMP)
```

## 作业日志与SYSOUT一起打印

要将作业日志和作业的sysout数据集打印在同一输出列表上，将它们放在同一个输出类中 ^[extracted]：

```jcl
//J1 JOB DF16,MSGCLASS=B
//S1 EXEC PGM=ABC
//OUT DD SYSOUT=*
```

## 保持输出（Holding Output）

可以使用以下方式保持输出供以后查看或打印：

- 在DD语句中指定 `SYSOUT=(class,INCLUDE)` 或使用OUTPUT JCL语句
- 在JES2中使用 `/*OUTPUT` 语句的 `HOLD` 参数
- 在JES3中使用 `//*FORMAT` 语句

## 使用SDSF查看输出

SDSF（System Display and Search Facility）是查看和管理z/OS作业输出的主要工具 ^[extracted]：

1. 选择 `ST` 查看作业状态
2. 输入 `?` 查看作业的数据集列表
3. 输入 `S` 选择要显示的数据集

### SDSF面板显示的数据集

| 数据集 | 内容 |
|---|---|
| `JESMSGLG` | JES消息 |
| `JESJCL` | 展开过程、应用覆盖和解析符号后的JCL |
| `JESYSMSG` | MVS系统消息 |
| `SYSOUT` | 程序产生的消息（如SORT程序的输出） |

## 相关页面

- [[concepts/zos-jcl-basics]] — SDSF概述
- [[concepts/zos-batch-processing]] — 批处理和JES2/JES3
- [[concepts/jcl-scheduling-timing]] — TIME参数和调度
- [[references/zos-jcl-user-guide]] — IBM官方JCL用户指南

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, Part 5 "Tasks for requesting sysout data set resources"