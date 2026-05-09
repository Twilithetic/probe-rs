# probe-rs 配置文件 `.probe-rs.toml` 使用指南

## 文件扫描顺序

probe-rs 启动时按以下顺序找 `.probe-rs.toml`（也支持 `.json`/`.yaml`/`.yml`）：

```
① 当前工作目录    →  .probe-rs.toml
② probe-rs.exe 目录 →  .probe-rs.toml
③ 用户家目录       →  ~/.probe-rs.toml
```

后找到的配置会**覆盖**先找到的（合并叠加）。全部找不到也没关系——用默认值。

## Config 结构体

```rust
pub(crate) struct Config {
    pub server:  ServerConfig,              // 远程 server 配置
    pub presets: HashMap<String, ConfigPreset>,  // 预设参数组
}
```

## presets 功能 — 免敲参数

在项目根目录放一个 `.probe-rs.toml`：

```toml
[presets.default]
chip = "CYT2BL3BAS"
protocol = "swd"
speed = 100
```

然后命令行简化为：

```pwsh
probe-rs download build/firmware.elf
```

不再需要 `--chip CYT2BL3BAS --protocol swd --speed 100`。

## presets 高级用法

可以定义多个预设：

```toml
[presets.fast]
chip = "CYT2BL3BAS"
protocol = "swd"
speed = 1000

[presets.safe]
chip = "CYT2BL3BAS"
protocol = "swd"
speed = 100
```

使用时：

```pwsh
probe-rs download --preset fast build/firmware.elf    # 高速
probe-rs download --preset safe build/firmware.elf    # 低速稳定
```

## 源码位置

| 文件 | 行号 | 说明 |
|------|:--:|------|
| `probe-rs-tools/src/bin/probe-rs/main.rs` | 40-46 | `Config` 结构体定义 |
| 同上 | 758-789 | `load_config()` 函数实现 |
| 同上 | 536 | `main()` 入口 |

## 加载流程

```
main()
  ├─ UtcOffset::current_local_offset()    ← 拿时区
  ├─ std::env::args_os().collect()       ← 收集命令行参数
  ├─ load_config()                        ← ★ 加载配置文件
  │    ├─ 默认 Config::default()
  │    ├─ 依次扫描 ①②③
  │    ├─ figment 合并
  │    └─ 返回 Config
  ├─ parse_and_resolve_cli_args(args)     ← 解析命令行 + preset 合并
  └─ cli.run(client, config, offset)      ← 执行命令
```
