# probe-rs 模块化架构与可配置性 — 全景分析报告

> **日期**: 2026-05-06  
> **probe-rs 版本**: 0.31.0  
> **源码路径**: `tools/probe-rs-src/`

---

## 1. 整体架构 — 五层模型

```
┌─────────────────────────────────────────────────────────┐
│                   CLI 层 (probe-rs-tools)                │
│  probe-rs download | run | info | erase | ...            │
│  参数: --chip --protocol --speed --probe --connect-...   │
├─────────────────────────────────────────────────────────┤
│                   Session 层                             │
│  Session::auto_attach(chip) → Session::core(n)           │
│  配置: SessionConfig, Permissions, TargetSelector        │
├─────────────────────────────────────────────────────────┤
│                   Target 层                              │
│  Target { cores, memory_map, flash_algorithms,           │
│           debug_sequence: DebugSequence }                │
│  来源: YAML 文件 (编译时嵌入 或 运行时加载)               │
├─────────────────────────────────────────────────────────┤
│                   Sequence 层                            │
│  ArmDebugSequence / RiscvDebugSequence / Xtensa           │
│  可重写: reset_system, debug_port_setup, 等 13 个方法     │
│  注册: Vendor trait → try_create_debug_sequence()         │
├─────────────────────────────────────────────────────────┤
│                   Probe 层                               │
│  Probe → CMSIS-DAP / ST-Link / J-Link / FTDI / ...       │
│  DapProbe → swj_sequence, raw_read_register, 等           │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Workspace 子 crate 结构

> 源码: `Cargo.toml`

| crate | 功能 |
|-------|------|
| **`probe-rs`** | 核心库：Probe, Session, Core, Flash, Config, Vendor |
| **`probe-rs-target`** | 目标描述数据结构定义 (Chip, Core, MemoryRegion, Architecture 等) |
| **`probe-rs-tools`** | CLI 工具: `probe-rs download/run/info/erase` + GDB Server + DAP Server |
| **`target-gen`** | CMSIS-Pack → YAML 目标描述生成器 |
| **`probe-rs-debug`** | VS Code 调试适配器 |
| **`probe-rs-mi`** | 机器接口 (Machine Interface) |
| **`probe-rs-espressif`** | Espressif 芯片专用支持 |
| **`smoke-tester`** | 集成测试框架 |

---

## 3. 配置点全景图

### 3.1 运行时配置 (CLI 参数)

> 源码: `probe-rs-tools/.../common_options.rs`

```
┌──────────────┬───────────────────────────────────────────┐
│ 参数          │ 作用                                     │
├──────────────┼───────────────────────────────────────────┤
│ --chip       │ 芯片名称 (如 CYT2BL3BAS)                  │
│ --protocol   │ 协议: swd / jtag                          │
│ --speed      │ SWD/JTAG 频率 (kHz)                       │
│ --probe      │ 指定探针 VID:PID:SERIAL                   │
│ --connect-   │ 复位状态下连接 (assert nRESET → connect)   │
│   under-reset│                                           │
│ --chip-      │ 外部目标描述 YAML 文件路径                 │
│   description│                                           │
│   -path      │                                           │
│ --cycle-     │ 连接前 USB 断电再上电                     │
│   power      │                                           │
│ --dry-run    │ 不实际执行，仅验证                         │
│ --allow-     │ 允许擦除只读区域                           │
│   erase-all  │                                           │
└──────────────┴───────────────────────────────────────────┘
```

### 3.2 代码级配置 (SessionConfig)

```rust
// probe-rs/src/session.rs
pub struct SessionConfig {
    // 会话级别配置
    pub permissions: Permissions,
    // ...
}

pub struct Permissions {
    pub allow_erase_all: bool,
}
```

### 3.3 目标描述 (YAML 配置)

> 路径: `probe-rs/targets/CYT2BL_Series.yaml`

这是**最丰富的配置层**：

```yaml
# CYT2BL_Series.yaml 结构示意
name: CYT2BL                          # 芯片族名
variants:                             # 变体列表
  - name: CYT2BL3BAS
    cores:                            # ← 核心配置
      - name: cortex-m0p
        type: armv6m
        core_access_options: !Arm
          ap: !v1 1                   # ← AP 地址 (可通过哪个 AP 访问)
      - name: cortex-m4
        type: armv7em
        core_access_options: !Arm
          ap: !v1 2
    memory_map:                       # ← 内存映射
      - !Ram
        name: IRAM1
        range: { start: 0x8000000, end: 0x8080000 }
        cores: [cortex-m0p, cortex-m4]
      - !Nvm
        name: IROM1
        range: { start: 0x10000000, end: 0x10410000 }
        cores: [cortex-m0p, cortex-m4]
    flash_algorithms:                 # ← 关联的 Flash 算法
      - cyt2bl
      - cyt2bx_wflash_128
      # ...更多算法

flash_algorithms:                     # ← Flash 算法定义
  - name: cyt2bl
    load_address: 0x8001008           # RAM 加载地址
    pc_init: 0x1                     # init() 偏移
    pc_program_page: 0x591           # program_page() 偏移
    pc_erase_sector: 0x4f1           # erase_sector() 偏移
    # ... 各入口点
    flash_properties:
      address_range: { start: 0x10000000, end: 0x10410000 }
      page_size: 0x200
      sectors:
        - { size: 0x8000, address: 0x0 }    # 大 Sector
        - { size: 0x2000, address: 0x3f0000 } # 小 Sector
```

### 3.4 Vendor Sequence (代码级扩展)

> 这是你需要的核心扩展点！

```rust
// 现有 Infineon vendor: vendor/infineon/mod.rs
impl Vendor for Infineon {
    fn try_create_debug_sequence(&self, chip: &Chip) -> Option<DebugSequence> {
        if chip.name.starts_with("XMC4") {
            Some(DebugSequence::Arm(XMC4000::create()))
        } else if chip.name.starts_with("PSE84") {
            Some(DebugSequence::Arm(PsocEdge::create(chip)))
        } else {
            None  // ← CYT2BL 落在这里，使用 DefaultArmSequence
        }
    }
}
```

**需要添加**: `chip.name.starts_with("CYT2BL") → DebugSequence::Arm(Cyt2bl::create())`

---

## 4. ArmDebugSequence Trait — 全部 13 个可重写方法

> 源码: `sequences.rs:444-1176`

### 4.1 方法分类

```
┌─────────────────────┬────────────────────────────────────────────────┐
│ 类别                 │ 方法                                           │
├─────────────────────┼────────────────────────────────────────────────┤
│ 🔌 DP 连接           │ debug_port_setup()                             │
│                     │ debug_port_connect()                           │
│                     │ debug_port_start()   (上电)                     │
│                     │ debug_port_stop()    (下电)                     │
├─────────────────────┼────────────────────────────────────────────────┤
│ 🧠 Core 管理         │ debug_core_start()  (使能调试 + halt)          │
│                     │ debug_core_stop()                              │
├─────────────────────┼────────────────────────────────────────────────┤
│ 🔄 复位              │ reset_system()      ← ★ 你最需要的             │
│                     │ reset_hardware_assert()   (nRESET LOW)         │
│                     │ reset_hardware_deassert() (nRESET HIGH)        │
│                     │ reset_catch_set()    (复位后捕获)               │
│                     │ reset_catch_clear()                            │
├─────────────────────┼────────────────────────────────────────────────┤
│ 🔓 特殊功能          │ debug_device_unlock() (解锁)                   │
│                     │ recover_support_start() (低功耗恢复)            │
│                     │ debug_erase_sequence() (返回自定义擦除序列)      │
├─────────────────────┼────────────────────────────────────────────────┤
│ 🪣 批量擦除          │ erase_all()                                    │
└─────────────────────┴────────────────────────────────────────────────┘
```

### 4.2 每个方法的签名与能力

```rust
pub trait ArmDebugSequence: Send + Sync + Debug {
    // ── 硬件复位 ──
    fn reset_hardware_assert(&self, interface: &mut dyn DapProbe)
        -> Result<(), ArmError>;
    
    fn reset_hardware_deassert(&self, probe: &mut dyn ArmDebugInterface,
        default_ap: &FullyQualifiedApAddress) -> Result<(), ArmError>;
    
    // ── DP 连接 ──
    fn debug_port_setup(&self, interface: &mut dyn DapProbe, dp: DpAddress)
        -> Result<(), ArmError>;
    
    fn debug_port_connect(&self, interface: &mut dyn DapProbe, dp: DpAddress)
        -> Result<(), ArmError>;
    
    fn debug_port_start(&self, interface: &mut dyn DapAccess, dp: DpAddress)
        -> Result<(), ArmError>;
    
    fn debug_port_stop(&self, interface: &mut dyn DapAccess, dp: DpAddress)
        -> Result<(), ArmError>;
    
    // ── Core 管理 ──
    fn debug_core_start(&self, interface: &mut dyn ArmDebugInterface,
        core_ap: &FullyQualifiedApAddress, core_type: CoreType,
        debug_base: Option<u64>, cti_base: Option<u64>) -> Result<(), ArmError>;
    
    fn debug_core_stop(&self, interface: &mut dyn ArmMemoryInterface)
        -> Result<(), ArmError>;
    
    // ── ★ 系统复位 (你最需要的) ──
    fn reset_system(&self, interface: &mut dyn ArmMemoryInterface,
        core_type: CoreType, debug_base: Option<u64>)
        -> Result<(), ArmError>;
    
    // 复位后捕获 (VC_CORERESET → halt at reset vector)
    fn reset_catch_set(&self, core: &mut dyn ArmMemoryInterface,
        core_type: CoreType, debug_base: Option<u64>) -> Result<(), ArmError>;
    
    fn reset_catch_clear(&self, core: &mut dyn ArmMemoryInterface,
        core_type: CoreType, debug_base: Option<u64>) -> Result<(), ArmError>;
    
    // ── 特殊功能 ──
    fn debug_device_unlock(&self, interface: &mut dyn ArmDebugInterface,
        default_ap: &FullyQualifiedApAddress,
        permissions: &Permissions) -> Result<(), ArmError>;
    
    fn recover_support_start(&self, interface: &mut dyn ArmMemoryInterface)
        -> Result<(), ArmError>;
    
    fn debug_erase_sequence(&self) -> Option<Arc<dyn DebugEraseSequence>>;
    
    fn erase_all(&self, interface: &mut dyn ArmDebugInterface)
        -> Result<(), ArmError>;
}
```

---

## 5. DapProbe Trait — 底层 probe 接口

> 源码: `communication_interface.rs` + `mod.rs` (CMSIS-DAP)

这是自定义序列中直接操作 SWD 总线的接口：

```rust
pub trait DapProbe: RawDapAccess + DebugProbe {
    // 继承自 DebugProbe:
    fn select_protocol(&mut self, protocol: WireProtocol) -> ...;
    fn active_protocol(&self) -> Option<WireProtocol>;
    fn speed_khz(&self) -> u32;
    fn set_speed(&mut self, speed_khz: u32) -> ...;
    fn target_reset(&mut self) -> ...;        // 硬件 nRESET

    // 继承自 RawDapAccess:
    fn raw_read_register(&mut self, addr: RegisterAddress) -> Result<u32, ArmError>;
    fn raw_write_register(&mut self, addr: RegisterAddress, value: u32) -> ...;
    fn raw_flush(&mut self) -> ...;

    // SWD 总线原始操作:
    fn swj_sequence(&mut self, bit_len: u8, bits: u64) -> ...;   // ★ 发送任意 bit 序列
    fn swj_pins(&mut self, pin_mask, pin_value, timeout) -> ...; // ★ 控制 SWD 引脚
    fn jtag_sequence(&mut self, bit_len: u8, tms: u8, tdi: u64) -> ...;
}

pub trait ArmDebugInterface {
    fn reinitialize(&mut self) -> Result<(), ArmError>;  // ★ 断开+重连 DP
    fn memory_interface(...) -> ...;
    fn current_debug_port(&self) -> Option<DpAddress>;
    fn close(self: Box<Self>) -> Probe;
}
```

---

## 6. Vendor 注册流程

### 6.1 已有 Vendor 列表

```
vendor/
├── amd/
├── holtek/
├── infineon/    ← ★ 已存在！只需添加 CYT2BL sequence
│   ├── mod.rs
│   └── sequences/
│       ├── mod.rs
│       ├── psoc_edge.rs    ← PSoC Edge 序列
│       └── xmc4000.rs      ← XMC4000 序列
├── microchip/
├── nordicsemi/
├── nuclei/
├── nxp/
├── raspberrypi/
├── renesas/
├── sifli/
├── silabs/
├── st/
├── ti/
└── vorago/
```

### 6.2 添加新 Sequence 的步骤

```rust
// 1. 创建 vendor/infineon/sequences/cyt2bl.rs
#[derive(Debug)]
pub struct Cyt2bl;

impl Cyt2bl {
    pub fn create() -> Arc<dyn ArmDebugSequence> {
        Arc::new(Self)
    }
}

impl ArmDebugSequence for Cyt2bl {
    fn reset_system(&self, interface: &mut dyn ArmMemoryInterface,
        core_type: CoreType, debug_base: Option<u64>) -> Result<(), ArmError> 
    {
        // ★ 自定义复位逻辑: 轮询 DPIDR
    }
}

// 2. 在 vendor/infineon/sequences/mod.rs 中导出
pub mod cyt2bl;

// 3. 在 vendor/infineon/mod.rs 中注册
impl Vendor for Infineon {
    fn try_create_debug_sequence(&self, chip: &Chip) -> Option<DebugSequence> {
        if chip.name.starts_with("XMC4") { ... }
        else if chip.name.starts_with("PSE84") { ... }
        else if chip.name.starts_with("CYT2BL") {       // ← 添加这行
            Some(DebugSequence::Arm(Cyt2bl::create()))
        }
        else { None }
    }
}
```

---

## 7. Target Registry — 目标发现机制

```rust
// config/registry.rs
pub struct Registry {
    families: Vec<ChipFamily>,  // 芯片族列表
}

// config/target.rs
pub enum TargetSelector {
    Unspecified(String),  // 按名称查找
    Specified(Target),    // 直接指定 (用于外部 YAML)
    Auto,                  // 自动检测 (读 ROM Table)
}

// Target 创建流程:
// ChipFamily (YAML) → Chip (variant) → Target
//    ↓
// vendor::try_create_debug_sequence(chip)
//    ↓
// DebugSequence::Arm(sequence)   ← 这里可以替换为自定义 sequence
```

---

## 8. 各层可配置程度总结

```
┌───────────────────┬───────────────────────────────────────┬──────────┐
│ 层次               │ 可配置内容                            │ 方式     │
├───────────────────┼───────────────────────────────────────┼──────────┤
│ CLI 参数           │ chip, protocol, speed, probe, 等      │ 命令行   │
│ SessionConfig     │ permissions (allow_erase_all)         │ 代码     │
│ Target (YAML)     │ cores, memory_map, flash_algorithms,  │ YAML 文件 │
│                   │   sector layout, page_size, 等        │          │
│ DebugSequence     │ 全部 13 个方法可重写 ★                │ Rust 代码│
│ Probe             │ protocol selection, speed, 探针参数   │ 代码     │
│ Flash Algorithm   │ load_address, 各入口点, 算法指令      │ YAML 文件 │
│ Memory Map        │ RAM/NVM 区域, 权限, 别名             │ YAML 文件 │
│ RTT Scan Regions  │ 默认扫描 RAM 或指定地址范围            │ YAML 文件 │
│ JTAG Scan Chain   │ IR 长度, TAP 数量                     │ YAML 文件 │
│ Chip Detection    │ 芯片识别方法 (ROM Table, SCS, 自定义)  │ YAML + 代码│
└───────────────────┴───────────────────────────────────────┴──────────┘
```

---

## 9. 针对 CYT2BL3 问题的扩展方案对比

| 方案 | 侵入性 | 实现难度 | 效果 |
|------|:--:|:--:|------|
| **改一行超时** (`600ms → 10000ms`) | 低 | ⭐ | 可能有效，但不优雅 |
| **新建 Vendor Sequence** | 中 | ⭐⭐⭐ | 最优雅，可复用 |
| **改 `cortex_m_wait_for_reset` 增加 DP reinit 重试** | 中 | ⭐⭐ | 通用改进 |
| **改 `flasher.rs` 跳过 `reset_and_halt`** | 低 | ⭐ | 快速但不够完整 |
| **改 `debug_port_connect` 超时** | 低 | ⭐ | 全局影响其他芯片 |

**推荐**: 路径 2 (新建 Vendor Sequence)，因为 Infineon vendor 已存在，只需加一个文件 + 一行注册。

---

## 10. 参考资料

| 资源 | 路径/URL |
|------|---------|
| probe-rs 官方文档 | https://probe.rs/docs/ |
| Library API 文档 | https://docs.rs/probe-rs/*/ |
| ArmDebugSequence trait | `probe-rs/src/architecture/arm/sequences.rs:444-1176` |
| Vendor trait | `probe-rs/src/vendor/mod.rs:40-73` |
| Infineon vendor (现有) | `probe-rs/src/vendor/infineon/mod.rs` |
| Target YAML 示例 | `probe-rs/targets/CYT2BL_Series.yaml` |
| DebugSequence enum | `probe-rs/src/config/target.rs:262-269` |
| ProbeOptions CLI | `probe-rs-tools/.../common_options.rs:89-138` |

*本报告由知心姐姐基于 probe-rs 0.31 源码和官方文档编写 💖*
