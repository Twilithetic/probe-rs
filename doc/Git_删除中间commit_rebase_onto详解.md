# Git 删除中间某个 commit — `rebase --onto` 详解

> 📅 整理日期：2026-05-10  
> 🎯 场景：删掉历史中间的某个 commit，保留后面的 commit

---

## 一、问题场景

```
cb0d151d  ← 好的
c78aa370  ← 😈 这个改错了，要删掉！
5c087d87  ← 好的，要保留
37d2a4e4  ← 好的，要保留
538eb703  ← 好的，要保留 (HEAD)
```

**需求**：删掉中间的 `c78aa370`，后面的 3 个 commit 不能丢。

---

## 二、核心思路

Git 的 commit 是**不可变的**。不能真正"删除"一个 commit，只能**重写历史**——把后面的 commit 重新"嫁接"到前面的 parent 上：

```
嫁接前:                          嫁接后:
                                  (新)
cb0d151d (parent)                cb0d151d ← 还是这个
    │                                │
c78aa370 ← 要被跳过 ❌             5c087d87' ← 重新嫁接
    │                                │
5c087d87 ← 要保留                  37d2a4e4' ← 重新嫁接
    │                                │
37d2a4e4 ← 要保留                  538eb703' ← 重新嫁接
    │
538eb703 ← 要保留

c78aa370 没有被嫁接 → 消失了！
```

---

## 三、用到的命令

### 命令拆解

```bash
git rebase --onto cb0d151d c78aa370
#          ─────        ────────
#            │              │
#    新 parent (接在哪)   旧 parent (从哪之后开始接)
```

完整含义：

```
git rebase --onto <new_parent> <old_parent>
```

把 `<old_parent>..HEAD` 之间的所有 commit，逐个 reapply 到 `<new_parent>` 上。

### 用图理解

```
git rebase --onto cb0d151d c78aa370

     cb0d151d                  c78aa370
     (新 parent)              (旧 parent)
         ↓                       ↓
    ┌────┴────┐              ┌───┴───┐
    │ 接在这里  │              │ 从这之后 │
    └────┬────┘              │ 的开始接 │
         │                   └───┬───┘
         │                       │
         │     5c087d87 ←────────┘
         ├──→ 37d2a4e4 ←────────┘
         ├──→ 538eb703 ←────────┘
         │
    c78aa370 被跳过！
```

---

## 四、完整操作流程

```bash
# ① 先建备份（好习惯！）
git branch backup-before-rebase

# ② 查看现状
git log --oneline --graph -6

# ③ 🔥 核心操作：跳过 c78aa370
git rebase --onto cb0d151d c78aa370

# ④ 验证
git log --oneline --graph -6
```

### 执行过程

```
$ git rebase --onto cb0d151d c78aa370
Rebasing (1/3)    ← 5c087d87 重新嫁接到 cb0d151d
Rebasing (2/3)    ← 37d2a4e4 重新嫁接到 5c087d87'
Rebasing (3/3)    ← 538eb703 重新嫁接到 37d2a4e4'
Successfully rebased and updated refs/heads/AI_commitocn.
```

---

## 五、结果对比

```
操作前:                             操作后:
* 538eb703 等待改错                 * 78af9c51 等待改错     ← 内容相同，hash 变了
* 37d2a4e4 新的知识                 * 6f8c4ac8 新的知识     ← 内容相同，hash 变了
* 5c087d87 找回了！                 * 32d779b8 找回了！     ← 内容相同，hash 变了
* c78aa370 😈 改错了                * cb0d151d Add CYT2BL...
* cb0d151d Add CYT2BL...            * 5bbfca34 Generalize...
```

- `c78aa370` → 消失！
- 后面 3 个 commit 内容保留，hash 改变（因为 parent 变了）
- 备份分支 `backup-before-rebase` 保留原历史

---

## 六、为什么 hash 会变？

Git 的 commit hash = `SHA1(内容 + parent + author + timestamp + ...)`

当 parent 从 `c78aa370` 变成 `cb0d151d`，hash 必然改变——但这不影响内容。

---

## 七、备选方案对比

| 方案 | 命令 | 适用场景 |
|------|------|---------|
| **rebase --onto**（本次用） | `git rebase --onto parent bad` | 删中间的 commit |
| reset + cherry-pick | `git reset --hard parent` → `git cherry-pick ...` | 同上，但更手动 |
| revert | `git revert c78aa370` | 已 push 到远程、不能改历史时 |
| rebase -i | `git rebase -i parent` → 标记 `drop` | 交互式，需要编辑器 |

---

## 八、如果出错了怎么办？

```bash
# 回到 rebase 前的状态
git reset --hard backup-before-rebase

# 或者用 reflog 找回
git reflog
git reset --hard <rebase前的HEAD>
```

---

## 九、记忆口诀

```
git rebase --onto 接到哪  从哪之后开始接
              ───        ───────────
           new parent    old parent (这个会被跳过！)
```

📌 **`--onto` 后面的第一个参数是 "接在谁后面"，第二个参数是 "从谁之后开始嫁接"。第二个参数本身不会被嫁接（被跳过了）！**
