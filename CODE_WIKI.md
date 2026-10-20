# Zhulong (烛龙) RISCV-CPU Code Wiki

> 本文档为 **Zhulong-RISCV-CPU** 项目的结构化代码维基，涵盖项目整体架构、模块职责、关键类与函数、依赖关系以及运行方式。

---

## 目录

1. [项目概述](#1-项目概述)
2. [目录结构](#2-目录结构)
3. [整体架构](#3-整体架构)
4. [核心模块详解](#4-核心模块详解)
5. [指令集架构 (RV64IM)](#5-指令集架构-rv64im)
6. [流水线控制与冒险处理](#6-流水线控制与冒险处理)
7. [存储系统](#7-存储系统)
8. [中断与异常处理](#8-中断与异常处理)
9. [模块依赖关系](#9-模块依赖关系)
10. [设计指标](#10-设计指标)
11. [构建与运行](#11-构建与运行)

---

## 1. 项目概述

**Zhulong (烛龙) CPU** 是一款基于 RISC-V 指令集架构的 **5 级流水线、顺序执行、单发射、单核 64 位** 处理器。

### 核心特性

| 特性 | 说明 |
|------|------|
| 流水线 | 5 级（IF → ID → EX → MEM → WB） |
| 数据位宽 | 64 位 |
| 指令集 | RV64IM（含 62 条指令） |
| 指令缓存 | 512 Bytes（2 路组相联） |
| 数据缓存 | 1024 Bytes（2 路组相联） |
| 总线协议 | AXI4-Lite |
| 乘法器 | 64 位移位相加乘法器 |
| 除法器 | 64 位移位相减除法器 |

### 流片指标（TSMC 130nm）

- 工作频率：**200 MHz**
- 总功耗：**26.6073 mW**
- 芯片面积：**821967.86 µm²**
- 流程：RTL → GDSII（逻辑综合 → 形式验证 → 静态时序分析 → 物理实现）

---

## 2. 目录结构

```
Zhulong_RISCV_CPU/
├── src/                      # RTL 源代码（核心）
│   ├── TOP_RISCV.v          # CPU 顶层模块
│   ├── RISCV_SOC.v          # SoC 顶层（含 Cache/AXI/ROM/RAM 的测试平台）
│   ├── tb.v                 # 测试平台
│   ├── defines.v            # 指令宏定义
│   ├── PC.v                 # 程序计数器
│   ├── IF.v                 # 取指级
│   ├── IF_ID.v              # IF/ID 流水寄存器
│   ├── ID.v                 # 译码级
│   ├── ID_EX.v              # ID/EX 流水寄存器
│   ├── EX.v                 # 执行级
│   ├── EX_MEM.v             # EX/MEM 流水寄存器
│   ├── MEM.v                # 访存级
│   ├── MEM_WB.v             # MEM/WB 流水寄存器
│   ├── WB.v                 # 写回级
│   ├── REGS.v               # 通用寄存器堆
│   ├── CHECK.v              # 数据冒险检测与前递
│   ├── CTRL.v               # 流水线控制（暂停/跳转）
│   ├── CSR_REGS.v           # CSR 寄存器堆
│   ├── CLINT.v              # 中断控制器
│   ├── DFF_SET.v            # 可配置流水寄存器单元
│   ├── Multiplier.v         # 乘法器
│   ├── Divider.v            # 除法器
│   ├── ICache.v             # 指令 Cache
│   ├── DCache.v             # 数据 Cache
│   ├── AXI4_Lite.v          # AXI4-Lite 总线互联
│   ├── ROM.v                # 只读存储器
│   └── RAM.v                # 随机存取存储器
├── scripts/                  # 编译脚本
│   ├── filelist             # 综合用文件列表
│   ├── FileList.f           # VCS 仿真文件列表
│   └── FileList_netlist.f   # 门级网表仿真文件列表
├── debug/                    # VCS 功能仿真
│   └── Makefile
├── debug_netlist/            # VCS 门级仿真
│   └── Makefile
├── syn/                      # 逻辑综合与物理实现
│   ├── exec/synthesis.tcl   # Design Compiler 综合脚本
│   ├── formality/           # Formality 形式验证
│   ├── icc/                 # IC Compiler 布局布线
│   ├── primetime/           # PrimeTime 静态时序分析
│   ├── rpts/                # 综合报告
│   ├── temp/                # 临时文件
│   └── work/                # 综合输出（网表/SDC/SDF/DDC）
├── modelsim_simulation_win11/  # ModelSim 仿真环境
├── figures/                  # 架构图/版图/原理图
├── resources/                # RISC-V 参考资料
├── LICENSE
└── README.md
```

---

## 3. 整体架构

### 3.1 五级流水线数据通路

```
        ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
PC ───►│  IF  │──►│  ID  │──►│  EX  │──►│ MEM │──►│  WB  │──► REGS
        └──────┘   └──────┘   └──────┘   └──────┘   └──────┘
           │          │          │          │
        ICache      REGS      ALU/MUL     DCache
                      │        /DIV
                   CHECK ◄─────┘
                   (前递)
```

### 3.2 模块层次

```
RISCV_SOC (SoC 顶层 / 测试平台)
├── TOP_RISCV (CPU 核)
│   ├── PC                程序计数器
│   ├── IF                取指
│   ├── IF_ID             流水寄存器
│   ├── ID                译码
│   ├── REGS              寄存器堆 (32×64bit)
│   ├── ID_EX             流水寄存器
│   ├── EX                执行 (含 ALU)
│   │   ├── Multiplier    乘法器
│   │   └── Divider       除法器
│   ├── CTRL              流水线控制
│   ├── EX_MEM            流水寄存器
│   ├── MEM               访存
│   ├── MEM_WB            流水寄存器
│   ├── WB                写回
│   ├── CHECK             数据前递/冒险检测
│   ├── CSR_REGS          CSR 寄存器
│   └── CLINT             中断控制器
├── ICache                指令缓存
├── DCache                数据缓存
├── AXI4_Lite             总线互联 (2 Master × 2 Slave)
├── ROM                   指令存储器
└── RAM                   数据存储器
```

### 3.3 关键信号流

- **取指**：`PC` 输出指令地址 → `IF` 通过 ICache 取指 → `IF_ID` 锁存
- **译码**：`ID` 解析指令，产生操作数、立即数、读写控制 → 读取 `REGS`
- **执行**：`EX` 进行 ALU 运算、分支判定、地址计算；乘除法由独立单元完成
- **访存**：`MEM` 通过 DCache 读写 RAM
- **写回**：`WB` 将结果写回 `REGS`
- **控制**：`CTRL` 汇总各级暂停请求，产生 `hold_flag` 与跳转信号
- **前递**：`CHECK` 将 EX/MEM 阶段结果前递到 ID 阶段解决数据冒险

---

## 4. 核心模块详解

### 4.1 TOP_RISCV — CPU 顶层

**文件**：[TOP_RISCV.v](file:///workspace/src/TOP_RISCV.v)

CPU 核的顶层模块，实例化所有流水线级与辅助单元，并完成模块间信号连接。对外暴露 ICache 和 DCache 的请求接口。

**对外端口**：

| 端口 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `clk` / `rst` | in | 1 | 时钟与复位 |
| `if_icache_req_addr` | out | 12 | 取指地址 |
| `if_icache_req_valid` | out | 1 | 取指请求有效 |
| `if_icache_req_rw` | out | 1 | 取指读写（恒为读） |
| `if_icache_data_read` | in | 32 | 取指数据 |
| `if_icache_ready` | in | 1 | ICache 就绪 |
| `if_icache_hit` | in | 1 | ICache 命中 |
| `cpu_jump` | out | 1 | 发生跳转（清空 IF） |
| `mem_dcache_req_addr` | out | 12 | 访存地址 |
| `mem_dcache_req_valid` | out | 1 | 访存请求有效 |
| `mem_dcache_req_rw` | out | 1 | 访存读写 |
| `mem_dcache_data_write` | out | 64 | 写数据 |
| `mem_dcache_data_read` | in | 64 | 读数据 |
| `mem_dcache_ready` | in | 1 | DCache 就绪 |
| `mem_dcache_hit` | in | 1 | DCache 命中 |

### 4.2 PC — 程序计数器

**文件**：[PC.v](file:///workspace/src/PC.v)

维护当前指令地址，支持正常递增、跳转和流水线暂停。

**行为**：
- 复位时 `pc = 0`
- `jump_en=1` 时加载 `jump_addr_i`
- `hold_flag_i == 2'b01`（暂停并插入气泡）或 `2'b10`（暂停保持）时保持不变
- 否则 `pc = pc + 4`

使用 `DFF_SET` 输出到 `inst_addr_o`，暂停时可注入 NOP。

### 4.3 IF — 取指级

**文件**：[IF.v](file:///workspace/src/IF.v)

将 PC 地址转换为 ICache 请求，并接收指令数据。

- `icache_req_addr = inst_addr_i[13:2]`（Cache 行地址）
- `icache_req_valid = 1`，`icache_req_rw = 0`（只读）
- 指令输出：`inst_o = icache_ready ? icache_data_read : NOP`
- 暂停标志：未命中且未就绪时请求暂停

### 4.4 ID — 译码级

**文件**：[ID.v](file:///workspace/src/ID.v)

核心译码单元，解析指令并产生所有控制信号与操作数。

**输入**：指令 `rom_inst_i`、指令地址、寄存器数据（来自 REGS 与前递 CHECK）、CSR 数据、中断信号。

**输出**：
- `rs1_addr_o` / `rs2_addr_o`：源寄存器地址
- `op1_o` / `op2_o`：操作数
- `rd_addr_o` / `reg_wen`：目标寄存器与写使能
- `base_addr_o` / `addr_offset_o`：访存/分支基址与偏移
- `csr_we_o` / `csr_raddr_o` / `csr_waddr_o`：CSR 控制
- `id_axi_araddr`：访存地址（最高位设为 1 标识 RAM 空间）
- `read_ram` / `write_ram`：访存读写标志

**译码逻辑**：以 `opcode`（`inst[6:0]`）为主分支，`funct3`、`funct7` 为子分支，覆盖 I/L/S/R-M/B/J/U/CSR 等全部指令类型。立即数按指令类型进行符号扩展。

### 4.5 EX — 执行级

**文件**：[EX.v](file:///workspace/src/EX.v)

执行算术逻辑运算、分支跳转判定、访存地址计算，并调度乘法器/除法器。

**主要功能**：

1. **ALU 运算**：加减、与/或/异或、逻辑/算术移位、比较（有符号/无符号）
2. **分支判定**：根据比较结果产生 `jump_en_o` 与 `jump_addr_o`
3. **地址计算**：`base_addr_add_addr_offset = base_addr_i + addr_offset_i`
4. **乘除法调度**：
   - 检测到乘除法指令时，向 Multiplier/Divider 发送操作数与 `ready` 信号
   - 乘除单元工作期间置位 `hold_flag`，并产生"伪跳转"使 PC 保持（重执行同一条指令）
   - 运算完成后接收结果并写回
5. **CSR 操作**：根据指令类型对 CSR 执行读改写（RW/RS/RC 及立即数版本）
6. **中断响应**：`int_assert_i` 有效时，跳转到中断入口地址，并禁止写寄存器/CSR

**关键信号**：

| 信号 | 说明 |
|------|------|
| `rd_data_o` | 运算结果（ALU / 乘法 / 除法 结果按位或合并） |
| `jump_addr_o` | 跳转目标地址（分支/跳转/中断） |
| `hold_flag_o` | 执行级暂停（乘除法运算中） |

### 4.6 MEM — 访存级

**文件**：[MEM.v](file:///workspace/src/MEM.v)

通过 DCache 执行 Load/Store 操作，支持字节/半字/字/双字的有符号与无符号访问。

**Store 操作**：根据 `funct3` 将 `op2_i` 按字节使能写入对应的 64-bit 数据通道，利用 `dcache_data_read` 实现读-改-写。

**Load 操作**：从 `dcache_data_read` 中按地址低 3 位选择对应字节/半字/字，并进行符号扩展或零扩展。

**Cache 握手**：未命中时持续请求，命中且就绪时完成操作，否则置位 `hold_flag_mem_o`。

### 4.7 WB — 写回级

**文件**：[WB.v](file:///workspace/src/WB.v)

纯组合逻辑，将 MEM_WB 寄存器输出的 `rd_addr`、`rd_data`、`rd_wen` 直接传递给寄存器堆 REGS。

### 4.8 REGS — 通用寄存器堆

**文件**：[REGS.v](file:///workspace/src/REGS.v)

32 个 64 位通用寄存器（x0~x31），x0 恒为 0。

- **读端口**：2 个异步读端口，支持写优先（当读地址等于写地址且写使能有效时，直接返回写数据）
- **写端口**：1 个同步写端口，`posedge clk` 写入，x0 不可写

### 4.9 CHECK — 数据前递与冒险检测

**文件**：[CHECK.v](file:///workspace/src/CHECK.v)

实现 EX/MEM → ID 的数据前递（forwarding），解决数据冒险。

**逻辑**：对 `rs1` 和 `rs2` 分别判断：
1. 若源寄存器地址等于 EX 级目标地址且 EX 写使能有效 → 前递 EX 结果
2. 否则若等于 MEM 级目标地址且 MEM 写使能有效 → 前递 MEM 结果
3. 若源寄存器为 x0 → 返回 0
4. 否则 → 无效（由 REGS 提供原值）

输出 `rs1_valid` / `rs2_valid` 指示 ID 阶段应使用前递数据还是寄存器堆数据。

### 4.10 CTRL — 流水线控制

**文件**：[CTRL.v](file:///workspace/src/CTRL.v)

汇总各级暂停请求，产生统一的流水线控制信号。

**暂停优先级**（`hold_flag_o` 编码）：

| 条件 | `hold_flag_o` | 含义 |
|------|---------------|------|
| `jump_en_i` / EX暂停 / CLINT暂停 | `2'b01` | 暂停 PC，插入气泡（NOP） |
| MEM 暂停 | `2'b10` | 暂停保持（不插入气泡） |
| IF 暂停 | `2'b10` | 暂停保持 |
| 无暂停 | `2'b11` | 正常流动 |

`mem_en` 信号用于控制 EX_MEM 流水寄存器在 MEM 暂停时的保持行为。

### 4.11 CSR_REGS — 控制状态寄存器

**文件**：[CSR_REGS.v](file:///workspace/src/CSR_REGS.v)

实现 Machine 模式下的 CSR 寄存器：`mtvec`、`mcause`、`mepc`、`mie`、`mstatus`、`mscratch`。

- 两个写端口：EX 级（指令写）和 CLINT（中断写）
- 两个读端口：ID 级（指令读）和 CLINT（中断读）
- 支持写优先的旁路逻辑
- `global_int_en_o = mstatus[3]`（MIE 位）

### 4.12 CLINT — 中断控制器

**文件**：[CLINT.v](file:///workspace/src/CLINT.v)

处理同步异常（ECALL/EBREAK）与异步中断（MRET 返回）。

**状态机**：
- `int_state`：中断仲裁（IDLE / SYNC_ASSERT / ASYNC_ASSERT / MRET）
- `csr_state`：CSR 写入序列（IDLE → MEPC → MSTATUS → MCAUSE → IDLE，或 MSTATUS_MRET → IDLE）

**中断流程**：
1. 检测到 ECALL/EBREAK 或全局中断使能
2. 依次写入 `mepc`（当前/跳转地址）、`mstatus`（关闭中断）、`mcause`（异常码）
3. 向 EX 发送 `int_assert_o`，跳转至 `mtvec`
4. MRET 时恢复 `mstatus` 并跳转至 `mepc`

### 4.13 DFF_SET — 可配置流水寄存器

**文件**：[DFF_SET.v](file:///workspace/src/DFF_SET.v)

参数化（`DW` 位宽）的流水寄存器单元，是所有流水寄存器的基础构件。

**行为**：
- `rst=1` 或 `hold_flag_i==2'b01`：输出 `set_data`（插入气泡）
- `hold_flag_i==2'b10`：保持原值
- 其他：输出 `data_i`

### 4.14 Multiplier — 乘法器

**文件**：[Multiplier.v](file:///workspace/src/Multiplier.v)

64 位移位相加乘法器，支持 MUL、MULH、MULHSU、MULHU、MULW。

**实现**：
- 对有符号操作数取绝对值，记录结果符号
- 通过 64 个时钟周期逐位移位相加得到 128 位乘积
- 根据指令类型选择乘积的高/低 64 位，并进行符号修正

**接口**：`mult_ready` 启动运算，`mult_finish` 指示完成，`busy_o` 指示忙。

### 4.15 Divider — 除法器

**文件**：[Divider.v](file:///workspace/src/Divider.v)

64 位移位相减除法器，支持 DIV、DIVU、REM、REMU 及 32 位 W 版本。

**实现**：
- 计数器 `counter` 从 0 计数到 67（64 次迭代）
- 处理有符号数的符号与绝对值转换
- 迭代中进行移位、比较与减法
- 除零特殊处理（商和余数均为全 1 或返回被除数）

**接口**：`div_ready` 启动，`div_finish` 完成，`busy_o` 忙信号，`div_rem_data` 输出商或余数。

### 4.16 ICache — 指令缓存

**文件**：[ICache.v](file:///workspace/src/ICache.v)

512 Bytes，2 路组相联，16 组 × 2 路 × 4 字 × 4 字节。

**Cache 行格式**（136 位）：`V(1) | D(1) | Tag(6) | Data(128)`

**状态机**：`IDLE → CompareTag → (WriteBack →) Allocate → CompareTag`

- **命中**：直接返回对应字的数据
- **未命中**：若被替换块为脏（V=1,D=1），先写回主存（ROM），再从主存分配新行
- **替换策略**：优先选无效路，两路均有效时替换第 0 路
- 跳转时返回 NOP（`32'h00000013`）以冲刷流水线

### 4.17 DCache — 数据缓存

**文件**：[DCache.v](file:///workspace/src/DCache.v)

1024 Bytes，2 路组相联，16 组 × 2 路 × 4 双字 × 8 字节。

**Cache 行格式**（264 位）：`V(1) | D(1) | Tag(6) | Data(256)`

结构与 ICache 类似，区别在于：
- 数据宽度 256 位（4 个 64-bit 双字）
- 支持读写操作
- 写命中时置脏位（D=1）

### 4.18 AXI4_Lite — 总线互联

**文件**：[AXI4_Lite.v](file:///workspace/src/AXI4_Lite.v)

2 主 2 从的 AXI4-Lite 交叉开关：
- **Master 0**（ICache）↔ **Slave 0**（ROM）：128 位数据
- **Master 1**（DCache）↔ **Slave 1**（RAM）：256 位数据

实现地址/写数据/写响应/读地址/读数据 5 个通道的信号直通，读数据通道加一级寄存器。

### 4.19 ROM — 指令存储器

**文件**：[ROM.v](file:///workspace/src/ROM.v)

1024 × 32 位 ROM，通过 AXI4-Lite Slave 接口访问。每次返回 128 位（4 条指令）。仿真时通过 `$readmemh` 加载程序。

### 4.20 RAM — 数据存储器

**文件**：[RAM.v](file:///workspace/src/RAM.v)

1024 × 64 位 RAM，通过 AXI4-Lite Slave 接口访问。每次返回 256 位（4 个双字）。支持读写，写时按 4 个双字写入。

### 4.21 RISCV_SOC — SoC 顶层

**文件**：[RISCV_SOC.v](file:///workspace/src/RISCV_SOC.v)

集成 CPU 核、ICache、DCache、AXI4-Lite、ROM、RAM 的完整 SoC 测试平台。

- 时钟：`#2.5` 翻转 → 200 MHz
- 复位：初始高电平，15ns 后拉低
- 程序加载：`$readmemh` 将测试指令文件加载到 ROM
- 自检测试：等待 x26=1 且 x27=1 时输出 `TEST_PASS`
- 支持波形转储：`DUMP_FSDB` / `DUMP_VPD`
- 支持门级仿真：`NET_SIM` 时反标 SDF

---

## 5. 指令集架构 (RV64IM)

### 5.1 指令类型与宏定义

**文件**：[defines.v](file:///workspace/src/defines.v)

| 类型 | Opcode 宏 | 说明 |
|------|-----------|------|
| I 型 | `INST_TYPE_I` (7'b0010011) | 算术立即数 |
| L 型 | `INST_TYPE_L` (7'b0000011) | 加载 |
| S 型 | `INST_TYPE_S` (7'b0100011) | 存储 |
| R/M 型 | `INST_TYPE_R_M` (7'b0110011) | 寄存器-寄存器运算与乘除 |
| 64I 型 | `INST_TYPE_64IW` (7'b0011011) | 64 位立即数运算 |
| 64R/M 型 | `INST_TYPE_R_M_64W` (7'b0111011) | 64 位寄存器运算与乘除 |
| B 型 | `INST_TYPE_B` (7'b1100011) | 分支 |
| J 型 | `INST_JAL`/`INST_JALR` | 跳转 |
| U 型 | `INST_LUI`/`INST_AUIPC` | 长立即数 |
| CSR 型 | `INST_CSR` (7'b1110011) | 控制状态寄存器 |
| 系统 | `INST_ECALL`/`INST_EBREAK`/`INST_MRET` | 异常与中断返回 |

### 5.2 指令清单（共 62 条）

**I 型（6 条）**：ADDI, SLTI, SLTIU, XORI, ORI, ANDI, SLLI, SRI(SRLI/SRAI)

**L 型（7 条）**：LB, LH, LW, LD, LBU, LHU, LWU

**S 型（4 条）**：SB, SH, SW, SD

**R 型（10 条）**：ADD/SUB, SLL, SLT, SLTU, XOR, SR(SRL/SRA), OR, AND

**M 型（8 条）**：MUL, MULH, MULHSU, MULHU, DIV, DIVU, REM, REMU

**64I 型（3 条）**：ADDIW, SLLIW, SRIW(SRLIW/SRAIW)

**64R/M 型（6 条）**：ADDW/SUBW/MULW, SLLW, SRW/SRLW/SRAW, DIVW, DIVUW, REMW, REMUW

**B 型（6 条）**：BEQ, BNE, BLT, BGE, BLTU, BGEU

**J 型（2 条）**：JAL, JALR

**U 型（2 条）**：LUI, AUIPC

**CSR 型（6 条）**：CSRRW, CSRRS, CSRRC, CSRRWI, CSRRSI, CSRRCI

**系统（3 条）**：ECALL, EBREAK, MRET

---

## 6. 流水线控制与冒险处理

### 6.1 暂停机制

`hold_flag` 为 2 位编码，由 `CTRL` 模块统一仲裁：

| 编码 | 行为 | 触发场景 |
|------|------|----------|
| `2'b01` | PC 暂停，流水线插入气泡 | 跳转、乘除法启动、中断处理 |
| `2'b10` | 全流水线保持 | Cache 未命中等待 |
| `2'b11` | 正常流动 | 无暂停 |

### 6.2 数据冒险

通过 `CHECK` 模块实现 **数据前递**（Forwarding），无需暂停即可解决 EX/MEM → ID 的写后读冒险。前递源优先级：EX 级 > MEM 级。

### 6.3 控制冒险

分支与跳转指令在 EX 级判定，通过 `jump_en_o` 通知 `CTRL`，将 IF 与 IF_ID 级冲刷为 NOP（`INST_NOP = 32'h00000013`），产生 2 周期分支惩罚。

### 6.4 结构冒险

乘除法为多周期运算单元，运算期间 EX 级置位 `hold_flag_o`，同时通过"伪跳转"使 PC 重执行该指令，待运算完成后写回结果。

---

## 7. 存储系统

### 7.1 地址映射

```
地址空间（12 位）:
┌────────────┬──────────────┬───────────┐
│  Tag[11:6] │ Index[5:2]   │ Offset[1:0]│
└────────────┴──────────────┴───────────┘
```

CPU 访存地址最高位（bit 31）为 1 时映射到 RAM 空间（在 ID 级设置 `id_axi_araddr[31]=1`），ROM 用于取指。

### 7.2 Cache 组织

| 参数 | ICache | DCache |
|------|--------|--------|
| 容量 | 512 B | 1024 B |
| 相联度 | 2 路 | 2 路 |
| 组数 | 16 | 16 |
| 块大小 | 128 bit (4 字) | 256 bit (4 双字) |
| 写策略 | 回写 (Write-Back) | 回写 (Write-Back) |
| 分配策略 | 读分配 + 写分配 | 读分配 + 写分配 |
| 替换策略 | 优先无效路，否则第 0 路 | 同左 |

### 7.3 总线

AXI4-Lite 交叉开关，ICache 与 DCache 分别独占 ROM 和 RAM 端口，无总线仲裁竞争。

---

## 8. 中断与异常处理

### 8.1 支持的异常

| 异常 | mcause | 触发条件 |
|------|--------|----------|
| ECALL | 11 (Environment call from M-mode) | `INST_ECALL` |
| EBREAK | 3 (Breakpoint) | `INST_EBREAK` |
| 其他 | 10 | 默认 |

### 8.2 中断处理流程

1. CLINT 检测到异常或全局中断使能
2. 状态机依次写入 CSR：
   - `mepc` ← 当前指令地址（或跳转目标 - 4）
   - `mstatus[3]` ← 0（关闭全局中断）
   - `mcause` ← 异常码
3. 向 EX 发送 `int_assert_o`，跳转至 `mtvec`
4. 中断服务程序执行
5. `MRET` 指令触发：恢复 `mstatus[3]`，跳转至 `mepc`

### 8.3 中断期间的流水线控制

- EX 级在 `int_assert_i` 有效时：禁止写寄存器、禁止写 CSR、跳转到中断入口
- CLINT 置位 `hold_flag_o` 直到 CSR 写入序列完成

---

## 9. 模块依赖关系

### 9.1 实例化层次图

```
RISCV_SOC
├── TOP_RISCV
│   ├── PC ──────────── DFF_SET
│   ├── IF
│   ├── IF_ID ───────── DFF_SET ×2
│   ├── ID
│   ├── REGS
│   ├── ID_EX ───────── DFF_SET ×14
│   ├── EX
│   │   ├── Multiplier (独立实例，由 TOP_RISCV 连接)
│   │   └── Divider    (独立实例，由 TOP_RISCV 连接)
│   ├── EX_MEM ──────── DFF_SET ×9
│   ├── MEM
│   ├── MEM_WB ──────── DFF_SET ×3
│   ├── WB
│   ├── CHECK
│   ├── CSR_REGS
│   └── CLINT
├── ICache
├── DCache
├── AXI4_Lite
├── ROM
└── RAM
```

### 9.2 模块间数据依赖

```
PC ──addr──► IF ──inst──► IF_ID ──inst──► ID ──op1/op2──► ID_EX ──► EX
                                                               │
REGS ──rdata──► ID ◄──forward── CHECK ◄──rd_data── EX/MEM
                                                               │
                                              EX ──rd_data──► EX_MEM ──► MEM ──► MEM_WB ──► WB ──wdata──► REGS
                                                               │
                                              MEM ◄──data──► DCache ◄──AXI──► RAM
```

### 9.3 控制信号依赖

```
IF ──hold_flag_if──► CTRL ◄──hold_flag_ex── EX
                      │  ◄──hold_flag_clint── CLINT
                      │  ◄──hold_flag_mem── MEM
                      │
                      ├──hold_flag──► PC, IF_ID, ID_EX
                      ├──jump_en────► PC
                      └──mem_en─────► EX_MEM
```

---

## 10. 设计指标

### 10.1 综合结果（Design Compiler, TSMC 130nm）

| 指标 | 数值 |
|------|------|
| 时钟周期 | 5.0 ns (200 MHz) |
| 关键路径长度 | 4.66 ns |
| 建立时间裕量 (WNS) | 0.00 ns |
| 保持时间违例 | 0 |
| 组合逻辑单元数 | 39,266 |
| 时序逻辑单元数 | 4,857 |
| 总单元数 | 44,123 |
| 组合逻辑面积 | 61,548.75 |
| 非组合逻辑面积 | 29,628.25 |
| 总单元面积 | 91,177.00 |
| 设计总面积（含线网） | 110,383.93 |

### 10.2 物理实现（IC Compiler）

- 输出 GDSII：`syn/icc/riscv_cpu.gdsii`
- 最终面积：**821,967.86 µm²**
- 功耗：**26.6073 mW**

---

## 11. 构建与运行

### 11.1 功能仿真（VCS）

**目录**：`debug/`

```bash
cd debug

# 编译
make com

# 运行仿真
make sim

# 查看波形（Verdi）
make verdi

# 查看覆盖率
make cov

# 清理
make clean
```

**编译选项**（见 [debug/Makefile](file:///workspace/debug/Makefile)）：
- `+define+DUMP_VPD`：输出 VCD+ 波形
- `+define+DUMP_FSDB`：输出 FSDB 波形
- `+NET_SIM`：门级仿真（反标 SDF）
- 覆盖率收集：`-cm line+cond+fsm+tgl`

### 11.2 门级仿真（VCS）

**目录**：`debug_netlist/`

使用 `scripts/FileList_netlist.f`，包含综合后的网表 `syn/work/TOP_RISCV.v` 与行为级存储器模型。

### 11.3 ModelSim 仿真（Windows）

**目录**：`modelsim_simulation_win11/`

```bash
# 单条指令测试
python sim/test_one_inst.py <bin_file>

# 全量指令测试
python sim/test_all.py
```

测试通过标准：RISC-V 官方测试程序向 x26 写入 1，再向 x27 写入 1，仿真器检测到该序列后输出 `TEST_PASS`。

### 11.4 逻辑综合（Design Compiler）

**目录**：`syn/exec/`

```bash
cd syn/exec
./run          # 执行 synthesis.tcl
```

**综合脚本**：[syn/exec/synthesis.tcl](file:///workspace/syn/exec/synthesis.tcl)

关键约束：
- 时钟：`create_clock -period 5.0 -waveform {0 2.5}`
- 输入延迟：1.5 ns
- 输出延迟：0.75 ns
- 最大转换时间：0.2 ns

**输出文件**（`syn/work/`）：
- `TOP_RISCV.v`：门级网表
- `TOP_RISCV.sdc`：时序约束
- `TOP_RISCV.sdf`：标准延时格式
- `TOP_RISCV.ddc`：DC 设计数据库

### 11.5 形式验证（Formality）

**目录**：`syn/formality/`

验证 RTL 与综合后门级网表的功能等价性。

### 11.6 静态时序分析（PrimeTime）

**目录**：`syn/primetime/`

对综合后网表进行精确时序分析。

### 11.7 布局布线（IC Compiler）

**目录**：`syn/icc/`

完成从网表到 GDSII 的物理实现，输出最终版图 `riscv_cpu.gdsii`。

### 11.8 文件列表说明

| 文件 | 用途 | 包含内容 |
|------|------|----------|
| `scripts/filelist` | 综合 | 仅 CPU 核 RTL |
| `scripts/FileList.f` | VCS 功能仿真 | CPU 核 + 存储器/总线 + SoC |
| `scripts/FileList_netlist.f` | VCS 门级仿真 | 门级网表 + 存储器/总线 + SoC |

---

## 附录：关键信号速查

### 流水线暂停编码

| `hold_flag` | PC | IF_ID | ID_EX | 说明 |
|-------------|-----|-------|-------|------|
| `2'b01` | 保持 | 注入 NOP | 注入 NOP | 跳转/乘除/中断 |
| `2'b10` | 保持 | 保持 | 保持 | Cache miss |
| `2'b11` | +4 | 锁存 | 锁存 | 正常 |

### 访存地址构造

```verilog
id_axi_araddr = {1'b1, (rs1_data + imm)[30:0]};
```
最高位 1 标识 RAM 空间，Cache 使用地址 `[13:2]` 作为行地址。

---

*文档生成时间：基于仓库源码分析*
