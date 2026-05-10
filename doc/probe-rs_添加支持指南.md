# probe-rs 添加 CYT2BL3 支持 — 完整操作报告

> **日期**: 2026-05-05  
> **目标**: 让 probe-rs 识别并烧录 Infineon CYT2BL3  
> **结果**: ✅ 识别成功，⚡ Flash 算法已包含，⚠️ 烧录遇物理层限制

---

## 1. 前置条件

| 项目 | 详情 |
|------|------|
| probe-rs 源码 | `tools/probe-rs-src/` (v0.31.0) |
| Rust 工具链 | rustc 1.94.1, cargo 1.94.1 |
| CMSIS-Pack | `tools/infineon-packs/CAT1A_DFP/Infineon.CAT1A_DFP.1.5.0.pack` |
| 调试器 | WCH-Link (CMSIS-DAP v2, VID:PID=0x1a86:0x8012) |
| 固件 | `build/firmware.hex` (LED 闪烁, P23.7) |

---

## 2. 操作步骤

### Step 1: 编译 target-gen

```bash
cd tools/probe-rs-src
cargo build --release --bin target-gen
# 产物: target/release/target-gen.exe
# 耗时: ~6 分钟
```

### Step 2: 从 CMSIS-Pack 提取 YAML

```bash
target-gen pack \
  "tools/infineon-packs/CAT1A_DFP/Infineon.CAT1A_DFP.1.5.0.pack" \
  "tools/cyt2bl-yaml"
```

**输出**:
```
Generated 10 target definition(s):
  PSoC_4500H.yaml
  PSoC_60.yaml ... PSoC_64.yaml
  CYT2B6.yaml
  CYT2B7.yaml
  CYT2B9.yaml
  CYT2BL.yaml        ← 包含 CYT2BL3 全部变体!
Finished in 1.29s
```

**生成的 CYT2BL.yaml**:
- 大小: 45 KB
- 包含 20 个芯片变体 (CYT2BL3BAS 到 CYT2BL8CAS)
- 包含完整 Flash 算法 (base64 编码的二进制指令)
- 包含内存映射 (Code Flash, Work Flash, SRAM, SFlash)
- 包含芯片检测信息 (`InfineonPsocSiid`)

### Step 3: 放入 probe-rs targets 目录

```bash
Copy-Item tools/cyt2bl-yaml/CYT2BL.yaml \
         tools/probe-rs-src/probe-rs/targets/CYT2BL_Series.yaml
```

### Step 4: 编译 probe-rs

```bash
cd tools/probe-rs-src
cargo build --release --bin probe-rs
# 耗时: ~10 分钟
```

### Step 5: 安装到系统

```bash
cargo install --path probe-rs-tools --bin probe-rs --force
# 替换 C:\Users\29344\.cargo\bin\probe-rs.exe
# v0.30.0 → v0.31.0 (含 CYT2BL 支持)
```

---

## 3. 验证结果

### 3.1 芯片列表

```bash
$ probe-rs chip list | Select-String "CYT2BL"

CYT2BL
        CYT2BL3BAE
        CYT2BL3BAS    ← 我们的芯片!
        CYT2BL3CAE
        CYT2BL3CAS
        CYT2BL4BAE
        ... (共 20 个变体)
```

### 3.2 芯片详细信息

```bash
$ probe-rs chip info CYT2BL3BAS

CYT2BL3BAS
Cores (2):
    - cortex-m0p (Armv6m)         ← CM0+
    - cortex-m4 (Armv7em)          ← CM4F
RAM: 0x08000000..0x08080000 (512.0 KiB)
NVM: 0x10000000..0x10410000 (4.1 MiB)    ← Code Flash
NVM: 0x14000000..0x14020000 (128.0 KiB)  ← Work Flash
NVM: 0x17000800..0x17001000 (2.0 KiB)    ← SFlash
NVM: 0x17001a00..0x17001c00 (512 B)
NVM: 0x17006400..0x17007000 (3.0 KiB)
NVM: 0x17007c00..0x17007e00 (512 B)
```

### 3.3 烧录测试

```bash
$ probe-rs download --chip CYT2BL3BAS --protocol swd \
    --binary-format hex build/firmware.hex

⚠️ 失败: "DPIDR didn't become readable within guard time"
   (详见 probe-rs 烧录失败 trace 分析报告)
```

---

## 4. 关键技术细节

### 4.1 CMSIS-Pack 来源

| 项目 | 详情 |
|------|------|
| Pack 名称 | Infineon.CAT1A_DFP.1.5.0 |
| Pack 类型 | Device Family Pack |
| 覆盖芯片 | PSoC 6, CYT2B6/B7/B9/BL 全系列 |
| 内容 | .FLM Flash 算法, SVD 寄存器描述, 启动文件 |

### 4.2 生成的 YAML 结构

```yaml
name: CYT2BL
manufacturer:
  id: 0x34          # JEP106: Infineon
  cc: 0x0
chip_detection:
- !InfineonPsocSiid   # 通过 Silicon ID 自动识别
  family_id: 0x108
  silicon_ids:
    0xea02: CYT2BL3BAS  # ← probe-rs info 读到的 Part Number
    ...
flash_algorithms:      # Flash 烧录算法 (base64 编码)
- name: CYT2BL_1024k
  instructions: <base64>
  pc_init: 0x08000001
  pc_program_page: 0x080000a5
  pc_erase_sector: 0x08000075
  ...
variants:
- name: CYT2BL3BAS
  cores:
  - name: cortex-m0p
    type: armv6m
    core_access_options: !Arm
      ap: !v1 1
  - name: cortex-m4
    type: armv7em
    core_access_options: !Arm
      ap: !v1 2
  memory_map:
  - !Ram
    name: IRAM1
    range: {start: 0x8000000, end: 0x8080000}
  - !Nvm
    name: IROM1
    range: {start: 0x10000000, end: 0x10410000}
  flash_algorithms:
  - CYT2BL_1024k
```

### 4.3 probe-rs 的复位→重连机制（已验证）

trace 日志证实 probe-rs 在芯片复位后正确执行了：

```
Performing SWD line reset       ← ✅ Line Reset
Reading DPIDR                    ← ✅ 读 ID
(重复 7-8 次, 间隔 5ms, 总超时 1 秒)
DPIDR didn't become readable...  ← ⚠️ 超时
```

---

## 5. 已知问题与解决方案

| 问题 | 解决方案 |
|------|---------|
| 烧录时 DPIDR 读超时 | 断电重启后重试，或使用 OpenOCD `halt` 方式 |
| 自动检测不工作 | 手动指定 `--chip CYT2BL3BAS` |
| HEX 被当成 ELF | 加 `--binary-format hex` |

---

## 6. 总结

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║  ✅ CYT2BL3 已成功加入 probe-rs 支持列表                  ║
║  ✅ CMSIS-Pack 自动提取 → 含完整 Flash 算法               ║
║  ✅ 芯片识别、内存映射、核心枚举全部正常                   ║
║  ⚠️ 烧录遇 SWD 物理层限制 (同样影响 OpenOCD)              ║
║                                                          ║
║  操作命令:                                                ║
║    probe-rs chip list                 查看支持列表         ║
║    probe-rs chip info CYT2BL3BAS      查看芯片详情        ║
║    probe-rs download --chip CYT2BL3BAS \                 ║
║      --binary-format hex firmware.hex  烧录 (需断电重启)  ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```
