# probe-rs `prepare_plan()` — Flash 算法与内存区域的匹配机制

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：`loader.commit()` 中 `prepare_plan()` 的完整逻辑  
> 📖 上下文：位于 `commit()` 执行的第一步，在 `initialize()` 和 `program()` 之前

---

## 一、一句话总结

`prepare_plan()` 是 **"烧录作战计划制定者"**：把 ELF 数据段、芯片 Flash 区域、可用的 Flash 算法三者配对，生成一个 `Vec<Flasher>`——每个 `Flasher` 负责一到多个 Flash 区域，用同一个算法烧录。

---

## 二、输入 → 处理 → 输出

```
输入                               处理                    输出
┌──────────────┐              ┌────────────┐          ┌───────────┐
│ ① ELF 数据段  │              │  地址范围   │          │ Flasher 1 │
│  builder.data │ ──────────→ │  匹配检查   │ ──────→ │ (cyt2bl)  │
│  (addr→bytes) │              │            │          │ + 数据段  │
│               │              │  核心归属   │          │           │
│ ② 芯片内存图  │              │  匹配检查   │          │ Flasher 2 │
│  memory_map   │              │            │          │ (wflash)  │
│  NvmRegions   │              │  算法选择   │          │ + 数据段  │
│               │              │  策略:      │          └───────────┘
│ ③ Flash 算法库│              │  1. 唯一    │
│  target.      │              │  2. default │
│  flash_       │              │  3. preferred│
│  algorithms   │              └────────────┘
└──────────────┘
```

---

## 三、源码逐行解析

**文件**: `probe-rs/src/flashing/loader.rs:808`

```rust
fn prepare_plan(
    &self,
    session: &mut Session,          // 芯片会话（包含 target 信息）
    restore_unwritten_bytes: bool,  // 是否保护不需写的字节
    opt_preferred_algos: &[String], // 用户命令行指定的优选算法
) -> Result<Vec<Flasher>, FlashError> {

    // ========== 0️⃣ 打印调试信息 ==========
    tracing::debug!("Contents of builder:");
    for (&address, data) in &self.builder.data {
        tracing::debug!(
            "    data: {:#010X}..{:#010X} ({} bytes)",
            address, address + data.len() as u64, data.len()
        );
    }
    // 打印所有可用的 Flash 算法
    tracing::debug!("Flash algorithms:");
    for algorithm in &session.target().flash_algorithms {
        let Range { start, end } = algorithm.flash_properties.address_range;
        tracing::debug!(
            "    algo {}: {:#010X}..{:#010X} ({} bytes)",
            algorithm.name, start, end, end - start
        );
    }

    // ========== 1️⃣ 检查内存图一致性 ==========
    if self.memory_map != session.target().memory_map {
        tracing::warn!("Memory map of flash loader does not match memory map of target!");
    }

    let mut algos = Vec::<Flasher>::new();

    // ========== 2️⃣ 遍历每个 NVM（Flash）区域 ==========
    for region in self.memory_map.iter()
        .filter_map(MemoryRegion::as_nvm_region)   // 只看 Flash，跳过 RAM
    {
        // ②-a: 这个区域没数据要写？跳过！
        //       比如 Work Flash 没用到，就不加载它的算法
        if !self.builder.has_data_in_range(&region.range) {
            tracing::debug!("     -- empty, ignoring!");
            continue;
        }

        let region = region.clone();

        // ②-b: 找到这个区域归哪个 CPU 核心管
        let Some(core_name) = region.cores.first() else {
            return Err(FlashError::NoNvmCoreAccess(region));
        };
        let target = session.target();
        let core = target.core_index_by_name(core_name).unwrap();

        // ②-c: 🔑 选择 Flash 算法！
        let algo = Self::get_flash_algorithm_for_region(
            &region, target, core_name, opt_preferred_algos,
        )?;

        tracing::debug!("     -- using algorithm: {}", algo.name);

        // ②-d: 复用 or 新建 Flasher
        //       如果两个区域用同一个算法 → 复用 Flasher（只加载一次算法到 RAM）
        if let Some(entry) = algos.iter_mut()
            .find(|e| e.flash_algorithm.name == algo.name && e.core_index == core)
        {
            // 追加区域到已有的 Flasher
            entry.add_region(region, &self.builder, restore_unwritten_bytes)?;
        } else {
            // 新建 Flasher
            let mut flasher = Flasher::new(target, core, algo)?;
            flasher.add_region(region, &self.builder, restore_unwritten_bytes)?;
            flasher.read_rtt_output(self.read_flasher_rtt);
            algos.push(flasher);
        }
    }

    Ok(algos)
}
```

---

## 四、算法选择逻辑：`get_flash_algorithm_for_region`

**文件**: `probe-rs/src/flashing/loader.rs:1024`

```rust
pub(crate) fn get_flash_algorithm_for_region<'a>(
    region: &NvmRegion,
    target: &'a Target,
    core_name: &String,
    preferred_algos: &[String],
) -> Result<&'a RawFlashAlgorithm, FlashError> {
    // ① 从芯片 target 定义中筛选匹配的算法
    let algorithms = target.flash_algorithms.iter()
        // 条件A: 算法覆盖的地址范围必须包含这个 Flash 区域
        .filter(|fa| fa.flash_properties.address_range.contains_range(&region.range))
        // 条件B: 算法支持这个核心（或不受核心限制）
        .filter(|fa| fa.cores.is_empty() || fa.cores.contains(core_name))
        .collect::<Vec<_>>();

    match algorithms.len() {
        // 情况1: 没匹配到 → 报错
        0 => Err(FlashError::NoFlashLoaderAlgorithmAttached {
            range: region.range.clone(),
            name: target.name.clone(),
        }),

        // 情况2: 恰好一个 → 直接用
        1 => Ok(algorithms[0]),

        // 情况3: 多个算法都匹配 → 需要进一步选择
        _ => {
            // ③-a: 用户有指定 preferred_algos？
            if !preferred_algos.is_empty() {
                // 在匹配的算法中找用户指定的
                let preferred = algorithms.iter()
                    .filter(|a| preferred_algos.contains(&a.name))
                    .collect::<Vec<_>>();

                match preferred.len() {
                    0 => {} // 用户指定的都不匹配，继续走默认逻辑
                    1 => return Ok(preferred[0]), // 就用用户指定的！
                    _ => return Err(FlashError::MultiplePreferredAlgos { ... }),
                }
            }

            // ③-b: 没有用户指定 → 找标记为 default 的
            let defaults = algorithms.iter()
                .filter(|fa| fa.default)
                .collect::<Vec<_>>();

            match defaults.len() {
                0 => Err(FlashError::MultipleFlashLoaderAlgorithmsNoDefault { ... }),
                1 => Ok(defaults[0]),
                _ => Err(FlashError::MultipleDefaultFlashLoaderAlgorithms { ... }),
            }
        }
    }
}
```

### 选择优先级

```
多个算法都能烧这个区域时:

  ① preferred_algos（用户显式指定）    ← 最高优先级
       ↓ 没指定或匹配不到
  ② default = true 的算法              ← target 定义里的默认值
       ↓ 没有或冲突
  ③ 报错！                            ← 不知道用哪个
```

---

## 五、以 CYT2BL3 为例

CYT2BL3 的典型 Flash 布局：

```
CYT2BL3 内存布局
┌─────────────────────────────────────┐
│ Flash 区域                           │
│                                     │
│ ┌─ NvmRegion ─────────────────────┐ │
│ │  Main Flash                     │ │
│ │  地址: 0x1000_0000 ~ 0x101F_FFFF │ │
│ │  大小: 2MB                      │ │
│ │  算法: cyt2bl                   │ │
│ │  核心: cm4                      │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─ NvmRegion ─────────────────────┐ │
│ │  Work Flash                     │ │
│ │  地址: 0x1400_0000 ~ 0x1401_FFFF │ │
│ │  大小: 128KB                    │ │
│ │  算法: cyt2bx_wflash_128        │ │
│ │  核心: cm4                      │ │
│ └─────────────────────────────────┘ │
│                                     │
│ RAM 区域 (跳过，不走 prepare_plan)     │
│ ┌─ RamRegion ─────────────────────┐ │
│ │  SRAM: 0x0800_0000 ~ 0x0807_FFFF │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**执行 trace**:

```
遍历区域 1: Main Flash (0x1000_0000)
  has_data_in_range? → ✅ YES (.text/.rodata 在这里)
  get_flash_algorithm_for_region →
    匹配算法: cyt2bl ✓ (覆盖 0x10000000~0x101FFFFF)
    只匹配到一个 → 直接用
  新建 Flasher(cyt2bl) → push

遍历区域 2: Work Flash (0x1400_0000)
  has_data_in_range? → ❌ NO (你没用 Work Flash)
  跳过!

遍历区域 3~N: RAM 区域
  filter_map(as_nvm_region) → 过滤掉，不走这里

结果: algos = [Flasher { algorithm: "cyt2bl", regions: [MainFlash] }]
```

---

## 六、为什么叫"Plan"而不直接烧？

| prepare_plan 做什么 | prepare_plan 不做什么 |
|---------------------|----------------------|
| ✅ 匹配 区域 ↔ 算法 | ❌ 不加载算法到 RAM |
| ✅ 分组（复用算法） | ❌ 不接触硬件 |
| ✅ 整理数据段到 Flasher | ❌ 不擦除/编程 |
| ✅ 跳过空区域 | ❌ 不发 ProgressEvent |
| ✅ 选择策略（default/preferred） | ❌ 不操作 SWD |

**真正的烧录在后面**：

```
prepare_plan()     → 制定计划
    ↓
initialize()       → 把算法加载到芯片 RAM，设好 SP/PC
    ↓
for flasher in algos:
    program()      → 擦除 → 编程 → 校验
```

---

## 七、复用算法机制

如果两个 Flash 区域用同一个算法（如 STM32 双 Bank），`prepare_plan` 会把它们合并到一个 Flasher 里：

```rust
// 复用逻辑
if let Some(entry) = algos.iter_mut()
    .find(|e| e.flash_algorithm.name == algo.name && e.core_index == core)
{
    entry.add_region(region, &self.builder, restore_unwritten_bytes)?;
    //    ↑ 追加第二个区域到已有的 Flasher
}
```

**好处**：Flash 算法只加载一次到芯片 RAM，节省 RAM 空间和加载时间。

---

## 八、流程图

```
                  prepare_plan() 开始
                        │
                        ▼
              ┌─ 遍历 memory_map ─┐
              │  filter NVM only  │
              └────────┬──────────┘
                       │
                  ┌────▼────┐
                  │ 有数据？ │
                  └─┬────┬──┘
                NO  │    │ YES
                跳过 │    │
                     │    ▼
                     │  ┌─────────────┐
                     │  │ 查算法列表   │
                     │  │ 匹配:        │
                     │  │ ① 地址覆盖   │
                     │  │ ② 核心归属   │
                     │  └──────┬──────┘
                     │         │
                     │    ┌────▼────┐
                     │    │ 几个匹配？│
                     │    └─┬──┬──┬─┘
                     │     0│ 1│  │N个
                     │      │  │  │
                     │   报错  │  ▼
                     │         │ ┌──────────┐
                     │         │ │ 选策略:   │
                     │         │ │ preferred │
                     │         │ │ → default │
                     │         │ │ → 报错    │
                     │         │ └────┬─────┘
                     │         │      │
                     │         ▼      ▼
                     │    ┌──────────────┐
                     │    │ 已有同名      │
                     │    │ Flasher?     │
                     │    └──┬───────┬───┘
                     │    YES│       │NO
                     │       │       │
                     │   追加区域  新建Flasher
                     │       │       │
                     │       └───┬───┘
                     │           │
                     └───────────┘
                            │
                            ▼
                    Ok(Vec<Flasher>)
```

---

## 九、源码位置速查

| 内容 | 文件 | 行号 |
|------|------|------|
| `prepare_plan()` | `probe-rs/src/flashing/loader.rs` | 808 |
| `get_flash_algorithm_for_region()` | `probe-rs/src/flashing/loader.rs` | 1024 |
| 调用处 (`loader.commit()`) | `probe-rs/src/flashing/loader.rs` | 673 |
| `Flasher::new()` | `probe-rs/src/flashing/flasher.rs` | — |
| `Flasher::add_region()` | `probe-rs/src/flashing/flasher.rs` | — |
| `MemoryRegion::as_nvm_region()` | `probe-rs-target` | — |
| `FlashBuilder::has_data_in_range()` | `probe-rs/src/flashing/builder.rs` | — |
