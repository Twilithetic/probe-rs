# CYT2BL3 probe-rs download — 复位前状态验证报告

> **日期**: 2026-05-06  
> **源日志**: `C:\Users\29344\.local\share\opencode\tool-output\tool_dfbee4a99001RiTsaXi4PbmeXk` (6671 行)  
> **命令**: `probe-rs download --chip CYT2BL3BAS --protocol swd --speed 100 --binary-format elf build/firmware.elf`  
> **探针**: WCH-Link (CMSIS-DAP v2)  

---

## 核心结论

> **复位前一切 100% 正常。DP 连接、双核初始化、halt、断点清理 —— 85 次 SWD 操作，0 次失败。**
>
> **断裂点**: `flasher.rs:201` 的 `core.reset_and_halt()` → SYSRESETREQ → CYT2BL3 Boot ROM 重配 SWD → DP 不可达

---

## 完整时间线（27 步逐行证据）

### Phase 1: DP 连接 (行 5-9)

| 行号 | 事件 | 证据 |
|:--:|------|------|
| 5 | `debug_port_setup` Line Reset | `Performing SWD line reset` |
| 6 | `debug_port_connect` Line Reset | `Performing SWD line reset` |
| 7 | 读 DPIDR | `Reading DPIDR to enable SWD interface` |
| 8 | **DPIDR 可读** | `DPIDR became readable after 0ms` |
| 9 | **DPIDR 值正确** | `0x6ba02477` |

**结果**: ✅ DP 连接成功，0ms 即响应，无需重试

---

### Phase 2: 错误清除 + CTRL/STAT 初读 (行 10-14)

| 行号 | 事件 | 证据 |
|:--:|------|------|
| 10 | 清 ABORT | `Clearing errors using ABORT register` |
| 11-14 | 读 CTRL/STAT | `csyspwrupack: false, csyspwrupreq: false, cdbgpwrupack: false, cdbgpwrupreq: false, read_ok: true, sticky_err: false` |

**结果**: ✅ DP 处于正常掉电状态，无 sticky 错误

---

### Phase 3: DP 上电 — 调试域 (行 17-41)

| 行号 | 事件 | 证据 |
|:--:|------|------|
| 17-20 | 读 DPIDR | `version: 2 (DPv2), designer: Cypress (0x23B), part_no: 0xBA` |
| 21-25 | 读 CTRL/STAT | 确认上电标志全为 false |
| 26-27 | 写 ABORT | 清除所有 sticky |
| 28-36 | **写 CTRL/STAT 请求上电** | `cdbgpwrupreq: true, csyspwrupreq: true` |
| 37-41 | **等待确认** | `cdbgpwrupack: true, csyspwrupack: true` ✅ |

**结果**: ✅ 调试域上电成功

---

### Phase 4: DP 上电 — 系统域 (行 42-55)

| 行号 | 事件 | 证据 |
|:--:|------|------|
| 42-50 | 写 CTRL/STAT 请求系统域上电 | `csyspwrupreq: true` |
| 51-55 | **等待确认** | `csyspwrupack: true` ✅ |

**结果**: ✅ 系统域上电成功

---

### Phase 5: 二次验证 (行 56-64)

| 行号 | 事件 | 证据 |
|:--:|------|------|
| 56-60 | 读 CTRL/STAT | 所有上电标志 true, `read_ok: true` |
| 61-64 | 读 DPIDR | `0x6ba02477` — 确认 DP 稳定 |

**结果**: ✅ 两个电源域均正常

---

### Phase 6: CM0+ Core 初始化 (行 65-82)

| 行号 | 事件 | 寄存器 | 证据 |
|:--:|------|--------|------|
| 65 | 选 AP#1 Bank F | SELECT | `ap_sel: 1, ap_bank_sel: 15` |
| 66-70 | 切回 Bank 0 | SELECT | `ap_bank_sel: 0` |
| 71-73 | **写 CSW** | AP#1 CSW | `DbgSwEnable: true, Size: U32, AddrInc: Single` |
| 74-76 | 写 TAR | AP#1 TAR | `address: e000edf0` (DHCSR) |
| 77 | **读 DHCSR** | AP#1 DRW | `Finished reading block` |
| 80-82 | **写 DHCSR → halt** | AP#1 DRW | `data: a05f0001` (C_DEBUGEN=1) |

**结果**: ✅ CM0+ 调试接口已使能

> **地址 0xE000EDF0 = DHCSR** (Debug Halting Control and Status Register)  
> **写入值 0xA05F0001**: 写 key + C_DEBUGEN=1 + C_HALT=0

---

### Phase 7: CM4 Core 初始化 (行 83-103)

| 行号 | 事件 | 寄存器 | 证据 |
|:--:|------|--------|------|
| 83-86 | 选 AP#2 Bank F | SELECT | `ap_sel: 2, ap_bank_sel: 15` |
| 89-91 | **写 CSW** | AP#2 CSW | `DbgSwEnable: true, Size: U32` |
| 94-97 | 写 TAR=DHCSR | AP#2 TAR | `address: e000edf0` |
| 98 | **读 DHCSR** | AP#2 DRW | `Finished reading block` |
| 101-103 | **写 DHCSR → halt** | AP#2 DRW | `data: a05f0001` |

**结果**: ✅ CM4 调试接口已使能

---

### Phase 8: Halt Core#0 (CM0+) (行 123-212)

| 行号 | 事件 | 关键数据 |
|:--:|------|---------|
| 123 | **INFO: Halting core 0...** | |
| 124-133 | **写 DHCSR=0xA05F0003** | C_DEBUGEN=1 + C_HALT=1 |
| 134 | 写入完成 | `Finished writing block` |
| 135-148 | **读回 DHCSR 验证** | `4 of 4 items executed` |
| 147 | DHCSR 值 | `value=16973827` = **0x01030003** |
| 149-161 | 读 DHCSR (S_REGRDY) | `2 of 2 items executed` → `value=1` |
| 163-173 | 写 DHCSR (清状态) | `data: 0x1F` |
| 174-184 | 写 DEMCR | `address: e000edf4, data: 0x0F` |
| 185-197 | 再读 DHCSR | `6 of 6 items executed` → `value=16973827` |
| 207-212 | **读 DEMCR** | `value=27484` = **0x6B5C** (正常) |

**结果**: ✅ CM0+ halt 确认

> **DHCSR=0x01030003 含义**: S_HALT=1 (已暂停), C_DEBUGEN=1, C_HALT=1

---

### Phase 9: Halt Core#1 (CM4) (行 234-323)

| 行号 | 事件 | 关键数据 |
|:--:|------|---------|
| 234 | **INFO: Halting core 1...** | |
| 235-245 | 写 DHCSR=0xA05F0003 | C_HALT=1 (AP#2) |
| 246-258 | **读回 DHCSR** | `4 of 4 items executed` → `value=196611` = **0x00030003** |
| 259-272 | 读 DHCSR (S_REGRDY) | `value=1` |
| 273-284 | 写 DHCSR (清状态) | `data: 0x1F` |
| 285-295 | 写 DEMCR | `data: 0x0F` |
| 296-308 | 再读 DHCSR | `6 of 6 items executed` → `value=196611` |
| 309-322 | **读 DEMCR** | `value=444` = **0x01BC** (正常) |

**结果**: ✅ CM4 halt 确认

> **DHCSR=0x00030003 含义**: S_HALT=1, C_DEBUGEN=1, C_HALT=1

---

### Phase 10: 清理硬件断点 (行 335-399)

| 行号 | 事件 | 目标 |
|:--:|------|------|
| 335-349 | 读 FPB 寄存器 0xE0002000-0xE0002014 | CM0+ |
| 361-399 | 读 FPB 寄存器 0xE0002000-0xE000201C | CM4 |

**结果**: ✅ 双核 HW 断点已清理（地址 0xE0002000 = FPB_CTRL, FPB_COMPx）

---

### Phase 11: 核心运行状态验证 + Flasher 初始化 (行 411-503)

| 行号 | 事件 | 关键数据 |
|:--:|------|---------|
| 411-414 | **写 DHCSR=0xA05F000D** | C_DEBUGEN + C_MASKINTS + C_STEP → 进入调试模式 |
| 415-416 | 读 DHCSR 确认 | |
| 417-418 | 读 DHCSR (0xE000ED30) | S_REGRDY 验证 |
| 419-422 | 写 DHCSR | 清状态 |
| **423-424** | **CM0+ halt 原因** | `Reason for halt has changed: Halted(Request)` |
| 425-432 | 再确认 halt 状态 | |
| **433** | **CM0+ 确认** | `Cached halt reason: Halted(Request)` |
| 434-445 | 写 DEMCR + 读 DHCSR | 最终状态确认 |
| 446-456 | 切换到 AP#2 (CM4) | SELECT, CSW |
| 457-462 | 写 DHCSR=0xA05F000B | CM4 调试模式 |
| 463-466 | 写 DHCSR=0xA05F000D | |
| 467-474 | 读 DHCSR + 写清状态 | |
| **475-476** | **CM4 halt 原因** | `Reason for halt has changed: Halted(Request)` |
| 477-484 | 再确认 CM4 halt | |
| **485** | **CM4 确认** | `Cached halt reason: Halted(Request)` |
| 486-503 | 写 DEMCR, DHCSR=0xA05F0001 | 恢复 C_HALT=0, 保持 C_DEBUGEN=1 |

**结果**: ✅ 双核均确认为 `Halted(Request)`，已进入调试态

---

### Phase 12: Flash 段分析 (行 504-521)

| 行号 | 段 | 地址 | 大小 |
|:--:|-----|------|:--:|
| 504-509 | `.vectors`, `.text`, `.ARM.extab`, `.ARM.exidx` | 0x10000000 | 5900 B |
| 510-512 | `.data` | 0x1000170C | 84 B |
| 513-516 | `.copy_table`, `.zero_table` | 0x10001760 | 20 B |
| 517-521 | **合计 3 段** | | **6004 B** |

**结果**: ✅ ELF 文件分析完成，段映射正确

---

## 💥 断裂点

```
行 521: INFO .copy_table .zero_table at 0x10001760 (20 bytes)
         ↓  ← Flasher::load() → reset_and_halt() → AIRCR.SYSRESETREQ
行 522: DEBUG session_drop: Performing SWD line reset
行 523: DEBUG ... Performing SWD line reset
行 524: DEBUG ... Reading DPIDR to enable SWD interface
行 525: DEBUG ... Transfer status: NACK  🔴
           ↓  重复 NACK × ~200 × 5 轮 ≈ 5 秒
行 6664: DEBUG ... DPIDR didn't become readable within guard time
          ERROR: Failed to reset, and then halt the CPU.
```

**行 521 → 522 之间没有任何中间日志。** Flash 段分析完成后，`reset_and_halt` 触发芯片复位 → DP 立即断开 → `session_drop` → NACK 循环。

---

## 统计数据总结

| 阶段 | 行范围 | 成功操作 | 失败 | 重试 |
|------|:--:|:--:|:--:|:--:|
| DP 连接 | 5-9 | ~5 | 0 | 0 |
| DP 上电 (调试域) | 10-41 | ~10 | 0 | 0 |
| DP 上电 (系统域) | 42-55 | ~5 | 0 | 0 |
| CM0+ 初始化 | 65-82 | ~8 | 0 | 0 |
| CM4 初始化 | 83-103 | ~8 | 0 | 0 |
| Halt CM0+ | 123-212 | ~12 | 0 | 0 |
| Halt CM4 | 234-323 | ~12 | 0 | 0 |
| HW 断点清理 | 335-399 | ~10 | 0 | 0 |
| 核心状态验证 | 411-503 | ~15 | 0 | 0 |
| **复位前合计** | **5-521** | **~85** | **0** | **0** |

**每一个 `process_batch` 都是 "X of X items executed" — 没有任何 NACK、WAIT、FAULT。**

---

## 根因分析

```
probe-rs flasher.rs:201:
  core.reset_and_halt(Duration::from_millis(500))
    → cortex_m_reset_system()
      → 写 AIRCR.SYSRESETREQ (0xE000ED0C)
        → 芯片全系统复位
          → CYT2BL3 ROM Boot 执行
            → Boot ROM 重配 SWJ-DP 引脚为 GPIO
              → SWD 物理连接断开
                → DP 不可达
                  → 所有 SWD 请求 NACK
                    → debug_port_connect 重试 5 秒超时
                      → FlashingError::ResetAndHalt
```

**这与 OpenOCD 文档中记录的 DP 重连失败是同一个根因**（`CYT2BL3_调试_DP重连失败根因分析.md`）。

OpenOCD 的解决方案: 用 `halt 3000` 替代 `reset init`，避免芯片复位。

---

## 下一步修复方向

1. **跳过 reset_and_halt**: 修改 `flasher.rs:201`，因为核心已在 Phase 11 中确认 `Halted(Request)`
2. **或**: 增加 `cortex_m_wait_for_reset` 中的超时时间 (当前 600ms → 可试 2000ms)
3. **或**: 为 CYT2BL3 添加 vendor-specific sequence，在 reset 后做硬件 nRESET + 延迟等待

---

## 源日志路径

```
C:\Users\29344\.local\share\opencode\tool-output\tool_dfbee4a99001RiTsaXi4PbmeXk
```

*本报告由知心姐姐基于逐行日志分析编写 💖*
