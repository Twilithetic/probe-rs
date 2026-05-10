# probe-rs FlashRequest 完整执行追踪

> 📅 整理日期：2026-05-10  
> 🎯 追踪目标：从 client 端 `flash()` 到芯片硬件烧录的全路径  
> 📖 三部曲之一：配合 `probe-rs_flash函数_RPC调用链详解.md` 和 `probe-rs_RPC传输层_postcard-rpc框架详解.md` 阅读

---
🗺️ FlashRequest 的完整旅程
```
你贴的代码就在这里 ─────────┐
                            ▼
┌─ Client 端 ──────────────────────────────────────────────────────────┐
│                                                                      │
│  cli.rs: flash()                                                     │
│    │                                                                 │
│    └→ session.flash(options, loader, rtt, on_msg)                    │
│         │                                                            │
│         └→ client.rs: pub async fn flash(...)   ← 🎯 你贴的代码      │
│              │                                                       │
│              └→ self.client.send_and_read_stream::<FlashEndpoint,    │
│                                         ProgressEventTopic, _>(...)  │
│                   │                                                  │
│                   │  ① 订阅 Topic "flash/progress"                   │
│                   │  ② 发送 Endpoint "flash/flash"                   │
│                   │     (并发执行 via tokio::join!)                   │
│                   │                                                  │
│     ════════════ 跨 channel / WebSocket / Unix Socket ═══════════    │
│                   │                                                  │
└───────────────────┼──────────────────────────────────────────────────┘
                    │
┌─ Daemon 端 ───────┼──────────────────────────────────────────────────┐
│                   ▼                                                  │
│  postcard-rpc Server::run() 事件循环                                 │
│    │                                                                 │
│    │ 读帧 → 解析 VarHeader → key 匹配路由表                          │
│    │                                                                 │
│    ├─ key "flash/flash" → FlashEndpoint → handler: flash()           │
│    │     │                                                           │
│    │     └→ functions/flash.rs:372                                   │
│    │          pub async fn flash(ctx, _header, request) {            │
│    │              ctx.run_blocking::<ProgressEventTopic, _, _, _>(   │
│    │                  request, flash_impl                            │
│    │              ).await                                             │
│    │          }                                                      │
│    │                                                                 │
│    └─ 同时: Topic "flash/progress" → ProgressEventTopic              │
│          等待 flash_impl 推送事件                                     │
│                                                                      │
│  ↓ 进入 flash_impl() (阻塞线程池)                                    │
│                                                                      │
│  functions/flash.rs:377 flash_impl()                                 │
│    │                                                                 │
│    ├─ ① let session = ctx.session_blocking(sessid);                  │
│    │      → 根据 Key<Session> 取出真正的 Session 对象                │
│    │                                                                 │
│    ├─ ② let loader = ctx.object_mut_blocking(loader_key);            │
│    │      → 根据 Key<FlashLoader> 取出 FlashLoader                   │
│    │      → 里面有: ELF段信息 + 匹配的Flash算法 + memory_map         │
│    │                                                                 │
│    ├─ ③ 设置进度回调 → sender.blocking_send(ProgressEvent)           │
│    │                                                                 │
│    └─ ④ loader.commit(&mut session, options)   ← 🔥 真正的硬件操作   │
│                                                                      │
└───────────────────────┼──────────────────────────────────────────────┘
                        │
┌─ 核心库 probe-rs ─────┼──────────────────────────────────────────────┐
│                       ▼                                              │
│  loader.rs:667  pub fn commit(&self, session, options) {              │
│    │                                                                 │
│    ├─ ① prepare_plan() — 匹配 Flash 算法到内存区域                   │
│    │     CYT2BL3 的 Flash 算法: cyt2bl / cyt2bx_wflash_128           │
│    │                                                                 │
│    ├─ ② initialize() — 加载 Flash 算法到芯片 RAM                     │
│    │     → 通过 SWD 写数据到芯片 SRAM                                │
│    │     → 设置 SP(堆栈指针)、PC(程序计数器)                         │
│    │                                                                 │
│    ├─ ③ for each flasher in algos:                                  │
│    │     │                                                           │
│    │     ├─ [可选] run_erase_all() → 整片擦除                        │
│    │     │                                                           │
│    │     └─ flasher.program()  ← 🎯 逐页编程                         │
│    │           │                                                     │
│    │           ├─ fill_unwritten() — 恢复不写的字节(如果开启)         │
│    │           ├─ sector_erase() — 擦除需要写的 sector               │
│    │           ├─ do_program() — 逐页编程                            │
│    │           │     │                                               │
│    │           │     ├─ program_simple: 单缓冲                       │
│    │           │     │    for page in pages:                         │
│    │           │     │      active.program_page(page)                │
│    │           │     │                                              │
│    │           │     └─ program_double_buffer: 双缓冲（更快）         │
│    │           │          for page in pages:                         │
│    │           │            ① load_page_buffer(data, buf) → 写RAM    │
│    │           │            ② wait_for_write_end(addr) → 等完成      │
│    │           │            ③ start_program_page_with_buffer(...)    │
│    │           │                                                │
│    │           └─ verify() — 读回校验                                │
│    │                                                                 │
│    └─ ④ commit_ram() — 如果有 RAM 段，写入 RAM                      │
│        → 如果是 RAM boot，设置向量表并 reset 核心                    │
│                                                                      │
└───────────────────────┼──────────────────────────────────────────────┘
                        │
┌─ 芯片硬件层 ──────────┼──────────────────────────────────────────────┐
│                       ▼                                              │
│  ActiveFlasher::program_page(page)                                   │
│    │                                                                 │
│    │  ① 把 page 数据通过 SWD 写到芯片 RAM 的 page buffer             │
│    │     core.write_8(buffer_address, page.data())                   │
│    │                                                                 │
│    │  ② 设置 CPU 寄存器 → 调用 Flash 算法里的 ProgramPage 函数:      │
│    │     ┌──────────────────────────────────────────┐               │
│    │     │ R0 = flash_address    (要烧到哪个地址)     │               │
│    │     │ R1 = page_size        (一页多大)          │               │
│    │     │ R2 = buffer_address   (数据在RAM的哪里)    │               │
│    │     │ PC = Flash算法入口    (芯片Flash控制器的  │               │
│    │     │       ProgramPage函数  操作代码)          │               │
│    │     │ SP = stack_top        (算法用的栈)        │               │
│    │     │ R9 = static_base      (算法全局变量区)    │               │
│    │     └──────────────────────────────────────────┘               │
│    │                                                                 │
│    │  ③ core.run() → 让芯片 CPU 开始执行 Flash 算法                  │
│    │     芯片内部: Flash 控制器 → 高压编程 → 写入 Flash 单元          │
│    │                                                                 │
│    │  ④ wait_for_completion() → 轮询 CPU 状态                        │
│    │     while core.status() != Halted:                              │
│    │         sleep(poll_interval)                                    │
│    │     → 芯片执行完 Flash 算法，自动 halt                          │
│    │                                                                 │
│    │  ⑤ 读 R0 返回值 → 0=成功, 非0=失败地址                         │
│    │                                                                 │
│    └─→ 发送 ProgressEvent::Progress { operation: Program, size }    │
│         → sender.blocking_send() → Topic → client on_msg 回调        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```
---

## 二、逐层详析

### 2.1 Client 端：打包发送

**文件**: `probe-rs-tools/src/bin/probe-rs/rpc/client.rs:517`

```rust
pub async fn flash(
    &self,
    options: DownloadOptions,
    loader: Key<FlashLoader>,
    rtt_client: Option<Key<RttClient>>,
    on_msg: impl AsyncFnMut(ProgressEvent),
) -> anyhow::Result<()> {
    self.client
        .send_and_read_stream::<FlashEndpoint, ProgressEventTopic, _>(
            &FlashRequest { sessid: self.sessid, loader, options, rtt_client },
            on_msg,
        )
        .await
}
```

`send_and_read_stream` 内部通过 `tokio::join!` 并发：
- **订阅** `ProgressEventTopic` → 接收进度推送
- **发送** `FlashRequest` → Endpoint `"flash/flash"`

### 2.2 RPC 路由：VarHeader key 匹配

postcard-rpc 的 `Server::run()` 事件循环：
1. 读帧 → 解析 VarHeader
2. 根据 `key` 查路由表（`endpoints!` 宏生成的 dispatch 代码）
3. `key("flash/flash")` → `FlashEndpoint` → handler: `flash()`

```rust
// functions.rs:544 — 端点注册
| FlashEndpoint | async | flash |
```

### 2.3 Daemon 端 handler

**文件**: `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs:372`

```rust
pub async fn flash(ctx: &mut RpcContext, _header: VarHeader, request: FlashRequest) -> NoResponse {
    ctx.run_blocking::<ProgressEventTopic, _, _, _>(request, flash_impl).await
}
```

**`run_blocking` 的作用**：把硬件操作（同步阻塞）放到 `tokio::spawn_blocking` 的独立线程池，不阻塞 async runtime。

### 2.4 flash_impl：对象取出 + 装配

**文件**: `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs:377`

```rust
fn flash_impl(ctx: RpcSpawnContext, request: FlashRequest, sender: Sender<ProgressEvent>) -> NoResponse {
    // ① 根据 Key 取回 Session
    let mut session = ctx.session_blocking(request.sessid);

    // ② 根据 Key 取回 FlashLoader
    let loader = ctx.object_mut_blocking(request.loader);

    // ③ 如果有 RTT，用 FlashLoader 信息配置它
    if let Some(rtt) = rtt_client.as_mut() {
        rtt.configure_from_loader(&loader);
    }

    // ④ 装配 DownloadOptions + 设置进度回调
    let mut options = request.download_options();
    options.progress = FlashProgress::new(move |event| {
        ProgressEvent::from_library_event(event, |event| {
            sender.blocking_send(event).unwrap(); // → 推送 ProgressEvent 给 Client
        });
    });

    // ⑤ 🔥 真正的硬件烧录
    loader.commit(&mut session, options)?;
    Ok(())
}
```

### 2.5 loader.commit()：Flash 烧录总入口

**文件**: `probe-rs/src/flashing/loader.rs:667`

```rust
pub fn commit(&self, session: &mut Session, mut options: DownloadOptions) -> Result<(), FlashError> {
    // ① 匹配 Flash 算法：给每个内存区域分配对应的 Flash Algorithm
    let mut algos = self.prepare_plan(session, options.keep_unwritten_bytes, &options.preferred_algos)?;

    // ② 初始化：把 Flash 算法加载到芯片 RAM，设置 SP/PC
    self.initialize(&mut algos, session, &mut options)?;

    // ③ 遍历所有 Flash 算法，逐个区域烧录
    for mut flasher in algos {
        // [可选] 整片擦除
        if do_chip_erase {
            flasher.run_erase_all(session, &mut options.progress)?;
        }

        // 🔥 逐页编程
        flasher.program(session, &mut options.progress, ...)?;
    }

    // ④ 写入 RAM 段（如果有 .data/.bss 等）
    for region in self.memory_map.iter().filter_map(MemoryRegion::as_ram_region) {
        // 写 RAM 数据
    }

    // ⑤ 如果是 RAM boot，reset 核心
    if let BootInfo::FromRam { .. } = self.boot_info() { ... }
}
```

### 2.6 flasher.program()：擦除 → 编程 → 校验

**文件**: `probe-rs/src/flashing/flasher.rs:397`

```rust
pub(super) fn program(&mut self, session, progress, restore_unwritten_bytes,
                       enable_double_buffering, skip_erasing, verify) -> Result<(), FlashError> {
    // ① 填充不需写的字节（恢复保护）
    if restore_unwritten_bytes { self.fill_unwritten(session, progress)?; }

    // ② 擦除目标 sector
    if !skip_erasing { self.sector_erase(session, progress)?; }

    // ③ 逐页编程（单缓冲或双缓冲）
    self.do_program(session, progress, enable_double_buffering)?;

    // ④ 读回校验
    if verify && !self.verify(session, progress, !restore_unwritten_bytes)? {
        return Err(FlashError::Verify);
    }
    Ok(())
}
```

**双缓冲机制**（`program_double_buffer`）:
```
时间 →
buf0: [load page1] [wait]           [load page3] [wait]
buf1:              [load page2] [wait]           [load page4] ...
       ← 编程 page1 → ← 编程 page2 → ← 编程 page3 → ← 编程 page4 →

优势: 编程 page1 的同时已经在往 buf1 加载 page2，CPU 和 SWD 传输并行！
```

### 2.7 芯片级操作：CMSIS Flash 算法执行

**文件**: `probe-rs/src/flashing/flasher.rs:968`

```
probe-rs 做的事:                    芯片做的事:
┌──────────────────┐              ┌─────────────────────┐
│ ① SWD 写算法到RAM │ ──数据──→  │ RAM: FlashAlgorithm │
│ ② SWD 写数据到RAM │ ──数据──→  │ RAM: page_buffer    │
│ ③ SWD 写 寄存器  │ ──指令──→  │ CPU: 设 R0=addr     │
│ ④ core.run()     │ ──启动──→  │      R1=size        │
│                  │            │      R2=buffer       │
│ ⑤ 轮询 core      │ ←──状态── │ CPU 执行 Flash 算法  │
│   .status()      │            │  → 写 Flash 控制器   │
│                  │            │  → 高压编程          │
│                  │            │  → 自动 Halt         │
│ ⑥ 读 R0 返回值   │            │                     │
└──────────────────┘            └─────────────────────┘
```

**call_function 寄存器设置**:

```rust
fn call_function(&mut self, registers: &Registers, init: bool) -> Result<(), FlashError> {
    let registers = [
        (PC,  Some(registers.pc)),        // Flash算法函数入口
        (R0,  registers.r0),              // 参数1: flash地址
        (R1,  registers.r1),              // 参数2: 大小
        (R2,  registers.r2),              // 参数3: buffer地址
        (R3,  registers.r3),              // 参数4
        (R9,  if init { base } else { None }),  // static_base
        (SP,  if init { stack_top } else { None }), // 栈顶
        (LR,  Some(load_address + 1)),    // 返回地址 (+1 for Thumb)
    ];
    for (reg, val) in registers {
        core.write_core_reg(reg, val)?;
    }
    core.run()?;  // 🔥 让芯片开始执行！
}
```

**wait_for_completion 轮询**:

```rust
fn wait_for_completion(&mut self, timeout: Duration) -> Result<u32, FlashError> {
    loop {
        match core.status()? {            // SWD 读 DHCSR 寄存器
            CoreStatus::Halted(_) => {    // 芯片执行完自动 Halt
                let r0 = core.read_core_reg(R0)?; // 读返回值
                return Ok(r0 as u32);
            }
            _ => {
                if start.elapsed() > timeout {
                    return Err(FlashError::Timeout);
                }
                // 打印 RTT 日志（如果有）
                if let Some(rtt) = &mut self.rtt {
                    rtt.poll(&mut core)?;
                }
                sleep(poll_interval);
            }
        }
    }
}
```

### 2.8 进度事件回传

```rust
// flash_impl 中设置的回调
options.progress = FlashProgress::new(move |event| {
    ProgressEvent::from_library_event(event, |event| {
        sender.blocking_send(event).unwrap();
        //     ↑
        //     └→ ProgressEventTopic → "flash/progress"
        //        → postcard-rpc 序列化 → WireTx → channel/WebSocket
        //        → Client 端的 on_msg 回调 → 进度条更新
    });
});
```

---

## 三、关键节点速查表

| 步骤 | 文件（相对 probe-rs 仓库） | 行号 | 做什么 |
|------|--------------------------|------|--------|
| client flash() | `probe-rs-tools/src/bin/probe-rs/rpc/client.rs` | 517 | 打包 FlashRequest，发 RPC |
| send_and_read_stream | `client.rs` | 327 | 并发：发请求 + 订阅进度 |
| Server 路由 | postcard-rpc 框架 | — | VarHeader.key → FlashEndpoint |
| handler 入口 | `rpc/functions/flash.rs` | 372 | 收到请求，转到 flash_impl |
| run_blocking | `rpc/functions.rs` | 238 | spawn_blocking 异步包装 |
| flash_impl | `rpc/functions/flash.rs` | 377 | 取 Session/Loader，装配选项 |
| **loader.commit()** | `probe-rs/src/flashing/loader.rs` | 667 | Flash 烧录总入口 |
| prepare_plan | `loader.rs` | 673 | 匹配 Flash 算法到内存区域 |
| initialize | `loader.rs` | — | 加载算法到芯片 RAM |
| program() | `probe-rs/src/flashing/flasher.rs` | 397 | 擦除→编程→校验 |
| do_program() | `flasher.rs` | 671 | 单缓冲 or 双缓冲编程 |
| call_function() | `flasher.rs` | 968 | 设 R0-R3,SP,PC → core.run() |
| wait_for_completion() | `flasher.rs` | 1052 | 轮询 core.status() |
| program_simple | `flasher.rs` | 693 | 单缓冲：逐页 program_page |
| program_double_buffer | `flasher.rs` | 729 | 双缓冲：load + program 并行 |
| ProgressEvent 推送 | `rpc/functions/flash.rs` | 398 | sender.blocking_send → Topic |
| endpoint 注册 | `rpc/functions.rs` | 544 | `FlashEndpoint → async → flash` |
| topic 注册 | `rpc/functions.rs` | 519 | `ProgressEventTopic → "flash/progress"` |

---

## 四、请求数据流简图

```
FlashRequest (序列化后的二进制)
│
├─ sessid:       Key<Session>        → daemon 查表取 Session
├─ loader:       Key<FlashLoader>    → daemon 查表取 FlashLoader
│                                       ├─ ELF 段信息(src/dst 地址)
│                                       ├─ 匹配的 Flash Algorithm
│                                       └─ MemoryMap(RAM/Flash 区域)
├─ options:      DownloadOptions
│   ├─ keep_unwritten_bytes: bool
│   ├─ do_chip_erase: bool
│   ├─ skip_erase: bool
│   ├─ verify: bool
│   └─ preferred_algos: Vec<String>
└─ rtt_client:   Option<Key<RttClient>> → 烧录时读 RTT 日志
```

---

## 五、烧录阶段与 ProgressEvent 对照

| 阶段 | ProgressEvent | 说明 |
|------|--------------|------|
| Flash 算法初始化 | `Started(Erase)` / `Started(Program)` | 准备阶段 |
| 加载算法到 RAM | — | 内部操作 |
| 擦除 sector | `Progress { operation: Erase, size }` | 每擦一个 sector |
| 擦除完成 | `Finished(Erase)` | |
| 编程 page | `Progress { operation: Program, size }` | 每写一页 |
| 编程完成 | `Finished(Program)` | |
| 校验 | `Progress { operation: Verify, size }` | 每校验一页 |
| 校验完成 | `Finished(Verify)` | |
| Flash 算法消息 | `DiagnosticMessage { message }` | 算法内部日志 |
