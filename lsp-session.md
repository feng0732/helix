# LSP 客户端会话代码路径追踪

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     helix-term / Application            │
│  ┌─────────────────┐     ┌──────────────────────────┐     │
│  │  commands/lsp.rs │────▶│   application.rs       │     │
│  │  (用户命令触发)   │     │  (事件循环/消息分发)  │     │
│  └─────────────────┘     └──────────────────────────┘     │
│                                 │                      │
└─────────────────────────────────┼──────────────────────────────┘
                              │
┌─────────────────────────────────▼──────────────────────────────┐
│                  helix-lsp / Client                   │
│  ┌───────────────────────────────────────────┐       │
│  │              client.rs                │       │
│  │  - 连接管理、请求封装、能力缓存        │       │
│  └───────────────────────────────────────────┘       │
│                     │                          │
│  ┌───────────────────────────────────────────┐       │
│  │             transport.rs             │       │
│  │  传输层：JSON-RPC 消息收发、           │       │
│  │  三个异步任务（收/发/错误              │       │
│  └───────────────────────────────────────────┘       │
│                     │                          │
│  ┌───────────────────────────────────────────┐       │
│  │              jsonrpc.rs              │       │
│  │  JSON-RPC 协议类型定义          │       │
│  └───────────────────────────────────────────┘       │
│                     │                          │
│  ┌───────────────────────────────────────────┐       │
│  │                lib.rs / Registry      │       │
│  │  多 LSP 客户端注册表           │       │
│  └───────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────▼──────────────────────────────┐
│              helix-view / handlers               │
│  ┌───────────────────────────────────────────┐       │
│  │            handlers/lsp.rs           │       │
│  │  事件钩子、诊断处理、WorkspaceEdit 应用 │       │
│  └───────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
                    语言服务器进程 (stdin/stdout)
```

---

## 二、连接建立流程

### 2.1 启动入口：[lib.rs::start_client](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/lib.rs#L895-L973)

```
用户打开文档 → Registry::get() → start_client()

步骤：
1. **工作目录查找 (`find_lsp_workspace)
2. **进程启动** (`Client::start)
3. **异步初始化** (`initialize + initialized)
```

### 2.2 进程启动：[client.rs::Client::start](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L209-L271)

```rust
pub fn start(...) -> Result<(Self, UnboundedReceiver<(LanguageServerId, Call)>, Arc<Notify>)
```

**关键步骤：**

1. **解析可执行文件路径**：`helix_stdx::env::which(cmd)

2. **启动子进程**：
   ```rust
   Command::new(cmd)
       .envs(environment)
       .args(args)
       .stdin(Stdio::piped())
       .stdout(Stdio::piped())
       .stderr(Stdio::piped())
       .current_dir(&root_path)
       .kill_on_drop(true)
       .spawn()
   ```

3. **包装 stdio**：
   - `BufWriter<ChildStdin> → 写入请求
   - `BufReader<ChildStdout> → 读取响应
   - `BufReader<ChildStderr> → 读取错误

4. **启动传输层**：`Transport::start()

5. **创建 Client 结构体：
   ```rust
   Client {
       id,
       name,
       _process: process,  // 持有子进程句柄
       server_tx,              // 发送 Payload 通道
       request_counter: AtomicU64::new(0),
       capabilities: OnceCell::new(),
       config,
       root_path,
       root_uri,
       workspace_folders: Mutex::new(...),
       initialize_notify,
       req_timeout,
   }
   ```

### 2.3 传输层启动：[transport.rs::Transport::start](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L50-L88)

启动 **三个独立的异步任务**：

```
                      ┌─────────────────────────┐
                      │   Transport::start   │
                      └─────────┬─────────────┘
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │  recv   │  │  err   │  │  send   │
     │(stdout)│  │(stderr)│  │(stdin) │
     └─────────┘  └─────────┘  └─────────┘
```

**1. **recv 任务** ([transport.rs::recv](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L255-L321)

   - 循环读取 LSP 消息头（Content-Length 协议)
   - 解析 JSON-RPC 消息
   - 区分 `ServerMessage::Output (响应) → 匹配 pending_requests 中的等待者
   - `ServerMessage::Call (服务端请求/通知) → 转发给 client_tx

   消息头解析：
   ```
   Content-Length: 123\r\n
   \r\n
   <JSON body>
   ```

**2. send 任务** ([transport.rs::send](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L338-L441)

   - 接收 `client_rx 接收 Payload:
     - `Payload::Request { chan, value }
     - `Payload::Notification(value)
     - `Payload::Response(output)

   - **初始化排队机制**：
     - `is_pending` 状态：服务器未初始化完成前
     - 只允许 `initialize` 和 `initialized` 消息通过
     - 其他请求缓存到 `pending_messages: Vec<Payload>
     - `initialize_notify` 触发后，一次性发送所有排队消息

**3. err 任务** ([transport.rs::err](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L323-L336)

   - 读取 stderr 并打印日志

### 2.4 异步初始化：[lib.rs::start_client](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/lib.rs#L948-L970)

```rust
tokio::spawn(async move {
    // 1. 发送 initialize 请求
    let value = _client.capabilities.get_or_try_init(|| {
        _client.initialize(enable_snippets)
            .map_ok(|response| response.capabilities)
    })

    // 2. 发送 initialized 通知
    _client.notify::<lsp::notification::Initialized>(lsp::InitializedParams {});

    // 3. 通知传输层可以发送其他请求了
    initialize_notify.notify_one();
});
```

**initialize 请求**：[client.rs::initialize](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L569-L773)

发送客户端能力声明：
- workspace 能力（workspace_folders, apply_edit, file_operations 等）
- text_document 能力（completion, hover, signature_help, code_action, rename, diagnostics 等）
- window 能力（work_done_progress, show_document 等）
- general 能力（position_encodings: [utf-8, utf-32, utf-16]）

---

## 三、请求分发机制

### 3.1 客户端 → 服务器 请求

**请求路径：commands/lsp.rs → client.rs → transport.rs

**示例：goto_definition**

1. **命令层**：[commands/lsp.rs
```rust
// 用户按 gd → 调用 language_server.goto_definition(...)
```

2. **客户端层**：[client.rs::goto_definition](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L1448-L1467)
```rust
pub fn goto_definition(...) -> Option<impl Future<Output = Result<Option<lsp::GotoDefinitionResponse>>> {
    // 检查 capability 检查
    Some(self.goto_request::<lsp::request::GotoDefinition>(...))
}
```

3. **通用请求方法**：[client.rs::call](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L437-L499)

```rust
fn call<R: lsp::request::Request>(&self, params: R::Params) -> impl Future<Output = Result<R::Result>>
```

**请求构造流程**：

```
1. 生成请求 ID: jsonrpc::MethodCall {
    jsonrpc: Some(Version::V2),
    id: next_request_id(),  // AtomicU64 自增
    method: R::METHOD,       // "textDocument/definition"
    params: value_into_params(params),
}

2. 创建 oneshot channel: (tx, rx)

3. 发送 Payload::Request { chan: tx, value: request }

4. 返回 Future: 等待 rx.recv() + timeout
```

**pending_requests 映射**：[transport.rs::Payload::Request](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L167-L171)

```rust
// send_payload_to_server 中：
self.pending_requests
    .lock()
    .await
    .insert(value.id.clone(), chan);
```

**响应匹配**：[transport.rs::process_request_response](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L223-L253)

```rust
// 收到 ServerMessage::Output 时：
let (id, result) = match output {
    Output::Success(...) => (id, Ok(result)),
    Output::Failure(...) => (id, Err(error.into())
};

if let Some(tx) = self.pending_requests.lock().await.remove(&id) {
    tx.send(result).await;
}
```

### 3.2 服务器 → 客户端 消息

**路径**：Transport::recv → client_tx → Registry::incoming → Editor 主循环 → Application::handle_language_server_message

**1. 传输层接收**：
```rust
// transport.rs::process_server_message
match msg {
    ServerMessage::Output(output) => process_request_response(...),
    ServerMessage::Call(call) => client_tx.send((self.id, call)),
}
```

**2. Registry 聚合**：[lib.rs::Registry](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/lib.rs#L581-L587)

```rust
pub struct Registry {
    inner: SlotMap<LanguageServerId, Arc<Client>>,
    incoming: SelectAll<UnboundedReceiverStream<(LanguageServerId, Call)>>,
    // ...
}
```

所有 LSP 客户端的 incoming receiver 被 select_all 合并到一个流中

**3. 主事件循环**：[editor.rs::wait_event](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-view/src/editor.rs#L2389-L2390)

```rust
tokio::select! {
    Some(message) = self.language_servers.incoming.next() => {
        return EditorEvent::LanguageServerMessage(message)
    }
}
```

**4. 应用层分发**：[application.rs::handle_language_server_message](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-term/src/application.rs#L762-L1110)

```rust
match call {
    Call::Notification(notification) => {
        let notification = Notification::parse(&method, params)?;
        match notification {
            Notification::Initialized => { /* 触发 LanguageServerInitialized 事件 }
            Notification::PublishDiagnostics(params) => { /* 处理诊断 }
            Notification::ShowMessage(params) => { /* 显示消息 }
            Notification::LogMessage(params) => { /* 日志 */ }
            Notification::ProgressMessage(params) => { /* 进度条 */ }
            Notification::Exit => { /* 清理资源 */ }
        }
    }
    Call::MethodCall(method_call) => {
        let method_call = MethodCall::parse(&method, params)?;
        match method_call {
            MethodCall::ApplyWorkspaceEdit(params) => { /* 应用 WorkspaceEdit }
            MethodCall::WorkspaceFolders => { /* 返回工作区目录 */ }
            MethodCall::WorkspaceConfiguration(params) => { /* 返回配置 */ }
            MethodCall::RegisterCapability(params) => { /* 注册能力 */ }
            // ... 其他方法调用
        }
        // 发送响应回服务器
        language_server.reply(id, reply);
    }
}
```

---

## 四、状态维护

### 4.1 Client 状态字段

[client.rs::Client](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L56-L71)

| 字段 | 类型 | 用途 |
|------|------|------|
| `id` | `LanguageServerId` | 唯一标识 |
| `name` | `String` | 服务器名称 |
| `_process` | `Child` | 子进程句柄 |
| `server_tx` | `UnboundedSender<Payload>` | 发送通道 |
| `request_counter` | `AtomicU64` | 请求 ID 计数器 |
| `capabilities` | `OnceCell<ServerCapabilities>` | 服务器能力缓存 |
| `file_operation_interest` | `OnceLock<FileOperationsInterest>` | 文件操作兴趣 |
| `config` | `Option<Value>` | 配置 |
| `root_path` | `PathBuf` | 根路径 |
| `root_uri` | `Option<Url>` | 根 URI |
| `workspace_folders` | `Mutex<Vec<WorkspaceFolder>>` | 工作区目录 |
| `initialize_notify` | `Arc<Notify>` | 初始化完成通知 |
| `req_timeout` | `u64` | 请求超时时间 |

### 4.2 Transport 状态

[transport.rs::Transport](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L43-L47)

| 字段 | 类型 | 用途 |
|------|------|------|
| `id` | `LanguageServerId` | 服务器 ID |
| `pending_requests` | `Mutex<HashMap<Id, Sender<Result<Value>>>>` | 等待响应的请求 |

**pending_requests 是请求-响应匹配的核心：
```
请求发送时插入 (id → Sender)
响应收到时移除并发送结果
```

### 4.3 初始化状态机

```
          start
           │
           ▼
    ┌─────────────┐
    │ is_pending │
    │  = true   │
    └──────┬──────┘
           │
           │ 只允许:
           │ - initialize 请求
           │ - initialized 通知
           │其他请求入队 pending_messages
           │
           ▼
    ┌──────────────┐
    │ initialize │
    │ 响应返回  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ notify()   │
    │ initialize_notify │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ is_pending │
    │  = false   │
    └──────┬───────┘
           │
           ▼ 发送 pending_messages
    ┌──────────────┐
    │ 正常处理请求 │
    └──────────────┘
```

### 4.4 文档生命周期状态

通过事件钩子维护：[handlers/lsp.rs::register_hooks](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-view/src/handlers/lsp.rs#L386-L430)

```
DocumentDidOpen → text_document_did_open
DocumentDidChange → text_document_did_change
DocumentDidClose → text_document_did_close
```

**text_document_did_change**：[client.rs::text_document_did_change](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L1054-L1097)

根据服务器能力选择同步策略：
- `TextDocumentSyncKind::FULL → 发送完整文档
- `TextDocumentSyncKind::INCREMENTAL → 发送增量变更
- `TextDocumentSyncKind::NONE → 不发送

### 4.5 诊断状态

[handlers/lsp.rs::handle_lsp_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-view/src/handlers/lsp.rs#L279-L365)

```rust
// Editor.diagnostics: HashMap<Uri, Vec<(lsp::Diagnostic, DiagnosticProvider)>>
```

- 按 URI 存储所有诊断
- 按服务器 ID 分组
- 版本检查防止过时诊断

### 4.6 多工作区状态

[client.rs::try_add_doc](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/client.rs#L74-L156)

```
打开新文档 → 检查是否已有匹配的 Client
→ 匹配 root_path
→ 或匹配 workspace_folders
→ 或检查 workspace_folders 能力
→ 支持则添加 folder
→ 不支持则创建新 Client
```

---

## 五、关键数据结构

### 5.1 Payload 枚举

[transport.rs::Payload](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/transport.rs#L22-L29)

```rust
pub enum Payload {
    Request {
        chan: Sender<Result<Value>>,
        value: jsonrpc::MethodCall,
    },
    Notification(jsonrpc::Notification),
    Response(jsonrpc::Output),
}
```

### 5.2 Call 枚举

[jsonrpc.rs::Call](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/jsonrpc.rs#L245-L254)

```rust
pub enum Call {
    MethodCall(MethodCall),    // 服务端 → 客户端 请求
    Notification(Notification), // 服务端 → 客户端 通知
    Invalid { id: Id },
}
```

### 5.3 MethodCall （客户端解析）

[lib.rs::MethodCall](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/lib.rs#L481-L491)

```rust
pub enum MethodCall {
    WorkDoneProgressCreate(...),
    ApplyWorkspaceEdit(...),
    WorkspaceFolders,
    WorkspaceConfiguration(...),
    RegisterCapability(...),
    UnregisterCapability(...),
    ShowDocument(...),
    WorkspaceDiagnosticRefresh,
    ShowMessageRequest(...),
}
```

### 5.4 Notification （客户端解析）

[lib.rs::Notification](file:///d:/fz/0601/solo-dogfeeding/code/266-helix/helix-lsp/src/lib.rs#L536-L545)

```rust
pub enum Notification {
    Initialized,    // 内部注入
    Exit,           // 内部注入
    PublishDiagnostics(...),
    ShowMessage(...),
    LogMessage(...),
    ProgressMessage(...),
}
```

---

## 六、代码路径总结

### 6.1 连接建立：
```
Registry::get()
  → start_client()
    → Client::start()
      → Command::spawn() 启动进程
      → Transport::start() 启动三个异步任务
        → recv/send/err 任务
    → tokio::spawn 初始化任务
      → client.initialize()
      → client.notify::<Initialized>()
      → initialize_notify.notify_one()
```

### 6.2 请求发送：
```
用户命令 → language_server.goto_definition()
  → client.goto_definition()
    → client.call::<GotoDefinition>()
      → 构造 MethodCall
      → 创建 oneshot channel
      → server_tx.send(Payload::Request)
        → Transport::send() 任务接收
          → send_payload_to_server()
            → pending_requests.insert(id, chan)
            → 写入 stdin

等待响应:
Transport::recv() 读取 stdout
  → process_server_message()
    → process_request_response()
      → pending_requests.remove(&id)
      → tx.send(result)
        → 原 Future 被唤醒
          → 解析结果返回给调用者
```

### 6.3 服务端通知：
```
Transport::recv() 读取 stdout
  → client_tx.send((id, Call::Notification))
    → Registry::incoming (SelectAll)
      → Editor::wait_event()
        → EditorEvent::LanguageServerMessage
          → Application::handle_editor_event()
            → handle_language_server_message()
              → Notification::parse()
              → 匹配处理（PublishDiagnostics/ShowMessage 等
```

### 6.4 服务端请求：
```
Transport::recv() 读取 stdout
  → client_tx.send((id, Call::MethodCall))
    → Registry::incoming
      → Editor::wait_event()
        → EditorEvent::LanguageServerMessage
          → Application::handle_editor_event()
            → handle_language_server_message()
              → MethodCall::parse()
              → 匹配处理（ApplyWorkspaceEdit 等
              → client.reply(id, result)
                → server_tx.send(Payload::Response)
```
