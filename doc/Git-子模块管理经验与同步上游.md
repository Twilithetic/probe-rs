# Git 子模块管理经验报告

> 📅 日期：2026-05-06  
> 🏠 项目：CYT2BL3 Wheel-leg  
> 👤 操作者：Twilithetic

---

## 一、问题：嵌套 Git 仓库警告

### 现象

执行 `git add tools/infineon-packs` 后，Git 提示：

```
hint: You've added another git repository inside your current repository.
hint: Clones of the outer repository will not contain the contents of
hint: the embedded repository and will not know how to obtain it.
```

### 原因

`tools/infineon-packs` 目录内包含独立的 `.git` 目录（从 `https://github.com/Infineon/cmsis-packs.git` clone 下来的），Git 检测到这是嵌套仓库。

**技术原理**：Git 不支持嵌套仓库。当你 `git add` 一个包含 `.git` 的子目录时，Git 只会在 index 中记录一个 **gitlink**（160000 模式），指向内层仓库的某个 commit，但**不会把内层仓库的文件纳入版本控制**。别人 clone 时拿不到这些文件。

### 解决方案

使用 **Git Submodule（子模块）**：

```bash
# 1. 先从暂存区移除旧的记录
git rm --cached -f tools/infineon-packs

# 2. 添加为正规子模块
git submodule add https://github.com/Infineon/cmsis-packs.git tools/infineon-packs
```

**注意**：如果目录已存在且有 `.git`，`git submodule add` 会智能地"吸收"现有仓库，不会重新下载。

---

## 二、排查：找出所有子模块和孤儿 gitlink

### 关键命令

```bash
# 1. 查看已注册子模块状态
git submodule status

# 2. 查看 .gitmodules 配置文件
cat .gitmodules    # Linux/Mac
Get-Content .gitmodules    # Windows PowerShell

# 3. 搜索嵌套 .git 目录（找出未注册的仓库）
Get-ChildItem -Recurse -Directory -Filter ".git" -Depth 5

# 4. 查找 index 中所有 gitlink 条目（160000 模式 = 子模块指针）
git ls-files --stage | Select-String "^160000"
```

### 发现的三种状态

| 状态 | 含义 | 如何处理 |
|------|------|----------|
| `.gitmodules` 中有，目录存在 | ✅ 正规子模块 | 无需处理 |
| `.gitmodules` 中无，index 中有 gitlink | ⚠️ 孤儿 gitlink | 补注册或清理 |
| `.gitmodules` 中无，目录也不存在 | 👻 幽灵 gitlink | `git rm --cached` 清理 |

---

## 三、操作：注册孤儿子模块

### 项目中发现的孤儿 gitlink 及其远程地址

| 路径 | 远程仓库 |
|------|----------|
| `libs/CMSIS_5` | `https://github.com/ARM-software/CMSIS_5.git` |
| `libs/FreeRTOS` | `https://github.com/FreeRTOS/FreeRTOS-Kernel.git` |
| `libs/hal` | `https://github.com/Infineon/mtb-hal-cat1.git` |
| `libs/pdl` | `https://github.com/Infineon/mtb-pdl-cat1.git` |
| `tools/CMSIS-DAP` | `https://github.com/ARM-software/CMSIS-DAP` |
| `tools/probe-rs-src` | `https://github.com/probe-rs/probe-rs.git` |

### 批量注册流程

对每个孤儿 gitlink，执行：

```bash
# 1. 从 index 移除旧 gitlink
git rm --cached <路径>

# 2. 用 submodule add 吸收本地仓库
git submodule add <远程URL> <路径>
```

---

## 四、进阶：子模块内修改的保存（Fork 模式）

### 问题

对子模块 `tools/probe-rs-src` 做了本地修改（新增 `CYT2BL_Series.yaml`），如何让修改被保存 + 别人 clone 时也能拿到？

### 子模块工作原理

```
父仓库只记录一个"指针"：
  tools/probe-rs-src → 指向子模块的某个 commit hash

当别人 clone + submodule update 时：
  去 .gitmodules 里的 URL 下载该 commit
  ❌ 如果你的修改 commit 只在你本地，远程不存在 → 失败！
```

### 解决方案：Fork 模式

#### 完整流程

```bash
# === 第1步：在 GitHub 上 Fork 原仓库 ===
# 浏览器打开 https://github.com/probe-rs/probe-rs → 点 Fork

# === 第2步：在子模块中 commit 修改 ===
cd tools/probe-rs-src
git add .
git commit -m "Add CYT2BL_Series target definition for probe-rs"

# === 第3步：添加你的 fork 为 remote ===
git remote add myfork https://github.com/Twilithetic/probe-rs.git

# === 第4步：同步上游最新 + 叠上你的修改 ===
git pull myfork master --rebase    # 把你的 commit rebase 到最新

# === 第5步：推送到你的 fork ===
git push myfork master

# === 第6步：在父仓库中注册子模块（指向 fork）===
cd ../..
git rm --cached tools/probe-rs-src
git submodule add https://github.com/Twilithetic/probe-rs.git tools/probe-rs-src
```

### 以后如何同步上游更新

```bash
cd tools/probe-rs-src
git fetch origin              # 只下载，不动你的代码（安全！）
git rebase origin/master      # 把你的 commit 叠到最新的上面
git push myfork master        # 推到你的 fork
```

---

## 五、关键知识点

### 1. `git fetch` vs `git pull`

| 命令 | 作用 | 是否修改工作区 | 安全性 |
|------|------|:---:|:---:|
| `git fetch` | 从远程下载数据到本地仓库（`.git/refs/remotes/`） | ❌ 不修改 | 🟢 绝对安全 |
| `git pull` | `git fetch` + `git merge/rebase` | ✅ 会修改 | 🟡 可能冲突 |

> **类比**：`fetch` = 去快递站看一眼有什么新包裹但不取；`pull` = 取包裹并当场拆开塞进工作区。

### 2. gitlink（160000 模式）

- Git 用 `160000` 文件模式标记子模块
- 在 index 中，子模块只有一个 commit hash 指针，不含实际文件
- `.gitmodules` 文件负责记录子模块的名称、路径和 URL

### 3. 子模块克隆注意事项

别人 clone 你的项目后，需要额外操作才能获得子模块内容：

```bash
# 方式1：clone 时一并初始化
git clone --recurse-submodules <仓库URL>

# 方式2：clone 后再初始化
git clone <仓库URL>
git submodule update --init --recursive
```

---

## 六、最终子模块清单

| # | 子模块路径 | 来源 | 
|---|-----------|------|
| 1 | `libs/core-lib` | `Infineon/core-lib` |
| 2 | `tools/infineon-packs` | `Infineon/cmsis-packs` |
| 3 | `libs/CMSIS_5` | `ARM-software/CMSIS_5` |
| 4 | `libs/FreeRTOS` | `FreeRTOS/FreeRTOS-Kernel` |
| 5 | `libs/hal` | `Infineon/mtb-hal-cat1` |
| 6 | `libs/pdl` | `Infineon/mtb-pdl-cat1` |
| 7 | `tools/CMSIS-DAP` | `ARM-software/CMSIS-DAP` |
| 8 | `tools/probe-rs-src` | `Twilithetic/probe-rs` 🔗（Fork） |

---

## 七、常用命令速查表

```bash
# 添加子模块
git submodule add <URL> <路径>

# 查看子模块状态
git submodule status

# 初始化并更新所有子模块
git submodule update --init --recursive

# 更新子模块到远程最新
git submodule update --remote

# 删除子模块
git rm <路径>
# 然后手动删除 .git/modules/<路径> 目录

# 查找所有 gitlink
git ls-files --stage | grep "^160000"

# 清除孤儿 gitlink（目录已不存在）
git rm --cached <路径>

# 查看 .gitmodules 内容
cat .gitmodules
```

---

> 💡 **经验总结**：遇到嵌套仓库不要慌，`git submodule add` 一键转正！修改子模块用 Fork 模式最稳妥，`git fetch` 比 `git pull` 安全一百倍~
