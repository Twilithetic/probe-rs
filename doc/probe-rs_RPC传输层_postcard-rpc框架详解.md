# probe-rs RPC 传输层详解 — postcard-rpc 框架 & 双模通信

> 📅 整理日期：2026-05-10  
> 🎯 讲解目标：daemon 监听机制、postcard-rpc 框架原理、本地/远程双模通信架构

---

## 一、核心答案速览

| 问题 | 答案 |
|------|------|
| **daemon 用什么库监听？** | **postcard-rpc**（自研框架），不是 axum |
| **跟 axum 什么关系？** | axum 只在远程模式做 HTTP → WebSocket 升级，不参与 RPC 逻辑 |
| **为什么本地+远程都能跑？** | 靠 **WireRx / WireTx 传输抽象层** — 同一套 RPC 逻辑，换不同传输实现 |

---

## 二、三层架构全景图

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 3: RPC 应用层 (RpcApp)                                │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Endpoints:  FlashEndpoint, BuildEndpoint, ...             ││
│  │ Topics:     ProgressEventTopic, TargetInfoDataTopic, ...  ││
│  │ Handlers:   flash(), build(), erase(), ...                ││
│  │ 对象注册表: Session, FlashLoader, RttClient               ││
│  └──────────────────────────────────────────────────────────┘│
│                          ↕                                    │
│  Layer 2: postcard-rpc 框架 (自研)                            │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Server<WireTx, WireRx, Buf, App>   ← 服务端               ││
│  │ HostClient<WireTx, WireRx>         ← 客户端               ││
│  │ postcard 序列化 (为嵌入式优化的 serde 格式)                 ││
│  └──────────────────────────────────────────────────────────┘│
│                          ↕                                    │
│  Layer 1: 传输抽象层 (WireRx / WireTx trait)                   │
│  ┌───────────────┬──────────────────┬───────────────────┐    │
│  │ tokio channel │ WebSocket 帧     │ Unix Socket        │    │
│  │ (本地模式)     │ (远程 TCP 模式)   │ (远程 Unix 模式)   │    │
│  └───────────────┴──────────────────┴───────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 三、postcard-rpc：自研 RPC 框架

### 3.1 基本信息

| 项目 | 描述 |
|------|------|
| crate 名 | `postcard-rpc` |
| 版本 | `v0.12.0` |
| 序列化格式 | **postcard**（专为 `#[no_std]` 嵌入式环境设计的 serde 二进制格式） |
| 维护者 | probe-rs 团队（在 `probe-rs-tools` 仓库中自研） |

### 3.2 为什么自研而不用现成方案？

| 需求 | 为什么不用现成的 |
|------|-----------------|
| **postcard 序列化** | 比 JSON/Protobuf 都轻量，Flash 算法里也能用（`#[no_std]` 兼容） |
| **多 transport 支持** | 需要同一个接口适配 tokio channel、WebSocket、Unix Socket |
| **Topic 流式推送** | 烧录需要持续推送 ProgressEvent，传统 RPC 只有 request/response |
| **对象注册表** | daemon 需要管理 Session、FlashLoader 等有生命周期的对象（Key\<T\>） |
| **HEADER 路由** | 自定义二进制帧头做路由，不依赖 HTTP 语义 |

### 3.3 帧格式

不依赖 HTTP 语义，直接用二进制帧：

```
┌──────────────────────┬─────────────────────────┐
│ VarHeader (1-9 bytes)│ postcard body (N bytes)  │
├──────────────────────┼─────────────────────────┤
│ key:   路由到哪个     │ serde 序列化的请求/响应   │
│        Endpoint/Topic │                         │
│ seq_no: 序列号        │                         │
│        (流控/去重)    │                         │
└──────────────────────┴─────────────────────────┘
```

这就是为什么 postcard-rpc 可以在任何字节流上工作 — 它根本不关心底层是 channel、WebSocket 还是 TCP 裸流。

---

## 四、传输抽象层：WireRx / WireTx

### 4.1 核心 trait 定义

这是整个架构最精妙的设计。`transport/memory.rs` 中定义了两个关键 trait：

```rust
// 接收端抽象 — 任何能"收到字节"的东西
pub trait PostcardReceiver {
    fn receive(&mut self) -> impl Future<Output = Result<Vec<u8>, WireRxErrorKind>> + Send;
}

// 发送端抽象 — 任何能"发出字节"的东西
pub trait PostcardSender {
    fn send(&self, buf: Vec<u8>) -> impl Future<Output = Result<(), WireTxErrorKind>> + Send;
}
```

然后 `WireRx<R>` 和 `WireTx<S>` 分别实现 `postcard-rpc` 框架要求的 `server::WireRx` 和 `server::WireTx` trait。

### 4.2 三种传输实现

| 实现 | PostcardSender | PostcardReceiver | 传输介质 | 用途 |
|------|---------------|-----------------|---------|------|
| **Memory** | `Sender<Vec<u8>>` | `Receiver<Vec<u8>>` | tokio channel | 本地模式 |
| **WebSocket** | `AxumWebsocketTx` | `WebsocketRx` | WebSocket 帧 | 远程 TCP |
| **Unix** | `UnixStreamTx` | `UnixStreamRx` | Unix 字节流 | 远程 Unix |

### 4.3 Memory 实现（本地模式）

```rust
// transport/memory.rs

// tokio channel 实现 PostcardSender
impl PostcardSender for Sender<Vec<u8>> {
    async fn send(&self, buf: Vec<u8>) -> Result<(), WireTxErrorKind> {
        Sender::send(self, buf).await
            .map_err(|_| WireTxErrorKind::ConnectionClosed)
    }
}

// tokio channel 实现 PostcardReceiver
impl PostcardReceiver for Receiver<Vec<u8>> {
    async fn receive(&mut self) -> Result<Vec<u8>, WireRxErrorKind> {
        match self.recv().await {
            Some(packet) => Ok(packet),
            None => Err(WireRxErrorKind::ConnectionClosed),
        }
    }
}
```

**本地通信就是进程内的 tokio mpsc channel，零网络开销。**

### 4.4 WebSocket 实现（远程模式）

```rust
// transport/websocket.rs
// 把 postcard 帧包装成 WebSocket Binary Message
// 网络层只看到 WebSocket 帧，RPC 层只看到字节流
```

### 4.5 WireTx 的 send 方法（序列化细节）

```rust
impl<S: PostcardSender> server::WireTx for WireTx<S> {
    async fn send<T: Serialize>(&self, hdr: VarHeader, msg: &T) -> Result<(), Self::Error> {
        // ① 先测量 body 长度（不分配）
        let mut length_counter = LengthCounter(0);
        postcard::to_io(msg, &mut length_counter).unwrap();

        // ② 分配 buffer：HEADER(最大9字节) + body
        let mut buffer = Vec::with_capacity(length_counter.0 + HEADER_MAX_LEN);

        // ③ 写 HEADER
        let header_bytes = hdr.write_to_slice(&mut buffer).unwrap();

        // ④ 写 body（postcard 序列化）
        let buffer = postcard::to_extend(msg, buffer).unwrap();

        // ⑤ 通过抽象 sink 发送
        self.sink.send(buffer).await
    }
}
```

**无论底层是 channel 还是 WebSocket，这一层代码完全不变。**

---

## 五、创建 Server：create_server()

### 5.1 源码 (functions.rs:594)

```rust
impl RpcApp {
    pub fn create_server(
        depth: usize,                         // channel 队列深度 (默认16)
        probe_access: ProbeAccess,            // 探针访问权限
    ) -> (ServerImpl, TxChannel, RxChannel) { // 返回 (server, tx, rx)
        // ① 创建一对 tokio channel（server↔client 的桥梁）
        let client_to_server = channel::<Result<Vec<u8>, WireRxErrorKind>>(depth);
        let server_to_client = channel::<Vec<u8>>(depth);

        // ② 用 channel 的接收端/发送端构造 WireRx / WireTx
        let client_to_server_rx = WireRx::new(client_to_server.1);
        let server_to_client_tx = WireTx::new(server_to_client.0);

        // ③ 创建 RPC 应用（注册所有 Endpoint/Topic 处理函数）
        let mut dispatcher = RpcApp::new(RpcContext::new(probe_access), TokioSpawner);

        // ④ 把 sender 注入 context（用于 Topic 推送）
        let vkk = dispatcher.min_key_len();
        dispatcher.context.set_sender(
            PostcardSender::new(server_to_client_tx.clone(), vkk)
        );

        // ⑤ 组装 Server
        (
            Server::new(
                server_to_client_tx,                   // server → client
                client_to_server_rx,                   // client → server
                vec![0u8; 1024 * 1024].into_boxed_slice(), // 1MB 接收缓冲区
                dispatcher,                            // RPC 应用
                vkk,                                   // 最小 key 长度
            ),
            client_to_server.0,   // tx — 给 client 发请求用
            server_to_client.1,   // rx — 给 client 收响应用
        )
    }
}
```

返回值的三个部分：
- `ServerImpl` — 服务端，调用 `.run()` 启动事件循环
- `TxChannel` — client 用来**向** daemon 发请求
- `RxChannel` — client 用来**从** daemon 收响应

---

## 六、本地模式 vs 远程模式：代码对比

### 6.1 本地模式 (main.rs)

```rust
// ① 创建 server
let (mut local_server, tx, rx) = RpcApp::create_server(16, ProbeAccess::All);

// ② daemon 跑在后台线程
let handle = tokio::spawn(async move { local_server.run().await });

// ③ client 直接用 channel 连上 — 注意这里！
let client = RpcClient::new_local_from_wire(tx, rx);

// ④ 跑命令
let result = cb(client).await;

// ⑤ 等 daemon 退出
_ = handle.await.unwrap();
```

**关键点**：`tx` 和 `rx` 就是 tokio channel 的两端，`new_local_from_wire` 直接把 channel 包成 `WireTx`/`WireRx`。

### 6.2 远程模式 (serve.rs)

```rust
// ① axum 启动 HTTP 服务器（只做 WebSocket 升级）
let app = Router::new()
    .route("/", get(server_info))        // 状态页面（HTML）
    .route("/worker", any(ws_handler))    // WebSocket 入口
    .with_state(state);

axum::serve(listener, app).await;

// ② WebSocket 连接建立后 (handle_socket):
//    认证 → 创建 server → 桥接 WebSocket ↔ channel

let (mut server, tx, mut rx) = RpcApp::create_server(16, user.access);

// ③ 桥接：WebSocket 帧 ⟷ tokio channel
let sender = async {
    // server 的输出 → rx → WebSocket → 网络 → client
    while let Some(msg) = rx.recv().await {
        writer.send(msg).await.unwrap();
    }
};
let receiver = async {
    // 网络 → WebSocket → reader → tx → server 的输入
    while let Some(msg) = reader.next().await {
        tx.send(msg).await.unwrap();
    }
};

// ④ 三路并发
tokio::select! {
    _ = server.run() => ...,   // RPC 服务运行
    _ = sender       => ...,   // server→network
    _ = receiver     => ...,   // network→server
}
```

**关键点**：`RpcApp::create_server()` 在本地和远程模式里**完全一样**！区别只在于 tx/rx 另一端接的是什么。

---

## 七、axum 的角色：只是"门童"

```
远程客户端                          probe-rs serve (daemon)
┌──────────┐                      ┌────────────────────────────┐
│          │  ① HTTP GET /worker  │  axum (门童)               │
│ RPC      │ ──────────────────→ │  ┌──────────────────────┐  │
│ Client   │ ←── 101 Upgrade ──── │  │ Router::route(...)   │  │
│          │                      │  │ ws_handler()         │  │
│          │  ② WebSocket 帧      │  └────────┬─────────────┘  │
│          │ ════════════════════ │           │ 升级完成        │
│          │  二进制 postcard     │           ▼                 │
│          │  (FlashRequest etc) │  ┌──────────────────────┐  │
│          │ ←═══════════════════→│  │ postcard-rpc Server  │  │
│          │  (ProgressEvent等)   │  │ (跟本地模式完全一样)   │  │
└──────────┘                      │  └──────────────────────┘  │
                                  └────────────────────────────┘
```

axum 不参与任何 RPC 路由、序列化、请求分发。它的全部工作：
1. 接收 HTTP GET `/worker`
2. 把连接升级成 WebSocket（101 Switching Protocols）
3. **退出舞台**，让 `postcard-rpc` 接管一切

---

## 八、独特机制总结

### 8.1 对象注册表（Key\<T\>）

daemon 内部维护类型化的对象存储，支持跨网络传递：

```
本地: Key 是内存索引
远程: Key 序列化后通过 WebSocket 发送，daemon 在注册表里查找
```

### 8.2 双通道通信（Endpoint + Topic）

```
Endpoint:  请求→响应  (一问一答)
  "flash/flash"  →  FlashRequest → NoResponse

Topic:     发布→订阅  (持续推送)
  "flash/progress" → ProgressEvent → ProgressEvent → ...
```

`send_and_read_stream` 通过 `tokio::join!` 并发执行请求和订阅。

### 8.3 认证（远程模式专属）

```
Client                          Server
  │                               │
  │ ① GET /worker                 │ 返回 Probe-Rs-Challenge header
  │    (64字节随机数)              │
  │                               │
  │ ② WebSocket 连接建立           │
  │    SHA512(challenge+token)    │ 验证所有已配置用户的 token
  │                               │
  │ ③ 认证通过 → create_server()  │
```

### 8.4 run_blocking — 阻塞任务异步化

```rust
pub async fn run_blocking<T, F, REQ, RESP>(&mut self, request: REQ, task: F) -> RESP {
    let (sender, publisher) = T::create(token);

    let ctx = self.clone();
    let blocking = tokio::task::spawn_blocking(move || task(ctx, request, sender));

    tokio::select! {
        _ = publisher.publish(&self.sender) => unreachable!(),
        response = blocking => response.unwrap(),
    }
}
```

烧录是 CPU 密集型操作，用 `spawn_blocking` 放到独立线程池，不阻塞 async runtime。

---

## 九、框架对比

| | postcard-rpc (probe-rs) | axum | gRPC | tarpc |
|--|------------------------|------|------|-------|
| 定位 | 嵌入式工具 RPC 框架 | 通用 Web 框架 | 通用 RPC 框架 | Rust RPC 框架 |
| 序列化 | postcard（嵌入式友好） | JSON/任意 | Protobuf | serde |
| 传输 | tokio ch / WS / Unix Socket | HTTP/WS | HTTP/2 | TCP |
| 流式推送 | Topic（原生支持） | SSE / WS | Streaming RPC | 不支持 |
| 对象注册表 | 内置 Key\<T\> | 无 | 无 | 无 |
| probe-rs 里角色 | **核心 RPC 框架** | WebSocket 升级门童 | 没用 | 没用 |

---

## 十、源码位置速查

| 文件 | 行号 | 内容 |
|------|------|------|
| `Cargo.toml` | 116 | `postcard-rpc = "0.12.0"` |
| `main.rs` | 627 | 本地模式 `create_server` + `tokio::spawn` |
| `main.rs` | 614 | `run_app()` — 本地/远程分支 |
| `cmd/serve.rs` | 137 | 远程 TCP 模式 (`run_tcp`) |
| `cmd/serve.rs` | 160 | axum Router 定义 |
| `cmd/serve.rs` | 205 | WebSocket handler (`ws_handler`) |
| `cmd/serve.rs` | 234 | WebSocket ↔ channel 桥接 (`handle_socket`) |
| `cmd/serve.rs` | 270 | `create_server`（远程模式也用同一个！） |
| `cmd/serve.rs` | 298 | Unix Socket 桥接 (`handle_unix_rpc`) |
| `rpc/functions.rs` | 590 | `ServerImpl` 类型别名 |
| `rpc/functions.rs` | 594 | `create_server()` 实现 |
| `rpc/functions.rs` | 238 | `run_blocking()` — 阻塞任务异步化 |
| `rpc/transport/memory.rs` | 19 | `PostcardReceiver` trait |
| `rpc/transport/memory.rs` | 41 | `WireRx<R>` — 接收抽象 |
| `rpc/transport/memory.rs` | 88 | `PostcardSender` trait |
| `rpc/transport/memory.rs` | 109 | `WireTx<S>` — 发送抽象 |
| `rpc/transport/memory.rs` | 126 | `WireTx::send()` — 序列化 + 发送 |
| `rpc/transport/websocket.rs` | — | WebSocket transport 实现 |
| `rpc/transport/unix.rs` | — | Unix Socket transport 实现 |
| `rpc/client.rs` | 327 | `send_and_read_stream` — 并发请求+订阅 |
