# probe-rs `prepare_plan()` 三样原材料溯源

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：ELF数据段、芯片内存图、Flash算法库分别从哪来，怎么被加载的  
> 📖 上下文：`prepare_plan()` 接收三样输入来制定烧录计划

---

## 一、总览

`prepare_plan()` 的三种输入，来自两个完全不同的源头：

```
              编译产物                         芯片定义 (YAML)
          firmware.elf                      CYT2BL_Series.yaml
               │                                   │
               ▼                                   ▼
       ┌──────────────┐                  ┌──────────────────┐
       │ extract_from_ │                  │ 编译期嵌入:       │
       │    elf()      │                  │ targets.bincode  │
       │      ↓        │                  │      ↓           │
       │ FlashBuilder  │                  │ Registry         │
       │  .add_data()  │                  │      ↓           │
       │      ↓        │                  │ Target::new()    │
       │ BTreeMap      │                  │   ├─ memory_map  │
       │ <addr, bytes> │                  │   └─ flash_      │
       └──────┬───────┘                  │       algorithms │
              │                          └────────┬─────────┘
              │                                   │
              │      session.target()             │
              │        ┌──────────────────────────┘
              ▼        ▼
       ┌──────────────────────┐
       │    prepare_plan()    │  ← 三样原材料在此汇合！
       └──────────────────────┘
```

---

## 二、原材料 ①：ELF 数据段 (`builder.data`)

**来源**：编译器输出的 `firmware.elf` 文件

**类型**：`BTreeMap<u64, Vec<u8>>` — 起始地址 → 二进制数据

### 加载流程

```
firmware.elf 文件
      │
      ▼
① build_loader()                          [probe-rs-tools/.../util/flash.rs:111]
      │
      ├─ ② FormatOptions::image_loader()  [main.rs:433]
      │     → 根据格式类型创建加载器:
      │       ELF → ElfLoader
      │       BIN → BinLoader
      │       HEX → IHexLoader
      │
      ├─ ③ flashing::build_loader()       [probe-rs/src/flashing/download.rs:151]
      │     ├─ Target::flash_loader()     [target.rs:176]
      │     │   → FlashLoader::new(memory_map, source)
      │     │   → 创建空的 FlashBuilder
      │     └─ FlashLoader::load_image()  [loader.rs:588]
      │
      └─ ④ 格式解析 → add_data()
            │
            ├─ ELF: extract_from_elf()    [loader.rs:253]
            │     → 解析 ELF header
            │     → 遍历 PT_LOAD 段
            │
            └─ FlashLoader::add_data(addr, bytes)  [loader.rs:564]
                  │
                  ├─ check_data_in_memory_map()    [loader.rs:544]
                  │   → 检查地址是否在芯片有效范围内
                  │
                  └─ FlashBuilder::add_data()      [builder.rs:180]
                        → 存入 BTreeMap<u64, Vec<u8>>
```

### 关键代码

```rust
// loader.rs:564 — FlashLoader::add_data
pub fn add_data(&mut self, address: u64, data: &[u8]) -> Result<(), FlashError> {
    // 验证：这段数据所在的地址是否在芯片内存图中有对应区域
    self.check_data_in_memory_map(address, data.len())?;
    // 存入 builder
    self.builder.add_data(address, data)
}

// builder.rs:180 — FlashBuilder::add_data
pub fn add_data(&mut self, address: u64, data: &[u8]) -> Result<(), FlashError> {
    if data.is_empty() { return Ok(()); }
    // 检查跟已有数据不重叠
    // 如果能跟前一段拼接 → extend
    // 否则 → insert 新条目
    self.data.insert(address, data.to_vec());
}

// builder.rs — FlashBuilder 的数据结构
pub struct FlashBuilder {
    pub data: BTreeMap<u64, Vec<u8>>,   // 有序的地址→数据映射
    // ...
}
```

### 数据示例

```
FlashBuilder.data:
┌──────────────────────────────────────────┐
│ 0x1000_2000 → [0x00, 0x48, 0x00, 0xf0, ...]  │ ← .text (代码段)
│ 0x1000_8000 → [0x48, 0x65, 0x6c, 0x6c, ...]  │ ← .rodata (只读数据)
│ 0x1001_0000 → [...]                          │ ← 更多段
│ ...                                           │
└──────────────────────────────────────────┘
```

### ELF 提取细节

```rust
// loader.rs:276 — extract_from_elf_inner
fn extract_from_elf_inner(elf_data: &[u8], opts: &ElfOptions) -> Vec<ExtractedFlashData> {
    // ① 解析 ELF 文件头
    let elf = object::read::elf::ElfFile::parse(elf_data)?;

    // ② 遍历 PT_LOAD 段（真正需要加载到内存的段）
    for segment in elf.elf_header().program_headers(elf.endianness())? {
        if segment.p_type != PT_LOAD || segment.p_filesz == 0 {
            continue;  // 跳过非 LOAD 段和空段
        }

        // ③ 提取段数据
        let segment_data = elf_data[segment.file_range()];

        // ④ 按 section 拆分，返回 (virtual_addr, data)
        for section in sections {
            results.push(ExtractedFlashData {
                address: p_paddr as u64,
                data: section_data.to_vec(),
                section_names: ...,
            });
        }
    }
}
```

---

## 三、原材料 ②：芯片内存图 (`memory_map`)

**来源**：芯片 YAML 目标定义文件（编译期嵌入）

**类型**：`Vec<MemoryRegion>` — 包含 `NvmRegion` (Flash) 和 `RamRegion` (RAM)

### 加载流程

```
编译期:
  CYT2BL_Series.yaml
      │
      ├─ build.rs (编译脚本)
      │   → 解析 YAML → 序列化为 bincode 格式
      │
      └─ targets.bincode ──include_bytes!()──→ BUILTIN_TARGETS

运行时:
  probe-rs download firmware.elf --chip CYT2BL3BAE
      │
      ▼
  Session::auto_attach("CYT2BL3BAE")    [session.rs:564]
      │
      ├─ Registry::get_target_by_name() [registry.rs:206]
      │   → 在 BUILTIN_TARGETS 中搜索
      │
      └─ Target::new(family, chip)      [target.rs:67]
            │
            ├─ memory_map = chip.memory_map.clone()  ← 🎯 这里！
            └─ flash_algorithms = ...                  ← 🎯 这里！
```

### YAML 定义

```yaml
# CYT2BL_Series.yaml — variants:
variants:
  - name: CYT2BL3BAE
    memory_map:
      # RAM 区域
      - !Ram
        name: IRAM1
        range:
          start: 0x08000000       # 512KB SRAM
          end: 0x08080000
        cores:
          - cortex-m0p
          - cortex-m4

      # Flash 区域 (NVM = Non-Volatile Memory)
      - !Nvm
        name: IROM1
        range:
          start: 0x10000000       # 4MB Code Flash
          end: 0x10410000
        cores:
          - cortex-m0p
          - cortex-m4
        access:
          write: false            # 不能像 RAM 一样直接写
          boot: true              # 这是启动内存
```

### Rust 结构体

```rust
// probe-rs-target/src/memory.rs

pub enum MemoryRegion {
    Ram(RamRegion),           // prepare_plan 跳过 → RAM 在 commit_ram() 单独处理
    Generic(GenericRegion),   // 跳过
    Nvm(NvmRegion),           // prepare_plan 处理这个！
}

pub struct NvmRegion {
    pub name: Option<String>,       // "IROM1"
    pub range: Range<u64>,          // 0x10000000..0x10410000
    pub cores: Vec<String>,         // ["cortex-m0p", "cortex-m4"]
    pub is_alias: bool,             // 是否是别名区域
    pub access: Option<MemoryAccess>,
}
```

### 运行时访问

```rust
// session.rs:917
impl Session {
    pub fn target(&self) -> &Target {
        &self.target
    }
}

// target.rs
pub struct Target {
    pub memory_map: Vec<MemoryRegion>,       // ← 这里！
    pub flash_algorithms: Vec<RawFlashAlgorithm>, // ← 这里！
    pub cores: Vec<Core>,
    // ...
}
```

---

## 四、原材料 ③：Flash 算法库 (`flash_algorithms`)

**来源**：同样来自芯片 YAML 定义，包含预编译的 CMSIS Flash 算法二进制码

**类型**：`Vec<RawFlashAlgorithm>` — 每个算法包含函数入口点 + 二进制指令

### 加载流程

跟 `memory_map` 完全一样，都在 `Target::new()` 里加载。

### YAML 定义（CYT2BL 有 6 个算法）

```yaml
# CYT2BL_Series.yaml — 顶层 flash_algorithms:
flash_algorithms:
  - name: cyt2bl                    # 主 Flash 算法
    description: CYT2BL
    default: true
    instructions: QkP... (base64)   # ← 🔥 算法的二进制指令！
    load_address: 0x8001008         # 加载到 RAM 的地址
    pc_init: 0x1                    # Init() 入口偏移
    pc_uninit: 0x599                # UnInit() 入口偏移
    pc_program_page: 0x591          # ProgramPage() 入口
    pc_erase_sector: 0x4f1          # EraseSector() 入口
    pc_erase_all: 0x4e9             # EraseChip() 入口
    pc_verify: 0x5a1                # Verify() 入口
    data_section_offset: 0x820      # 数据段在 RAM 中的偏移
    flash_properties:
      address_range:                # 这个算法能烧的地址范围
        start: 0x10000000
        end: 0x10410000
      page_size: 0x200              # 512 字节/页
      erased_byte_value: 0xff       # 擦除后值为 0xFF
      program_page_timeout: 20      # 编程超时 (ms)
      erase_sector_timeout: 320     # 擦除超时 (ms)
      sectors:                      # sector 布局
        - size: 0x8000              # 前 4MB-64KB: 32KB 一个扇区
          address: 0x0
        - size: 0x2000              # 最后 64KB: 8KB 一个扇区
          address: 0x3f0000

  - name: cyt2bx_wflash_128         # Work Flash 算法
    description: CYT2Bx WFlash 128 KB
    instructions: ... (base64)
    flash_properties:
      address_range:
        start: 0x14000000
        end: 0x14020000
      page_size: 0x80               # 128 字节/页

  - name: cyt2bx_sflash_user        # SFlash 用户区
  - name: cyt2bx_sflash_nar         # SFlash NAR
  - name: cyt2bx_sflash_pkey        # SFlash 保护密钥
  - name: cyt2bx_sflash_toc2        # SFlash TOC2
```

### CYT2BL 的 6 个算法一览

| 算法 | 地址范围 | 页大小 | 用途 |
|------|---------|--------|------|
| `cyt2bl` | `0x10000000..0x10410000` | 512 B | 主 Code Flash（默认） |
| `cyt2bx_wflash_128` | `0x14000000..0x14020000` | 128 B | Work Flash (128KB) |
| `cyt2bx_sflash_user` | `0x17000800..0x17001000` | 512 B | SFlash 用户区 |
| `cyt2bx_sflash_nar` | `0x17001a00..0x17001c00` | 512 B | SFlash NAR |
| `cyt2bx_sflash_pkey` | `0x17006400..0x17007000` | 512 B | SFlash 保护密钥 |
| `cyt2bx_sflash_toc2` | `0x17007c00..0x17007e00` | 512 B | SFlash TOC2 |

### Rust 结构体

```rust
// probe-rs-target/src/flash_algorithm.rs
pub struct RawFlashAlgorithm {
    pub name: String,                    // "cyt2bl"
    pub description: String,             // "CYT2BL"
    pub default: bool,                   // 是否是默认算法
    pub instructions: Vec<u8>,           // 🔥 二进制机器码 (从 base64 解码)
    pub load_address: Option<u64>,       // 加载到 RAM 的基地址
    pub pc_init: Option<u64>,            // Init 入口 (偏移)
    pub pc_uninit: Option<u64>,          // UnInit 入口
    pub pc_program_page: u64,            // ProgramPage 入口
    pub pc_erase_sector: u64,            // EraseSector 入口
    pub pc_erase_all: Option<u64>,       // EraseChip 入口
    pub pc_verify: Option<u64>,          // Verify 入口
    pub flash_properties: FlashProperties,
    pub cores: Vec<String>,             // 支持的核心
    pub stack_size: Option<u32>,        // 需要的栈大小
    pub transfer_encoding: Option<TransferEncoding>, // Raw 或 Miniz 压缩
    pub rtt_location: Option<u64>,      // RTT 控制块地址
    // ...
}

// probe-rs-target/src/flash_properties.rs
pub struct FlashProperties {
    pub address_range: Range<u64>,       // 0x10000000..0x10410000
    pub page_size: u32,                  // 512
    pub erased_byte_value: u8,           // 0xFF
    pub program_page_timeout: u32,       // 编程超时 ms
    pub erase_sector_timeout: u32,       // 擦除超时 ms
    pub sectors: Vec<SectorDescription>, // sector 布局
}

// probe-rs-target/src/memory.rs
pub struct SectorDescription {
    pub size: u64,    // sector 大小 (e.g. 0x8000 = 32KB)
    pub address: u64, // 从 Flash 基地址的偏移
}
```

### `instructions` 是什么？

就是用 base64 编码的 **ARM Thumb 机器码**。运行时解码成 `Vec<u8>`，然后通过 SWD 写到芯片 RAM，让芯片 CPU 执行它来操作 Flash 控制器。

这是 **CMSIS-Pack 标准**的一部分——芯片厂商提供 Flash 算法二进制，probe-rs 只需加载和执行。

---

## 五、三者在 `prepare_plan()` 中的使用

```rust
// loader.rs:808
fn prepare_plan(&self, session, ...) -> Vec<Flasher> {
    // ① self.builder.data       ← ELF 数据段 (BTreeMap<地址→字节>)
    // ② memory_map              ← 芯片内存图 (Vec<MemoryRegion>)
    // ③ flash_algorithms        ← 算法库 (Vec<RawFlashAlgorithm>)

    for region in session.target().memory_map.iter()
        .filter_map(MemoryRegion::as_nvm_region)  // ② 只看 Flash 区域
    {
        // ① 这个 Flash 区域有 ELF 数据吗？
        if !self.builder.has_data_in_range(&region.range) {
            continue;  // 没数据 → 跳过这个区域
        }

        // ③ 找一个能烧这个区域的 Flash 算法
        let algo = get_flash_algorithm_for_region(&region, target, core_name, preferred);
        //  匹配条件:
        //  - algo.flash_properties.address_range 覆盖 region.range
        //  - algo.cores 包含 region.cores

        // 创建 Flasher（算法 + 区域 + 数据的组合）
        algos.push(Flasher::new(target, core, algo));
    }
}
```

### 数据流对照

| 来源 | 结构 | 关键字段 | 在 prepare_plan 的用途 |
|------|------|---------|----------------------|
| ELF 文件 | `BTreeMap<u64, Vec<u8>>` | 地址、二进制数据 | 判断哪个区域有数据 (`has_data_in_range`) |
| YAML → memory_map | `Vec<MemoryRegion>` | `NvmRegion.range`, `.cores` | 迭代每种 Flash 区域 |
| YAML → flash_algorithms | `Vec<RawFlashAlgorithm>` | `.flash_properties.address_range`, `.cores`, `.default` | 匹配算法到区域 |

---

## 六、完整时间线

```
┌─ 编译期 ───────────────────────────────────────────────────┐
│  CYT2BL_Series.yaml ───build.rs───→ targets.bincode       │
│                                         │                  │
│                                    include_bytes!()        │
│                                         │                  │
│                                    BUILTIN_TARGETS         │
│                                    (内存图 + 算法库)        │
└───────────────────────────────────────┬────────────────────┘
                                        │
┌─ 运行时 ──────────────────────────────┼────────────────────┐
│                                       ▼                    │
│  ① probe-rs download firmware.elf --chip CYT2BL3BAE       │
│                                                             │
│  ② Session::auto_attach()                                  │
│     → Registry::get_target_by_name()                       │
│     → Target::new() ← 加载 memory_map + flash_algorithms   │
│                                                             │
│  ③ build_loader()                                          │
│     → extract_from_elf()                                   │
│     → FlashBuilder::add_data() ← 加载 ELF 数据段            │
│                                                             │
│  ④ prepare_plan() ← 🎯 三者汇合                            │
│     → 匹配区域 ↔ 算法                                       │
│     → 分配 ELF 数据到各 Flasher                             │
│     → 返回 Vec<Flasher>                                    │
│                                                             │
│  ⑤ initialize() → 把算法 instructions 写到芯片 RAM         │
│  ⑥ program() → 让芯片执行算法，逐页烧录                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、源码位置速查

| 内容 | 文件 | 行号 |
|------|------|------|
| `build_loader()` — CLI 入口 | `probe-rs-tools/.../util/flash.rs` | 111 |
| `flashing::build_loader()` — 公共 API | `probe-rs/src/flashing/download.rs` | 151 |
| `Target::flash_loader()` — 创建 FlashLoader | `probe-rs/src/config/target.rs` | 176 |
| `FlashLoader::new()` | `probe-rs/src/flashing/loader.rs` | 494 |
| `FlashLoader::load_image()` | `probe-rs/src/flashing/loader.rs` | 588 |
| `FlashLoader::add_data()` | `probe-rs/src/flashing/loader.rs` | 564 |
| `check_data_in_memory_map()` | `probe-rs/src/flashing/loader.rs` | 544 |
| `FlashBuilder::add_data()` | `probe-rs/src/flashing/builder.rs` | 180 |
| `extract_from_elf()` | `probe-rs/src/flashing/loader.rs` | 253 |
| `extract_from_elf_inner()` | `probe-rs/src/flashing/loader.rs` | 276 |
| `Target::new()` — 加载 memory_map + algorithms | `probe-rs/src/config/target.rs` | 67 |
| `Registry::get_target_by_name()` | `probe-rs/src/config/registry.rs` | 206 |
| `BUILTIN_TARGETS` | `probe-rs/src/config/registry.rs` | 157 |
| CYT2BL target YAML | `probe-rs/targets/CYT2BL_Series.yaml` | — |
| `NvmRegion` 结构体 | `probe-rs-target/src/memory.rs` | 6 |
| `MemoryRegion` 枚举 | `probe-rs-target/src/memory.rs` | 386 |
| `RawFlashAlgorithm` 结构体 | `probe-rs-target/src/flash_algorithm.rs` | 30 |
| `FlashProperties` 结构体 | `probe-rs-target/src/flash_properties.rs` | 11 |
| `SectorDescription` 结构体 | `probe-rs-target/src/memory.rs` | 307 |
| `prepare_plan()` — 汇合点 | `probe-rs/src/flashing/loader.rs` | 808 |
