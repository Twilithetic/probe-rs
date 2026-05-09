# probe-rs multicall binary 机制详解

> 摘自 `probe-rs-tools/src/bin/probe-rs/main.rs` 和 `src/bin/cargo-*.rs`

## 一句话

probe-rs 是一个 **multicall binary**——一个可执行文件 + 两个壳程序，共同提供 `probe-rs`、`cargo flash`、`cargo embed` 三个命令。

## 三层调度架构

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Cargo 自动发现                                  │
│                                                         │
│ src/bin/cargo-flash.rs  → 编译为 cargo-flash.exe        │
│ src/bin/cargo-embed.rs  → 编译为 cargo-embed.exe        │
│ src/bin/probe-rs/main.rs → 编译为 probe-rs.exe          │
│                                                         │
│ Cargo 规则: src/bin/*.rs 下的每个文件自动成为 binary     │
│ 不需要在 Cargo.toml 里写 [[bin]]                        │
├─────────────────────────────────────────────────────────┤
│ Layer 2: 壳程序转发 (Windows)                             │
│                                                         │
│ cargo-flash.exe (32 行)                                 │
│   ├─ argv[0] = "cargo-flash"                            │
│   ├─ 去掉 cargo 插入的 "flash"                          │
│   └─ 启动 probe-rs.exe cargo-flash --chip xxx            │
│                                                         │
│ cargo-embed.exe (32 行)                                 │
│   └─ 同上，改为 "cargo-embed"                            │
├─────────────────────────────────────────────────────────┤
│ Layer 3: multicall_check (Linux/macOS 软链接)             │
│                                                         │
│ main.rs:536                                             │
│   if argv[0] 的文件名 == "cargo-flash" → cargo flash 模式│
│   if argv[1] == "cargo-flash"            → 同上         │
│   if argv[0] 的文件名 == "cargo-embed"  → cargo embed 模式│
│   if argv[1] == "cargo-embed"            → 同上         │
└─────────────────────────────────────────────────────────┘
```

## 源码速查

| 文件 | 说明 |
|------|------|
| `src/bin/cargo-flash.rs` | 32 行壳程序，改 argv → 启动 probe-rs |
| `src/bin/cargo-embed.rs` | 32 行壳程序，同上 |
| `src/bin/probe-rs/main.rs:518-533` | `multicall_check()` 函数 |
| `src/bin/probe-rs/main.rs:536` | `#[tokio::main] async fn main()` 入口 |
| `src/bin/probe-rs/main.rs:549-556` | 入口处调用 multicall_check |

## main.rs 启动流程

```
#[tokio::main]
async fn main() -> Result<()> {
    probe_rs_espressif::register_plugin();      // 注册 Espressif 插件

    let utc_offset = UtcOffset::current_local_offset()
        .unwrap_or(UtcOffset::UTC);              // 拿时区

    let args: Vec<_> = std::env::args_os().collect();  // 收集命令行参数

    let config = load_config()                   // 加载 .probe-rs.toml
        .context("Failed to load configuration.")?;

    // ★ multicall_check — 检测是否被叫做 cargo-flash/cargo-embed
    if let Some(args) = multicall_check(&args, "cargo-flash") {
        return cmd::cargo_flash::main(args, config).await;
    }
    if let Some(args) = multicall_check(&args, "cargo-embed") {
        cmd::cargo_embed::main(args, config, utc_offset).await;
        return Ok(());
    }

    // 都不是 → 正常 probe-rs 模式
    let mut cli = parse_and_resolve_cli_args::<Cli>(args, &config)?;
    cli.run(client, config, utc_offset).await
}
```

## multicall_check 源码 (main.rs:518)

```rust
fn multicall_check(args: &[OsString], want: &str) -> Option<Vec<OsString>> {
    let argv0 = Path::new(&args[0]);
    // 情况 A: 自己就叫 cargo-flash (软链接)
    if let Some(command) = argv0.file_stem().and_then(|f| f.to_str())
        && command == want
    {
        return Some(args.to_vec());
    }
    // 情况 B: 第一个参数是 cargo-flash (壳程序转发过来的)
    if let Some(command) = args.get(1).and_then(|f| f.to_str())
        && command == want
    {
        return Some(args[1..].to_vec());
    }
    None
}
```

## 壳程序源码 (cargo-flash.rs, 32 行)

```rust
fn main() {
    let mut args: Vec<_> = std::env::args_os().collect();
    args[0] = "cargo-flash".into();          // 改写 argv[0]
    if args.get(1) == Some("flash") {        // cargo 插入的 "flash" 去掉
        args.remove(1);
    }
    let mut cmd = Command::new("probe-rs");   // 启动真正的 probe-rs
    cmd.args(&args);

    #[cfg(unix)]
    let err = cmd.exec();                     // Unix: exec 替换自身
    #[cfg(not(unix))]
    let err = cmd.spawn();                    // Windows: spawn 子进程

    eprintln!("Error launching `probe-rs`: {err}");
    exit(99);
}
```

## cargo install 装了哪些文件

`cargo install probe-rs-tools` 会安装所有 `src/bin/*.rs` 编译出的二进制到 `~/.cargo/bin/`：

```
~/.cargo/bin/
├── cargo-flash.exe   (231KB) ← 壳程序
├── cargo-embed.exe   (231KB) ← 壳程序
└── probe-rs.exe      (大得多) ← 真正逻辑
```

## 为什么需要两层（壳程序 + multicall_check）

| 平台 | 支持软链接？ | 使用方案 |
|------|:--:|------|
| Linux/macOS | ✅ | 软链接 probe-rs → cargo-flash → multicall_check 检测 argv[0] |
| Windows | ❌ | 壳程序 cargo-flash.exe → 启动 probe-rs → multicall_check 检测 argv[1] |

壳程序和 `multicall_check` 互为补充，保证所有平台都能正常工作。
