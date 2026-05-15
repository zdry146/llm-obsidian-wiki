---
title: EXEC CICS API
category: concepts
tags: [cics, api, exec-cics, cobol, response-codes, translator]
sources: [/home/openclaw/下载/CICS.pdf]
summary: EXEC CICS API是CICS应用与资源交互的接口，包含命令级编程接口、翻译器、EIB响应码机制，支持340+命令。
provenance:
  extracted: 0.82
  inferred: 0.13
  ambiguous: 0.05
base_confidence: 0.80
lifecycle: draft
lifecycle_changed: 2026-05-14
created: 2026-05-14
updated: 2026-05-14
---

# EXEC CICS API

## API架构概述

CICS作为应用服务器，通过API将资源管理责任从应用程序员手中抽象出来 ^[extracted]。例如，CICS自动打开和关闭文件，COBOL程序不需要在FILE SECTION中定义文件描述符 ^[extracted]。

EXEC CICS API由以下组件构成 ^[extracted]：

```
用户程序 --EXEC CICS命令--> EXEC接口 --> CICS服务模块
                              ↑
                          EIB（执行接口块）
```

### 核心组件

1. **EXEC CICS命令** — 写在COBOL程序中的API调用
2. **EXEC接口模块（DFHEIP）** — 处理命令路由
3. **EXEC处理器模块** — 各组件对应的处理器（如文件控制组件DFHEIFC）
4. **执行接口块（EIB, DFHEIBLK）** — 存储当前CICS请求的状态
5. **EXEC存根（DFHEI1）** — 链接到COBOL程序中的调用点
6. **命令级翻译器** — 将EXEC CICS命令转换为COBOL语句

## 命令格式

```cobol
EXEC CICS command option(arg) ... END-EXEC
```

- `EXEC CICS` 标识符告诉翻译器这是一个CICS API命令 ^[extracted]
- `command` 是要执行的功能（如READ、WRITE、LINK）
- `option(arg)` 是选项和参数
- `END-EXEC` 标识命令结束

**示例：**
```cobol
EXEC CICS READ FILE('PAYROLL') RIDFLD(ws-key) INTO(payroll-record) END-EXEC
```

这个命令从名为'PAYROLL'的文件中读取记录，键值为ws-key变量，返回数据到payroll-record变量 ^[extracted]。

## COBOL翻译器

EXEC CICS命令**不是COBOL保留字**，因此需要翻译器将它们转换为COBOL语句才能被COBOL编译器理解 ^[extracted]。

翻译器可以集成到COBOL编译器中，也可以作为独立的预编译步骤运行 ^[extracted]。

### 翻译过程

```cobol
* 原文:
EXEC CICS READ FILE('PAYROLL') RIDFLD(ws-key) INTO(payroll-record) END-EXEC

* 翻译后变成:
CALL 'DFHEI1' USING KEY-I AREA-O ...
```

翻译器同时会将EIB复制书（DFHEIBLK和DFHCOMMAREA）拷贝到LINKAGE SECTION ^[extracted]。

## 响应码（Response Codes）

命令执行后，响应信息存放在EIB字段中：

- **EIBRESP** — 主响应码，0表示成功，非零表示异常（如NOT-FOUND = 13）
- **EIBRESP2** — 辅助响应码，用于进一步细分异常原因

### 检查响应码的两种方式

**方式1：RESP选项（推荐）**
```cobol
EXEC CICS READ FILE('PAYROLL') RIDFLD(ws-key) INTO(payroll-record)
    RESP(WC) RESP2(RC2) END-EXEC
IF WC = DFHRESP(NORMAL) THEN
    ...
END-IF
```

**方式2：nohandle选项 + 直接检查EIB**
```cobol
EXEC CICS READ FILE('PAYROLL') RIDFLD(ws-key) INTO(payroll-record)
    nohandle END-EXEC
* 然后检查EIBRESP和EIBRESP2字段
```

如果没有处理条件（既不用RESP也不用nohandle），CICS会发出abend并终止程序 ^[extracted]。

## 常用命令分类

CICS提供超过**340个命令** ^[extracted]。主要类别：

| 类别 | 示例命令 | 用途 |
|---|---|---|
| 文件控制 | READ, WRITE, REWRITE, DELETE | 访问VSAM文件 |
| 程序控制 | LINK, XCTL, RETURN | 程序间调用和流程控制 |
| 终端I/O | SEND, RECEIVE | 与3270终端通信 |
| 存储 | GETMAIN, FREEMAIN | 动态内存管理 |
| 事务管理 | START, RETRIEVE, WAIT | 启动和等待异步任务 |
| 容器 | PUT CONTAINER, GET CONTAINER | 通道和容器数据传递 |
| Web服务 | WEB SEND, WEB RECEIVE | HTTP/REST交互 |
| 事件 | SIGNAL EVENT | 事件处理 |

## API调用流程

以READ命令为例 ^[extracted]：

1. COBOL程序执行`EXEC CICS READ...`，调用存根DFHEI1
2. DFHEI1定位到EXEC接口模块DFHEIP
3. DFHEIP分支到相应的处理器模块（如文件控制的DFHEIFC）
4. 处理器模块验证参数列表，然后分支到CICS服务模块
5. 如果参数正确，从VSAM文件检索记录
6. 控制通过处理器模块返回DFHEIP
7. DFHEIP用命令状态更新EIB（DFHEIBLK）
8. 控制返回应用程序

## 新增API

随着CICS演进，不断有新的API加入，如web命令（支持web协议）、异步命令、事件信号命令（`SIGNAL EVENT`）等 ^[extracted]。

## 相关概念

- [[concepts/cics-hello-world-cobol]] — API使用示例
- [[concepts/cics-programming-cobol]] — COBOL编程实践
- [[concepts/cics-channels-containers]] — Channels和Containers API
- [[concepts/cics-async-event-processing]] — 异步编程和事件处理API
- [[references/modernizing-applications-ibm-cics]] — 原始红皮书来源

## 来源

- IBM Redbooks REDP-5628-00, Chapter 3 "Coding applications to run in IBM CICS", December 2020