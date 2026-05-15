---
title: System/360 History and Mainframe Evolution
category: concepts
tags: [ibm, mainframe, history, system360]
sources: [下载/zOS Basics.pdf]
summary: IBM System/360于1964年发布，开创通用计算时代，引入微码概念，此后每十年一次重大架构升级：S/370、ESA/390、z/Architecture，持续兼容旧应用是核心特征。
provenance:
  extracted: 0.80
  inferred: 0.15
  ambiguous: 0.05
base_confidence: 0.57
lifecycle: draft
lifecycle_changed: 2026-05-15
created: 2026-05-15T01:21:00Z
updated: 2026-05-15T01:21:00Z
---

# System/360 History and Mainframe Evolution

## 背景：1964年之前的mainframe

第一代大型机（1950年代），如IBM 705（1954年）和IBM 1401（1959年），与后来的强大机器相去甚远。IBM 1401被称为计算机业的"Model T"，是首款可被众多企业负担的大规模生产的晶体管化商用计算机 ^[extracted]。

1960年代，mainframe制造厂商开始在硬件和软件上推行标准化，这彻底改变了计算历史的进程。

## System/360：转折点

IBM于1964年推出**System/360（S/360）**，标志着第三代计算的开始——首个**通用计算机**。此前，系统要么专用于商业计算，要么专用于科学计算。S/360能够同时执行两种类型的计算 ^[extracted]。

**S/360命名由来：** 名称中的"360"指其覆盖范围的广度——360度，覆盖所有可能的用途 ^[extracted]。

### S/360的关键创新

1. **微码（Microcode）引入** — S/360是首个使用微码实现机器指令的计算机，而非将所有指令硬连线到电路。微码（又称固件）由存储的微指令组成，在硬件和软件之间提供功能层。微码的优势在于灵活性——任何更正或新功能只需更改微码，无需更换计算机 ^[extracted]
2. **统一架构** — 结束了商业和科学计算分离的历史，一台机器满足所有需求
3. **向后兼容的先例** — 成为mainframe设计的核心原则 ^[inferred]

## 架构演进：每十年一次跨越

自1964年S/360推出以来，IBM大约每十年对平台进行一次重大扩展 ^[extracted]：

| 年份 | 架构 | 重大变化 |
|------|------|---------|
| 1964 | System/360 | 通用计算、微码、32位架构 |
| 1970 | System/370 | 虚拟存储、扩展内存 |
| 1983 | System/370 Extended Architecture (370-XA) | 扩展寻址能力 |
| 1990 | Enterprise Systems Architecture/390 (ESA/390) | 更强的I/O和内存管理 |
| 2000 | **z/Architecture** | 64位寻址、持续至今 |

每次升级都保持了与早期应用的兼容性——1964年编写的应用程序今天仍可在System z上运行 ^[extracted]。

## 现代mainframe的成长

现代mainframe在四个维度上持续改进 ^[extracted]：

1. **更多更快的处理器** — 从单处理器发展到64路处理器
2. **更大的物理内存和内存寻址能力** — 最大3 TB内存
3. **动态升级能力** — 可在不中断工作负载的情况下升级硬件和软件
4. **增强的I/O能力** — FICON通道、InfiniBand互连

Figure 1-1（见原书）展示了mainframe的均衡设计——在所有四个维度上同时实现改进，而非偏重某一方向 ^[inferred]。

## 架构的定义

**架构（Architecture）** 是用于构建产品的一套已定义的术语和规则。在计算机科学中，架构描述了系统的组织结构 ^[extracted]。

大型机架构的演进涵盖：
- 处理器数量和速度
- 内存容量和寻址能力
- 动态升级能力（硬件和软件）
- 自动化和错误检查恢复能力
- I/O设备数量和通道速度
- 虚拟化和逻辑分区技术
- 集群技术（如Parallel Sysplex）

## "Big Iron"与术语演变

"Big Iron"一词始于1960年代，用于指代大型机，以区别于较小的部门级系统 ^[inferred]。

"Mainframe"这个术语最初来源于这些机器的物理形态——早期mainframe被安装在巨大的房间级金属箱子或框架中，需要大量电力和空调制冷 ^[extracted]。1990年代起，处理器和I/O设备小型化，mainframe的物理尺寸大幅缩小，现在大约只有大型冰箱的尺寸 ^[extracted]。

## 关键洞察

- **S/360是现代计算的分水岭** — 统一了商业和科学计算，引入了微码和可扩展架构的理念
- **兼容性是mainframe的灵魂** — 54年来，应用始终可以向前兼容，这是其他任何计算平台无法匹敌的
- **架构演进而非革命** — IBM采用进化而非革命的策略，每次扩展都保留核心原则 ^[inferred]

## 相关页面

- [[mainframe-hardware-architecture]] — 硬件架构的详细内容
- [[mainframe-high-availability-sysplex]] — Sysplex集群和高可用技术
- [[zos-overview]] — z/OS操作系统概述
- [[mainframe-roles-workloads]] — mainframe角色和工作负载