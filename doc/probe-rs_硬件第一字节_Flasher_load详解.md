# probe-rs 硬件第一字节 — `initialize()` 的真相 & `Flasher::load()` 是谁

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：澄清 `initialize()` 的真实职责，追踪第一个 SWD/JTAG 字节的发送位置  
> 📖 上下文：补充第三篇 "FlashRequest 完整执行追踪" 的关键细节

---

## 一、一句话总结

`loader::initialize()` **不碰硬件**，只算进度条。第一个 SWD/JTAG 字节在 `Flasher::load()` 里的 `core.write(algo.load_address, algo.instructions)` 发出——把 Flash 算法二进制码写入芯片 RAM。

---

## 二、`loader::initialize()` 的真实面目

**文件**: `probe-rs/src/flashing/loader.rs:910`

这个函数虽然叫 "initialize"，但全程没有 `core.read()` / `core.write()` / `core.run()`：

```rust
fn initialize(&self, algos: &mut [Flasher], session: &mut Session,
              options: &mut DownloadOptions) -> Result<(), FlashError> {

    // ① 检查芯片是否支持整片擦除
    if options.do_chip_erase && !flasher.is_chip_erase_supported(session) {
        options.do_chip_erase = false;
    }

    // ② 遍历所有区域，计算各操作总字节数
    for flasher in algos.iter_mut() {
        fill_size += ...;     // 填充字节数
        erase_size += ...;    // 擦除字节数
        program_size += ...;  // 编程字节数
    }

    // ③ 给每个操作创建一个进度条
    options.progress.add_progress_bar(Fill, Some(fill_size));
    options.progress.add_progress_bar(Erase, Some(erase_size));
    options.progress.add_progress_bar(Program, Some(program_size));

    // ④ 发送 ProgressEvent::FlashLayoutReady
    options.progress.initialized(phases);

    Ok(())
}
```

**它的真名应该叫 `calculate_progress_bars()`** — 只是一个 UI 预告。

---

## 三、真正的硬件第一字节：`Flasher::load()`

**文件**: `probe-rs/src/flashing/flasher.rs:192`

```rust
fn load(&mut self, session: &mut Session) -> Result<(), FlashError> {
    let algo = &self.flash_algorithm;
    let mut core = session.core(self.core_index)?;

    // ① 如果芯片还没 halt → halt 它
    //    🔥 SWD 读 DHCSR / RISC-V DMSTATUS / Xtensa XDM → 第一个读操作！
    if !core.core_halted().unwrap_or(true) {
        core.reset_and_halt(Duration::from_millis(500))?;
    }

    // ② 🔥🔥🔥 第一个写操作！把 Flash 算法二进制码写到芯片 RAM
    //     → MemoryInterface::write()
    //       → ArmCommunicationInterface / RiscvCommunicationInterface
    //         → DAP / DM 协议
    //           → SWD / JTAG 引脚电平变化！
    core.write(algo.load_address, algo.instructions.as_bytes())?;

    // ③ 读回来逐字节校验
    let mut data = vec![0; algo.instructions.len()];
    core.read(algo.load_address, data.as_mut_bytes())?;
    for (i, (orig, read)) in algo.instructions.iter().zip(data.iter()).enumerate() {
        if orig != read {
            // 校验失败 → 数据没写对
            return Err(...);
        }
    }

    self.loaded = true;  // 标记已加载，后续跳过
}
```

### 然后是 Init 函数

`Flasher::init()` → `ActiveFlasher::init()` 让芯片执行 `Init()`：

```rust
// flasher.rs:837
fn init(&mut self, clock: Option<u32>) -> Result<(), FlashError> {
    let Some(pc_init) = algo.pc_init else { return Ok(()); }; // 没有就跳过

    self.call_function_and_wait(&Registers {
        pc: pc_init,                // Init 函数入口
        r0: Some(address),          // Flash 起始地址
        r1: clock.or(Some(0)),      // 时钟
        r2: Some(O::OPERATION),     // 1=Erase, 2=Program, 3=Verify
        r3: None,
    }, true, INIT_TIMEOUT)?;
}
```

---

## 四、`ensure_loaded` — 懒加载

```rust
// flasher.rs:175
fn ensure_loaded(&mut self, session: &mut Session) -> Result<(), FlashError> {
    if !self.loaded {
        self.load(session)?;   // ← 只执行一次！
        self.loaded = true;
    }
    Ok(())
}
```

每次擦除/编程前调 `ensure_loaded()`，但 `loaded` 标记保证算法只加载一次。

---

## 五、完整时间线

```
loader::commit()
│
├─ prepare_plan()           ← 🧠 纯内存
├─ initialize()             ← 🧠 纯内存（算进度条）
│
└─ for flasher in algos:
     │
     ├─ run_erase_all() 
     │    └─ ensure_loaded() → Flasher::load()  ← 🔥🔥🔥 第一个 SWD/JTAG 字节！
     │         ├─ core.reset_and_halt()
     │         ├─ core.write(load_addr, instructions)  ← 🔥 写算法到 RAM
     │         └─ core.read() 校验
     │
     ├─ program()
     │    ├─ ensure_loaded()   loaded=true，跳过
     │    ├─ init() → ActiveFlasher::init()
     │    │    └─ call_function_and_wait(pc_init)
     │    │         → core.run() → 芯片执行 Init()
     │    │
     │    ├─ fill_unwritten()
     │    ├─ sector_erase() → call_function_and_wait(pc_erase_sector)
     │    ├─ do_program()   → call_function_and_wait(pc_program_page)
     │    └─ verify()       → call_function_and_wait(pc_verify)
     │
     └─ commit_ram()
```

### 各架构的传输路径

```
core.write(addr, data)
  → MemoryInterface::write()
    ├─ ARM:  ArmCommunicationInterface → DAP → DP → AP → MemoryAP → SWD/JTAG
    ├─ RISC-V: RiscvCommunicationInterface → DM → abstract command → JTAG
    └─ Xtensa: XtensaCommunicationInterface → XDM → JTAG
```

---

## 六、补充：双缓冲加速 & 多 Flasher 循环

### 6.1 双缓冲 vs 单缓冲

```rust
// loader.rs — commit() 中
let mut do_use_double_buffering = flasher.double_buffering_supported();
//     ↑ page_buffers.len() > 1 → 芯片 RAM 放得下两个 buffer
if do_use_double_buffering && options.disable_double_buffering {
    do_use_double_buffering = false;  // 用户强制关
}
```

**单缓冲 (`program_simple`)**：串行
```
page1: [SWD传数据→RAM] [CPU烧Flash] [等]
page2:                  [SWD传数据]  [CPU烧Flash] [等]
                        ↑ 空闲等待！
```

**双缓冲 (`program_double_buffer`)**：并行
```
buf0: [SWD传page1] [CPU烧page1→等]        [SWD传page3] [CPU烧page3→等]
buf1:              [SWD传page2→等] [CPU烧page2]         [SWD传page4→等]
       ← SWD传输 与 CPU烧录 重叠！速度翻倍！→
```

前提：Flash 算法 YAML 定义了两个 `page_buffers`。用户可用 `--disable-double-buffering` 强制关闭。

### 6.2 `for mut flasher in algos` — 为什么可能多个？

`algos` 来自 `prepare_plan()`：每个不同的 Flash 算法一个 Flasher。

| 情况 | algos 数量 | 例子 |
|------|-----------|------|
| 只烧 Main Flash | 1 | 你的 CYT2BL3 |
| Main + Work Flash 都有数据 | 2 | CYT2BL3 用到了 Work Flash |
| 多 Bank + 多 Flash 类型 | N | ESP32, 某些 STM32 |

```rust
for mut flasher in algos {
    // 每个 Flasher 处理它负责的区域:
    flasher.run_erase_all()?;   // 整片擦除（只做一次）
    flasher.program()?;         // 逐页烧录
}
// → 按算法分组，一组一组烧
```

---

## 七、补充：烧录循环前的"五大件"准备工作

到 `for mut flasher in algos` 时，已经准备好了：

```
❶ Session   — 探针+芯片已连接，已 halt
❷ Target    — 芯片"说明书"已加载
❸ FlashLoader — ELF 数据段已就位 (builder.data)
❹ Flasher[] — 算法已加载到 RAM 且 Init 完成
❺ Progress  — 进度条已配好
```

### 7.1 五大件清单

| # | 准备好了什么 | 怎么来的 | 芯片差异 |
|---|-------------|---------|---------|
| ❶ | USB 探针已打开 | `lister.open()` | CMSIS-DAP/STLink/JLink/EspUsbJtag |
| ❶ | SWD/JTAG 协议已选 | `probe.select_protocol()` | 默认 SWD，ESP32 仅 JTAG |
| ❶ | 芯片已识别 (IDCODE) | SWD 读 DPIDR / JTAG 扫 IDCODE | 每种芯片 ID 不同 |
| ❶ | 芯片已 halt | `core.reset_and_halt()` | ARM 写 DHCSR / RISC-V 写 DM / Xtensa 写 XDM |
| ❷ | 内存布局 `memory_map` | targets.bincode → `Target::new()` | 每种芯片不同 |
| ❷ | Flash 算法列表 | YAML → `flash_algorithms` | 每种芯片算法不同 |
| ❷ | 调试序列 | `Cyt2bl::create()` 等 | ARM/RISC-V/Xtensa 完全不同 |
| ❸ | `builder.data` 地址→数据 | `extract_from_elf()` → `add_data()` | ESP32 多一步 IdfLoader 合并 |
| ❸ | 地址已验证 | `check_data_in_memory_map()` | 不同芯片地址范围不同 |
| ❹ | 算法二进制码已写 RAM | `core.write(load_addr, instructions)` | ARM=Thumb2, RISC-V=RV32, Xtensa=Xtensa |
| ❹ | Init() 已执行 | `call_function_and_wait(pc_init)` | Init 代码芯片厂商写，每个不同 |
| ❺ | 进度条已创建 | `add_progress_bar(Fill/Erase/Program/Verify)` | 无差异 |
| ❺ | `initialized` 事件已发 | UI 收到 "准备开始" | 无差异 |

### 7.2 芯片差异大吗？

到这个循环时，**差异已被抽象层消化**：

| 差异层 | 统一层 |
|--------|--------|
| 芯片不同 → 不同 YAML | 都变成 `Vec<MemoryRegion>` |
| 核心不同 → 不同寄存器 | 都实现 `CoreInterface` trait |
| 算法不同 → 不同指令集 | 都通过 `call_function_and_wait()` 调用 |
| 格式不同 → IDF vs ELF | 都变成 `builder.data` |
| 探针不同 → 不同 USB 协议 | 都实现 `DebugProbe` trait |

→ `for mut flasher in algos { flasher.program(...) }` **所有芯片都一样**！

### 7.3 循环前做了什么的完整清单

```
✓ 探针已连接 + 协议已选 + 速度已设
✓ 芯片已识别 + halted
✓ 内存布局已加载 (哪里是 Flash, 哪里是 RAM)
✓ Flash 算法已加载到芯片 RAM
✓ Init() 已执行 (芯片 Flash 控制器已初始化)
✓ ELF 数据段已解析并验证 (地址都在有效范围内)
✓ 算法已匹配到区域 (哪个算法管哪个 Flash 区)
✓ 进度条已配好 (知道要烧多少字节)
```

此时就像厨房备好菜、锅已热好——**只剩 "炒" 这个动作了！**

---

## 八、源码位置

| 内容 | 文件 | 行号 |
|------|------|------|
| `initialize()` (算进度条) | `probe-rs/src/flashing/loader.rs` | 910 |
| `Flasher::load()` (写算法到 RAM) | `probe-rs/src/flashing/flasher.rs` | 192 |
| `core.write()` — 第一个写操作 | `probe-rs/src/flashing/flasher.rs` | 221 |
| `core.read()` — 校验 | `probe-rs/src/flashing/flasher.rs` | 225 |
| `ensure_loaded()` (懒加载) | `probe-rs/src/flashing/flasher.rs` | 175 |
| `Flasher::init()` | `probe-rs/src/flashing/flasher.rs` | 280 |
| `ActiveFlasher::init()` | `probe-rs/src/flashing/flasher.rs` | 837 |
| `core.reset_and_halt()` | `probe-rs/src/flashing/flasher.rs` | 212 |
| `commit()` 主循环 | `probe-rs/src/flashing/loader.rs` | 689-722 |
