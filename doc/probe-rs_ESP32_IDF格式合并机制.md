# ESP32 IDF 格式的合并机制 — 为什么 `prepare_plan()` 不需要特殊处理

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：esp32 的 bootloader + 分区表在哪一步合并、`prepare_plan()` 为什么是通用的  
> 📖 上下文：补充第五篇 "原材料溯源" — ESP32 的三段式固件如何变成统一的数据段

---

## 一、一句话总结

ESP32 的 bootloader + partition table + app 在 `IdfLoader::load()` 阶段就被 `espflash` crate 合并了。到 `prepare_plan()` 时，它们已经是 `builder.data` 里的普通段，没有任何特殊标记。

---

## 二、ARM vs ESP32 对比

```
普通 ARM (CYT2BL)                         ESP32 (IDF 格式)
───────────────                           ────────────────
一个文件: firmware.elf                     三个文件:
                                              ├─ app.elf
                                              ├─ bootloader.bin
                                              └─ partition.bin

ElfLoader::load()                         IdfLoader::load()
  extract_from_elf()                        │
    → PT_LOAD 段                            ├─ ① 读 ELF → check_idf_bootloader()
  add_data(addr, data)                      ├─ ② 芯片上执行 FlashSize vendor 函数
    ↓                                       │     → 自动检测 Flash 大小
  prepare_plan()                            ├─ ③ 芯片兼容性检查 (ELF metadata)
    ↓                                       ├─ ④ IdfBootloaderFormat::new(
  ⚡ 烧录                                     │       elf, bootloader, partition, flash_settings)
                                            │     → espflash crate 合并三段
                                            └─ ⑤ image.flash_segments() 返回合并后的段列表
                                                   for segment in segments:
                                                     flash_loader.add_data(addr, data)
                                                       ↓
                                                  prepare_plan()  ← 跟 ARM 一样！
                                                       ↓
                                                  ⚡ 烧录
```

---

## 三、`IdfLoader::load()` 源码

**文件**: `probe-rs-espressif/src/image_format.rs:63`

```rust
impl ImageLoader for IdfLoader {
    fn load(&self, flash_loader: &mut FlashLoader, session: &mut Session,
            file: &mut dyn ImageReader) -> Result<(), FileDownloadError> {

        // ① 解析芯片名 (如 "ESP32-S3" 去掉尾缀 "-xxx")
        let chip = espflash::target::Chip::from_str(target_name)?;

        // ② 在芯片上执行 FlashSize vendor 函数，检测 Flash 容量
        let flash_size_result = algo.call_vendor_function("FlashSize", [None; 4])?;
        let flash_size = match flash_size_result {
            0x200000 => Some(FlashSize::_2Mb),
            0x400000 => Some(FlashSize::_4Mb),
            // ...
        };

        // ③ 检查 ELF 元数据中的芯片名是否匹配
        check_chip_compatibility_from_elf_metadata(session, &elf_buffer)?;

        // ④ 🔥 espflash crate 合并三段！
        let image = IdfBootloaderFormat::new(
            &elf_buffer,                         // app.elf
            &flash_data,                         // Flash 参数 (大小/频率/模式)
            self.partition_table.as_deref(),     // 可选: 分区表
            self.bootloader.as_deref(),          // 可选: bootloader
            None,
            self.target_app_partition.as_deref(),
        )?;

        // ⑤ 合并后的所有段 → 逐个 add_data
        for data in image.flash_segments() {
            flash_loader.add_data(data.addr.into(), &data.data)?;
        }

        Ok(())
    }
}
```

---

## 四、合并后的 `builder.data`

```
普通 ARM 的 builder.data:                ESP32 的 builder.data:
┌──────────────────────┐                ┌─────────────────────────┐
│ 0x1000_2000: .text   │                │ 0x0000_0000: 2nd boot   │ ← 来自 bootloader.bin
│ 0x1000_8000: .rodata │                │ 0x0000_8000: partition  │ ← 来自 partition.bin
│ ...                  │                │ 0x0001_0000: app code   │ ← 来自 app.elf
└──────────────────────┘                │ ...                     │
                                        └─────────────────────────┘

对 prepare_plan() 来说，两者完全一样：都是 BTreeMap<地址, 字节>
```

---

## 五、为什么设计成这样？

这是 **关注点分离** 的经典应用：

| 层 | 职责 | 知道 ESP32 特殊吗？ |
|----|------|-------------------|
| `ImageLoader` (IdfLoader/ElfLoader) | 解析文件格式，合并多段 | ✅ ESP32 知道 |
| `FlashLoader` / `FlashBuilder` | 存储数据段 (地址→字节) | ❌ 不知道 |
| `prepare_plan()` | 匹配区域 ↔ 算法 | ❌ 不知道 |
| `Flasher::program()` | 执行 Flash 算法 | ❌ 不知道 |

**每一层只关心自己的事**：`IdfLoader` 负责把复杂的 IDF 格式 "翻译" 成统一的 `add_data()` 调用，后续所有层都不需要知道数据是怎么来的。

---

## 六、bootloader / partition 是怎么传进来的？

CLI 端 `build_flash_loader()` 负责上传文件：

```rust
// client.rs — build_flash_loader()
pub async fn build_flash_loader(&self, path, format, ...) {
    // ① 上传主 ELF
    path = self.client.upload_file(&path).await?;

    // ② ESP32 only: 上传 bootloader
    if let Some(ref mut idf_bootloader) = format.idf_options.idf_bootloader {
        *idf_bootloader = self.client.upload_file(&*idf_bootloader).await?
            .display().to_string();
    }

    // ③ ESP32 only: 上传分区表
    if let Some(ref mut idf_partition_table) = format.idf_options.idf_partition_table {
        *idf_partition_table = self.client.upload_file(&*idf_partition_table).await?
            .display().to_string();
    }

    // ④ 发 BuildRequest → daemon 端调用 IdfLoader::load()
    self.client.send_resp::<BuildEndpoint, _>(&BuildRequest { ... }).await
}
```

---

## 七、时间线

```
ARM 时间线:                            ESP32 时间线:
─────────                             ────────────
① CLI 上传 firmware.elf               ① CLI 上传 app.elf + bootloader.bin + partition.bin
② ElfLoader 解析 ELF                  ② IdfLoader 解析 + espflash 合并
③ add_data → builder.data             ③ add_data → builder.data  (同上！)
④ prepare_plan → 匹配算法             ④ prepare_plan → 匹配算法   (同上！)
⑤ commit → 烧录                      ⑤ commit → 烧录            (同上！)
```

**从第③步开始，两者完全一致。**
