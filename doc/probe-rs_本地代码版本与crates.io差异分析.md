# probe-rs 本地代码版本判定与 crates.io 发布版差异分析

> **日期**: 2026-05-06  
> **目的**: 解释本地 probe-rs 代码与 docs.rs/crates.io 上 v0.31.0 的关系

---

## 1. Git 历史还原

### 1.1 关键提交

| 提交 | 日期 | 说明 |
|------|------|------|
| `eab9c5e0` | 2025-10-31 | Release 0.30.0 (#3618) |
| `e1358c0f` | 2026-01-17 | Release 0.31.0 (#3758) — 首次尝试 |
| `130b5f59` | 2026-01-17 | Revert "Release 0.31.0" (#3760) |
| `4f008cdf` | 2026-01-17 | **Release 0.31.0 (#3762) — 实际发布** |
| `0a81a32c` | 2026-01-31 | Vendor code as plugins (#3813) — **plugin 模块诞生** |
| `c78aa370` | 2026-05-06 | **本地 HEAD** (含 Cyt2bl + flasher 改动) |

### 1.2 时间线

```
2025-10-31  ■ v0.30.0 发布 (eab9c5e0)
                ↓
2026-01-17  ■ v0.31.0 发布 (4f008cdf)
                │  crate.io 发布: 2026-01-17 19:09 UTC
                │  docs.rs 构建用的就是这个
                │
                │  ← CRATES.IO / DOCS.RS 版本在此
                │
2026-01-31  ■ plugin 模块添加 (0a81a32c, PR #3813)
                │  距 v0.31.0 发布仅 14 天
                │
                ↓  ... 154 个提交 ...
                │
2026-05-06  ■ 本地 HEAD (c78aa370)
                │  距 v0.31.0 发布 109 天 (~3.5 个月)
                │  共 156 个新提交
                │  含 plugin、Cyt2bl、flasher 改动等
```

---

## 2. 判断方法（用到的 Git 命令）

### 2.1 查看 Cargo.toml 版本号

```bash
# 当前本地版本
grep "^version" Cargo.toml
# → version = "0.31.0"
```

### 2.2 查找版本变更历史

```bash
# 搜索所有改动过 "0.31.0" 字符串的提交
git log --all --oneline -S "0.31.0" -- Cargo.toml
# 结果:
#   0a81a32c Vendor code as plugins (#3813)
#   4f008cdf Release 0.31.0 (#3762)      ← 发布提交
#   130b5f59 Revert "Release 0.31.0" (#3760)
#   e1358c0f Release 0.31.0 (#3758)
```

`-S "0.31.0"` 是 "pickaxe" 搜索——查找**添加或删除**了指定字符串的提交。

### 2.3 查看提交日期

```bash
# 查看特定提交的日期
git log --format="%h %ai %s" 4f008cdf -1
# → 4f008cdf 2026-01-17 19:57:36 +0100 Release 0.31.0 (#3762)

git log --format="%h %ai %s" 0a81a32c -1
# → 0a81a32c 2026-01-31 21:46:15 +0100 Vendor code as plugins (#3813)
```

### 2.4 计算发布后的提交数

```bash
# 从 v0.31.0 发布到 HEAD 有多少个提交
git rev-list --count 4f008cdf..HEAD
# → 156
```

### 2.5 查看 plugin.rs 的创建时间

```bash
# 查看 plugin.rs 第一次被添加的提交
git log --oneline --diff-filter=A -- probe-rs/src/plugin.rs
# → 0a81a32c Vendor code as plugins (#3813)
```

`--diff-filter=A` 意思是只显示 **Added**（新建文件）的提交。

### 2.6 查询 crates.io 发布时间

```bash
# 用 crates.io API 查询
curl https://crates.io/api/v1/crates/probe-rs/versions
# → 0.31.0: created_at=2026-01-17T19:09:11
```

### 2.7 检查是否有 git tag

```bash
# 列出所有 tag
git tag -l
# → (空)  ← 没有 tag!
```

没有 tag 说明开发者没有用 `git tag v0.31.0` 标记发布点，只改了 Cargo.toml 版本号就发布到了 crates.io。

---

## 3. 总结

### 3.1 本地代码是什么版本

```
Cargo.toml 写的: v0.31.0
实际代码:      v0.31.0 + 156 个未发布提交
                    = "v0.31.0-dev" 或 "master 分支"
```

### 3.2 为什么 docs.rs 没有 plugin

```
crates.io v0.31.0 发布于 2026-01-17
    → 无 plugin 模块
    → docs.rs 构建此版本 → 无 plugin 文档

plugin 模块 添加于 2026-01-31
    → v0.31.0 发布后 14 天
    → 仅在 git master，未发布到 crates.io
    → docs.rs 无法看到
```

### 3.3 判断是否同步的通用方法

| 要判断什么 | 命令/方法 |
|-----------|---------|
| 本地版本号 | `grep "^version" Cargo.toml` |
| 版本什么时候发布的 | `git log -S "0.31.0" -- Cargo.toml` |
| 某个文件什么时候加的 | `git log --diff-filter=A -- path/to/file` |
| 发布后有多少新提交 | `git rev-list --count <release-commit>..HEAD` |
| crates.io 发布时间 | `curl https://crates.io/api/v1/crates/<name>/versions` |
| 有没有 git tag | `git tag -l` |

*本报告由知心姐姐编写 💖*
