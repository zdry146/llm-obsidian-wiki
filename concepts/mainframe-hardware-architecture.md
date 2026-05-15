---
title: Mainframe Hardware Architecture
category: concepts
tags: [ibm, mainframe, hardware, architecture]
sources: [下载/zOS Basics.pdf]
summary: mainframe硬件架构：Central Processor Complex（CPC）、LPAR逻辑分区、通道子系统（CSS）、CHPID寻址、处理器类型（CP/IFL/zAAP/zIIP/ICF）、多路复用I/O、DASD和存储层次结构。
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

# Mainframe Hardware Architecture

## 系统术语澄清

**CPC（Central Processor Complex）** — 物理硬件的集合，包含主存储、一个或多个中央处理器、定时器和通道 ^[extracted]。

mainframe术语丰富且重叠 ^[extracted]：
- **Box, CEC, CPC, CPU, Machine, Processor, Sysplex, System** — 均可指代整个机器
- **Processors, CPUs, Engines, PUs, CPs** — 指单个处理器内核

历史上，system、processor、CPU可互换使用。随着多处理器系统出现，这些术语产生了歧义。IBM使用CPC指代"盒子"或集中处理 hub ^[extracted]。

## 处理器单元（PU）的角色分类

所有处理器均从等效的处理器单元（PU）开始，由IBM在安装时或之后进行角色分类 ^[extracted]：

| 类型 | 用途 | 许可证激励 |
|------|------|------------|
| **CP** (Central Processor) | 通用操作系统和应用软件 | 按全价计入 |
| **IFL** (Integrated Facility for Linux) | 仅供Linux LPAR或z/VM下的Linux使用 | 有特殊许可优惠 |
| **zAAP** (z/OS Application Assist Processor) | Java工作负载（降低许可证成本） | 不计入机器容量 |
| **zIIP** (z/OS Integrated Information Processor) | 数据库/BI/ERP/CRM工作负载 | 不计入机器容量 |
| **ICF** (Integrated Coupling Facility) | 运行Coupling Facility Control Code | 不计入机器容量 |
| **SAP** (System Assist Processor) | 内部I/O子系统驱动代码 | 对操作系统不可见 |
| **Spare** | 故障时自动替换失败的CP | 热备 |

## 通道子系统（Channel Subsystem）

通道提供I/O设备与内存之间的独立数据和控制路径 ^[extracted]。通道发展历程：

1. **Parallel Channels（并行通道）** — 使用bus和tag两根重铜电缆，最大速率4.5 MBps，最大距离122米，已淘汰
2. **ESCON（Enterprise Systems CONnection）** — 光纤连接，速度更快
3. **FICON（FIber CONnection）** — 当前主流，集成到主处理器盒中

现代mainframe可有多达1024个CHPID（Channel Path Identifier），每个CHPID使用两个十六进制数字 ^[extracted]。

## 逻辑分区（LPAR）

LPAR由PR/SM（Processor Resource/Systems Manager）功能实现 ^[extracted]：

- 最多60个LPAR（实际受内存和I/O限制）
- 每个LPAR运行独立的操作系统实例（z/OS、z/VM、Linux等）
- 内存（Central Storage）不可跨LPAR共享
- 处理器可在LPAR间共享或专用分配
- 分配权重可控制LPAR间的处理器时间比例

**Type 1 hypervisor（原生超visor）** — 直接运行在硬件上的控制程序（如PR/SM），区别于运行在操作系统内的Type 2 hypervisor（如VMware） ^[extracted]。

## 设备寻址

设备地址由四个十六进制数字组成（0000-FFFF），由系统程序员分配 ^[extracted]。

```
地址格式: CCUU (Channel + Control Unit + Unit Address)
例如: 132 = Channel 1, Control Unit 3, Device 2
```

多路径访问：一个设备可通过多个通道地址访问（如132、532、632），操作系统会自动选择可用路径 ^[extracted]。

## DASD存储架构

IBM 3390磁盘驱动器配合3990控制单元，是当前mainframe的标准磁盘配置 ^[extracted]。

现代存储架构（如IBM System Storage DS8000）：
- 多主机适配器（HA）提供控制单元接口
- 内部使用 commodity SCSI磁盘，RAID 5阵列
- 多GB缓存 + 非易失性存储（NVS）写缓冲
- 模拟大量3390磁盘驱动器，对软件完全兼容旧应用

**Parallel Access Volumes（PAV）** — 允许同一磁盘设备并行执行多个I/O，而非排队等待 ^[extracted]。

## 多路复用与I/O虚拟化

现代mainframe的I/O虚拟化包括：
- **Channel paths动态附加** — 根据工作负载需求动态连接
- **ESCON/FICON Director（开关）** — 允许多系统共享控制单元和设备
- **Multiple Allegiance** — 多系统同时访问同一磁盘设备

## 相关页面

- [[mainframe-history-s360]] — 从S/360到今天的演进历史
- [[mainframe-high-availability-sysplex]] — LPAR、Sysplex和高可用性
- [[zos-overview]] — z/OS软件视角的虚拟存储和系统概念
- [[zos-io-data-management]] — I/O和数据管理