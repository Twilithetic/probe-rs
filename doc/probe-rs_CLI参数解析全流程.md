# probe-rs CLI 参数解析全流程
## 十四、Flash 布局可视化 — `--flash-layout-output-path`
> 从终端字符串到 Rust 结构体的每一步
`flash_layout` 变量初始为 `None`，在烧录过程中被 daemon 的进度事件偷偷填上，最后用来画图：
## 一、完整调用链
```
用户敲: probe-rs download --chip CYT2BL3BAS --speed 100 build/firmware.elf
main.rs:536  async fn main()
  │
  ├─ std::env::args_os().collect()
  │     从操作系统拿原始字符串，切成 Vec<OsString>
  │     → ["probe-rs", "download", "--chip", "CYT2BL3BAS", "--speed", "100", "build/firmware.elf"]
  │
  ├─ multicall_check() — 检测是否是 cargo-flash / cargo-embed
  │
  ├─ load_config() — 加载 .probe-rs.toml
  │     → Config { presets: { "default": { chip="CYT2BL3BAS", ... } }, ... }
  │
  ├─ parse_and_resolve_cli_args::<Cli>(args, &config)?
  │   │
  │   ├─ 第一步：Cli::command().get_matches_from(&args)
  │   │     字符串 → ArgMatches (中间格式)
  │   │
  │   ├─ 第二步：apply_config_preset() — 注入预设参数
  │   │     如果有 [presets.default] → 把 --chip, --protocol, --speed 追加到 args
  │   │
  │   ├─ 第三步：重新 get_matches_from(扩展后的 args)
  │   │     预设参数和命令行参数合并后重新解析
  │   │
  │   └─ 第四步：Cli::from_arg_matches(&matches)?
  │         ArgMatches → Rust struct Cli
  │
  └─ cli.run(client, config, utc_offset)
        ↓
      match subcommand { Download(cmd) → cmd.run() → attach_probe → flash }
```
## 二、Cli 结构体 (main.rs:55-109)
```rust
#[derive(clap::Parser)]          // ★ 编译时生成解析代码
#[clap(name = "probe-rs", about = "The probe-rs CLI")]
struct Cli {
    // ── 全局参数 (global = true 表示对所有子命令生效) ──
    #[clap(long, global = true)]
    log_file:      Option<PathBuf>,   // --log-file <PATH>
let mut flash_layout = None;   // 初始空着
    #[clap(long, global = true)]
    log_to_folder: bool,              // --log-to-folder
    #[clap(short, long, global = true)]
    report:        Option<PathBuf>,   // -r / --report <PATH>
    #[cfg(feature = "remote")]
    #[arg(long, global = true)]
    host:  Option<String>,            // --host
    #[cfg(feature = "remote")]
    #[arg(long, global = true)]
    token: Option<String>,            // --token
    #[arg(long, global = true)]
    preset: Option<String>,           // --preset <NAME>
    // ── 子命令 ──
    #[clap(subcommand)]
    subcommand: Subcommand,           // download | info | run | erase | ...
}
```
## 二A、`Cli` 的双重身份 — struct 两用
一个 struct，两个角色：
```rust
// 身份一：参数容器 — #[derive(Parser)] 给的
#[derive(clap::Parser)]
struct Cli {
    log_file: Option<PathBuf>,
    subcommand: Subcommand,
    // ...
}
// 身份二：命令调度器 — 自己写的 impl 块
impl Cli {
    async fn run(self, client: RpcClient, config: Config, utc_offset: UtcOffset) -> Result<()> {
        match self.subcommand {          // 根据子命令派发
            Subcommand::Download(cmd) => cmd.run(client).await,
            Subcommand::Info(cmd)     => cmd.run(client).await,
            Subcommand::Run(cmd)      => cmd.run(client, utc_offset).await,
            // ... 全部 21 个子命令
        }
// preverify / flash 过程中，daemon 发 FlashLayoutReady 事件
session.flash(..., async |event| {
    if let ProgressEvent::FlashLayoutReady { flash_layout: layout } = &event {
        flash_layout = Some(layout.clone());  // ← 回调里偷偷填
    }
}
```
}).await?;
**`#[derive(Parser)]` 只加了参数解析能力（`command()`、`from_arg_matches()`）。`run()` 是自己写的方法。** Cli 既是"填满参数的容器"也是"派发任务的调度器"。
## 四、子命令枚举
```
Subcommand::Download(cmd::download::Cmd)   → probe-rs download
Subcommand::Info(cmd::info::Cmd)           → probe-rs info
Subcommand::Run(cmd::run::Cmd)             → probe-rs run
Subcommand::Erase(cmd::erase::Cmd)         → probe-rs erase
Subcommand::Reset(cmd::reset::Cmd)         → probe-rs reset
Subcommand::Verify(cmd::verify::Cmd)       → probe-rs verify
Subcommand::Debug(cmd::debug::Cmd)         → probe-rs debug
Subcommand::Attach(cmd::attach::Cmd)       → probe-rs attach
Subcommand::List(cmd::list::Cmd)           → probe-rs list
Subcommand::Gdb(cmd::gdb_server::Cmd)      → probe-rs gdb
Subcommand::Trace(cmd::trace::Cmd)         → probe-rs trace
Subcommand::Itm(cmd::itm::Cmd)             → probe-rs itm
Subcommand::Chip(cmd::chip::Cmd)           → probe-rs chip
Subcommand::DapServer(cmd::dap_server::Cmd)→ probe-rs dap-server
Subcommand::Serve(cmd::serve::Cmd)         → probe-rs serve
Subcommand::Read(cmd::read::Cmd)           → probe-rs read
Subcommand::Write(cmd::write::Cmd)         → probe-rs write
Subcommand::Benchmark(cmd::benchmark::Cmd) → probe-rs benchmark
Subcommand::Profile(cmd::profile::Cmd)     → probe-rs profile
Subcommand::Complete(cmd::complete::Cmd)   → probe-rs complete
Subcommand::Mi(cmd::mi::Cmd)               → probe-rs mi
```
## 五、Download 子命令的结构 (cmd/download.rs)
```
Subcommand::Download(Cmd {
    probe_options:    ProbeOptions {
        chip:               Option<String>,              // --chip
        chip_description_path: Option<PathBuf>,          // --chip-description-path
        protocol:           Option<WireProtocol>,         // --protocol
        speed:              Option<u32>,                  // --speed
        probe:              Option<DebugProbeSelector>,   // --probe VID:PID
        connect_under_reset: bool,                        // --connect-under-reset
        cycle_power:        bool,                         // --cycle-power
        dry_run:            bool,                         // --dry-run
        non_interactive:    bool,                         // --non-interactive
    },
    path:             PathBuf,          // 裸参数: build/firmware.elf
    download_options: BinaryDownloadOptions {
        binary_format:     Option<Format>,               // --binary-format
        restore_unwritten: bool,                         // --restore-unwritten
        chip_erase:        bool,                         // --chip-erase
        verify:            bool,                         // --verify
        preverify:         bool,                         // --preverify
        disable_double_buffering: bool,                  // --disable-double-buffering
        disable_progressbars: bool,                      // --disable-progressbars
        allow_erase_all:   bool,                         // --allow-erase-all
        prefer_flash_algorithm: Vec<String>,             // --prefer-flash-algorithm
        flash_layout_output_path: Option<PathBuf>,        // --flash-layout-output-path
        read_flasher_rtt:  bool,                         // --read-flasher-rtt
    },
    format_options:   FormatOptions,
})
```
## 五、`#[derive(clap::Parser)]` 做了什么
`#[derive(clap::Parser)]` 在**编译时**展开代码，为 struct 自动实现两个 trait：
### 5.1 `CommandFactory` trait → 生成 `command()` 方法
```rust
// 编译时自动生成（伪代码）:
impl CommandFactory for Cli {
    fn command() -> Command {
        Command::new("probe-rs")
            .about("The probe-rs CLI")
            .arg(arg!(--log_file <PATH>).global(true))
            .arg(arg!(--log_to_folder).global(true))
            .arg(arg!(-r --report <PATH>).global(true))
            .arg(arg!(--preset <NAME>).global(true))
            .subcommand(Download::command())
            .subcommand(Info::command())
            .subcommand(Run::command())
            // ... 所有子命令
    }
// 后面用 --flash-layout-output-path 导出 SVG
if let Some(path) = download_options.flash_layout_output_path {
    let visualizer = flash_layout.visualize();
    visualizer.write_svg(path)?;  // 输出 Flash 布局图
}
```
**命名规则**：
- 字段名 `log_file` → 自动生成 `--log-file`（下划线变连字符）
- `bool` 类型 → flag 参数（出现就是 true）
- `Option<T>` → 可选参数（可以不出现）
- `PathBuf` → 参数后面跟值
- `#[clap(subcommand)]` → 子命令分支
### 用法
### 5.2 `FromArgMatches` trait → 生成 `from_arg_matches()` 方法
```rust
// 编译时自动生成（伪代码）:
impl FromArgMatches for Cli {
    fn from_arg_matches(m: &ArgMatches) -> Result<Self> {
        Ok(Cli {
            log_file:      m.get_one::<PathBuf>("log_file").cloned(),
            log_to_folder: m.get_flag("log_to_folder"),
            report:        m.get_one::<PathBuf>("report").cloned(),
            preset:        m.get_one::<String>("preset").cloned(),
            subcommand:    Subcommand::from_arg_matches(
                m.subcommand().expect("subcommand required")
            )?,
        })
    }
}
```bash
probe-rs download --flash-layout-output-path flash.svg build/firmware.elf
```
### 5.3 clap 库自带的方法（不需要 derive）
### 生成的 SVG 包含
```rust
// clap 库的 Command 结构体自带:
Command::get_matches_from(args: &[OsString]) → ArgMatches
                             // 字符串匹配 → 中间格式
- 每个 Sector 的范围和擦除状态
- 每个 Page 的地址和被写入的数据大小
- Fill 操作的范围
Command::ignore_errors(true)                // 忽略未知参数
Command::get_matches_from(args)             // 重新解析
```
不指定 `--flash-layout-output-path` 时，`flash_layout` 填了也没人看，但不影响性能。
## 六、Download 子命令的所有参数
### 来源：`ProbeOptions` (探针/芯片配置)
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--chip` | `Option<String>` | `None` | 芯片型号 |
| `--chip-description-path` | `Option<PathBuf>` | `None` | 外部 YAML 目标文件 |
| `--protocol` | `Option<WireProtocol>` | `None` | SWD / JTAG |
| `--speed` | `Option<u32>` | `None` | 调试接口频率 (kHz) |
| `--probe` | `Option<DebugProbeSelector>` | `None` | 指定探针 VID:PID |
| `--connect-under-reset` | `bool` | `false` | 复位状态下连接 |
| `--cycle-power` | `bool` | `false` | 连接前断电重启 |
| `--dry-run` | `bool` | `false` | 不实际操作 |
| `--non-interactive` | `bool` | `false` | 不弹交互选探针 |
### 来源：`BinaryDownloadOptions` (烧录选项)
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--binary-format` | `Option<Format>` | `None` | ELF / HEX / BIN |
| `--restore-unwritten` | `bool` | `false` | 保留未写的 flash 内容 |
| `--chip-erase` | `bool` | `false` | 整片擦除 |
| `--verify` | `bool` | `false` | 烧录后校验 |
| `--preverify` | `bool` | `false` | 烧录前校验 |
| `--disable-double-buffering` | `bool` | `false` | 禁用双缓冲 |
| `--disable-progressbars` | `bool` | `false` | 禁用进度条 |
| `--allow-erase-all` | `bool` | `false` | 允许擦除保护区 |
| `--prefer-flash-algorithm` | `Vec<String>` | `vec![]` | 首选算法名 |
| `--flash-layout-output-path` | `Option<PathBuf>` | `None` | 布局输出路径 |
| `--read-flasher-rtt` | `bool` | `false` | 读 Flash 算法 RTT |
### 默认值规则
**clap 不填就是 Rust 类型的默认值：**
| Rust 类型 | 默认值 |
|----------|--------|
| `bool` | `false` |
| `Option<T>` | `None` |
| `Vec<T>` | 空 vec |
| `u32` / `i32` | `0` |
| `String` | `""` |
### 你的 VS Code task 实际生效的参数
```
你敲的:                download --chip CYT2BL3BAS --protocol swd --speed 100 --binary-format elf build/firmware.elf
解析后:
  ProbeOptions:
    chip = "CYT2BL3BAS"    ← 你敲了
    protocol = SWD          ← 你敲了
    speed = 100             ← 你敲了
    probe = None            ← 默认
    connect_under_reset = false ← 默认
    ...                     ← 全是默认
  BinaryDownloadOptions:
    binary_format = Elf     ← 你敲了
    verify = false          ← 默认
    chip_erase = false      ← 默认
    ...                     ← 全是默认
  path = "build/firmware.elf" ← 你敲了
```
## 六A、DownloadOptions — 传递给 Flasher 的结构
`get_matches_from()` 的返回值 `ArgMatches` 是 clap 库里的一个 struct，存"匹配完的结果"：
```rust
struct ArgMatches {
    // ① 子命令 (递归嵌套)
    subcommand:    Option<(String, Box<ArgMatches>)>,
    //                 ↑ 子命令名          ↑ 子命令自己的参数又是个 ArgMatches
    // ② 参数值 (--chip CYT2BL3BAS、--speed 100)
    args:          HashMap<Id, Vec<String>>,
    //              ↑ 参数名             ↑ 值列表（一般就一个值）
    // ③ flag (--log-to-folder 这种出现就是 true 的)
    flags:         HashSet<Id>,
    //              ↑ 参数出现在命令行里 → 塞进这个集合
    // ④ 裸参数 (不跟 -- 的，如 build/firmware.elf)
    positionals:   Vec<String>,
}
```
### 填充示例
输入: `probe-rs download --chip CYT2BL3BAS --speed 100 build/firmware.elf`
```
ArgMatches {
    subcommand: Some(("download", ArgMatches {
        args: {
            "--chip":  ["CYT2BL3BAS"],
            "--speed": ["100"],
        },
        flags: {},
        positionals: ["build/firmware.elf"],
    })),
}
```
**子命令是递归的**——`download` 和 `info` 的参数不同，所以 clap 给每个子命令维护独立的 `ArgMatches`。根命令没有子命令专属参数，所以只有 `subcommand` 字段指向 `download` 的 `ArgMatches`。
## 八、parse_and_resolve_cli_args 详解 (main.rs:640)
```rust
fn parse_and_resolve_cli_args<T: FromArgMatches + CommandFactory>(
    mut args: Vec<OsString>,       // ["probe-rs", "download", "build/firmware.elf"]
    config: &Config,               // Config { presets: { "default": {...} } }
) -> Result<T> {
    // ┌── Step 1: 字符串 → ArgMatches (中间格式) ──┐
    // │  T::command() = Cli::command()              │
    // │  → 根据 Cli struct 生成"命令行定义"          │
    // │  .get_matches_from(&args)                   │
    // │  → clap 遍历 args，匹配规则，填 ArgMatches   │
    let mut matches = T::command().get_matches_from(&args);
    // ┌── Step 2: 注入预设参数 ──┐
    // │  apply_config_preset()   │
    // │  从 .probe-rs.toml 读取预设，追加到 args    │
    if apply_config_preset(config, &matches, &mut args)? {
        // ┌── Step 3: 重新解析 ──┐
        // │ 用扩展后的 args 重跑 │
        matches = T::command()
            .ignore_errors(true)    // 忽略子命令特有的参数
            .get_matches_from(args);
    }
    // ┌── Step 4: ArgMatches → Rust 结构体 ──┐
    // │  T::from_arg_matches(&matches)        │
    // │  → 把中间格式的值填到 struct 字段       │
    Ok(T::from_arg_matches(&matches)?)
}
```
## 九、apply_config_preset 详解 (main.rs:657)
```
1. 判断用哪个预设:
   --preset fast  → 用 [presets.fast]
   没指定但存在 [presets.default] → 隐式用
   都没有 → 跳过，返回 false
2. 遍历预设的每个键值对:
   [presets.default]
   chip = "CYT2BL3BAS"
   protocol = "swd"
   speed = 100
   "chip"     → --chip CYT2BL3BAS
   "protocol" → --protocol swd
   "speed"    → --speed 100
3. 跳过命令行已指定的参数（不覆盖）
   如果 args 里已经有 --speed 500 → 预设的 speed = 100 被忽略
4. 返回 true（表示 args 被修改过，需要重新解析）
```
## 十、参数优先级
```
命令行显式指定 > 预设配置 > 环境变量 > 默认值
例: --speed 1000 (命令行) > speed = 500 (preset) > PROBE_RS_SPEED (env)
```
## 十一、日志文件自动创建 (main.rs 563-574)
```rust
if cli.log_file.is_none() && (cli.log_to_folder || cli.report.is_some()) {
    let location = default_logfile_location()?;   // 生成默认路径
    prune_logs(location.parent()?)?;              // 清理旧日志（保留最近20个）
    cli.log_file = Some(location);                // 设上
}
```
触发条件：用了 `--log-to-folder` 或 `--report`，但没指定 `--log-file`。
## 十二、源码位置速查
| 文件 | 行号 | 说明 |
|------|:--:|------|
| `main.rs` | 48-109 | `struct Cli` 定义 + `#[derive(clap::Parser)]` |
| `main.rs` | 536 | `main()` 入口 |
| `main.rs` | 563-574 | 日志自动创建 |
| `main.rs` | 640-655 | `parse_and_resolve_cli_args()` |
| `main.rs` | 657-712 | `apply_config_preset()` |
| `cmd/download.rs` | 10-23 | `Cmd` (Download 子命令) 结构体 |
| `rpc/functions/flash.rs` | 46-63 | `sanitize()` — 清理 `--prefer-flash-algorithm` 输入 |
## 十三、`sanitize()` — 参数清理 (rpc/functions/flash.rs:47)
`--prefer-flash-algorithm` 允许用户指定首选 Flash 算法名，但用户可能手抖多打了引号或空格。`sanitize()` 把输入洗干净：
```rust
pub fn sanitize(&mut self) {
    for algo in self.preferred_algos.iter_mut() {
        *algo = algo
            .trim()                                   // 去首尾空格
            .trim_matches(|c| c == '\'' || c == '"')  // 去首尾引号
            .chars().filter(|c| !c.is_whitespace())    // 去中间空格
            .collect();
    }
    self.preferred_algos.retain(|s| !s.is_empty());   // 去空串
}
```
| 用户输入 | sanitize 之后 |
|---------|-------------|
| `"cyt2bl"` | `cyt2bl` |
| `'  algo1 , algo2 '` | `algo1,algo2` |
| `algo1,, algo2` | `algo1` `algo2`（空串被移除） |
没传 `--prefer-flash-algorithm` 时该字段为空 vec，`sanitize()` 直接跳过。
| `util/common_options.rs` | 89-138 | `ProbeOptions` (探针配置) |