# probe-rs RPC 架构：为什么有个 daemon

## 一句话

probe-rs 用**客户端-服务端**架构。CLI 是 client，只发命令。真正操作探针、读写芯片的是 daemon（后台线程）。同一个进程，两个角色。

## 为什么这么设计

1. **硬件独占**：USB 探针同一时间只能一个线程操作
2. **职责分离**：CLI 管用户交互（进度条、日志、参数解析），daemon 管硬件
3. **远程支持**：同一套代码，本地用 channel 通信，远程用 TCP 通信

## 架构图

```
你的终端              后台线程（同一个 probe-rs.exe 进程）
┌──────────┐  channel  ┌──────────────────────────┐
│  CLI     │ ════════→ │  RPC Daemon              │
│ (client) │ ←════════ │                          │
│          │  RPC 消息  │  ∘ 打开探针 (USB)         │
│ main.rs  │  来回传递  │  ∘ SWD 通信              │
│ download │           │  ∘ 读写芯片寄存器         │
│  .run()  │           │  ∘ 执行 Flash 算法        │
└──────────┘           │  ∘ 持有真正的硬件状态     │
                       └──────────────────────────┘
```

## 启动流程 (main.rs:626-632)

```rust
// ① 创建 daemon — channel 的深度是 16（最多排队 16 个请求）
let (mut local_server, tx, rx) = RpcApp::create_server(16, ProbeAccess::All);

// ② 在后台线程启动 daemon
let handle = tokio::spawn(async move { local_server.run().await });

// ③ client 通过 channel 连上 daemon
let client = RpcClient::new_local_from_wire(tx, rx);

// ④ 用 client 发命令 — 命令通过 channel 传到 daemon 执行
cb(client).await;
```

## 本地 vs 远程

```
本地模式:                         远程模式:
┌──────┐  channel  ┌──────┐      ┌──────┐   TCP    ┌──────┐
│client│ ═══════→ │daemon│      │client│ ──────→ │daemon│
│      │ ←═══════ │      │      │      │ ←────── │      │
└──────┘          └──────┘      └──────┘         └──────┘
 同一个进程                      不同机器
```

## download 命令的 RPC 调用序列

```
CLI (client)                    Daemon (后台线程)
│                                    │
├─ BuildFlashLoader ────────────────→│ 读 ELF → 分析段 → 选算法
│←────────────── BuildResult ───────┤
│                                    │
├─ EraseSector ─────────────────────→│ 执行 Flash 算法擦除 sector
│←────────────── OK ────────────────┤
│                                    │
├─ ProgramPage ─────────────────────→│ 执行 Flash 算法编程 page
│←────────────── OK ────────────────┤
│                                    │
├─ Verify ──────────────────────────→│ 读回比对
│←────────────── OK ────────────────┤
│                                    │
```

## 为什么日志里的 DPIDR / NACK 来自 daemon

```rust
// client 端 — 处理用户交互
eprintln!("Erasing ✔ 100%");

// daemon 端 — 真正干活
tracing::debug!("DPIDR became readable after 0ms");
tracing::debug!("Transfer status: NACK");
```

**所有 `probe_rs::architecture::arm::sequences`、`probe_rs::flashing` 开头的日志都是 daemon 线程打印的。** client 只负责格式化输出（进度条、`Finished in 1.62s`）。

## 生命周期：用完即焚 vs 长期运行

### 本地模式 — daemon 跟 CLI 同生共死

```rust
let handle = tokio::spawn(async move { local_server.run().await });  // 启动
let result = cb(client).await;  // 干活
_ = handle.await.unwrap();      // 等 daemon 退出
// 进程结束，daemon 没了
```

**daemon 的生命 = 一次命令的执行时间。** 跑完就没了。每次 `probe-rs download` 都是重新启动 daemon。

### 远程模式 — daemon 可以一直活着

```bash
# 终端A — 启动持久 daemon
probe-rs serve

# 终端B — 反复远程连接
probe-rs download --host localhost:5555 build/firmware.elf
probe-rs info --host localhost:5555
probe-rs erase --host localhost:5555
# daemon 一直运行，等下一个命令
```

### 对比

| | 本地模式 | 远程模式 |
|--|---------|---------|
| 启动 | 每次命令自动启动 | `probe-rs serve` 手动启动 |
| 退出 | 命令结束自动退出 | 手动 Ctrl+C 或 kill |
| 通信 | 进程内 channel | TCP |
| 生命周期 | 用完即焚 | 像服务器 |

## `build_flash_loader` 详解

### 职责：上传文件 + 发起分析请求

位于 `rpc/client.rs:479`，是 RPC client 端的一个封装函数。

#### 完整源码（30 行）

```rust
pub async fn build_flash_loader(
    &self, path, format, image_target, read_flasher_rtt
) -> anyhow::Result<BuildResult> {
    // ① 上传主固件文件到 daemon
    path = self.client.upload_file(&path).await?;

    // ② ESP32 only: 上传 bootloader → 你跳过
    if let Some(ref mut idf_bootloader) = format.idf_options.idf_bootloader {
        *idf_bootloader = self.client.upload_file(&*idf_bootloader).await?
            .display().to_string();
    }

    // ③ ESP32 only: 上传分区表 → 你跳过
    if let Some(ref mut idf_partition_table) = format.idf_options.idf_partition_table {
        *idf_partition_table = self.client.upload_file(&*idf_partition_table).await?
            .display().to_string();
    }

    // ④ 发 BuildRequest RPC → daemon 分析怎么烧
    self.client.send_resp::<BuildEndpoint, _>(&BuildRequest {
        sessid: self.sessid,
        path: path.display().to_string(),
        format, image_target, read_flasher_rtt,
    }).await
}
```

#### daemon 端收到 BuildRequest 后做的事 (rpc/functions/flash.rs:82)

```rust
async fn build(ctx, _header, request: BuildRequest) -> BuildResponse {
    let mut session = ctx.session(request.sessid).await;
    let mut loader = build_loader(
        &mut session, &request.path, request.format,
        request.image_target.and_then(InstructionSet::from_target_triple),
    )?;
    // build_loader() 内部:
    //   → 读 ELF 文件 → 分析 .vectors, .text, .data 等段
    //   → 匹配 Flash Algorithm (cyt2bl / cyt2bx_wflash_128 / ...)
    //   → 返回 FlashLoader（知道"哪里烧什么数据"）

    Ok(BuildResult {
        boot_info: loader.boot_info().into(),   // 固件入口点等信息
        loader: ctx.store_object(loader).await,  // FlashLoader 存到 daemon
    })
}
```

### 为什么有 ESP32 特殊处理

| 芯片 | 固件组成 | `build_flash_loader` 做什么 |
|------|---------|--------------------------|
| **ARM** (CYT2BL3, STM32) | 一个 ELF | 上传 ELF → 发请求 |
| **ESP32** (IDF 格式) | ELF + bootloader + partition | 上传三个文件 → 发请求 |

ESP32 的两段 `if let` 对 ARM 芯片来说是 `if let None → 跳过`，不消耗运行时。

### `build_flash_loader` 不做什么

- ❌ 不烧录（那是 `flash()` 函数的事）
- ❌ 不擦除（那是 `erase()` 函数的事）
- ❌ 不接触硬件（那是 daemon 的事）

**它只做一件事：把"烧什么"的问题整理好，交给 daemon 制定计划。**

## 源码位置

| 文件 | 说明 |
|------|------|
| `main.rs:626-632` | 启动 daemon + client |
| `main.rs:614-638` | `run_app()` — 本地/远程分支 |
| `rpc/client.rs:479` | `build_flash_loader` RPC client |
| `rpc/functions/flash.rs` | daemon 端 Flash 请求处理 |
| `rpc/functions.rs:595` | `create_server()` — 创建 daemon |
