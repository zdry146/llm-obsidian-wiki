---
title: Programming Languages on z/OS
category: concepts
tags: [ibm, zos, cobol, assembler, pli, java, rexx]
sources: [下载/zOS Basics.pdf]
summary: z/OS支持多种编程语言：COBOL（商业主力）、Assembler（系统级）、PL/I、C/C++、Java、REXX（脚本）和Language Environment统一运行时。不同语言有不同的编译和链接过程。
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

# Programming Languages on z/OS

## 语言概览

z/OS支持多种编程语言 ^[extracted]：

| 语言 | 主要用途 | 特点 |
|------|----------|------|
| **COBOL** | 商业应用（事务处理、批处理） | 主导语言，数十亿行代码在生产 |
| **Assembler** | 系统编程、性能关键代码 | 直接控制硬件，效率最高 |
| **PL/I** | 通用编程 | IBM开发，融合多种语言特性 |
| **C/C++** | 系统和应用开发 | 与UNIX环境集成 |
| **Java** | 现代应用、web服务 | 可跨平台，在zAAP上运行 |
| **REXX** | 脚本、自动化、命令过程 | 解释型，易学易用 |

## COBOL

**COBOL（Common Business Oriented Language）** 是mainframe商业应用开发的主导语言 ^[extracted]：

- 专为商业数据处理设计（财务、库存、订单处理等）
- 英文式语法，可读性高
- 大量生产系统依赖COBOL（估计全球70%以上的商业数据处理由COBOL完成）^[inferred]
- 标准化程度高（COBOL 85、COBOL 2002）

**COBOL程序结构**：
```
IDENTIFICATION DIVISION.
PROGRAM-ID. HELLOWORLD.
ENVIRONMENT DIVISION.
DATA DIVISION.
PROCEDURE DIVISION.
    DISPLAY 'Hello, world'.
    STOP RUN.
```

COBOL在mainframe上与CICS和DB2深度集成 ^[extracted]：
- 参见：[[cics-programming-cobol]]、[[cics-application-development]]

## Assembler

**Assembler（汇编语言）** 直接翻译为机器指令 ^[extracted]：

- 最高性能，适用于硬件级操作
- 系统编程必备（I/O驱动、通道程序）
- 硬件相关功能的唯一选择
- 学习曲线陡峭，但可完全控制CPU和内存

## PL/I

**PL/I** 由IBM于1964年开发，融合了COBOL、FORTRAN和ALGOL的特性 ^[extracted]：
- 多用途语言，支持商业和科学计算
- 结构化编程特性
- 更好的错误处理机制
- 相对COBOL使用较少，但在某些大型安装中仍很重要

## C/C++

**C/C++** 在z/OS上用于系统应用开发 ^[extracted]：
- 需使用OS/390 C/C++ Compiler
- 与z/OS UNIX环境（OMVS）集成
- 可调用z/OS服务
- 企业级应用的现代选择

## Java

**Java** 在z/OS上的运行方式 ^[extracted]：

- 使用IBM SDK for z/OS
- 可在专用**zAAP**处理器上运行（降低许可证成本）
- Java程序通过JNI调用COBOL/C/Assembler程序
- Web应用、web服务和微服务的现代选择

z/OS Java还支持：
- JDBC（数据库连接）
- JCA（企业信息系统连接器）
- WebSphere Application Server for z/OS

## REXX

**REXX（Restructured Extended Executor）** 是z/OS的脚本语言 ^[extracted]：

- 解释型语言，无需编译
- 易于学习和使用
- 常用于：
  - 系统管理自动化
  - 命令过程
  - 数据处理
  - 工具开发
- 可交互式运行（TSO）或批处理运行

## Language Environment（LE）

**Language Environment** 是z/OS的统一运行时环境 ^[extracted]：

- 所有LE支持的语言共享同一运行时
- 提供一致的错误处理、内存管理
- C/C++、COBOL、PL/I可互相调用
- 简化混合语言编程

## 语言选择因素

选择语言的考虑因素 ^[extracted]：
- 应用类型（批处理 vs 在线事务）
- 性能要求
- 现有代码库和团队技能
- 与现有系统（CICS、DB2、IMS）的集成需求
- 许可证成本

## 相关页面

- [[zos-application-development]] — 应用开发生命周期和编译过程
- [[cics-programming-cobol]] — CICS COBOL编程实践
- [[zos-jcl-basics]] — 编译和链接的JCL
- [[cics-overview]] — CICS事务处理系统