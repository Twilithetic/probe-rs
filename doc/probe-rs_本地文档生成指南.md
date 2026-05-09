# probe-rs 本地文档生成指南

## 一键生成

在项目根目录（`tools/probe-rs-src/`）下运行：

```pwsh
cargo doc --no-deps -p probe-rs
```

生成后在浏览器打开：

```pwsh
start target/doc/probe_rs/index.html
```

## 命令拆解

| 参数 | 作用 | 可以不写吗？ |
|------|------|:--:|
| `cargo doc` | 核心：生成 Rustdoc 文档 | 必须 |
| `--no-deps` | 跳过依赖库（不然等很久） | 建议写 |
| `-p probe-rs` | 只生成 probe-rs 这个 crate | 建议写 |

## 其他常用变体

```pwsh
# 只看 vendor 相关
start target/doc/probe_rs/vendor/index.html

# 只看 ArmDebugSequence trait
start target/doc/probe_rs/architecture/arm/sequences/trait.ArmDebugSequence.html

# 只看 plugin 模块（本地独有）
start target/doc/probe_rs/plugin/index.html

# 搜索功能：浏览器里按 S 键
```

## 和 docs.rs 的关系

```
docs.rs 做的事:
  1. 从 crates.io 下载源码
  2. 跑 cargo doc --no-deps
  3. 把 target/doc/ 里的 HTML 托管到网站上

你做的事:
  cargo doc --no-deps -p probe-rs
  → 效果完全一样，只是搭在了本地 target/doc/
```

## 生成全部 crate 的文档

```pwsh
cargo doc --no-deps
```

会生成 workspace 里全部 9 个 crate 的文档：
`probe-rs` / `probe-rs-target` / `probe-rs-tools` / `probe-rs-espressif` / `probe-rs-debug` / `probe-rs-mi` / `target-gen` / `xtask` / `smoke-tester-macros`

### ⚠️ 全量生成后要补一刀

全量生成有个坑：`probe-rs-tools`（二进制）和 `probe-rs`（库）的文档会互相覆盖——因为 `probe-rs-tools` 的 crate 名也叫 `probe-rs`，它俩的 HTML 都写到 `target/doc/probe_rs/` 下面。后生成的会盖掉先生成的，导致 library 文档里的 `plugin`、`vendor` 等模块可能丢失。

**所以在全量生成后，必须再单独跑一次 library 的文档盖回来：**

```pwsh
cargo doc --no-deps               # 生成全部 → probe-rs library 文档可能被盖
cargo doc --no-deps -p probe-rs   # ← 补刀！把 library 文档盖回来
```

## 注意

- 生成的文件在 `target/doc/` 下面
- 每次跑 `cargo clean` 会删掉
- 本地文档比 docs.rs 新——包含你改的 Cyt2bl 序列、plugin 模块等
