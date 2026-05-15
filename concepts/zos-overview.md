---
title: z/OS Overview
category: concepts
tags: [ibm, zos, operating-system, mainframe]
sources: [下载/zOS Basics.pdf]
summary: z/OS是IBM mainframe的主流操作系统，基于z/Architecture，提供虚拟存储、工作负载管理（WLM）、I/O数据管理、交叉内存服务等核心功能，是高度模块化设计的64位操作系统。
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

# z/OS Overview

## 什么是z/OS

z/OS是IBM mainframe上使用最广泛的操作系统 ^[extracted]。它基于z/Architecture（64位架构），设计用于支持大规模商业数据处理，提供最高级别的安全性、可用性和性能 ^[extracted]。

**核心定位：** "Businesses use mainframes to host commercial databases, transaction servers, and applications that require a greater degree of security and availability than is commonly found on smaller-scale machines." ^[extracted]

## z/OS与其他操作系统

一台mainframe可同时运行多个操作系统 ^[extracted]：
- **z/OS** — 主力商业操作系统
- **z/VM** — 虚拟机/超visor，可运行多个客户操作系统
- **z/VSE** — 面向较小规模mainframe的批处理和事务处理系统
- **Linux on System z** — 在mainframe上运行的Linux（31位或64位）
- **z/TPF** — 高事务量专用系统（如航空公司预订、信用卡处理）

## 虚拟存储（Virtual Storage）

虚拟存储是z/OS的核心概念 ^[extracted]。z/OS使用虚拟存储技术，使应用程序认为系统有更大的内存：

- **虚拟存储** — 应用程序可寻址的空间，远大于实际物理内存
- **实际内存（Central Storage/CSTOR）** — 硬件实际内存
- **辅助存储（AUX）** — 页面换出到DASD的空间

z/OS的虚拟存储架构允许应用程序使用大于实际内存的地址空间，这是从S/370的虚拟存储扩展发展而来 ^[extracted]。

### 虚拟存储的优势

1. **内存保护** — 每个用户的地址空间相互隔离，防止未授权访问
2. **多用户支持** — 数千用户可同时使用系统，内存资源按需分配
3. **程序独立性** — 程序不需要了解物理内存布局

## 工作负载管理（Workload Management, WLM）

WLM是z/OS的核心组件，根据服务级别目标自动管理资源分配 ^[extracted]。

WLM功能包括：
- 定义服务类别（service classes）和响应时间目标
- 动态调整处理器、内存和I/O资源分配
- 在系统内和跨Sysplex实现工作负载平衡
- 支持批量和在线工作负载的混合运行

参见：[[zos-workload-management]]

## I/O和数据管理

z/OS提供复杂的I/O子系统 ^[extracted]：

- **Channel Subsystem（CSS）** — 管理I/O通道和设备访问
- **Access Methods（访问方法）** — 如VSAM、BSAM、QSAM，提供数据读写接口
- **DFSMS** — Data Facility Storage Management Subsystem，负责存储管理

详见：[[zos-io-data-management]]

## 交叉内存服务（Cross-Memory Services）

允许在一个地址空间运行的程序调用另一个地址空间中的服务，同时保持地址空间隔离 ^[extracted]。这是z/OS实现安全内核服务和应用程序间通信的关键机制。

## z/OS的关键定义特征

z/OS的设计体现了以下关键特征 ^[extracted]：

1. **持续兼容性** — 新版本必须运行数十年前的应用程序
2. **模块化设计** — 系统组件可独立升级而不影响其他组件
3. **安全内置** — 硬件和软件多层安全架构，EAL5认证
4. **RAS优先** — 系统设计以可靠性、可用性和可服务性为核心

## 与UNIX的比较

z/OS与UNIX系统有一些相似之处 ^[extracted]：
- 层级文件系统
- 进程和任务概念
- IPC（进程间通信）能力

但关键区别：
- z/OS原生使用EBCDIC字符集（非ASCII）
- z/OS使用3270终端架构（非TTY）
- z/OS的I/O处理和内存管理模式与UNIX完全不同 ^[inferred]

## 相关页面

- [[mainframe-hardware-architecture]] — z/OS运行的硬件基础
- [[zos-workload-management]] — WLM工作负载管理详解
- [[zos-io-data-management]] — I/O和数据管理
- [[zos-tso-e-ispf-unix]] — 交互界面TSO/E、ISPF、z/OS UNIX
- [[zos-jcl-basics]] — JCL作业控制语言
- [[zos-batch-processing]] — 批处理和JES2/JES3