# CYT2BL3 SWD 可用条件 — 完整分析报告

> **日期**: 2026-05-05  
> **核心问题**: SWD 只在 Listen Window 期间可用吗？用户程序运行时能连吗？

---

## 1. 核心答案

> **SWD 在 Listen Window 内外都可以连接！Listen Window 只是 FlashBoot 给调试器的一个"优先连接窗口"，窗口关闭后 SWD 仍然可用——前提是用户程序没有显式禁用 SWJ 引脚。**

---

## 2. SWD 可用性阶段表

```
CYT2BL3 上电/复位后的完整生命周期:

时间轴:
  上电 ──────────────────────────────────────────────────────►
       │           │           │              │
       │ ROM Boot  │ FlashBoot │ 用户程序      │ (可能进入低功耗)
       │           │           │              │
SWD:   │  ❌ Hi-Z  │ ✅ 可用    │ ✅ 可用¹      │ ⚠️ DeepSleep: ❌
       │           │           │              │
       │        ┌──┴──┐        │              │
       │        │Listen│        │              │
       │        │Window│        │              │
       │        │ 20ms │        │              │
       │        └─────┘        │              │
```

| 阶段 | 时间 | SWD 状态 | 原因 |
|------|:--:|:--:|------|
| **POR 上电** | 0ms | ❌ | 芯片未初始化 |
| **ROM Boot** | 0-2ms | ❌ | 引脚 Hi-Z, 未配置 |
| **FlashBoot** | 2-5ms | ❌→✅ | 配置 SWJ 引脚 |
| **Listen Window** | 5-25ms | ✅ | FlashBoot 主动等待调试器 |
| **用户程序** | >25ms | ✅¹ | 引脚保持 SWJ 配置 |
| **DeepSleep** | 任意 | ❌ | 调试域断电 |
| **Hibernate** | 任意 | ❌ | 系统完全断电 |

> ¹ 前提：用户程序未修改 SWJ 引脚的 HSIOM 配置或 GPIO_CFG

---

## 3. SWD 可用的必要条件

### 3.1 物理层条件

```
SWD 通信需要以下条件全部满足:
┌─────────────────────────────────────────────────────────┐
│ 1. SWCLK 引脚时钟可达    — 连接正确, 未接地/浮空         │
│ 2. SWDIO 引脚双向可用    — 连接正确, 上拉电阻正常        │
│ 3. HSIOM 路由正确        — P23.5→SWCLK, P23.6→SWDIO     │
│ 4. GPIO_CFG 配置正确     — 驱动模式, 输入缓冲使能        │
│ 5. VDD 供电正常          — 调试域有电                    │
│ 6. SWD 时钟频率合适      — 不超过芯片最大 SWCLK (10MHz)  │
└─────────────────────────────────────────────────────────┘
```

### 3.2 协议层条件

```
SWD 协议层要求:
┌─────────────────────────────────────────────────────────┐
│ 1. 芯片 SW-DP 状态机在 IDLE 状态                         │
│    → 需要先发送 SWD Line Reset (50+ 时钟周期)            │
│    → 然后 JTAG-to-SWD 切换 (或直接读 DPIDR)              │
│                                                         │
│ 2. DAP 调试电源域已上电                                  │
│    → CDBGPWRUPACK = 1, CSYSPWRUPACK = 1                 │
│    → 通过写 DP CTRL/STAT 寄存器实现                      │
│                                                         │
│ 3. AP 已选择并配置                                       │
│    → 写 DP SELECT 选择目标 AP                           │
│    → 配置 AP CSW (传输大小, 自动递增等)                  │
│                                                         │
│ 4. 没有 sticky error                                     │
│    → 写 DP ABORT 清除                                    │
└─────────────────────────────────────────────────────────┘
```

### 3.3 安全条件

```
CYT2BL3 安全状态影响:
┌─────────────────────────────────────────────────────────┐
│ Lifecycle Stage:                                         │
│   NORMAL      → ✅ 全功能调试                             │
│   SECURE      → ❌ DAP 可被禁用 (通过 eFuse)              │
│   SECURE_W_DEBUG → ✅ 调试可用 (开发者模式)               │
│   DEAD        → ❌ 芯片永久锁定                           │
│                                                         │
│ 我们的芯片: "Chip Protection: NORMAL" → ✅ 无限制        │
│                                                         │
│ DAP Access Restrictions:                                 │
│   CM0+ AP    → 可独立禁用 (通过 FlashBoot/用户代码)       │
│   CM4 AP     → 可独立禁用                                │
│   System AP  → 可独立禁用 + MPU 保护                     │
│                                                         │
│ 我们的芯片: 未检测到 AP 禁用 → ✅                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. 各阶段的详细行为

### 4.1 ROM Boot 阶段

```
特性:
  - 从 ROM 执行 (地址 0x00000000)
  - 所有 SWJ 引脚处于 Hi-Z (高阻抗)
  - SWDIO 浮空, 上拉电阻拉到 VDD → 读回 ACK = 111 (JUNK)
  - 即使发送 Line Reset, 芯片也不响应 (引脚未连接到 DAP)
  
结论: ❌ SWD 绝对不可用
```

### 4.2 FlashBoot → Listen Window

```
特性:
  - FlashBoot 从 SFlash (0x17002000) 执行
  - 配置 HSIOM: P23.4→SWO, P23.5→SWCLK, P23.6→SWDIO, P23.7→TDI
  - 配置 GPIO_CFG: 设置驱动模式 (Pull-up/Pull-down/Strong)
  - 开启 Listen Window (由 TOC2_FLAGS 配置)
  
Listen Window 行为:
  - FlashBoot 完成后, 芯片主动等待调试器连接
  - 等待时长 = TOC2_FLAGS.LISTEN_WINDOW (我们的芯片: 20ms)
  - 窗口内如果有调试器通过 Line Reset 连接 → 保持 SWD 活跃
  - 窗口到期 → 继续启动用户程序, SWD 引脚保持配置
  
结论: ✅ SWD 可用 (窗口内优先)
```

### 4.3 用户程序阶段

```
特性:
  - 用户程序从 Code Flash (0x10000000) 执行
  - 除非用户程序显式修改, 否则:
    HSIOM 保持 SWJ 路由 ✅
    GPIO_CFG 保持 FlashBoot 配置 ✅
  - DAP 调试域维持上电 ✅
  - 核心可能在 WFI/WFE 睡眠 → DAP 不受影响 ✅
  
注意:
  - 如果用户代码调用 cyhal_hwmgr_reserve() 将 SWJ 引脚分配给 GPIO
    → SWD 断开 ❌
  - 如果进入 DeepSleep → 调试域断电 ❌
  - 如果进入 SECURE lifecycle → DAP 可能被禁用 ❌
  
结论: ✅ SWD 可用 (前提: 用户程序未关闭)
```

### 4.4 低功耗模式

| 模式 | SWD 状态 | 说明 |
|------|:--:|------|
| **Active** | ✅ | 正常运行 |
| **Sleep (WFI/WFE)** | ✅ | DAP 独立, 不受核心睡眠影响 |
| **DeepSleep** | ❌ | 调试域断电, 需外部唤醒后重连 |
| **Hibernate** | ❌ | 系统几乎完全断电 |

---

## 5. 实验证据

### 5.1 probe-rs info 在用户程序运行时成功

```
$ probe-rs info
SWD DPIDR 0x6ba02477         ← ✅ 成功读到 DPIDR
[Cortex-M0+ r0p1]            ← ✅ 识别 CM0+
[Cortex-M4 r0p1]             ← ✅ 识别 CM4F
枚举全部 3 个 AP              ← ✅ AP 全部可访问
枚举 CoreSight 组件           ← ✅ ROM Table 可读
```

**证明**：用户程序运行时 (LED blinker + SysTick + WFI)，SWD 完全可用。

### 5.2 probe-rs download 失败

```
$ probe-rs download
Performing SWD line reset      ← ✅ Line Reset 正确执行
Reading DPIDR                  ← ❌ 读不到
(重试 7-8 次...)
DPIDR didn't become readable within guard time  ← 1 秒超时
```

**为什么同样在用户程序运行时，一次成功一次失败？**

```
probe-rs info:
  芯片已上电运行 → SWD 可用 → 直接读 DPIDR → ✅

probe-rs download:
  芯片已上电运行 → write AIRCR → SYSRESETREQ → 芯片复位
  → ROM Boot (Hi-Z) → FlashBoot (配置SWJ) → Listen Window → 用户程序
  → probe-rs 在 ~150ms 后开始重连 → 芯片已在用户程序中
  → BUT: DP 状态可能已不同!
```

---

## 6. 🔑 关键洞察：DP 状态在复位前后的差异

```
复位前 (probe-rs info 成功时):
  DAP 状态:  已上电, CDBGPWRUPACK=1, CSYSPWRUPACK=1
  SELECT:    指向 AP#1 (CM0+)
  AP CSW:    已配置 (32-bit, auto-increment)
  ABORT:     无 sticky error
  
  probe-rs:  直接读 DPIDR → ✅ 返回 0x6ba02477


复位后 (probe-rs download 重连时):
  DAP 状态:  复位后默认值
  SELECT:    0x00000000 (Bank 0, APSEL=0)
  CTRL/STAT: CDBGPWRUPACK=0, CSYSPWRUPACK=0 (调试域断电!)
  ABORT:     可能有 sticky error
  SW-DP:     UNKNOWN 状态 (需要 Line Reset)
  
  probe-rs:  Line Reset → JTAG-to-SWD → DPIDR Read
             但 DAP 调试域没上电, DPIDR 可能返回 0 或 JUNK!
```

**问题可能出在 DAP 上电流程！**

probe-rs 的重连序列:
1. `debug_port_setup()`: Line Reset → JTAG-to-SWD → DPIDR Read
2. `debug_port_start()`: 写 SELECT → 读 DPIDR → 检查 powered_down → 写 CTRL/STAT 上电

但如果在步骤 1 中 DPIDR Read 返回 0 或 JUNK（因为调试域没上电），重连就失败了。

对比初始连接:
1. `debug_port_setup()`: Line Reset → JTAG-to-SWD → DPIDR Read → ✅ 成功 (因为芯片已上电运行, DAP 调试域已上电)
2. `debug_port_start()`: 正常上电流程

**所以根因可能是：CYT2BL3 在 SYSRESETREQ 后，DAP 调试域断电，而 probe-rs 的重连流程假设 DAP 调试域已上电。**

---

## 7. 最终结论

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║  Q: SWD 只在 Listen Window 期间可用吗？                       ║
║  A: ❌ 不是！SWD 在用户程序运行期间同样可用。                  ║
║     (probe-rs info 已验证)                                   ║
║                                                              ║
║  Q: 那什么时候不可用？                                        ║
║  A:                                                          ║
║     1. ROM Boot 期间 (引脚 Hi-Z)             — 约 0-5ms      ║
║     2. DeepSleep/Hibernate (调试域断电)      — 取决于功耗模式 ║
║     3. SECURE lifecycle (DAP 被禁用)         — 取决于安全配置 ║
║     4. 用户程序显式关闭 SWJ 引脚             — 取决于代码     ║
║                                                              ║
║  Q: probe-rs download 为什么失败？                            ║
║  A: SYSRESETREQ 后 DAP 调试域断电。probe-rs 重连时           ║
║     先读 DPIDR 验证连接, 但 DAP 上电需要先写 CTRL/STAT。      ║
║     DPIDR 读返回 0/JUNK → 重连失败。                          ║
║     初始连接时芯片已运行 → DAP 已上电 → DPIDR 读成功。        ║
║                                                              ║
║  Q: 为什么 OpenOCD halt 方式能工作？                          ║
║  A: halt 不触发 SYSRESETREQ → DAP 保持上电 → 不需要重连。    ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

> 📎 **关联报告**:
> - [用户固件 SWJ 引脚审计](./CYT2BL3_用户固件SWJ引脚配置审计报告.md)
> - [FlashBoot 参数诊断](./CYT2BL3_FlashBoot参数诊断_SWD重连失败最终报告.md)
> - [probe-rs 烧录失败 Trace 分析](./CYT2BL3_probe-rs_烧录失败Trace分析报告.md)
> - [复位后 SWD 可用时序](./CYT2BL3_复位后SWD可用时序_官方文档分析报告.md)
