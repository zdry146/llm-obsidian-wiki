---
title: Introduction to the New Mainframe: z/OS Basics
category: references
tags: [ibm, zos, mainframe, redbook, textbook]
sources: [下载/zOS Basics.pdf]
summary: IBM Redbooks SG24-6366-02 (March 2011), z/OS基础教材，系统介绍 mainframe硬件架构、z/OS操作系统、批处理/OLTP工作负载、JCL、编程语言和系统管理。出处：ibm.com/redbooks
provenance:
  extracted: 0.95
  inferred: 0.05
  ambiguous: 0.00
base_confidence: 0.57
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:21:00Z
updated: 2026-05-15T01:21:00Z
---

# Introduction to the New Mainframe: z/OS Basics

## 书籍概览

**书名：** Introduction to the New Mainframe: z/OS Basics
**作者：** Mike Ebbers, John Kettner, Wayne O'Brien, Bill Ogden
**出版方：** IBM International Technical Support Organization (ITSO), Poughkeepsie Center
**红皮书编号：** SG24-6366-02
**版本：** Third Edition, March 2011
**来源：** ibm.com/redbooks

本书是IBM mainframe入门教材的代表作，面向信息系统的学生和初学者，系统讲解大型机的基础知识和技能。书中涵盖z/OS操作系统的核心概念，是进一步学习z/OS高级主题（系统管理或应用编程）的先修教材。

## 书籍结构

全书分为四个部分，共19章：

### Part 1: Introduction to z/OS and the mainframe environment
- **Chapter 1** — Introduction to the new mainframe： mainframe历史、用途、角色与职责
- **Chapter 2** — Mainframe hardware systems and high availability：硬件架构、Sysplex、高可用
- **Chapter 3** — z/OS overview：z/OS操作系统、虚拟存储、工作负载管理、I/O和数据管理
- **Chapter 4** — TSO/E, ISPF, and UNIX：交互界面
- **Chapter 5** — Working with data sets：数据集、访问方法、VSAM、DFSMS
- **Chapter 6** — Using Job Control Language (JCL) and SDSF：JCL基础
- **Chapter 7** — Batch processing and the job entry subsystem：批处理、JES2/JES3、initiator

### Part 2: Application programming on z/OS
- **Chapter 8** — Designing and developing applications for z/OS：应用开发生命周期
- **Chapter 9** — Using programming languages on z/OS：COBOL、Assembler、PL/I、C/C++、Java、REXX、Language Environment
- **Chapter 10** — Compiling and link-editing a program on z/OS：编译、链接、执行

### Part 3: Online workloads for z/OS
- **Chapter 11** — Transaction management systems on z/OS：CICS和IMS概述
- **Chapter 12** — Database management systems on z/OS：DB2、SQL、IMS数据库
- **Chapter 13** — z/OS HTTP Server：Web服务
- **Chapter 14** — IBM WebSphere Application Server on z/OS：J2EE应用服务器
- **Chapter 15** — Messaging and queuing：WebSphere MQ

### Part 4: System programming on z/OS
- **Chapter 16** — Overview of system programming：系统程序员的角色
- **Chapter 17** — Using System Modification Program/Extended (SMP/E)：软件管理
- **Chapter 18** — Security on z/OS：安全设施、RACF
- **Chapter 19** — Network communications on z/OS：TCP/IP、VTAM、通讯服务器

## 核心主题

1. **System/360的革命性意义（1964）** — 首次提出通用计算架构，360度覆盖所有可能用途，引入微码（microcode）概念 ^[inferred]
2. **RAS特性** — Reliability（可靠性）、Availability（可用性）、Serviceability（可服务性）是mainframe设计的核心原则
3. **虚拟化** — LPAR（逻辑分区）、PR/SM、z/VM实现硬件资源共享，是现代云计算的先驱 ^[inferred]
4. **Parallel Sysplex** — 32台z/OS镜像共享数据，提供近连续可用性（99.999%）
5. **z/OS是share-everything架构** — 处理器、内存、I/O设备均可跨LPAR共享，区别于分布式环境的share-nothing架构
6. **兼容性至上** — 1964年的应用仍可在今天的System z上运行，这是mainframe区别于其他平台的根本特征

## 与本Wiki其他内容的关系

- 本书Part 3的CICS章节与现有 [[cics-overview]] 页面形成互补，本书侧重概念理论，现有页面侧重实践
- 本书的DB2章节（P.444）与现有 [[db2-12-architecture-design]] 页面可交叉引用
- COBOL编程语言（Chapter 9）是 [[cics-programming-cobol]] 的语言基础
- 本书Chapter 7的批处理模型与 [[continuous-delivery]] 的自动化部署概念相关

## 延伸阅读

- IBM z/OS Internet Library: http://www-03.ibm.com/systems/z/os/zos/bkserv/
- Appendix A: IBM mainframe历史（从System/360到System z）
- 红皮书SG24-8383-00: DB2 12 for z/OS Technical Overview（[[ibm-db2-12-zos-technical-overview]]）