---
title: CICS COBOL Hello World
category: concepts
tags: [cics, cobol, hello-world, programming, zos]
sources: [/home/openclaw/下载/CICS.pdf]
summary: 对比普通COBOL batch程序与CICS COBOL程序的Hello World实现，展示EXEC CICS API的基本用法。
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# CICS COBOL Hello World

## 普通COBOL Batch程序

```cobol
IDENTIFICATION DIVISION.
PROGRAM-ID. HELLO.
PROCEDURE DIVISION.
DISPLAY 'HELLO WORLD'.
STOP RUN.
```

这是一个普通的COBOL批处理程序，直接输出到标准输出。 ^[extracted]

## CICS COBOL程序

```cobol
IDENTIFICATION DIVISION.
PROGRAM-ID. HELLO.
PROCEDURE DIVISION.
EXEC CICS SEND TEXT ('HELLO-WORLD') END-EXEC.
EXEC CICS RETURN END-EXEC.
```

同一个逻辑，使用CICS API重写 ^[extracted]。

### 两行CICS API调用解析

#### `EXEC CICS SEND TEXT ('HELLO-WORLD') END-EXEC`

将字符串"Hello World"发送到启动该程序的终端 ^[extracted]。

#### `EXEC CICS RETURN END-EXEC`

通知CICS当前程序已完成执行 ^[extracted]。

## 程序结构

### IDENTIFICATION DIVISION

程序名称为"Hello" ^[extracted]。

### PROCEDURE DIVISION

这是实际业务逻辑所在的位置 ^[extracted]。

## 与Batch程序的本质区别

| 特性 | 普通COBOL Batch | CICS COBOL程序 |
|---|---|---|
| 输出目标 | 标准输出/Job log | 启动程序的3270终端 |
| 运行环境 | 批处理JCL | CICS Region |
| 事务管理 | 无 | CICS自动管理 |
| API调用 | 无 | EXEC CICS命令 |

CICS提供API让程序与各种资源交互，而不是直接进行I/O操作 ^[inferred]。

## 后续演进

这个Hello World示例过于简单。真实的应用需要处理：

- 多个程序之间的调用（通过`EXEC CICS LINK`）
- 状态管理（通过COMMAREA或Channels/Containers）
- 文件和数据库访问（通过`EXEC CICS READ/WRITE`）
- 错误处理和响应码检查 ^[inferred]

参见[[concepts/cics-exec-api]]了解完整的EXEC CICS API模型。

## 相关概念

- [[concepts/cics-overview]] — CICS概述
- [[concepts/cics-exec-api]] — EXEC CICS API详解
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 1, Example 1-1 and 1-2, December 2020