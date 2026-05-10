# probe-rs `flash()` 函数 — RPC 调用链详解

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：client 端 `flash()` 函数的工作原理、RPC 架构、完整调用链路

---

## 一、项目概览

probe-rs 是一个用 Rust 写的嵌入式调试工具包，支持连接各种 USB 探针（DAPLink、STLink、JLink 等），通过 SWD/JTAG 跟 ARM、RISC-V、Xtensa 芯片通信。

### 目录结构

```
probe-rs-src/
├── probe-rs/               ← 核心库：硬件抽象、Flash算法、Session
├── probe-rs-tools/         ← CLI 工具 + RPC 架构
│   └── src/bin/probe-rs/
│       ├── main.rs         ← 入口：本地/远程模式分支，启动 daemon
│       ├── rpc/
│       │   ├── client.rs   ← ** RPC 客户端实现 — flash() 在这里 (L517) **
│       │   ├── functions/  ← daemon 端的处理函数
│       │   │   └── flash.rs ← flash 的 RPC 处理实现 (L372)
│       │   └── functions.rs ← 端点/主题注册表 (macro_rules)
│       └── util/
│           └── cli.rs      ← CLI 命令的调用逻辑 (L347)
├── probe-rs-target/        ← 芯片数据库 (CYT2BL3 等)
├── probe-rs-debug/         ← VSCode DAP 调试器支持
├── probe-rs-mi/            ← GDB 调试器支持
└── doc/                    ← 项目文档
```

---

## 二、核心架构：客户端-服务端 (RPC)

### 2.1 为什么有 daemon？

即使在同一进程中，probe-rs 也采用 **client-daemon** 分离架构：

```
你的终端                   后台线程（同一个 probe-rs.exe 进程）
┌──────────┐   channel     ┌──────────────────────────┐
│  CLI     │ ════════════→ │  RPC Daemon              │
│ (client) │ ←════════════ │                          │
│          │  RPC 消息     │  ∘ 打开 USB 探针          │
│ 进度条   │  来回传递      │  ∘ SWD 通信               │
│ 在这里   │              │  ∘ 读写芯片寄存器          │
└──────────┘              │  ∘ 执行 Flash 算法         │
                          └──────────────────────────┘
```

**设计原因：**

| 原因 | 说明 |
|------|------|
| 硬件独占 | USB 探针同一时间只能一个线程操作，daemon 做串行化 |
| 职责分离 | CLI 管用户交互（进度条、日志），daemon 管硬件 |
| 远程支持 | 本地用 channel，远程用 TCP — 同一套代码 |

### 2.2 本地 vs 远程模式

```
本地模式:                         远程模式:
┌──────┐  channel  ┌──────┐      ┌──────┐   TCP    ┌──────┐
│client│ ═══════→ │daemon│      │client│ ──────→ │daemon│
│      │ ←═══════ │      │      │      │ ←────── │      │
└──────┘          └──────┘      └──────┘         └──────┘
 同一个进程                      不同机器
 daemon 用完即焚                 daemon 常驻运行
```

### 2.3 启动流程（main.rs:626-632）

```rust
// ① 创建 daemon — channel 深度为 16（最多排队 16 个请求）
let (mut local_server, tx, rx) = RpcApp::create_server(16, ProbeAccess::All);

// ② 在后台线程启动 daemon
let handle = tokio::spawn(async move { local_server.run().await });

// ③ client 通过 channel 连上 daemon
let client = RpcClient::new_local_from_wire(tx, rx);

// ④ 用 client 发命令 — 命令通过 channel 传到 daemon 执行
cb(client).await;
```

---

## 三、`flash()` 函数源码分析

### 3.1 Client 端 (client.rs:517)

```rust
pub async fn flash(
    &self,
    options: DownloadOptions,           // 烧录选项
    loader: Key<FlashLoader>,           // FlashLoader 的轻量句柄
    rtt_client: Option<Key<RttClient>>, // 可选的 RTT 客户端句柄
    on_msg: impl AsyncFnMut(ProgressEvent), // 进度回调
) -> anyhow::Result<()> {
    self.client
        .send_and_read_stream::<FlashEndpoint, ProgressEventTopic, _>(
            &FlashRequest {
                sessid: self.sessid,
                loader,
                options,
                rtt_client,
            },
            on_msg,
        )
        .await
}
```

### 3.2 参数说明

| 参数 | 类型 | 含义 |
|------|------|------|
| `options` | `DownloadOptions` | 烧录选项：`keep_unwritten_bytes`、`do_chip_erase`、`verify`、`preferred_algos` 等 |
| `loader` | `Key<FlashLoader>` | 指向 daemon 中 FlashLoader 对象的**轻量句柄**（ID） |
| `rtt_client` | `Option<Key<RttClient>>` | 可选的 RTT 客户端句柄，烧录时读取芯片日志 |
| `on_msg` | `impl AsyncFnMut(ProgressEvent)` | 异步回调：daemon 每有进度更新就调用一次 |

### 3.3 `Key<T>` 是什么？

daemon 端维护一个对象注册表，存储真正的 `Session`、`FlashLoader`、`RttClient` 等重量级对象。
client 端只拿**轻量的 Key 句柄**（本质是个 ID）。

```
Client 端                     Daemon 端
Key<Session>(id=3)  ───RPC───→  Session { probe, target_info, ... }
Key<FlashLoader>(id=7) ───RPC───→ FlashLoader { segments, algo, ... }
```

**本地模式**：Key 就是个内存索引  
**远程模式**：Key 通过 TCP 发送，daemon 在注册表里查找

### 3.4 `send_and_read_stream` 并发机制

```rust
// client.rs:327
async fn send_and_read_stream<E, T, R>(
    &self,
    req: &E::Request,
    on_msg: impl AsyncFnMut(T::Message),
) -> anyhow::Result<R>
```

它通过 `tokio::join!` **并发执行两件事**：

```
                        time →
┌──────────────────────────────────────────────────────────┐
│  ① 订阅 ProgressEventTopic ("flash/progress")           │
│     → daemon 推送的进度事件流 → 驱动 on_msg 回调          │
│                                                        │
│  ② 发送 FlashRequest 到 daemon (Endpoint "flash/flash") │
│     → daemon 收到后开始真正烧录                          │
│     → 烧录过程中持续推送 ProgressEvent                   │
│     → 烧录完成后返回 NoResponse                          │
└──────────────────────────────────────────────────────────┘
```

### 3.5 Daemon 端处理 (functions/flash.rs:372)

```rust
pub async fn flash(ctx: &mut RpcContext, _header: VarHeader, request: FlashRequest) -> NoResponse {
    ctx.run_blocking::<ProgressEventTopic, _, _, _>(request, flash_impl).await
}

fn flash_impl(ctx: RpcSpawnContext, request: FlashRequest, sender: Sender<ProgressEvent>) -> NoResponse {
    let mut session = ctx.session_blocking(request.sessid);  // ① 取 Session
    let mut rtt_client = request.rtt_client
        .map(|rtt| ctx.object_mut_blocking(rtt));             // ② 取 RTT（可选）
    let loader = ctx.object_mut_blocking(request.loader);     // ③ 取 FlashLoader

    if let Some(rtt) = rtt_client.as_mut() {
        rtt.configure_from_loader(&loader);                   // ④ 配置 RTT
    }

    let mut options = request.download_options();
    options.progress = FlashProgress::new(move |event| {       // ⑤ 设置进度回调
        ProgressEvent::from_library_event(event, |event| {
            sender.blocking_send(event).unwrap();              // → 推送给 client
        });
    });

    loader.commit(&mut session, options)                       // ⑥ 真正的硬件烧录！
        .map_err(FileDownloadError::Flash)?;

    Ok(())
}
```

**`loader.commit()` 内部做的事：**
- 按 FlashLayout 依次执行：擦除 → 编程 → 校验
- 每个阶段触发 `ProgressEvent`（Started / Progress / Finished）
- 通过 ARM Debug Interface (ADI) 把 Flash 算法加载到芯片 RAM
- 让 CPU 执行 Flash 算法完成实际擦写

---

## 四、ProgressEvent 类型

```rust
// functions/flash.rs:243
pub enum ProgressEvent {
    FlashLayoutReady {
        flash_layout: Vec<FlashLayout>,      // 布局分析完成
    },
    AddProgressBar {
        operation: Operation,               // 新进度条
        total: Option<u64>,                 // None 表示不确定长度
    },
    Started(Operation),                     // 某操作开始
    Progress {
        operation: Operation,
        size: u64,                          // 已处理字节数
    },
    Failed(Operation),                      // 操作失败
    Finished(Operation),                    // 操作完成
    DiagnosticMessage {
        message: String,                    // Flash 算法诊断消息
    },
}
```

CLI 端用这些事件画进度条：

```rust
async |event| {
    if let ProgressEvent::FlashLayoutReady { flash_layout: layout } = &event {
        flash_layout = Some(layout.clone());
    }
    if let Some(ref pb) = pb {
        pb.handle(event);  // 更新进度条 UI
    }
}
```

---

## 五、完整调用链

```
用户运行: probe-rs download firmware.elf
  │
  └─► main.rs → run_app()
        │
        ├─ 启动 daemon 后台线程
        │     RpcApp::create_server(16, ProbeAccess::All)
        │     tokio::spawn(local_server.run())
        │
        └─► client = RpcClient::new_local_from_wire(tx, rx)
              │
              └─► cli.rs: flash()  [L347]
                    │
                    ├─ ① session.build_flash_loader(path, format, ...)
                    │     │
                    │     ├─ client 端: upload_file → send(BuildRequest)
                    │     └─ daemon 端: 读 ELF → 分析段信息 → 匹配 Flash 算法
                    │                   → 返回 Key<FlashLoader>
                    │
                    ├─ ② session.flash(options, loader, rtt, on_msg)  ← 🎯 核心！
                    │     │
                    │     ├─ client 端: send_and_read_stream
                    │     │   ├─ 订阅 ProgressEventTopic ("flash/progress")
                    │     │   └─ 发送 FlashRequest → Endpoint "flash/flash"
                    │     │
                    │     └─ daemon 端: flash_impl()
                    │         ├─ 取 Session / FlashLoader / RttClient
                    │         ├─ 设置进度回调 (ProgressEvent → Topic → Client)
                    │         └─ loader.commit(&mut session, options)
                    │             ├─ 擦除 sector
                    │             ├─ 编程 page
                    │             └─ 校验
                    │
                    └─ ③ 输出 "Finished in 1.62s"
```

---

## 六、源码位置速查表

| 文件 | 行号 | 内容 |
|------|------|------|
| `probe-rs-tools/src/bin/probe-rs/main.rs` | 626-632 | 启动 daemon + client |
| `probe-rs-tools/src/bin/probe-rs/rpc/client.rs` | 517 | **`flash()` — RPC 客户端方法** |
| `probe-rs-tools/src/bin/probe-rs/rpc/client.rs` | 327 | `send_and_read_stream` — 并发请求+订阅 |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs` | 372 | daemon 端 `flash()` 入口 |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs` | 377 | `flash_impl()` — 实际硬件操作 |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs` | 108 | `FlashRequest` 结构体 |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions/flash.rs` | 243 | `ProgressEvent` 枚举 |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions.rs` | 474 | `FlashEndpoint` 注册 ("flash/flash") |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions.rs` | 519 | `ProgressEventTopic` 注册 ("flash/progress") |
| `probe-rs-tools/src/bin/probe-rs/rpc/functions.rs` | 544 | endpoint handler 绑定 |
| `probe-rs-tools/src/bin/probe-rs/util/cli.rs` | 347 | CLI `flash()` 调用入口 |

---

## 七、关键概念速记卡片

| 概念 | 一句话解释 |
|------|-----------|
| **RPC Daemon** | 后台线程，独占硬件，串行处理请求 |
| **RPC Client** | 前端，只管发命令和处理回调 |
| **Key\<T\>** | 对象的轻量句柄/ID，真正的对象在 daemon 那边 |
| **Endpoint** | 请求→响应，一问一答（如 `"flash/flash"`） |
| **Topic** | 发布→订阅，持续推送（如 `"flash/progress"`） |
| **FlashLoader** | 包含 ELF 段信息 + 匹配的 Flash 算法 |
| **ProgressEvent** | 进度事件：Started → Progress → Finished |
| **loader.commit()** | 执行真正的硬件烧录操作 |
