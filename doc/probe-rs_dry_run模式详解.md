# probe-rs `dry_run` 模式详解

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：`--dry-run` 的完整链路、用途、与正常模式的区别  
> 📖 上下文：`loader.commit()` 中的 dry_run 分支，补充第三篇执行追踪

---

## 一、一句话总结

`dry_run` 是 "彩排模式"：解析 ELF、匹配算法、制定计划全部走一遍，但不碰硬件、不写芯片，最后返回 `Ok(())`。

---

## 二、完整链路

```
CLI                            Daemon                        核心库
───                            ──────                        ────
probe-rs download --dry-run
    │
    ├─ attach 阶段:
    │    ProbeOptions.dry_run = true
    │      ↓
    │    FakeProbe::with_mocked_core()
    │    不连 USB，模拟一个"完美芯片"
    │
    ├─ 存标记:
    │    ctx.set_session(session, dry_run=true)  ──→
    │                                              ConnectionState
    │                                              .dry_run_sessions
    │                                              .insert(key)
    │
    └─ flash 阶段:
         flash_impl():
           dry_run = ctx.dry_run(sessid)       ←── 读回
           options.dry_run = true                ←── 传入
             ↓
         loader.commit():
           if options.dry_run {
               progress.failed_filling();        ← 跳过填充
               progress.failed_erasing();        ← 跳过擦除
               progress.failed_programming();    ← 跳过编程
               return Ok(());                    ← ✅ 成功!
           }
```

---

## 三、源码节点

### ① CLI 入口

```rust
// common_options.rs:140
#[arg(long, env = "PROBE_RS_DRY_RUN")]
pub dry_run: bool,
```

```bash
probe-rs download firmware.elf --dry-run --chip CYT2BL3BAE
# 或
PROBE_RS_DRY_RUN=1 probe-rs download firmware.elf --chip CYT2BL3BAE
```

### ② 假探针

```rust
// common_options.rs:274
let mut probe = if self.0.dry_run {
    Probe::from_specific_probe(Box::new(FakeProbe::with_mocked_core()))
    //                           ↑ 模拟芯片，不连 USB
} else {
    lister.open(selector)?  // 真正打开探针
};
```

### ③ 存标记到 Session

```rust
// rpc/mod.rs:174
pub async fn set_session(&mut self, session: Session, dry_run: bool) -> Key<Session> {
    if dry_run {
        self.dry_run_sessions.insert(key);
    }
    // ...
}
```

### ④ flash_impl 读回

```rust
// rpc/functions/flash.rs:382
let dry_run = ctx.dry_run(request.sessid);
options.dry_run = dry_run;
```

### ⑤ commit() 判断

```rust
// probe-rs/src/flashing/loader.rs:679
if options.dry_run {
    tracing::info!("Skipping programming, dry run!");
    options.progress.failed_filling();
    options.progress.failed_erasing();
    options.progress.failed_programming();
    return Ok(());
}
```

---

## 四、为什么叫 `failed_xxx` 而不是 `skipped_xxx`？

`FlashProgress` API 的状态机只有三种终态：

```rust
progress.started_filling();     // 开始
progress.finished_filling();    // ✅ 正常完成
progress.failed_filling();      // ❌ 没正常完成（包括"跳过"）
```

`failed_xxx()` 本质上告诉进度条："这个操作没有执行，不用等了"。在 dry run 场景语义是 **"跳过"**，但复用同一 API。

---

## 五、正常 vs dry run

```
正常模式:                          dry run 模式:
┌──────────────────────┐          ┌──────────────────────┐
│ ① 解析 ELF ✅        │          │ ① 解析 ELF ✅        │
│ ② 匹配算法 ✅        │          │ ② 匹配算法 ✅        │
│ ③ 擦除 Flash 🔥      │          │ ③ 跳过 ⏭️            │
│ ④ 编程 Flash 🔥      │          │ ④ 跳过 ⏭️            │
│ ⑤ 校验 ✅           │          │ ⑤ 跳过 ⏭️            │
│ ⑥ 写 RAM ✅         │          │ ⑥ 跳过 ⏭️            │
│ → 返回 Ok(())        │          │ → 返回 Ok(())        │
│ 耗时: 5-30秒         │          │ 耗时: <1秒            │
│ 硬件: 需要探针+芯片   │          │ 硬件: 不需要！        │
└──────────────────────┘          └──────────────────────┘
```

---

## 六、用途

| 场景 | 说明 |
|------|------|
| CI/CD | 无硬件的服务器上验证 ELF 可解析、算法可匹配 |
| 预检 | 烧录前确认 ELF 地址合法、有对应 Flash 算法 |
| 开发调试 | 不连硬件测试 RPC 链路和构建逻辑 |
| 演示 | 展示流程但不需要实际芯片 |
