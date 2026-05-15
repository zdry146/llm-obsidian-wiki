---
title: z/OS JCL User's Guide
category: references
tags: [ibm, zos, jcl, jes, ibm-official]
sources: [下载/jcl user guide.pdf]
summary: IBM z/OS 3.1 MVS JCL User's Guide (SA23-1386-60)，官方JCL用户指南，涵盖作业提交、控制语句、数据集资源请求、SPOOL输出管理。
provenance:
  extracted: 0.95
  inferred: 0.03
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:35:00Z
updated: 2026-05-15T01:35:00Z
---

# z/OS JCL User's Guide

## 文档概述

本参考页面基于 **IBM z/OS 3.1 MVS JCL User's Guide**（文档号 SA23-1386-60，更新日期 2025-05-15，IBM版权 1988-2024）。

该指南是z/OS JCL的**用户手册**，配套文档是 **z/OS MVS JCL Reference**，后者提供语句编码详情。本书旨在帮助程序员决定如何执行作业控制任务，而非如何编写语句语法 ^[extracted]。

## 文档结构

指南分为六个部分：

| 部分 | 内容 |
|---|---|
| Part 1: Introduction | JCL语句介绍、JCL入门、作业控制任务概览 |
| Part 2: Tasks for entering jobs | 作业识别、执行、输入控制、通信、保护、资源控制 |
| Part 3: Tasks for processing jobs | 条件执行、性能控制 |
| Part 4: Tasks for requesting data set resources | 数据集识别、描述、保护、分配、处理控制、结束处理 |
| Part 5: Tasks for requesting sysout data set resources | SYSOUT识别、目的地控制、保护、性能控制、输出格式、输出限制 |
| Part 6: Examples | 汇编/链接/运行示例、多输出示例、JES2/JES3输出获取 |

附录涵盖：GDG（生成数据组）、VSAM数据集、SMS数据集、可访问性说明。

## 核心概念映射

本指南涵盖以下主要概念领域，可通过以下wiki页面深入了解：

- [[concepts/jcl-introduction]] — JCL和JECL语句概述
- [[concepts/jcl-job-proc-step]] — JOB/EXEC/DD三类核心语句
- [[concepts/jcl-data-set-disposition]] — 数据集DISP参数和编目机制
- [[concepts/jcl-conditional-execution]] — IF/THEN/ELSE/ENDIF和COND条件执行
- [[concepts/jcl-spooling-sysout]] — SPOOLing和SYSOUT输出处理
- [[concepts/jcl-scheduling-timing]] — TIME参数、deadline/periodic调度、JES2/JES3差异
- [[concepts/jcl-restart-checkpoint]] — 检查点/重启机制
- [[concepts/jcl-procedures-instream]] — In-stream过程和cataloged过程
- [[concepts/jcl-storage-resources]] — REGION存储请求、私有库资源控制
- [[concepts/jcl-utilities]] — IBM提供的实用程序（IEBGENER、IEBCOPY等）
- [[concepts/jcl-sdsf-output]] — SDSF查看和管理作业输出

## 关键引用说明

- **配套文档**：z/OS MVS JCL Reference — 提供完整的JCL语句语法参考
- **JES2 vs JES3**：两者的控制语句和调度机制存在显著差异，指南中分别说明
- **存储管理**：当SMS（Storage Management Subsystem）活动时，数据集属性由数据类、存储类、管理类决定

## 来源

- IBM z/OS 3.1 MVS JCL User's Guide, SA23-1386-60, © Copyright IBM Corp. 1988, 2024