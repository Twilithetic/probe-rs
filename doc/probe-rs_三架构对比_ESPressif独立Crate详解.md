# probe-rs 三架构对比 & probe-rs-espressif 独立 crate 详解

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：ARM / RISC-V / Xtensa 三架构的异同 + 为什么 Espressif 是独立 crate

---

## 一、三架构概览：同一个 trait，不同的硬件

### 1.1 统一的 `CoreInterface` trait

三者都实现同一个 `CoreInterface` trait（`probe-rs/src/core.rs`），flash 流程**完全通用**：

```
FlashLoader::commit()
  → Flasher::program()
    → ActiveFlasher::call_function()
        │
        ├─ core.write_core_reg(PC, ...)    ← 架构不同，接口一样
        ├─ core.write_core_reg(R0, ...)    ← 同上
        ├─ core.write_core_reg(SP, ...)    ← 同上
        ├─ core.run()                       ← 同上
        └─ wait_for_completion()            ← 同上
```

### 1.2 核心差异对照表

| 方面 | ARM | RISC-V | Xtensa |
|------|-----|--------|--------|
| **调试传输** | DAP (DP→AP) via SWD/JTAG | Debug Module via JTAG/AP | XDM via JTAG only |
| **PC 寄存器** | R15 (ID=15) | PC (CSR 0x7b1) | PC (XDM 0xFF00) |
| **SP 寄存器** | R13 (ID=13) | x2 (CSR 0x1002) | a1 (XDM 0x0001) |
| **返回地址** | R14/LR (ID=14) | x1/ra (CSR 0x1001) | a0 (XDM 0x0000) |
| **参数寄存器** | R0, R1, R2, R3 | x10-x13 / a0-a3 | a2, a3, a4, a5 |
| **Halt 机制** | DHCSR.C_HALT | DM haltreq | XDM halt |
| **单步** | DHCSR.C_STEP | DCSR step + program buffer | XDM step(1, intlevel) |
| **断点** | FP_COMP | Trigger 模块 | IBreakA + IBreakEn |
| **64位支持** | ARMv8-A (A64) | RV64 (Xlen64) | 无 |
| **窗口寄存器** | 无 | 无 | ⚠️ **有**，需 spill |
| **向量捕获** | DEMCR 支持 | 不支持 | 不支持 |
| **Semihosting** | BKPT 0xAB | ebreak + slli/srai | break 窄编码 |
| **FPU 检测** | CPACR/MVFR0 | MISA CSR 扩展位 | 目前 hardcode false |

### 1.3 Flash 流程中唯一的 ISA 分支

在 `Flasher::call_function()` 里，**只有一个地方**区分架构：

```rust
// flasher.rs:989-997
(
    self.core.return_address(),
    if self.instruction_set == InstructionSet::Thumb2 {
        Some(algo.load_address + 1)   // ← ARM Thumb: bit[0] 必须为 1
    } else {
        Some(algo.load_address)       // ← RISC-V / Xtensa: 直接用
    },
),
```

ARM Cortex-M 在 Thumb 模式下跳转地址 bit[0] 必须为 1（指示 Thumb 状态），其余架构不需要。

### 1.4 架构代码量对比

| 架构 | 文件数 | 最大文件 | 特点 |
|------|--------|---------|------|
| **ARM** | 48 个文件 | `sequences.rs` (1186行) | 最复杂：5种核心变体、CoreSight 组件、SWO 追踪 |
| **RISC-V** | 9 个文件 | `mod.rs` (1175行) | 最干净：XlenMode trait → 一个泛型结构体适配 RV32/RV64 |
| **Xtensa** | 12 个文件 | `communication_interface.rs` (1503行) | 最新：窗口寄存器、XDM 独占、JTAG only |

---

## 二、ARM 架构详解

### 2.1 目录结构

```
probe-rs/src/architecture/arm/
├── mod.rs                        ← 总入口，ArmError
├── core/
│   ├── mod.rs                    ← CortexMState, CortexARState
│   ├── armv6m.rs                 ← Cortex-M0/M0+
│   ├── armv7m.rs                 ← Cortex-M3/M4/M7
│   ├── armv8m.rs                 ← Cortex-M23/M33
│   ├── armv7ar.rs                ← Cortex-A/R 32位
│   ├── armv8a.rs                 ← Cortex-A 64位
│   └── registers/
│       └── cortex_m.rs           ← 寄存器常量定义
├── dp/mod.rs                     ← Debug Port
├── ap/                           ← Access Port (v1, v2, MemoryAP)
├── traits.rs                     ← RawDapAccess, DapAccess
├── sequences.rs                  ← 调试序列（1186行！）
├── communication_interface.rs    ← ArmCommunicationInterface
├── swo/                          ← 串行线输出追踪
├── component/                    ← CoreSight 组件 (SCS, DWT, ITM, TPIU...)
└── memory/
    ├── romtable.rs               ← CoreSight ROM 表解析
    └── adi_memory_interface.rs   ← ADI 内存接口
```

### 2.2 调试传输：DAP 层级

```
Probe (USB)
  └→ Debug Port (DP)
       ├→ SWD (2线) 或 JTAG (4线)
       └→ Access Port (AP)
            └→ Memory AP (AHB3/AHB5/APB2-5/AXI3-5)
                 └→ 芯片内存总线
```

### 2.3 5 种核心变体

```rust
pub enum CoreType {
    Armv6m,   // Cortex-M0, M0+
    Armv7m,   // Cortex-M3, M4, M7
    Armv8m,   // Cortex-M23, M33
    Armv7a,   // Cortex-A7, A9, R4, R5 (32位)
    Armv8a,   // Cortex-A53, A72 (64位)
}
```

### 2.4 关键特征

- **Thumb bit**: Flash 算法返回地址必须 `+1`，因为 Cortex-M 的 bit[0] 区分 ARM/Thumb 状态
- **向量捕获**: 唯一支持 `enable_vector_catch()` 的架构，可以捕获 HardFault、MemManage 等异常
- **CoreSight 生态**: ROM 表、DWT、ITM、TPIU、SWO 等调试组件，arm 专属

---

## 三、RISC-V 架构详解

### 3.1 目录结构

```
probe-rs/src/architecture/riscv/
├── mod.rs                        ← RiscvCore<X: XlenMode>, CoreInterface
├── registers.rs                  ← RV32 寄存器定义
├── registers64.rs                ← RV64 寄存器定义
├── assembly.rs                   ← 指令汇编器 (EBREAK, lw, sw, csr...)
├── communication_interface.rs    ← RiscvCommunicationInterface
├── dtm/
│   ├── mod.rs
│   ├── jtag_dtm.rs               ← JTAG Debug Transport Module
│   └── mem_ap_dtm.rs             ← 通过 ARM AP 访问 DM
└── sequences.rs                  ← RiscvDebugSequence
```

### 3.2 泛型设计：XlenMode

```rust
// 一个泛型结构体适配 32 位和 64 位！
pub struct RiscvCore<'state, X: XlenMode> {
    // ...
}

pub trait XlenMode: Debug + Send + Sync + 'static {
    type CSR: CsrAccess;      // CSR 寄存器的读写方式 (32/64位不同)
    fn registers() -> &'static CoreRegisters;
    fn pc_value(value: RegisterValue) -> u64;
    fn sp_value(value: RegisterValue) -> u64;
    // ...
}

pub struct Xlen32;
pub struct Xlen64;
```

### 3.3 调试传输：DM (Debug Module)

```
Probe
  ├→ JTAG DTM (直接连接 RISC-V DM)
  └→ ARM AP → Memory-AP DTM (通过 ARM AP 间接访问 RISC-V DM)
```

### 3.4 关键特征

- **最简洁**: 9 个文件搞定，泛型设计让 RV32/RV64 共享一套代码
- **Trigger 模块**: 硬件断点通过 RISC-V 标准的 Trigger 模块 (TSELECT/TDATA1/TDATA2)
- **Abstract Command**: 通过 DM 的 abstract command 机制访问内存，比 ARM 的 AP 简单
- **Semihosting**: `ebreak` 后面跟 `slli x0, x0, 0x1f` + `srai x0, x0, 7` 序列来标识

---

## 四、Xtensa 架构详解

### 4.1 目录结构

```
probe-rs/src/architecture/xtensa/
├── mod.rs                        ← XtensaCore, CoreInterface
├── registers.rs                  ← 寄存器定义
├── communication_interface.rs    ← XtensaCommunicationInterface (1503行！)
├── xdm.rs                        ← Xtensa Debug Module 原始访问
├── sequences.rs                  ← XtensaDebugSequence
├── arch/
│   ├── mod.rs                    ← 架构常量
│   └── instruction/
│       ├── mod.rs
│       └── format.rs             ← 指令编码/解码
└── register_cache.rs             ← 延迟寄存器读缓存
```

### 4.2 窗口寄存器（Xtensa 独有）

Xtensa 的物理寄存器文件比可见的逻辑窗口大。切换函数时，不是压栈，而是滑动寄存器窗口。probe-rs 调试时必须 **spill（溢出）** 不可见窗口：

```rust
// mod.rs
pub struct RegisterFile {
    // 物理寄存器文件 (64个 AR 寄存器 = 8 窗口 × 8 regs/窗口)
    // 可见窗口: 4 组 (a0-a3 = 上一窗口, a4-a7 = 当前窗口, a8-a11 = 下一窗口, a12-a15 = 下下窗口)
}

pub fn spill_registers(&mut self) -> Result<(), Error> {
    // 读取不可见窗口的寄存器 → 保存到内存
    // 这是调试 Xtensa 的前提！
}
```

### 4.3 调试传输：XDM only

```
Probe
  └→ JTAG (唯一的选项！)
       └→ Xtensa Debug Module (XDM)
```

### 4.4 关键特征

- **JTAG only**: 不支持 SWD
- **窗口寄存器**: 调试前必须 spill，否则看不到完整寄存器状态
- **最新加入**: 12 个文件，代码量不小，专门为 ESP32 系列设计
- **批量读写优化**: `register_cache.rs` 实现延迟读，减少 JTAG 往返

---

## 五、`InstructionSet` 枚举

```rust
// probe-rs-target/src/chip_family.rs
pub enum InstructionSet {
    Thumb2,   // ARM Cortex-M (也用于 A/R 的 Thumb 模式)
    A32,      // ARM 32位 (传统 ARM)
    A64,      // ARM 64位 (AArch64)
    RV32,     // RISC-V 32位 未压缩
    RV32C,    // RISC-V 32位 压缩指令
    RV64,     // RISC-V 64位 未压缩
    RV64C,    // RISC-V 64位 压缩指令
    Xtensa,   // Xtensa
}
```

### 各架构如何判定

| 架构 | 判定方式 |
|------|---------|
| ARM M-profile | 始终 `Thumb2` |
| ARM A-profile | 读 CPSR → `Thumb2` 或 `A32` |
| ARMv8-A (64位) | 当前状态 → `A64`, `Thumb2`, 或 `A32` |
| RISC-V | 读 MISA CSR bit[2] ("C"扩展) → 决定是否压缩 |
| Xtensa | 目前始终 `Xtensa` |

---

## 六、Flash 算法的语言

| 架构 | 算法二进制格式 | 来源 |
|------|---------------|------|
| ARM | ARM Thumb2 机器码 | CMSIS-Pack (芯片厂商提供) |
| RISC-V | RISC-V 机器码 | CMSIS-Pack |
| Xtensa | Xtensa 机器码 | ESP-IDF / 芯片厂商 |

**probe-rs 不生成 Flash 算法**，只负责：加载到 RAM → 设寄存器 → `core.run()` → 等结果。

---

## 七、`probe-rs-espressif`：独立 crate 的原因

### 7.1 它比普通 vendor 多提供什么？

| 能力 | 普通 vendor (如 Infineon) | Espressif |
|------|--------------------------|-----------|
| 调试序列 (关WDT, 配内存, 复位) | ✅ | ✅ (且更复杂) |
| 芯片检测 | ✅ | ✅ (ROM 魔法值 0x40001000) |
| **自定义探针驱动** | ❌ | ✅ EspUsbJtag (USB bulk 协议) |
| **自定义固件格式** | ❌ | ✅ IDF 格式 (bootloader + partition) |
| **外部重依赖** | ❌ | ✅ `espflash` (v4) |
| | | ✅ `nusb` |
| **Semihosting 扩展** | ❌ | ✅ Panic/断点处理 |
| **RTC 内存 Reset 程序** | ❌ | ✅ (ESP32 需要下载 reset stub) |
| **目标芯片数** | 通常 1 家族 | 12 个家族 (ESP32/S2/S3/C2/C3/C5/C6/C61/H2/P4...) |

### 7.2 四个注册项

```rust
// probe-rs-espressif/src/lib.rs
pub fn register_plugin() {
    plugin::register_plugin(plugin::Plugin {
        vendors:       &[&Espressif],          // ① 芯片检测 + Xtensa/RISC-V 序列
        image_formats: &[&IdfLoaderFactory],   // ② ESP-IDF 多段固件格式
        targets:       &targets,               // ③ 12 种芯片目标定义
        probe_drivers: &[&EspUsbJtagFactory],  // ④ USB-JTAG 探针 (VID:0x303A)
    });
}
```

### 7.3 插件架构

```
probe-rs (核心库)              probe-rs-espressif (插件)
┌─────────────────────┐       ┌─────────────────────────┐
│ 轻量依赖:            │       │ 重依赖:                  │
│ - bitfield           │       │ - espflash v4           │
│ - jep106             │       │ - nusb                  │
│ - object             │       │ - bincode               │
│ - probe-rs-target    │       │ - bitvec                │
│                      │       │ - probe-rs (核心)        │
│                      │       │                         │
│ Plugin 扩展点:       │  ───→ │ 通过 register_plugin()   │
│ register_vendor()    │  注册  │ 注入到全局注册表         │
│ register_format()    │       │                         │
│ register_target()    │       │ CLI 启动时自动调用       │
│ register_probe()     │       │ main.rs:537             │
└─────────────────────┘       └─────────────────────────┘
```

### 7.4 EspUsbJtag 探针：自定义 USB 协议

| 方面 | CMSIS-DAP | ESP USB JTAG |
|------|-----------|-------------|
| **VID:PID** | 各厂商不同 | `0x303A:0x1001` (内置) / `0x303A:0x1002` (桥接) |
| **协议层** | 标准 CMSIS-DAP 命令 | 4-bit nibble 指令流 |
| **传输** | HID 或 WinUSB Bulk | USB Bulk |
| **支持协议** | SWD + JTAG | **仅 JTAG** |
| **重复编码** | 无 | 内建 RLE (最多 1023 次重复) |
| **捕获缓冲** | 无限制 | 128 字节 (1024 bits) |

命令编码：
```
Clock:  0b0C_TM_TD    (capture, tms, tdi)
Reset:  0b1000_0SRST  (软件复位)
Flush:  0b1010         (刷新缓冲)
Repeat: 0b1100 + 2bit base-4 计数器
```

### 7.5 IDF 固件格式

ESP-IDF 固件不是单个 ELF，而是：
- **Bootloader** (单独二进制)
- **Partition Table** (分区表)
- **Application ELF** (应用固件)

probe-rs 需要用 `espflash` crate 将三者合并成连续的 Flash 段，才能正常烧录。

### 7.6 ESP32 复位序列的特殊性

ESP32 系统复位会**禁用 JTAG**，所以 probe-rs 需要：
1. 下载一个 "reset stub" 到 RTC SLOW 内存 (0x50000000)
2. 在 RTC 内存中执行它
3. 程序重新使能 JTAG
4. 轮询 `RTC_CNTL_RESET_STATE_REG` 等完成

这个复杂流程仅在 ESP32 (Xtensa) 上需要。

---

## 八、普通 vendor 对比：CYT2BL (Infineon)

```rust
// probe-rs/src/vendor/infineon/sequences/cyt2bl.rs
impl ArmDebugSequence for Cyt2bl {
    fn reset_system_and_halt(&self, memory, timeout) -> Result<(), ArmError> {
        // CYT2BL 在 SWD 复位后需要特殊处理：
        // 通过 HSIOM 寄存器配置引脚，确保 SWD 重连
    }
}
```

Infineon CYT2BL 只需要处理 SWD 重连，不需要：
- ❌ Reset stub 下载
- ❌ 关三个看门狗
- ❌ ROM 魔法值检测
- ❌ 自定义探针
- ❌ 自定义固件格式

所以几行调试序列就够了，无需独立 crate。

---

## 九、源码位置速查

### 三架构

| 内容 | 文件 |
|------|------|
| `CoreInterface` trait | `probe-rs/src/core.rs:38` |
| `InstructionSet` enum | `probe-rs-target/src/chip_family.rs:109` |
| ARM `call_function` 中的 +1 | `probe-rs/src/flashing/flasher.rs:989-997` |
| ARM core 目录 | `probe-rs/src/architecture/arm/core/` |
| RISC-V core (`RiscvCore`) | `probe-rs/src/architecture/riscv/mod.rs` |
| RISC-V `XlenMode` trait | `probe-rs/src/architecture/riscv/mod.rs` |
| Xtensa core (`Xtensa`) | `probe-rs/src/architecture/xtensa/mod.rs` |
| Xtensa `spill_registers()` | `probe-rs/src/architecture/xtensa/mod.rs` |

### Espressif

| 内容 | 文件 |
|------|------|
| crate 入口 + `register_plugin()` | `probe-rs-espressif/src/lib.rs` |
| 插件系统 (`register_plugin`) | `probe-rs/src/plugin.rs` |
| IDF 固件格式 | `probe-rs-espressif/src/image_format.rs` |
| 调试序列 | `probe-rs-espressif/src/sequences/` |
| ESP USB JTAG 探针 | `probe-rs-espressif/src/espusbjtag/` |
| CLI 启动 (`register_plugin`) | `probe-rs-tools/src/bin/probe-rs/main.rs:537` |
| workspace Cargo.toml | `Cargo.toml` (根) |
| probe-rs Cargo.toml (无 espressif!) | `probe-rs/Cargo.toml` |
| probe-rs-tools Cargo.toml | `probe-rs-tools/Cargo.toml:100` |
