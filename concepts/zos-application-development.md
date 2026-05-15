---
title: z/OS Application Development
category: concepts
tags: [ibm, zos, development, compilation, linker]
sources: [下载/zOS Basics.pdf]
summary: z/OS应用开发流程：需求分析→设计→编码→编译→链接编辑→测试→投产。编译输出目标模块，链接编辑生成可执行load module，存储在LOADLIB中。
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

# z/OS Application Development

## 应用开发生命周期

z/OS应用开发遵循标准软件工程流程 ^[extracted]：

```
1. 需求分析
      ↓
2. 应用设计
      ↓
3. 编码（源代码编写）
      ↓
4. 编译（Compilation）
      ↓
5. 链接编辑（Link-editing）
      ↓
6. 测试（单元测试、集成测试、系统测试）
      ↓
7. 验收测试
      ↓
8. 投产（进入生产）
```

## 源代码管理

### 源代码库类型

源代码存储在**分区数据集（PDS）** 中 ^[extracted]：

| 类型 | 用途 |
|------|------|
| **SOURCE** | 源代码（COBOL、PL/I、C等） |
| **COPY** | COBOL COPY库（共享数据结构） |
| **INCLUDE** | Assembler/C的共享宏 |
| **LOADMODULE** | 编译后的可执行模块 |

### COPY库和INCLUDE库

程序可共享公共数据结构 ^[extracted]：
- **COBOL COPY** — 共享FD、WS、PK区定义
- **Assembler INCLUDE** — 共享宏定义
- 修改公共定义后，所有引用它的程序需要在下次编译时重新编译

## 编译过程

### COBOL编译

```jcl
//COBOL   EXEC PGM=IGYCRCTL,REGION=8M
//STEPLIB  DD DSN=IGY.V4R4M0.SIGYCOMP,DISP=SHR
//SYSIN    DD DSN=USER.SOURCE(COBOLPGM),DISP=SHR
//SYSLIB   DD DSN=USER.COPYLIBS,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSPUNCH DD DSN=USER.OBJ(CBL1),DISP=(NEW,CATLG,DELETE),
//             SPACE=(TRK,(1,1)),UNIT=SYSDA
```

编译输出：
- **目标模块（Object Module）** — 编译器输出，未链接不可执行
- **编译列表** — 诊断和调试信息

### C/C++编译

```jcl
//CCOMP   EXEC PGM=CCNDRVR,PARM='OPTFILE(...)'
//STEPLIB  DD DSN=CEE.SCEERUN,DISP=SHR
//SYSIN    DD DSN=USER.SOURCE(MYPROG),DISP=SHR
//OBJLIB   DD DSN=USER.OBJ,DISP=SHR
```

## 链接编辑过程

**链接编辑器（Linkage Editor）** 将一个或多个目标模块组合成**可执行模块（Load Module）** ^[extracted]：

```jcl
//LNKEDT EXEC PGM=HEWL,PARM='RMODE=ANY,AMODE=31'
//SYSLIB   DD DSN=USER.OBJ,DISP=SHR        ← 目标模块库
//         DD DSN=SYS1.SCEELKED,DISP=SHR   ← 系统运行单元库
//SYSLMOD  DD DSN=USER.LOADLIB(PGMNAME),DISP=SHR
//SYSPRINT DD SYSOUT=*
```

Load Module存储在**LOADLIB（Load Library）** 中，通常命名为 `*.LOADLIB` 或 `*.LOAD` ^[extracted]。

## 执行过程

Load module通过JCL执行 ^[extracted]：

```jcl
//MYJOB   JOB
//STEP1   EXEC PGM=MYPROGRAM,PARM='INPUT FILE'
//DD1     DD DSN=USER.DATA,DISP=SHR
//STEPLIB  DD DSN=USER.LOADLIB,DISP=SHR
```

## 测试环境

z/OS应用测试分层 ^[extracted]：

| 级别 | 内容 | 典型工具 |
|------|------|----------|
| **单元测试** | 单个程序功能 | IDE（VSAM、RD/Z） |
| **集成测试** | 模块间交互 | CICS、DB2测试环境 |
| **系统测试** | 完整业务流程 | 模拟生产环境 |
| **验收测试** | 用户确认 | 真实业务场景 |

## CICS应用开发

CICS应用开发有其特殊性 ^[extracted]：

1. COBOL程序中使用 `EXEC CICS` 接口调用CICS服务
2. 编译前需使用**CICS预编译器**处理 `EXEC CICS` 语句
3. CICS系统定义（CSD）管理资源
4. 需将程序安装到CICS region才能执行

参见：[[cics-application-development]]、[[cics-programming-cobol]]

## 数据库应用开发

### DB2应用

DB2应用使用嵌入式SQL ^[extracted]：

```cobol
EXEC SQL
  SELECT NAME INTO :WS-NAME
  FROM EMP
  WHERE EMPNO = :WS-EMPNO
END-EXEC.
```

开发流程需要：
- **DB2预编译器**处理SQL语句
- **DB2运行时库**（DBRM）绑定到计划（Plan）
- 计划授权给程序

参见：[[ibm-db2-12-zos-technical-overview]]

## 现代开发工具

- **IBM Developer for z/OS (IDz)** — Eclipse IDE，支持z/OS开发
- **Zowe** — z/OS CLI和API工具链，支持DevOps
- **IBM Z Open Editor** — VS Code扩展，编辑z/OS文件

参见：[[cics-devops]]

## 相关页面

- [[zos-programming-languages]] — 各编程语言详解
- [[zos-jcl-basics]] — 编译和链接相关JCL
- [[cics-application-development]] — CICS应用开发
- [[cics-devops]] — IBM Z上的DevOps工具链