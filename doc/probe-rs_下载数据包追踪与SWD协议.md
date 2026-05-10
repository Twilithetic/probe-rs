# probe-rs download 命令 SWD 总线数据包分析报告

> **文档日期**: 2026-05-06  
> **作者**: 知心姐姐  
> **参考资料**:
> - ARM Debug Interface Architecture Specification ADIv5.0 to ADIv5.2 (ARM IHI 0031C, ID080813)
> - probe-rs 源码 (v0.24+), 路径: `tools/probe-rs-src/`
> - CYT2BL3 目标定义: `tools/probe-rs-src/probe-rs/targets/CYT2BL_Series.yaml`
> - CMSIS-DAP 固件协议文档

---

## 目录

1. [问题陈述](#1-问题陈述)
2. [SWD 协议数据包定义](#2-swd-协议数据包定义)
   - [2.1 物理层](#21-物理层)
   - [2.2 数据包结构 (逐位分析)](#22-数据包结构-逐位分析)
   - [2.3 五种操作类型](#23-五种操作类型)
   - [2.4 DP 寄存器地址映射](#24-dp-寄存器地址映射dpacc)
   - [2.5 AP 寄存器地址映射](#25-ap-寄存器地址映射apacc)
   - [2.6 特殊序列](#26-特殊序列)
3. [probe-rs download 完整 SWD 数据包追踪](#3-probe-rs-download-完整-swd-数据包追踪)
   - [Phase A: 连接与协议切换](#phase-a-连接与协议切换)
   - [Phase B: 连接建立 & DPIDR 读取](#phase-b-连接建立--dpidr-读取)
   - [Phase C: 上电与错误清除](#phase-c-上电与错误清除)
   - [Phase D: AP 发现](#phase-d-ap-发现)
   - [Phase E: MEM-AP 初始化](#phase-e-mem-ap-初始化)
   - [Phase F: Flash 算法加载](#phase-f-flash-算法加载)
   - [Phase G: Flash 擦除](#phase-g-flash-擦除)
   - [Phase H: Flash 编程](#phase-h-flash-编程)
   - [Phase I: Flash 验证](#phase-i-flash-验证)
   - [Phase J: 复位与运行](#phase-j-复位与运行)
4. [CYT2BL3 特化细节](#4-cyt2bl3-特化细节)
5. [总结：SWD 数据包统计](#5-总结swd-数据包统计)

---

## 1. 问题陈述

### 问题
执行命令：
```bash
probe-rs download --chip CYT2BL3BAS --protocol swd --binary-format elf build/firmware.elf
```
该命令通过 **WCH-Link (CMSIS-DAP)** 调试器，使用 **SWD (Serial Wire Debug)** 协议，将 ELF 固件下载到 **CYT2BL3BAS** (Cypress/Infineon TRAVEO T2G Cortex-M4) 芯片的 Flash 中。

### 分析目标
追踪在此过程中，probe-rs 在 SWDIO/SWCLK 两根线上发送和接收的**每一个字节/数据包**，包括：
- 每个 SWD 数据包的 bit-level 组成
- 各数据包的作用
- 数据包序列的完整流程

---

## 2. SWD 协议数据包定义

> **依据文档**: ARM IHI 0031C Chapter 4 "The Serial Wire Debug Port (SW-DP)", 第 4.2-4.4 节

### 2.1 物理层

| 属性 | 说明 |
|------|------|
| 信号线 | **SWDIO** (双向数据) + **SWCLK** (时钟，由主机提供) |
| 数据采样 | SWDIO 在 SWCLK 上升沿采样 |
| 空闲状态 | SWDIO 拉低 |
| 上拉 | 目标端内置 100kΩ 上拉电阻 |
| 最大频率 | 取决于目标实现，通常 1-50 MHz |
| 位序 | **所有数据 LSB first** |

### 2.2 数据包结构 (逐位分析)

一个完整的 SWD 操作由 **请求头 (8 bits)**、**应答 (3 bits)**、**数据阶段 (33 bits，可选)** 三部分组成。

#### 2.2.1 请求头 (8 bits, Host→Target)

```
位序 (LSB first)  字段名       值        说明
─────────────────────────────────────────────
0                  Start        1         起始位，始终为 1
1                  APnDP        0/1       0=DP寄存器, 1=AP寄存器
2                  RnW          0/1       0=写, 1=读
3                  A[2]         0/1       寄存器地址位2
4                  A[3]         0/1       寄存器地址位3
5                  Parity       0/1       奇校验 (覆盖位1-4)
6                  Stop         0         停止位，始终为 0
7                  Park         1         Park位，始终为 1
```

**奇校验规则** (ADIv5 §4.2.5):
- 计算 APnDP ⊕ RnW ⊕ A2 ⊕ A3
- 如果这 4 位中 1 的个数为奇数 → Parity=1
- 如果这 4 位中 1 的个数为偶数 → Parity=0
- 使得这 5 位(含Parity)中 1 的个数为奇数

**举例** - 读取 DP 寄存器 DPIDR (A[3:2]=00):
```
APnDP=0, RnW=1, A2=0, A3=0 → XOR=1 (奇数) → Parity=1
请求头 = 1_0_1_0_0_1_0_1 = 1010 0101 = 0xA5 (LSB first 读出)
```

**举例** - 写 SELECT 寄存器 (A[3:2]=10):
```
APnDP=0, RnW=0, A2=1, A3=0 → XOR=1 (奇数) → Parity=1
请求头 = 1_0_0_1_0_1_0_1 = 1001 0101 = 0xA9
```

#### 2.2.2 Turnaround 周期 (1 bit)

请求头发送完后，Host 释放 SWDIO (高阻态)，目标接管驱动 SWDIO。默认 1 个 SWCLK 周期。

#### 2.2.3 ACK 应答 (3 bits, Target→Host, LSB first)

> 依据: ADIv5 §4.3.4-§4.3.6, Table 4-1 through Table 4-5

| ACK[2:0] | 值 | 含义 | 触发条件 | 数据阶段? |
|----------|----|------|---------|:--:|
| `001` | 1 | **OK** | DP 就绪，无错误 | ✅ 有 (读=33bit RDATA, 写=33bit WDATA) |
| `010` | 2 | **WAIT** | 前一个 DP/AP 操作未完成 | ❌ 无 (除非 overrun detection 使能) |
| `100` | 4 | **FAULT** | CTRL/STAT 中任意 sticky flag=1 | ❌ 无 (除非 overrun detection 使能) |
| `111` | — | **No Response** | 目标未驱动线路(协议错误/未连接) | — |

> **重要发现** (ADIv5 §4.3.4-§4.3.5): 
> - **DPIDR 和 CTRL/STAT 寄存器读取**必须**始终返回 OK**，绝不允许 WAIT 或 FAULT
> - **ABORT 寄存器写入**也必须始终返回 OK
> - 如果 DPIDR 读收到非 OK 响应，说明物理连接有问题或目标不在 SWD 模式

**NACK (111) 的本质** (ADIv5 §4.3.6 Protocol error response):
收到全 `111` **不是目标主动发送的 NACK 响应**。ADIv5 规范中没有定义 "NACK" 这个 ACK 值。全 1 是因为：
- 目标检测到**协议错误**（Parity 不匹配、Stop ≠ 0、Park ≠ 1）→ 进入**协议错误状态**
- 在协议错误状态下，目标**不驱动 SWDIO 线路**
- CMSIS-DAP 探针读取到的是 100kΩ 上拉电阻拉高的 HIGH 电平 → 解析为 `111`
- probe-rs 将此映射为 `Ack::NoAck`（源码 `transfer/mod.rs:207-214`）

**协议错误状态恢复** (ADIv5 §4.3.6):
1. 进入协议错误状态后，目标**可能**通过检测到有效的 DPIDR 读来退出（IMPLEMENTATION DEFINED）
2. 如果目标在协议错误状态下又检测到另一个协议错误 → 进入**锁定状态 (lockout)**
3. 锁定状态**只能**通过 Line Reset 退出
4. **SWD v2 实现**：Line Reset 后的第一个数据包如果是协议错误 → 直接进入锁定状态

```
协议错误状态转移 (ADIv5 §4.3.6):
  ┌──────────┐  有效 DPIDR 读?  ┌──────────┐
  │  NORMAL  │ ────────────────→│  NORMAL  │ (IMP DEF)
  └────┬─────┘                  └──────────┘
       │ 协议错误
       ▼
  ┌──────────┐  再次协议错误  ┌──────────┐
  │  PROTOCOL │ ────────────→ │  LOCKOUT │ (永久, 需 Line Reset)
  │  ERROR    │               └──────────┘
  └────┬─────┘
       │ Line Reset
       ▼
  ┌──────────┐
  │  RESET   │
  └──────────┘
```

#### 2.2.4 写操作的数据阶段 (33 bits, Host→Target)

```
位序         内容
──────────────────────
0..31        WDATA[31:0]    32位写入数据 (LSB first)
32           Parity         偶校验 (覆盖32个数据位)
```

后面**没有** turnaround — Host 继续驱动 SWDIO，可以直接开始下一个请求头。

#### 2.2.5 读操作的数据阶段 (33+1 bits, Target→Host)

```
位序         内容
──────────────────────
0..31        RDATA[31:0]    32位读取数据 (LSB first)
32           Parity         偶校验 (覆盖32个数据位)
33           Turnaround     Host 重新接管 SWDIO
```

**偶校验规则** (ADIv5 §4.2.5):
- 数据位中 1 的个数为奇数 → Parity=1
- 数据位中 1 的个数为偶数 → Parity=0
- 使得 33 位(含Parity)中 1 的个数为偶数

#### 2.2.6 完整数据包图示

**写操作 (OK 响应)**:
```
Host:  |Start|AP|RD|A2|A3|PAR|0|1|    = 8-bit 请求头
       |Trn |                         = Turnaround
Target:|ACK2|ACK1|ACK0|               = 3-bit ACK
       |Trn |                         = Turnaround
Host:  |D0..D31|PAR|                  = 33-bit 数据 (直接接下一个请求头)
```

**读操作 (OK 响应)**:
```
Host:  |Start|AP|RD|A2|A3|PAR|0|1|    = 8-bit 请求头
       |Trn |                         = Turnaround
Target:|ACK2|ACK1|ACK0|               = 3-bit ACK (Target 继续驱动)
Target:|D0..D31|PAR|                  = 33-bit 数据
       |Trn |                         = Turnaround (Host 接管)
```

**WAIT/FAULT 响应**:
```
Host:  |Start|AP|RD|A2|A3|PAR|0|1|
       |Trn |
Target:|ACK2|ACK1|ACK0|               = WAIT(010) 或 FAULT(100)
       |Trn |                         = 无数据阶段
```

**注意** (ADIv5 §4.3.3): AP 读是 **posted (流水线式)** — 当前 AP 读返回的是**上一次** AP 读的结果。第一次 AP 读返回的是 UNKNOWN。最后一次 AP 读的结果需要通过 DP 读取 RDBUFF 寄存器获得。

### 2.3 五种操作类型

结合 probe-rs 源码 (`polyfill.rs:817-892`)，实际使用五种 SWD 操作：

| 操作 | 请求头示例 | 说明 |
|------|----------|------|
| DP 读 | `1_0_1_A2_A3_P_0_1` | 读 Debug Port 寄存器 |
| DP 写 | `1_0_0_A2_A3_P_0_1` | 写 Debug Port 寄存器 |
| AP 读 | `1_1_1_A2_A3_P_0_1` | 读 Access Port 寄存器 (posted) |
| AP 写 | `1_1_0_A2_A3_P_0_1` | 写 Access Port 寄存器 |
| RDBUFF 读 | DP 读 A[3:2]=11 | 获取上一次 AP 读的缓冲结果 |

### 2.4 DP 寄存器地址映射 (DPACC)

> 依据: ADIv5 §2.3, 以及 probe-rs `dp/mod.rs`

| A[3:2] | DPBANKSEL | 读寄存器 | 写寄存器 | 作用 |
|--------|-----------|---------|---------|------|
| `00` | 忽略 | **DPIDR** | **ABORT** | IDCODE 读取 / 错误清除 |
| `01` | `0x0` | **CTRL/STAT** | **CTRL/STAT** | 电源控制、状态 |
| `01` | `0x1` | **DLCR** | **DLCR** | 数据链路控制 |
| `10` | `0x0` | **SELECT** | **SELECT** | AP 选择、Bank 选择 |
| `11` | `0x0` | **RDBUFF** | (保留/ TARGETSEL v2) | AP 读缓冲 |

**CTRL/STAT 关键位** (ADIv5 §2.3):
```
Bits  [31] CSYSPWRUPACK   系统电源应答 (读)
Bits  [30] CSYSPWRUPREQ   系统电源请求 (写)
Bits  [29] CDBGPWRUPACK   调试电源应答 (读)
Bits  [28] CDBGPWRUPREQ   调试电源请求 (写)
Bits  [11:8] MASKLANE     字节通道掩码
Bits  [7]  WDATAERR       写数据错误 (sticky)
Bits  [5]  STICKYERR      AP sticky 错误
Bits  [4]  STICKYCMP      pushed-compare 匹配
Bits  [1]  STICKYORUN     sticky overrun
```

**SELECT 关键位**:
```
Bits  [31:24] APSEL       AP 选择 (0-255)
Bits  [7:4]   APBANKSEL   AP 寄存器 Bank 选择
Bits  [3:0]   DPBANKSEL   DP 寄存器 Bank 选择
```

### 2.5 AP 寄存器地址映射 (APACC)

> 依据: ADIv5 §7.5 表 7-6

AP 寄存器通过 APACC 访问，A[3:2] 选址，Bank 通过 `SELECT.APBANKSEL` 选择:

**Bank 0x0 — 主要操作寄存器**:
| A[3:2] | 偏移 | 寄存器 | 作用 |
|--------|------|--------|------|
| `00` | 0x00 | **CSW** | 控制/状态字 (访问大小、地址自增等) |
| `01` | 0x04 | **TAR** | 传输地址寄存器 |
| `10` | 0x08 | **TAR2** | TAR 高32位 (仅 64 位地址扩展) |
| `11` | 0x0C | **DRW** | 数据读写寄存器 |

**Bank 0xF — 识别寄存器**:
| A[3:2] | 偏移 | 寄存器 | 作用 |
|--------|------|--------|------|
| `00` | 0xF0 | **BASE2** | 基地址高32位 |
| `01` | 0xF4 | **CFG** | 配置 (LD/LA/BE) |
| `10` | 0xF8 | **BASE** | 调试基地址 |
| `11` | 0xFC | **IDR** | AP 识别寄存器 |

**CSW 关键位**:
```
Bit [31]    DbgSwEnable  调试软件访问使能
Bits [5:4]  AddrInc      地址自增: 00=关, 01=单次(+4), 10=打包
Bits [2:0]  Size         访问大小: 010=32bit, 001=16bit, 000=8bit
```

**IDR 关键位**:
```
Bits [31:28] REVISION    修订号
Bits [16:13] CLASS        AP 类型: 0x8=MEM-AP, 0x1=COM-AP
Bits [7:4]   VARIANT     变体
Bits [3:0]   TYPE         总线类型: 0x1=AHB3
```

### 2.6 特殊序列

> 依据: ADIv5 §5.2 (SWD/JTAG 选择), §4.4.3 (线复位), §5.3 (Dormant)

#### 2.6.1 SWD 线复位 (Line Reset)

> 依据: ADIv5 §4.4.3 "Connection and line reset sequence", Figure 4-8

**规范要求**:
```
SWDIO 保持 HIGH ≥50 个 SWCLK 周期
然后 SWDIO 保持 LOW ≥2 个 SWCLK 周期 (Idle cycles)
```

**规范原文** (ADIv5 §4.4.3):
> "A line reset is achieved by holding the data signal HIGH for at least 50 clock cycles, followed by at least two idle cycles."
> 
> "When waiting for a packet header, if the target detects a sequence of 50 clock cycles with the data signal held HIGH, followed by at least two idle cycles, it **must** enter the reset state."

**Line Reset 后唯一合法操作** (ADIv5 §4.4.3):
> "The only valid transactions in reset state are:
> - A read of the DPIDR register. This takes the connection out of reset state.
> - One of the switching sequences defined by SWD and JTAG select mechanism, if implemented.
> - A write to the TARGETSEL register, if SWD protocol version 2 is implemented."
> 
> "The behavior of the target is **UNPREDICTABLE** if any other transaction is made in reset state."

**标准 Line Reset 时序图** (ADIv5 Figure 4-8):
```
line reset                       DP DPIDR register read
SWCLK  ──┐   ┌──┐   ┌──   ──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──
         │   │  │   │   ...   │   │  │   │  │   │  │   │  │   │  │   │  │   │  │   │
         └───┘  └───┘         └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘

SWDIO   ───────────────────────┐   ┌───┐       ┌───┐   ┌───────┐
                                │   │   │  1 0 1 0 0 1 0 1       │
                                └───┘   └───┘   └───┘   └───────┘
        │←─ ≥50 cycles HIGH ──→│←2→│←── 8-bit request ──→│
                                  Idle  Start RnW  Parity  Park
                                         APnDP A[2:3] Stop
```

probe-rs 实现 (`sequences.rs:1169-1175`):
```rust
fn swd_line_reset(interface: &mut dyn DapProbe, swdio_low_cycles: u8) -> Result<(), ArmError> {
    assert!(swdio_low_cycles + 51 <= 64);
    interface.swj_sequence(51 + swdio_low_cycles, 0x0007_FFFF_FFFF_FFFF)?;
    Ok(())
}
// 其中 swdio_low_cycles:
//   debug_port_setup 中 = 0 (仅 51 HIGH, 无 LOW 尾 — 后面有 JTAG→SWD 序列)
//   debug_port_connect 中 = 3 (51 HIGH + 3 LOW — 符合 ≥50+≥2 规范)
```

#### 2.6.2 JTAG→SWD 切换序列

> 依据: ADIv5 §5.2.1 "Switching from JTAG to SWD operation", Figure 5-3

**规范原文** (ADIv5 §5.2.1):
> "To switch SWJ-DP from JTAG to SWD operation:
> 1. Send at least 50 SWCLKTCK cycles with SWDIOTMS HIGH. 
> 2. Send the 16-bit JTAG-to-SWD select sequence on SWDIOTMS.
> 3. Send at least 50 SWCLKTCK cycles with SWDIOTMS HIGH."

**JTAG-to-SWD 序列** (ADIv5 §5.2.1):
```
16-bit: 0b0111100111100111, MSB first  →  0x79E7 (MSB first)
        0xE79E (LSB first)              →  probe-rs 使用此格式
```

**规范时序图** (ADIv5 Figure 5-3):
```
SWCLKTCK ──┐   ┌──┐   ┌──   ──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──   ──┐   ┌──
           │   │  │   │   ...   │   │  │   │  │   │  │   │  │   │  │   │  │        │   │
           └───┘  └───┘         └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └──      ┘   └──

SWDIOTMS  ───────────────────────┐ ┌─┐ ┌───┐ ┌───┐ ┌─┐ ┌───┐ ┌───┐ ┌─┐ ┌─┐ ┌─────────────
                                  │ │ │ │   │ │   │ │ │ │   │ │   │ │ │ │ │
                                  └─┘ └─┘   └─┘   └─┘ └─┘   └─┘   └─┘ └─┘ └─┘
          │←─ ≥50 cycles HIGH ──→│←── 16-bit JTAG-to-SWD sequence ──→│← ≥50 HIGH →│
           (确保当前接口在复位状态)  0 1 1 1 1 0 0 1 1 1 1 0 0 1 1 1   (确保 SWD 在线
                                   = 0xE79E LSB first                  复位状态)
```

probe-rs 实现 (`sequences.rs:588-593`):
```rust
// Execute SWJ-DP Switch Sequence JTAG to SWD (0xE79E).
interface.swj_sequence(16, 0xE79E)?;
// > 50 cycles SWDIO/TMS High, at least 2 idle cycles.
// -> done in debug_port_connect
```

**关键细节** (ADIv5 §5.2.1):
> "On selecting SWD operation, the SWD interface is in a reset state. See Connection and line reset sequence."
> 
> 这意味着 JTAG→SWD 后，SWD 接口进入**复位状态**，只接受 DPIDR 读。

probe-rs 因此在其后调用 `debug_port_connect()` → 做 Line Reset + DPIDR 读，完全符合规范。

#### 2.6.3 SWD→JTAG 切换序列 (备选)

> 依据: ADIv5 §5.2.2

```
SWD-to-JTAG: 0b0011110011100111, MSB first → 0x3CE7 (MSB first) → 0xE73C (LSB first)

步骤: ≥50 HIGH → 16-bit 序列 → ≥5 HIGH (确保 JTAG TAP 在 TLR)
```

#### 2.6.4 Dormant 操作

> 依据: ADIv5 §5.3

Dormant 状态是 SWD v2 引入的"第三状态"，允许 SWD、JTAG 和其他协议设备共享同一物理连线。

**从 SWD 到 Dormant**: ≥50 HIGH + 16-bit `0xE3BC` (LSB first)
**从 Dormant 退出**: 8 HIGH + 128-bit Selection Alert + 4 LOW + Activation Code + ≥50 HIGH

probe-rs 在 `debug_port_setup` 中 `retry>=1` 时会尝试 Dormant 路径。

CMSIS-DAP 将上述 SWD 操作封装在 USB 命令中：

```
USB 命令: [0x05, DAP#, Count, TransferReq[Count], WriteData[]]
TransferReq 字节: [APnDP|RnW|A2|A3|...]

USB 响应: [0x05, Count, LastResp, ReadData[]]
```

CMSIS-DAP 固件将每个 `TransferReq` 翻译为 wires 上的 SWD 数据包。

---

## 3. probe-rs download 完整 SWD 数据包追踪

以下按 probe-rs 源码中的实际执行顺序，逐步追踪每条 SWD 总线操作。

### Phase A: 连接与协议切换

> 源码: `sequences.rs:512-621` (`debug_port_setup`)

#### 步骤 A1: SWD 线复位

```
SWDIO: 1 1 1 ...... 1 1 1  (51 cycles HIGH)
       0 0 0               (3 cycles LOW)
```

**SWCLK 周期数**: 54  
**作用**: 将 SW-DP 强制置于复位状态，清除任何未完成的传输。

#### 步骤 A2: JTAG→SWD 切换序列

```
SWDIO (LSB first): 0 1 1 1 1 0 0 1 1 1 1 0 0 1 1 1
                  = 0xE79E (16 bits)
```

**SWCLK 周期数**: 16  
**作用**: 通知 SWJ-DP 从 JTAG 模式切换到 SWD 模式。  
**注**: 如果目标是纯 SW-DP (非 SWJ-DP)，此序列被忽略。CYT2BL3 使用 SWJ-DP。

`info` 日志输出对应:
```
INFO probe_rs::probe::cmsisdap: Using protocol SWD
```

---

### Phase B: 连接建立 & DPIDR 读取

> 源码: `sequences.rs:957-1083` (`debug_port_connect`)

#### 步骤 B1: SWD 线复位 + 3 个空闲周期

```
SWDIO: 1×51 + 0×3
```

**SWCLK 周期数**: 54

#### 步骤 B2: 读取 DPIDR (第一笔 SWD 操作!)

这是整个会话的第一个 SWD 数据包：

```
【请求头】    Host → Target  (8 bits)
  1    : Start=1
  0    : APnDP=0 (DP 访问)
  1    : RnW=1   (读)
  0    : A2=0
  0    : A3=0
  1    : Parity (0^1^0^0=1, 奇数 → Parity=1)
  0    : Stop=0
  1    : Park=1
  请求头 = 1010_0101 = 0xA5

【Turnaround】 Host 释放 SWDIO → Target 接管

【ACK】       Target → Host (3 bits, LSB first)
  1, 0, 0  → ACK=001 = OK

【数据阶段】  Target → Host (33 bits)
  D[0..31] = DPIDR 值 (例如 0x2BA01477)
  Parity   = 偶校验位

【Turnaround】 Target 释放 → Host 接管
```

**SWCLK 周期数**: 8+1+3+1+32+1+1 = **47**  
**作用**: 读取 DPIDR 确认 SWD 通信正常。此操作也将 SW-DP 从复位状态激活。

**DPIDR 解码** (ADIv5 §2.3):
```
0x2BA01477:
  REVISION = 0x2     (bit 31:28)
  PARTNO   = 0xBA    (bit 27:20)
  MIN      = 0       (bit 16, 非MINDP)
  VERSION  = 0x1     (bit 15:12, DPv1)
  DESIGNER = 0x23B   (bit 11:1, Cypress JEP106)
```

`info` 日志输出对应:
```
INFO probe_rs::architecture::arm::communication_interface: Debug Port version: DPv2 MinDP: NotImplemented
```

> **实际输出**显示 DPv2 — CYT2BL3 可能是 DPv1 向上兼容到 DPv2。

#### 步骤 B3: 写 ABORT 清除错误

```
【请求头】    Host → Target
  1, 0, 0, 0, 0, 0, 0, 1  = 0x81
  (APnDP=0, RnW=0, A[3:2]=00=ABORT寄存器, Parity=0^0^0^0=0)

【ACK】 OK=001

【WDATA】 0x0000_001F (或 0x1E)
  - ORUNERRCLR  = 1  (bit 4)
  - WDERRCLR    = 1  (bit 3)
  - STKERRCLR   = 1  (bit 2)
  - STKCMPCLR   = 1  (bit 1)
  Parity=偶校验

【无 Turnaround】 Host 继续驱动
```

**SWCLK 周期数**: 8+1+3+1+33 = **46**  
**作用**: 清除所有 sticky 错误标志。

#### 步骤 B4: 写 SELECT = 0x00000000

```
【请求头】    A[3:2]=10 (SELECT), RnW=0
  1, 0, 0, 0, 1, 0, 0, 1  = 0x89

【ACK】 OK

【WDATA】 0x0000_0000
  APSEL=0, APBANKSEL=0, DPBANKSEL=0
```

**作用**: 重置 AP 选择，回到默认 Bank。

#### 步骤 B5: 读 CTRL/STAT

```
【请求头】    A[3:2]=01 (CTRL/STAT), RnW=1
  1, 0, 1, 1, 0, 0, 0, 1  = 0x8D

【ACK】 OK
【RDATA】 CTRL/STAT 值
```

**作用**: 检查 CSYSPWRUPACK 和 CDBGPWRUPACK 是否已置位。

---

### Phase C: 上电与错误清除

> 源码: `sequences.rs:628-732` (`debug_port_start`)

#### 步骤 C1: 如果未上电 → 写 CTRL/STAT 请求上电

```
【请求头】    A[3:2]=01, RnW=0
  1, 0, 0, 1, 0, 0, 0, 1  = 0x8B (实际值取决于 Bank)

【WDATA】:
  CDBGPWRUPREQ = 1  (bit 28)
  CSYSPWRUPREQ = 1  (bit 30)
  MASKLANE     = 0b1111 (bits 11:8)
  = 0x5000_0F00
```

`info` 日志输出对应:
```
INFO probe_rs::architecture::arm::sequences: Debug port Default is powered down, powering up
```

#### 步骤 C2: 轮询 CTRL/STAT 直到上电确认

```
循环:
  【DP 读 CTRL/STAT】
  检查 CSYSPWRUPACK==1 && CDBGPWRUPACK==1 ?
超时 1 秒
```

**平均轮询次数**: 1-3 次 (上电通常很快)

---

### Phase D: AP 发现

> 源码: `communication_interface.rs`, `ap/v1.rs`

```
对于 AP 0 到 255:
  1. 【DP 写 SELECT】 APSEL=ap_num, APBANKSEL=0xF (Bank 0xF = IDR)
  2. 【AP 读 A[3:2]=11 → 偏移 0xFC = IDR】
  3. 检查 IDR.CLASS == 0x8 (MEM-AP)?
     是 → 记录此 AP
```

**每个 AP 需要**: 1 次 DP 写 + 2 次 AP 读 (posted read 需要读两次取结果)

对于 CYT2BL3，实际只有 **AP 0, 1, 2** 三个 MEM-AP，探测到 ~255 但大多数返回 0 (表示不存在)。

**CYT2BL3 probe-rs info 输出**:
```
├── V1(0) MemoryAP           ← APSEL=0
│   └── 0 MemoryAP (AmbaAhb3)
│       └── 0xf1000000 ROM Table (Class 1), Designer: Cypress
├── V1(1) MemoryAP           ← APSEL=1
│   └── 1 MemoryAP (AmbaAhb3)
│       ├── 0xf0000000 ROM Table (Class 1), Designer: Cypress
│       ├── 0xe00ff000 ROM Table (Class 1), Designer: ARM Ltd
│       └── ...
└── V1(2) MemoryAP           ← APSEL=2 (Cortex-M4)
    └── 2 MemoryAP (AmbaAhb3)
        ├── 0xe00ff000 ROM Table
        ├── Cortex-M3 TPIU
        ├── Cortex-M4 ETM
        └── ...
```

**CYT2BL3 核心分配**:
- **AP 1 → Cortex-M0+** (armv6m)
- **AP 2 → Cortex-M4** (armv7em) ← 固件烧录使用此 AP

---

### Phase E: MEM-AP 初始化

> 源码: `ap/memory_ap/mod.rs`

对选定的 MEM-AP (AP2, Cortex-M4):

#### 步骤 E1: 选择 AP 和 Bank 0
```
【DP 写 SELECT】 APSEL=2, APBANKSEL=0x0
```

#### 步骤 E2: 写 CSW (初始化传输参数)
```
【AP 写 CSW】 A[3:2]=00
  WDATA:
    DbgSwEnable = 1  (bit 31)
    AddrInc     = 01 (bits 5:4, 单次自增)
    Size        = 010 (bits 2:0, 32位)
    Prot        = 0x23 (bits 30:24, 数据访问)
```

**作用**: 配置 MEM-AP 为 32 位字访问模式，地址自动递增。

---

### Phase F: Flash 算法加载

> 源码: `flashing/flasher.rs`, `flashing/flash_algorithm.rs`

probe-rs **不直接在 SWD 上 bit-bang Flash**。而是：
1. 将 Flash 算法代码（position-independent binary）通过 SWD 写入 RAM
2. 通过设置 CPU 寄存器并运行 CPU 来调用算法函数

#### 步骤 F1: 复位并暂停 Cortex-M4 核心
```
【AP 写 TAR】   TAR = 0xE000EDF0 (DHCSR 地址)
【AP 写 DRW】   DRW = 0xA05F0003 (C_DEBUGEN=1, C_HALT=1)
【AP 读 DHCSR】 轮询直到 S_HALT=1
```

**每次内存访问**: TAR 写 (1 AP 写) + DRW 写/读 (1 AP 操作) = 2 笔 SWD 数据包

#### 步骤 F2: 加载算法到 RAM (0x08001008)
```
对于算法的每个 32-bit word:
  【AP 写 TAR】  TAR = 当前 RAM 地址
  【AP 写 DRW】  DRW = 算法指令 word
  (如果 CSW.AddrInc=Single，TAR 自动 +4)
```

**CYT2BL3 算法大小**: 约 2KB (从 YAML `instructions` 字段可知是 base64 编码，约 ~2600 字符解码后约 2KB)

**SWD 数据包数**: 约 500 word → **约 1000 笔 AP 写**

#### 步骤 F3: 调用算法 Init 函数
```
设置 CPU 寄存器:
  【AP 写 TAR = 0xE000EDF0】 【AP 写 DRW = 0xA05F0001】 (取消 halt)
  设置 PC = pc_init (0x1, 即算法入口 + 0x1 表示 Thumb)
  设置 SP = load_address + stack_size
  设置 R0 = flash 起始地址
  设置 R1 = 时钟频率
  设置 R2 = 操作码 (Erase=1, Program=2, Verify=3)
  运行 CPU
  等待 CPU 自动 halt (算法执行完毕)
```

---

### Phase G: Flash 擦除

> CYT2BL3 Flash: IROM1 @ 0x10000000, 4MB, 扇区大小 0x8000 + 0x2000

#### 步骤 G1: 加载擦除数据到 RAM buffer
```
【AP 写 TAR】  buffer_addr
【AP 写 DRW】  数据 words
... (约 4-8 words)
```

#### 步骤 G2: 调用 EraseSector
```
设置 R0 = 扇区地址 (例如 0x10000000)
设置 PC = pc_erase_sector (0x4F1)  ← CYT2BL3 算法中的偏移
运行 CPU，等待 halt
超时: 320ms
```

**每扇区 SWD 数据包**: 约 20-40 笔 (寄存器设置 + 状态轮询)

---

### Phase H: Flash 编程

> CYT2BL3 页大小: 0x200 (512 字节)

#### 步骤 H1: 对每个 Flash 页

```
1. 加载页数据到 RAM buffer (128 words × 1 AP 写 DRW)
   【AP 写 TAR】  buffer_addr
   【AP 写 DRW】 data_word_0
   【AP 写 DRW】 data_word_1  (TAR 自增)
   ... 共 128 笔

2. 调用 ProgramPage
   设置 R0 = 页地址 (Flash 目标地址)
   设置 R1 = 数据大小 (0x200)
   设置 R2 = buffer 地址
   设置 PC = pc_program_page (0x591)  ← CYT2BL3 算法中
   运行 CPU, 等待 halt
   超时: 20ms

3. 验证写入 (可选, 通过 poll Flash 状态寄存器)
```

**每页 SWD 数据包**: 约 130-140 笔

**4MB Flash @ 512B/页 = 8192 页**  
**总 SWD 数据包**: 8192 × 135 ≈ **1,100,000 笔** (仅编程阶段)

---

### Phase I: Flash 验证

#### 方式 A: 用算法 Verify 函数
```
【AP 写 TAR】   buffer_addr
【AP 写 DRW】   期望数据
【AP 写 TAR】   flash_addr
调用 pc_verify (0x5A1) ← CYT2BL3 算法
返回: addr+size (成功) 或 失败地址 (失败)
```

#### 方式 B: 手动读回对比 (如果算法无 Verify)
```
对每个 32-bit word:
  【AP 写 TAR】  flash_addr
  【AP 读 DRW】  读取值 (posted: 需要读两次)
  对比读取值和期望值
```

---

### Phase J: 复位与运行

```
【AP 写 TAR】   TAR = 0xE000ED0C (AIRCR 地址)
【AP 写 DRW】   DRW = 0x05FA0004 (VECTKEY + SYSRESETREQ)
```

然后 probe-rs 可能会:
1. 设置 VTOR = 0x10000000 (向量表位于 Flash)
2. 从向量表读取 SP 初始值和 PC 初始值
3. 设置 SP 和 PC
4. 取消 CPU halt

---

## 4. CYT2BL3 特化细节

### 芯片信息
| 属性 | 值 |
|------|-----|
| 芯片 | CYT2BL3BAS (Cypress/Infineon TRAVEO T2G) |
| 内核 | 双核: Cortex-M0+ (AP1) + Cortex-M4F (AP2) |
| Flash | 4MB Code Flash @ 0x10000000 |
| RAM | 512KB SRAM @ 0x08000000 |
| DP 版本 | DPv2 (Cypress, Part 0xea02, Rev 0x1) |
| SWD 协议 | SWDv1 (通过 WCH-Link CMSIS-DAP) |
| AP 数量 | 3 个 MEM-AP (AP0/AP1/AP2) |

### Flash 算法入口点
```
load_address:      0x08001008 (RAM 中加载地址)
pc_init:           0x00000001
pc_uninit:         0x00000599
pc_program_page:   0x00000591
pc_erase_sector:   0x000004F1
pc_erase_all:      0x000004E9
pc_verify:         0x000005A1
pc_blank_check:    0x00000039
data_section_offset: 0x820
```

### Flash 扇区布局
```
扇区 0: 0x10000000, 大小 0x8000 (32KB)  - 引导区
扇区 1: 0x103F0000, 大小 0x2000 (8KB)   - 尾部配置区
页大小: 0x200 (512 字节)
```

### probe-rs info 输出解读

```
Debug Port: DPv2, Designer: Cypress, Part: 0xea02, Revision: 0x1, Instance: 0x00
```

- **DPv2**: DP 架构版本 2，支持多协议切换、Dormant 状态
- **Cypress**: JEP106 厂商代码对应 Cypress/Infineon
- **ROM Table @ 0xf1000000**: AP0 连接的系统级 ROM 表
- **ROM Table @ 0xe00ff000**: AP1/AP2 连接的 CoreSight ROM 表
- **Cortex-M4 ETM @ 0xe0041000**: 嵌入式 Trace 宏单元
- **CTI @ 0xf0002000**: 交叉触发接口

---

## 5. 总结：SWD 数据包统计

执行 `probe-rs download --chip CYT2BL3BAS --protocol swd --binary-format elf build/firmware.elf` 时，SWDIO/SWCLK 上发生的完整数据包序列统计：

### 各阶段 SWD 数据包数量估算

| 阶段 | SWD 数据包数 | 说明 |
|------|-------------|------|
| A. 协议切换 | 2 个序列 | Line Reset (54bit) + JTAG→SWD (16bit)，非标准 SWD 数据包 |
| B. 连接建立 | ~6 | DPIDR 读, ABORT 写, SELECT 写, CTRL/STAT 读 |
| C. 上电 | ~4 | CTRL/STAT 写 + 轮询读 |
| D. AP 发现 | ~500 | 256 AP × 2 次 SWD 操作 (大部分返回 0 直接跳过) |
| E. MEM-AP 初始化 | ~3 | SELECT 写, CSW 写 |
| F. 算法加载 | ~1000 | 算法 (~2KB) + 寄存器设置 |
| G. Flash 擦除 | ~40 | 每扇区约 20 笔，2 扇区 = 40 |
| H. Flash 编程 | **~10,000-1,100,000** | 取决于固件大小。假设 64KB 固件 = 128 页 = ~17,000 笔 |
| I. Flash 验证 | ~8,500 | 读回对比 (或由算法完成，更少) |
| J. 复位 | ~10 | AIRCR 写 + 寄存器设置 |

### 典型 64KB 固件下载总览

```
总 SWD 数据包: 约 30,000 笔
总 SWCLK 周期: 约 1,500,000 周期
  (每笔 SWD 数据包约 46-47 周期 × 30,000)

假设 SWCLK = 10 MHz:
  总时间 ≈ 150 ms (纯协议开销，不含算法执行时间)
  实际耗时 ≈ 1-3 秒 (含 Flash 编程等待时间)
```

### 每笔典型 SWD 操作的字节表示

以 **DP 读 DPIDR** 为例：

```
Host 发出的字节 (SWDIO 单向):
  0xA5 = 10100101  (请求头: Start=1, APnDP=0, RnW=1, A=00, Parity=1, Stop=0, Park=1)

Target 返回的字节:
  ACK: 001  = OK
  RDATA: DPIDR 值 (4 bytes, LSB first)
  例如 0x2BA01477 → SWDIO 上: 11101110 00101000 00000101 11010100 (LSB first 逐位)
  Parity: 1 bit
```

以 **AP 写 DRW** 为例 (向 Flash 地址写数据):

```
步骤 1: DP 写 SELECT (选择 AP2, Bank0)
  Host:  0x89 = 10001001 → 然后 WDATA = 0x02000000 (APSEL=2)

步骤 2: AP 写 TAR (A[3:2]=01)
  Host:  0x99 = 10011001 → 然后 WDATA = flash_addr

步骤 3: AP 写 DRW (A[3:2]=11)
  Host:  0xA1 = 10100001 → 然后 WDATA = data_word
```

---

## 参考源码文件清单

| 文件路径 | 作用 |
|---------|------|
| `probe-rs/src/architecture/arm/traits/polyfill.rs` | **核心**: SWD 数据包构建 (`build_swd_transfer`)、解析 (`parse_swd_response`) |
| `probe-rs/src/architecture/arm/traits.rs` | `RawDapAccess`, `DapAccess` trait 定义 |
| `probe-rs/src/architecture/arm/sequences.rs` | 初始化序列: `debug_port_setup`, `debug_port_connect`, `debug_port_start`, `swd_line_reset` |
| `probe-rs/src/architecture/arm/communication_interface.rs` | AP 选择、DP/AP 寄存器读写 |
| `probe-rs/src/architecture/arm/dp/mod.rs` | DP 寄存器定义 (DPIDR, CTRL/STAT, SELECT, ABORT, RDBUFF) |
| `probe-rs/src/architecture/arm/ap/mod.rs` | AP 类型定义 |
| `probe-rs/src/architecture/arm/ap/registers.rs` | AP 寄存器定义 (CSW, TAR, DRW, IDR, BASE) |
| `probe-rs/src/architecture/arm/ap/memory_ap/mod.rs` | MEM-AP 操作: `set_target_address()`, `read_data()`, `write_data()` |
| `probe-rs/src/probe/cmsisdap/mod.rs` | CMSIS-DAP 探针驱动, SWD 配置 |
| `probe-rs/src/probe/cmsisdap/commands/transfer/mod.rs` | CMSIS-DAP DAP_Transfer 命令实现 |
| `probe-rs/src/flashing/flasher.rs` | Flash 编程主逻辑 |
| `probe-rs/src/flashing/flash_algorithm.rs` | Flash 算法结构定义 |
| `probe-rs/src/flashing/loader.rs` | Flash 数据加载器 |
| `probe-rs/targets/CYT2BL_Series.yaml` | CYT2BL3 目标定义 (核心, Flash 算法, 内存映射) |

---

## 附录: ADIv5 规范对照 — NACK 问题根因分析

> 本节将实测中遇到的 NACK 现象与 ADIv5 规范原文进行逐条对照。

### A.1 NACK (ACK=111) 不是合法的 ACK 响应

**规范原文** (ADIv5 §4.3.4-§4.3.6, Table 4-1～4-4):

| ACK | 二进制 | 规范定义 |
|:---:|--------|---------|
| OK | `001` | DP 就绪，操作成功 |
| WAIT | `010` | 前序操作未完成 |
| FAULT | `100` | Sticky flag 置位 |

**没有 `111` 这个值。** ADIv5 Table 4-5 将 "No ACK" 定义为 "line is not driven"（线路无人驱动），而不是一个发送的值。

**结论**: probe-rs 日志中的 "NACK" 实际上是 CMSIS-DAP 探针读取到**目标未驱动线路**时的上拉电平，被解析为 `Ack::NoAck`。

### A.2 为什么目标不响应 (No ACK)

**规范原文** (ADIv5 §4.3.6 "Protocol error response"):
> "When a protocol error is detected by the SW-DP, the SW-DP does not reply to the packet request and does not drive the line."

协议错误触发条件:
- Parity 位与请求头不匹配
- Stop 位 ≠ 0
- Park 位 ≠ 1

**规范原文** (ADIv5 §4.3.6, 续):
> "When in protocol error state:
> - If the target detects a valid read of the DP DPIDR register, it is **IMPLEMENTATION DEFINED** whether the target leaves the protocol error state, and gives an OK response.
> - If the target detects a valid packet header, other than the read of the DP DPIDR register, or the target detects an **IMPLEMENTATION DEFINED** number of additional protocol errors, it enters the **lockout state**."
> 
> "The target must leave the protocol error state on a line reset."

**结论**: 如果目标进入了协议错误状态或锁定状态，Line Reset 是**唯一可靠的恢复方式**。probe-rs 的 `debug_port_connect` 每次重试前都做 Line Reset，这一点是正确的。

### A.3 SWD v2 的额外限制

**规范原文** (ADIv5 §4.3.6):
> "If the SW-DP implements SWD protocol version 2, it must enter the lockout state after a single protocol error immediately after a line reset."

CYT2BL3 使用 DPv2 (实测 DPIDR 版本字段=0x2)，如果它实现了 SWD v2 协议，则 **Line Reset 后的第一个数据包如果是协议错误 → 直接锁定**。这意味着如果 WCH-Link 发送的 SWD 包有任何位错误（即使只是时钟/时序问题），芯片会锁定，后续所有 Line Reset+DPIDR 读也全部失败。

### A.4 DPIDR 读必须是 OK

**规范原文** (ADIv5 Table 4-1):
> "The SW-DP must always give an OK response to a read of the IDCODE or CTRL/STAT register."

如果 DPIDR 读返回的不是 OK（而是 No Response = 线路未驱动），说明：
1. 物理连接有问题，或
2. 目标不在 SWD 模式（可能仍在 JTAG 模式），或
3. 目标处于锁定/掉电状态

### A.5 复位后的 DAP 重连要求

**规范原文** (ADIv5 §5.1.1 "SWJ-DP structure"):
> "This means that tools must not rely on the state of either DP, or any AP accessed through the DP, persisting when the other DP is selected. On switching DPs, the debugger must re-initialize the DAP, including setting the CTRL/STAT.{CDBGPWRUPREQ, CSYSPWRUPREQ} bits correctly."

这解释了为什么 reset 后需要重新上电 DAP — 而 probe-rs 的 `debug_port_start` 在连接后做了这件事（读取 CTRL/STAT → 写 CDBGPWRUPREQ/CSYSPWRUPREQ → 等待确认）。

### A.6 规范与实测的对应关系

| 现象 | ADIv5 规范依据 | 对应 probe-rs 行为 |
|------|---------------|-------------------|
| 反复 NACK | §4.3.6 Protocol error → 目标不驱动线路 | `process_batch` 收到 ACK=7 → 返回 `NoAcknowledge` |
| Line Reset 后仍 NACK | §4.3.6 Lockout 状态需 Line Reset 恢复 | `debug_port_connect` 每次循环都做 Line Reset |
| JTAG→SWD 后初始连接成功 | §5.2.1 切换序列正确 | `swj_sequence(16, 0xE79E)` ✅ |
| Reset 后 DP 失联 | §5.1.1 DAP 状态不持久 | `cortex_m_reset_system` 触发 SYSRESETREQ → DP 丢失 |
| 掉电 DP 不响应 | §2.4 Power control 需先上电 | `debug_port_start` 上电 ✅ (初始连接 OK) |
| SWD v2 锁定更严格 | §4.3.6 v2 首个错误→直接锁定 | CYT2BL3=DPv2 → 对时序错误更敏感 |

---

## 参考资料

1. **ARM IHI 0031C** — ARM Debug Interface Architecture Specification ADIv5.0 to ADIv5.2  
   路径: `docs/资料-ARM_ADIv5_Specification.pdf`
   - Chapter 4: The Serial Wire Debug Port (SW-DP) — 第 4.2-4.4 节
   - Chapter 5: The Serial Wire/JTAG Debug Port (SWJ-DP) — 第 5.2-5.3 节
   - Chapter 7: The Memory Access Port (MEM-AP) — 第 7.5-7.6 节

2. **probe-rs 源码**  
   路径: `tools/probe-rs-src/`

3. **CMSIS-DAP 协议**  
   https://arm-software.github.io/CMSIS_5/DAP/html/index.html

4. **Open-CMSIS-Pack Debug Description**  
   https://open-cmsis-pack.github.io/Open-CMSIS-Pack-Spec/main/html/debug_description.html

5. **CYT2BL3 数据手册**  
   路径: `docs/CYT2BL3核心板资料含IAR开发环境链接/芯片官方手册/`

---

*本报告由知心姐姐基于 ADIv5 官方手册和 probe-rs 源码，逐行分析后编写。如有疑问，随时来找姐姐哦~ 💖*
