# Helix DAP 调试集成代码理解

## 整体架构概览

Helix 的 DAP (Debug Adapter Protocol) 调试集成横跨 4 个 crate，按照从底层到顶层的依赖关系为：

```
helix-dap-types    ← DAP 协议类型定义（纯数据结构）
    ↓
helix-dap          ← DAP 客户端、传输层、注册表（协议通信核心）
    ↓
helix-view         ← Editor 状态、事件处理、断点管理（视图层桥梁）
    ↓
helix-term         ← 命令入口、UI 渲染、键盘映射（终端交互层）
```

数据流方向：

```
用户按键/命令 → helix-term 命令层 → helix-view/editor 状态变更 → helix-dap Client 发请求
                                                                           ↓
                                                             DAP Adapter 进程 (stdio/TCP)
                                                                           ↓
UI 刷新 ← helix-view 事件处理 ← helix-dap Transport 接收 ← Adapter 响应/事件
```

---

## 一、会话启动（Session Startup）

### 1.1 入口：所有可用的启动方式

用户可以通过以下三类入口触发调试会话启动，它们最终都汇聚到同一个核心函数 `dap_start_impl`。

#### 1.1.1 键盘快捷键（Space 菜单）

在 Normal 模式下按 **`Space`** 进入 Space 菜单，再按 **`G`** 进入 Debug 子菜单（该子菜单标记为 `sticky=true`，即进入后可连续执行操作，无需每次重新按 Space）。

启动相关的按键如下（位于 [keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/keymap/default.rs#L239-L259)）：

| 按键序列 | 对应函数 | 说明 |
|---------|---------|------|
| `Space G l` | [dap_launch](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L234-L287) | 通过交互式 Picker 选择调试模板并启动 |
| `Space G r` | [dap_restart](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L289-L317) | 重启当前调试会话（需 Adapter 支持 restart 能力） |

#### 1.1.2 命令行（Typed 命令）

在 Normal 模式下按 `:` 进入命令行，可以使用以下调试相关命令（位于 [commands/typed.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L3598-L3630)）：

| 命令 | 别名 | 对应函数 | 说明 |
|------|------|---------|------|
| `:debug-start` | `dbg` | [debug_start](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2051-L2062) | 本地启动调试会话（传输方式由 `languages.toml` 中的 `debugger.transport` 配置决定，可能是 stdio 也可能是 tcp），可指定模板名和参数 |
| `:debug-remote` | `dbg-tcp` | [debug_remote](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2064-L2083) | 通过 TCP 地址连接已运行的远程 Adapter，再指定模板名和参数 |
| `:debug-eval` | （无别名） | [debug_eval](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2029-L2049) | 在当前调试上下文（栈帧）中求值表达式，**非启动命令** |

命令参数格式：
- `:debug-start [模板名] [参数1] [参数2] ...`
- `:debug-remote [host:port] [模板名] [参数1] [参数2] ...`

> **注意**：不要将命令入口等同于传输类型。`:debug-start` 只是"本地启动"的意思，实际使用 stdio 还是 tcp 取决于 `languages.toml` 中 `debugger.transport` 的配置。

#### 1.1.3 鼠标交互（仅用于设断点，非启动）

左键点击 gutter 区域可以切换断点（见 [ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/ui/editor.rs#L1260-L1276)），但无法启动会话。

### 1.2 启动流程详解

根据入口不同，启动流程分为两条路径，但核心都是 `dap_start_impl`。

#### 1.2.1 路径一：键盘 `Space G l` → dap_launch

[dap_launch](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L234-L287) 是交互式启动：

```
dap_launch()
  ├─ 1. 检查 editor.debug_adapters.get_active_client()，已有活跃调试器则报错返回
  ├─ 2. 从当前文档 language_config().debugger 获取 DebugAdapterConfig
  │     └─ 若不存在 → set_error("No debug adapter available for language") 并返回
  ├─ 3. 构建 Picker 列出 config.templates 供用户选择
  └─ 4. 用户选择模板后：
        ├─ 若 template.completion 为空 → 直接调用 dap_start_impl(cx, Some(&name), None, None)
        └─ 若 template.completion 非空 → 通过 debug_parameter_prompt() 逐项弹出 Prompt 收集参数
               └─ 参数收集完成后 → 调用 dap_start_impl(cx, Some(&name), None, Some(params))
```

#### 1.2.2 路径二：命令行 `:debug-start` / `:debug-remote` → debug_start / debug_remote

这两个函数直接解析命令行参数并调用 `dap_start_impl`。区别在于 `debug_remote` 第一个参数是 TCP 地址（解析为 SocketAddr 传入 socket 参数），而 `debug_start` 的 socket 参数为 None。

两个函数最终都调用：
```rust
dap_start_impl(cx, name.as_deref(), socket /* None 或 Some(address) */, Some(args))
```

### 1.3 dap_start_impl 核心逻辑

[dap_start_impl](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L116-L187) 是所有启动路径的汇合点，执行以下关键步骤：

```
dap_start_impl(cx, name, socket, params)
  │
  ├─ 步骤 A：创建 Client 并完成 initialize 握手（同步阻塞）
  │     ├─ editor.debug_adapters.start_client(socket, config)
  │     │    └─ Registry::start_client → Client 创建（见 1.4 节）
  │     │       ├─ 启动 Transport 层建立 stdio/TCP 连接
  │     │       ├─ block_on(client.initialize())  ← **同步等待 initialize 响应**
  │     │       └─ 将 Client 的 receiver 注册到 Registry.incoming 流
  │     └─ 此处已拿到 DebugAdapterId，Adapter 能力已存入 client.caps
  │
  ├─ 步骤 B：选择 DebugTemplate 并组装参数
  │     ├─ 根据 name 在 config.templates 中查找（name=None 则取第一个）
  │     ├─ 将 template.args 与 params 合并（支持 {0}/{1} 占位符替换）
  │     └─ 插入 cwd = 当前工作目录
  │
  └─ 步骤 C：异步发送 launch 或 attach 请求（非阻塞）
        ├─ 根据 template.request 判断：
        │     "launch" → debugger.launch(args)
        │     "attach" → debugger.attach(args)
        └─ 通过 dap_callback() 将 Future 提交给 Jobs 系统异步执行
              └─ dap_start_impl 函数此时立即返回 Ok(())，不等待 launch/attach 响应
```

### 1.4 传输路径与分发逻辑

所有 Client 的创建都经过 [Registry::start_client](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/registry.rs#L33-L59)，它根据 `socket` 参数是否存在，将请求分发到两条不同的路径：

```
Registry::start_client(socket, config)
   │
   ├─ socket == Some(addr) → Client::tcp(addr, id)   ← 纯 TCP 连接，不启动进程
   │
   └─ socket == None → Client::process(transport, command, args, port_arg, id)
                          │
                          ├─ transport=="tcp" && port_arg.is_some()
                          │    → Client::tcp_process(...)   ← 启动进程 + TCP 连接
                          │
                          ├─ transport=="stdio"
                          │    → Client::stdio(...)        ← 启动进程 + stdio 通信
                          │
                          └─ 其他 → 报错 "Incorrect transport"
```

因此，**共有三种实际的传输路径**，它们的触发条件、行为和适用场景各不相同。

#### 1.4.1 路径一：stdio 本地进程通信

**触发条件**：`socket == None` 且 `config.transport == "stdio"`

**方法**：[Client::stdio](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L114-L145)

**行为**：
- 使用 `helix_stdx::env::which()` 解析 Adapter 可执行文件路径
- 启动 Adapter 子进程，设置 `kill_on_drop(true)`（Client 销毁时自动杀死进程）
- 子进程的 stdin/stdout 用于 DAP 协议通信
- 子进程的 stderr 单独捕获，通过 Transport 层转发为日志输出
- `client.socket = None`（stdio 模式没有 TCP 端口）

**特点**：最常见的本地调试方式，通信延迟最低，进程生命周期与 Client 绑定。

#### 1.4.2 路径二：tcp 本地进程（自启动 + port_arg + 监听端口）

**触发条件**：`socket == None` 且 `config.transport == "tcp"` 且 `config.port_arg` 不为空

**方法**：[Client::tcp_process](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L165-L203)

**行为**：
- 先调用 `Self::get_port()` 分配一个随机可用的本地端口号（127.0.0.1 上，实际通过绑定 `:0` 后立即释放的方式获取空闲端口）
- 启动 Adapter 子进程，将 port_arg 中的 `{}` 替换为实际端口号作为额外参数传入（如 `--port 12345`）。Adapter 收到该参数后**在该端口上建立 TCP 服务器监听**（accept 循环）
- 子进程的 stdin/stdout/stderr 全部设为 null（不由 Helix 接管，DAP 消息全部走 TCP）
- **不设置** `kill_on_drop`（注释说明"adapter should exit automatically"，即期望被调试进程退出后 Adapter 自行退出）
- 等待 500ms 让 Adapter 启动并开始监听端口
- 通过 `TcpStream::connect(socket)` 建立 Helix 到 Adapter 监听端口的**第一条 TCP 连接**
- `client.socket = Some(socket)`（保存 TCP 监听端口地址，供后续子调试会话建立**新的**独立连接使用）

**特点**：适用于只能通过 TCP 通信的 Adapter，是唯一支持子调试会话的路径（因为需要监听端口 accept 多条连接）。

#### 1.4.3 路径三：tcp 客户端连接（连接到已存在的监听端）

**触发条件**：`socket == Some(address)`（即调用方显式传入了 TCP 地址和端口）

**方法**：[Client::tcp](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L105-L112)

**行为**：
- 直接通过 `TcpStream::connect(addr)` 连接到**已在运行的** Adapter 的 TCP 监听端
- 不启动任何子进程（Adapter 已由外部方式启动）
- 没有 stderr 捕获（stderr 不在 Helix 的控制范围内）
- `client.process = None`
- ⚠️ **注意**：此方法不会设置 `client.socket` 字段，`client.socket` 保持为 `None`

**适用场景**：
- `:debug-remote`：连接到另一台机器或后台已启动的 Adapter
- StartDebugging 子调试会话：父调试器通过 tcp_process 路径启动后，复用同一个端口地址建立**第二条独立 TCP 连接**

**特点**：该路径建立的 Client 由于 `client.socket == None`，**无法进一步启动子调试会话**（见 2.6.2 节）。

#### 1.4.4 命令入口与传输路径的对应关系

| 命令入口 | socket 参数 | 实际传输路径 | 说明 |
|---------|------------|-------------|------|
| `Space G l` | `None` | 取决于 `config.transport` | 本地启动，传输方式由配置决定 |
| `:debug-start` | `None` | 取决于 `config.transport` | 同左 |
| `:debug-remote host:port` | `Some(host:port)` | tcp 客户端连接 | 连接已运行的 Adapter |
| StartDebugging 反向请求 | `Some(父调试器.socket)` | tcp 客户端连接（同一监听端口，独立 TCP 连接） | 子调试会话建立到 Adapter 的第二条独立连接 |

> **关键区分**：命令入口描述的是"用户如何触发"，传输路径描述的是"底层如何通信"。二者不是一一对应的。`:debug-start` 可能走 stdio 也可能走 tcp 本地进程，取决于 `languages.toml` 配置。
>
> **另一层关键区分**："复用同一 TCP 端口地址" ≠ "复用同一条 TCP 连接"。同一端口地址上可以建立多条独立的 TCP 连接（这是 TCP 协议的标准 accept 机制），父子调试会话之间互不干扰。

#### 1.4.5 统一出口：Client::streams

三种路径最终都汇聚到 [Client::streams](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L72-L103)，它：
- 创建 `Transport` 层处理底层的消息编解码
- 启动 `recv` 协程转发 Adapter 消息到上层
- 返回 `(Client, UnboundedReceiver<(DebugAdapterId, Payload)>)` 对

所有路径创建的 Client 结构在语义上是等价的，上层代码无需关心底层使用哪种传输方式。

### 1.5 初始化握手协议（initialize）

DAP 规范要求所有操作之前先完成 initialize 握手。在 Helix 中，这个握手是 **同步阻塞**执行的，发生在 [Registry::start_client](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/registry.rs#L53-L54)：

```rust
block_on(client.initialize(config.name.clone()))?;
client.quirks = config.quirks.clone();
```

时序如下：

```
Helix (Client)                          DAP Adapter
     │                                       │
     │  ① 请求：initialize                   │
     │  ──────────────────────────────────► │
     │     (client_id="hx", adapter_id,     │
     │      supports_run_in_terminal=true,  │
     │      supports_progress_reporting=...) │
     │                                       │
     │  ② 响应：DebuggerCapabilities         │
     │  ◄────────────────────────────────── │
     │     (存入 client.caps)                │
     │                                       │
```

[Client::initialize](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L371-L392) 发送 `InitializeArguments`，声明 Helix 作为调试客户端的能力：
- `client_id/client_name = "hx"/"helix"`
- `lines_start_at_one/columns_start_at_one = true`（行列从 1 开始计数）
- `path_format = "path"`
- `supports_variable_type = true`
- `supports_run_in_terminal_request = true`（允许 Adapter 反向请求在终端运行程序）
- `supports_progress_reporting = true`

Adapter 返回 `DebuggerCapabilities`，存入 `client.caps`，后续操作（如条件断点、exception breakpoint 等）会根据这些能力判断是否可用。

**关键注意**：initialize 是 block_on 同步执行的，在其返回之前，Helix 主循环被阻塞，不会处理用户输入或渲染。这保证了后续 launch/attach 请求发送时，Adapter 能力已经确定。

### 1.6 launch/attach 请求的异步发送

initialize 完成后，`dap_start_impl` 根据 `DebugTemplate.request` 字段的值（`"launch"` 或 `"attach"`）通过 [dap_callback](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L94-L114) 异步提交请求：

```rust
match &template.request[..] {
    "launch" => {
        let call = debugger.launch(args);
        dap_callback(cx.jobs, call, callback);   // 异步执行，立即返回
    }
    "attach" => {
        let call = debugger.attach(args);
        dap_callback(cx.jobs, call, callback);   // 异步执行，立即返回
    }
    request => bail!("Unsupported request '{}'", request),
};
// dap_start_impl 立即返回 Ok(())
```

[Client::launch](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L410-L414) 和 [Client::attach](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L416-L420) 除了发送 DAP 请求外，还会设置：
- `connection_type = Some(ConnectionType::Launch)` 或 `ConnectionType::Attach`
- `starting_request_args = Some(args.clone())`（用于后续 restart）

### 1.7 Initialized 事件与配置完成

Adapter 在 launch/attach 请求处理完成、**准备好接收配置**（如断点、异常过滤）时，会发送 `Initialized` **事件**（注意是 Event 不是 Response）。

[handle_debugger_message](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L375-L396) 处理 `Initialized` 事件时，完成最后的配置阶段：

```
Initialized 事件处理
  ├─ 1. set_status("Debugger initialized...")
  ├─ 2. 同步已有断点：
  │     └─ 遍历 editor.breakpoints 中所有文件的断点
  │        └─ 对每个 path 调用 breakpoints_changed(debugger, path, breakpoints)
  │              └─ 发送 setBreakpoints 请求给 Adapter
  ├─ 3. 发送 configurationDone 请求（仅当 Adapter 支持时）
  │     └─ 通知 Adapter：所有初始配置已发送完毕，可以开始执行被调试程序
  ├─ 4. 成功则 set_status("Debugged application started")
  └─ 5. debug_adapters.set_active_client(id)  ← 标记当前 Client 为活跃调试器
```

这一步完成后，调试会话才算真正建立，用户可以进行步进、查看变量等操作。

### 1.8 启动全流程时序图

综合 1.3–1.7 节，完整时序如下：

```
用户输入
   │
   ▼
dap_start_impl()
   │
   ├─► Registry::start_client() ──┐
   │     （同步阻塞）              │
   │                               │
   │          ┌────────────────────┴────────────┐
   │          │  ① Transport 建立连接            │
   │          │     stdio / TCP                  │
   │          │                                   │
   │          │  ② block_on(initialize)          │
   │          │     Client → "initialize"        │
   │          │     Adapter → DebuggerCapabilities│
   │          │     存入 client.caps             │
   │          └────────────────────┬────────────┘
   │                               │
   ├─ 选择 DebugTemplate、组装参数
   │
   ├─► dap_callback(launch/attach) ──┐
   │     （异步提交，立即返回）       │
   │                                  │
   │          ┌───────────────────────┴──────────────┐
   │          │  ③ Client → "launch" 或 "attach"      │
   │          │     （异步，通过 Jobs 调度）          │
   │          └───────────────────────┬──────────────┘
   │                                  │
dap_start_impl 返回 Ok(())            │
   ▼                                  │
（主循环继续处理事件）                 │
                                      │
          ┌───────────────────────────┴───────────────┐
          │  ④ Adapter → "initialized" 事件           │
          │     （通过 Registry.incoming 流到达）      │
          └───────────────────────────┬───────────────┘
                                      │
                                      ▼
                       handle_debugger_message()
                       ├─ 同步已有断点 → setBreakpoints
                       ├─ configurationDone
                       ├─ set_status("Debugged application started")
                       └─ set_active_client(id)
                                      │
                                      ▼
                              调试会话就绪
```

---

## 二、协议通信（Protocol Communication）

### 2.1 Transport 层

[Transport](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/transport.rs#L52-L57) 是底层通信引擎，负责：

- **消息格式**：DAP 使用 HTTP 风格的头部 + JSON body 格式：`Content-Length: N\r\n\r\n{json}`
- **发送** ([send_payload_to_server](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/transport.rs#L151-L163))：将 Payload 序列化为 JSON，加上 Content-Length 头，写入 stdin/TCP 流
- **接收** ([recv_server_message](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/transport.rs#L85-L136))：按行读取 header 解析 Content-Length，再读取指定长度的 body，反序列化为 Payload
- **请求-响应匹配**：使用 `pending_requests: HashMap<u64, Sender<Result<Response>>>` 将 seq 与回调 channel 对应

### 2.2 Payload 三种类型

[Payload](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/transport.rs#L41-L50) 枚举：

```rust
pub enum Payload {
    Event(Event),      // Adapter → Client 的单向通知（如 stopped、continued、initialized）
    Response(Response), // Adapter 对 Client 请求的响应
    Request(Request),   // Adapter → Client 的反向请求（如 runInTerminal）
}
```

### 2.3 消息流向

**Client → Adapter（请求发送）**：

```
Client::call<R>() 
  → 构造 Request{seq, command, arguments, back_ch}
  → server_tx.send(Payload::Request(req))
  → Transport::send_payload_to_server() 
    → 注册 back_ch 到 pending_requests[seq]
    → 序列化并写入流
  → 在 back_ch 上等待响应（20s 超时）
```

**Adapter → Client（响应返回）**：

```
Transport::recv_server_message() 读到 Payload::Response
  → process_server_message()
    → 从 pending_requests 中取出 request_seq 对应的 back_ch
    → 通过 back_ch 发送 Response
  → Client::call() 中的 callback_rx 收到响应
```

**Adapter → Client（事件通知）**：

```
Transport::recv_server_message() 读到 Payload::Event
  → process_server_message()
    → 通过 client_tx 转发
  → Client::recv() 协程
    → 加上 DebugAdapterId 标识
    → 通过 client_tx 转发到 Registry.incoming 流
```

**Adapter → Client（反向请求）**：

```
Transport 读到 Payload::Request
  → 同 Event 一样通过 client_tx 转发到上层
  → 在 handle_debugger_message 中处理（如 RunInTerminal、StartDebugging）
  → 处理完后通过 Client::reply() 发回响应
```

### 2.4 事件循环集成

[Editor::wait_event](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/editor.rs#L2375-L2415) 中的 `tokio::select!` 同时监听多个事件源：

```rust
Some(event) = self.debug_adapters.incoming.next() => {
    return EditorEvent::DebuggerEvent(event)
}
```

在 [Application 的事件循环](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/application.rs#L662-L667) 中：

```rust
EditorEvent::DebuggerEvent((id, payload)) => {
    let needs_render = self.editor.handle_debugger_message(id, payload).await;
    if needs_render { self.render().await; }
}
```

### 2.5 异步回调模式

[debugger 宏](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L17-L24)：`debugger!(editor)` 从 `editor.debug_adapters` 获取活跃 Client 的可变引用，如果不存在则提前 return。

[dap_callback](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L94-L114)：将 DAP 请求封装为异步回调，通过 `Jobs` 系统调度：

```rust
jobs.callback(async {
    let json = call.await?;                          // 等待 DAP 响应
    let response = serde_json::from_value(json)?;    // 反序列化
    Ok(Callback::EditorCompositor(Box::new(|editor, compositor| {
        callback(editor, compositor, response)        // 在主循环中执行回调
    })))
});
```

### 2.6 反向请求与子调试会话

DAP 协议是双向的：不仅 Client 可以向 Adapter 发请求，Adapter 也可以向 Client 发**反向请求**（Reverse Request）。Helix 目前处理两种反向请求：`RunInTerminal` 和 `StartDebugging`。

反向请求的通用处理流程（位于 [handlers/dap.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L484-L582)）：

```
收到 Payload::Request
  ├─ Request::parse(command, arguments) 解析请求类型
  ├─ 匹配具体请求类型，执行处理逻辑，返回 Result<Value, Error>
  └─ 通过 debugger.reply(request.seq, &command, reply) 发回响应
```

#### 2.6.1 RunInTerminal 反向请求

**触发条件**：
- Adapter 需要在外部终端中运行被调试程序时发送
- 常见场景：调试模板中设置了 `runInTerminal: true`（如 `languages.toml` 中的 `"binary (terminal)"` 模板）
- Helix 需要在 `config.terminal` 中配置了外部终端命令

**处理逻辑**（位于 [handlers/dap.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L486-L512)）：
1. 检查 `self.config().terminal` 是否配置了外部终端
2. 使用 `std::process::Command::new` 启动外部终端，传入 Adapter 提供的 `args`
3. 返回 `RunInTerminalResponse`，包含 `process_id`

**失败处理**：
- 未配置终端 → `set_error("No external terminal defined")`
- 启动失败 → `set_error("Error starting external terminal: ...")`

#### 2.6.2 StartDebugging 反向请求（子调试会话）

**触发条件**：
- Adapter 需要启动一个**子调试会话**（child debug session）时发送
- 常见场景：调试父进程 fork 出子进程、调试多进程程序时需要同时调试子进程
- 前置限制：**父调试器必须使用 TCP 传输且 `client.socket` 不为 None**

**处理逻辑**（位于 [handlers/dap.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L513-L572)）：

```
StartDebugging 请求处理
  ├─ 1. 从 self.debug_adapters 获取父调试器 Client
  ├─ 2. 检查 debugger.socket 是否存在
  │     └─ 若为 None → set_error("Child debugger can only be started if the parent debugger is using TCP transport.")
  ├─ 3. 获取父调试器的 config（克隆）
  ├─ 4. 调用 debug_adapters.start_client(Some(socket), &config) 创建子调试器
  │     └─ 传入父调试器的 TCP socket 地址（同一个端口号）
  │        走 Client::tcp 路径：TcpStream::connect(socket)
  │        → 与 Adapter 监听端口建立**第二条独立的 TCP 连接**
  ├─ 5. start_client 内部：
  │     ├─ 为子调试器分配新的独立 DebugAdapterId
  │     ├─ 子调试器独立执行 block_on(client.initialize())  ← 独立的握手
  │     └─ 子调试器的独立 receiver 推入 Registry.incoming (SelectAll)
  ├─ 6. 根据 arguments.request 判断启动方式（在子调试器上）：
  │     ├─ ConnectionType::Launch → child_client.launch(arguments.configuration)
  │     └─ ConnectionType::Attach → child_client.attach(arguments.configuration)
  ├─ 7. 返回 { success: true } 响应
  └─ 8. 通过父调试器的 debugger.reply() 将响应发回给 Adapter
```

**连接模型：复用同一监听端口，建立独立 TCP 连接**

子调试器的连接模型可以类比为 HTTP 服务器与多个浏览器：
- **Adapter 端**：启动时监听一个 TCP 端口（如 `127.0.0.1:12345`），该端口可接受多条独立的 TCP 连接（使用标准的服务器 Socket accept 机制）
- **父调试器**：通过 `TcpStream::connect` 建立**第一条** TCP 连接，用于父进程调试会话
- **子调试器**：通过 `TcpStream::connect` 连接到**同一个端口**，建立**第二条独立的** TCP 连接，用于子进程调试会话
- **每条连接完全独立**：各自拥有独立的 BufReader/BufWriter、独立的 Transport、独立的消息队列、独立的请求-响应匹配（pending_requests）

**⚠️ 纠正**：不是"同一条连接里的多逻辑通道"，而是"同一监听端口上的多条独立 TCP 连接"。DAP 协议本身没有多路复用机制，每条连接对应一个独立的 DAP 会话。

**父子调试会话的关系**：

| 维度 | 父调试会话 | 子调试会话 |
|------|-----------|-----------|
| 触发方式 | 用户命令（Space G l / :debug-start） | Adapter 发送 StartDebugging 反向请求 |
| DebugAdapterId | 独立分配 | 再次独立分配（SlotMap 新键） |
| TCP 连接 | 第一条，通过 tcp_process 建立 | 第二条，通过 Client::tcp 建立到同一端口 |
| initialize 握手 | 独立执行，存入父 client.caps | 再次独立执行，存入子 client.caps |
| 消息通道 | 独立的 Transport、独立的 receiver | 独立的 Transport、独立的 receiver |
| 消息汇聚 | receiver 推入 SelectAll | 同一 SelectAll，每条消息自带 id 区分来源 |
| launch/attach | 由 dap_start_impl 异步触发 | 由 StartDebugging 处理逻辑直接 await |
| 调试目标 | 原始被调试进程（父进程） | fork 出的子进程或附加的目标 |
| 断点管理 | 独立的断点集合？* | 独立的断点集合？* |
| 活跃状态 | 初始被设为 active_client | 不会自动设为 active，需用户切换 |

*注：断点存储在 Editor.breakpoints 中是全局的（按文件路径索引），所有调试 Client 共享同一断点列表，但每个 Client 会各自发送 setBreakpoints 请求，Adapter 端区分哪些断点属于哪个会话。

**initialize 的独立性**：
- 父调试器：在 `Registry::start_client` 中 `block_on(client.initialize(config.name.clone()))` 同步执行
- 子调试器：同样在 `Registry::start_client` 中独立执行 `block_on(client.initialize(config.name.clone()))`，不依赖父调试器的状态
- 两次 initialize 之间没有任何共享数据，完全独立

**消息接收与区分**：
- 每个 Client 创建时都会将独立的 receiver 通过 `self.incoming.push(...)` 推入 Registry 的 `SelectAll<UnboundedReceiverStream<...>>`
- `SelectAll` 将多条流合并为一条，任何一条 receiver 上有消息都会被取出
- 每条消息都附带 `DebugAdapterId`（由 `Client::recv` 协程在转发时加上 `(id, Payload)`），因此上层可以准确区分消息来源
- 反向请求的回复通过 `debugger.reply(request.seq, ...)` 发送到**收到该请求的特定 Client**，不会干扰其他会话

**限制与注意事项**：
| 限制项 | 说明 |
|--------|------|
| 传输方式限制 | 父调试器必须使用 TCP 传输（client.socket 不为 None） |
| stdio 模式 | 不支持子调试（stdio 是一对一的，无法 accept 多条连接） |
| `:debug-remote` | 不支持子调试（`Client::tcp` 创建的 Client 的 `socket` 字段为 None，即使底层是 TCP） |
| 配置来源 | 子调试器复用父调试器的 `DebugAdapterConfig` |
| 会话数量 | 理论上可以有多个子调试器，由 SlotMap 统一管理，每条独立 TCP 连接 |
| Adapter 支持 | 需要 Adapter 自身实现支持多连接（服务器端 Socket accept 循环） |

> ⚠️ **代码细节**：`Client::tcp` 方法（远程连接路径）不会设置 `client.socket` 字段，因此通过 `:debug-remote` 连接的调试器即使底层是 TCP，也无法启动子调试会话。只有通过 `Client::tcp_process` 路径（本地 tcp+port_arg）创建的 Client 才会设置 `socket` 字段。这是当前实现的一个特性，可能是为了避免远程场景下假设端口可达性问题。

---

## 三、断点状态管理（Breakpoint State）

### 3.1 断点数据结构

[Breakpoint](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/editor.rs#L1173-L1184) 存储在 Editor 中：

```rust
pub struct Breakpoint {
    pub id: Option<usize>,          // DAP Adapter 分配的 ID
    pub verified: bool,             // Adapter 是否确认断点有效
    pub message: Option<String>,    // Adapter 返回的断点状态消息
    pub line: usize,                // 0-indexed 行号
    pub column: Option<usize>,
    pub condition: Option<String>,  // 条件断点表达式
    pub hit_condition: Option<String>, // 命中次数条件
    pub log_message: Option<String>,   // 日志点消息
}
```

存储位置：`Editor.breakpoints: HashMap<PathBuf, Vec<Breakpoint>>`

### 3.2 断点设置方式

有两种方式设置断点：

1. **键盘命令** `Space G b`：[dap_toggle_breakpoint](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L390-L402) 在当前光标行切换断点
2. **鼠标点击 gutter**：[ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/ui/editor.rs#L1260-L1276) 中，左键点击 gutter 区域时调用 `dap_toggle_breakpoint_impl`

### 3.3 断点切换逻辑

[dap_toggle_breakpoint_impl](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L404-L428)：

```
1. 在 editor.breakpoints[path] 中查找当前行
2. 如果已存在 → 移除（toggle off）
3. 如果不存在 → 添加默认 Breakpoint（toggle on）
4. 调用 breakpoints_changed() 同步到 Adapter
```

### 3.4 断点同步到 Adapter

[breakpoints_changed](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L90-L144) 是断点同步的核心函数：

1. **能力检查**：验证 Adapter 是否支持条件断点、命中条件断点、日志点
2. **0→1 索引转换**：Helix 内部使用 0-indexed 行号，DAP 协议使用 1-indexed，在构造 `SourceBreakpoint` 时 `line + 1`
3. **发送 setBreakpoints 请求**：对整个文件的所有断点一次性发送（DAP 协议要求按文件设置断点，每次发送该文件的全部断点列表）
4. **回写 Adapter 返回的断点信息**：将 Adapter 返回的 `verified`、`message`、修正后的 `line` 等信息更新到 Editor 的 breakpoints 中

### 3.5 Adapter 主动推送的断点事件

Adapter 可以通过 `Breakpoint` 事件主动通知断点状态变化，[handle_debugger_message](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L242-L317) 处理三种 reason：

| reason | 处理 |
|--------|------|
| `new` | 在对应 path 的 breakpoints 中新增 Breakpoint |
| `changed` | 根据 id 找到已有断点，更新 verified/message/line/column |
| `removed` | 根据 id 找到并移除断点 |

### 3.6 会话终止时的断点状态

当调试会话终止（`Terminated` 事件），如果不需要重启，会遍历所有断点将 `verified` 设为 `false`，这样 UI 上断点标记从实心 `●` 变为空心 `◯`，提示用户断点不再生效。

---

## 四、界面联动（UI Interaction）

### 4.1 断点 Gutter 渲染

[breakpoints gutter 函数](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/gutter.rs#L230-L272)：

- 从 `editor.breakpoints` 中获取当前文档的断点列表
- 对每一行，查找是否有匹配行号的断点
- 根据 `breakpoint.verified` 选择符号：已验证 `●`（实心圆）、未验证 `◯`（空心圆）
- 根据断点类型选择样式：
  - 既有 condition 又有 log_message → `error` + 下划线
  - 只有 condition → `error` 样式
  - 只有 log_message → `info` 样式
  - 普通断点 → `ui.debug.breakpoint` 样式

### 4.2 当前执行位置指示器

[execution_pause_indicator](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/gutter.rs#L274-L308)：

- 从 `editor.current_stack_frame()` 获取当前暂停的栈帧
- 比较栈帧的 source path 与当前文档 path
- 在暂停行显示 `▶` 符号，使用 `ui.debug.active` 样式
- 仅在当前焦点视图、首次视觉行时显示

### 4.3 Gutter 合并渲染

[diagnostics_or_breakpoints](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/gutter.rs#L310-L326) 将三个 gutter 层合并，优先级从高到低：

1. **执行暂停指示器** `▶`（最高优先级）
2. **断点标记** `●` / `◯`
3. **诊断信息**（最低优先级）

### 4.4 状态栏信息

各种 DAP 事件都会更新 Editor 的状态消息：

- `Stopped` 事件 → `"Thread X stopped because of Y"`
- `Continued` 事件 → resume_application 清除线程状态
- `Output` 事件 → `"Debug (category): output"`
- `ProgressStart/Update/End` → 进度信息
- `Initialized` 事件 → `"Debugger initialized..."` → `"Debugged application started"`

### 4.5 调试操作与界面联动（完整键盘映射）

Normal 模式下按 **`Space G`** 进入 Debug sticky 子菜单，以下是该子菜单下所有可用操作（见 [keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/keymap/default.rs#L239-L259)）：

| 操作 | 子菜单内按键 | 函数 | 界面效果 |
|------|------------|------|----------|
| 启动调试 | `l` | [dap_launch](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L234-L287) | 弹出模板选择 Picker |
| 重启调试 | `r` | [dap_restart](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L289-L317) | 重启当前调试会话 |
| 切换断点 | `b` | [dap_toggle_breakpoint](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L390-L402) | gutter 显示/隐藏断点标记 |
| 继续执行 | `c` | [dap_continue](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L430-L447) | 清除暂停指示器 |
| 暂停 | `h` | [dap_pause](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L449-L458) | 弹出线程 Picker |
| 单步进入 | `i` | [dap_step_in](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L460-L473) | 清除暂停指示器，等待 Stopped 事件 |
| 单步跳出 | `o` | [dap_step_out](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L475-L487) | 同上 |
| 单步跳过 | `n` | [dap_next](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L489-L501) | 同上 |
| 查看变量 | `v` | [dap_variables](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L503-L585) | 弹出变量 Popup |
| 终止调试 | `t` | [dap_terminate](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L587-L608) | 清除断点验证状态 |
| 编辑条件 | `C-c` | [dap_edit_condition](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L644-L683) | 弹出条件输入 Prompt |
| 编辑日志 | `C-l` | [dap_edit_log](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L685-L723) | 弹出日志消息 Prompt |
| 切换线程 | `s t` | [dap_switch_thread](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L725-L729) | 弹出线程 Picker |
| 切换栈帧 | `s f` | [dap_switch_stack_frame](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L730-L783) | 弹出栈帧 Picker |
| 启用异常 | `e` | [dap_enable_exceptions](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L610-L627) | 设置异常断点过滤器 |
| 禁用异常 | `E` | [dap_disable_exceptions](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L629-L641) | 清除异常断点过滤器 |

**注意**：因为 Debug 子菜单 `sticky=true`，进入 `Space G` 后可以连续按键。例如按 `Space G` 进入 Debug 菜单后，直接按 `b` 切换断点，再按 `n` 单步跳过，无需每次都按 `Space G`。

### 4.6 Stopped 事件后的界面联动

当 Adapter 发送 `Stopped` 事件时，[handle_debugger_message](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L168-L218) 的处理链路：

```
Stopped 事件
  ├─ all_threads_stopped?
  │     ├─ true  → 获取所有线程 + 每个线程 fetch_stack_trace
  │     └─ false → 仅对 thread_id 的线程 fetch_stack_trace
  ├─ select_thread_id() → 设置当前线程 + 跳转到栈帧位置
  │     ├─ debugger.thread_id = Some(thread_id)
  │     ├─ fetch_stack_trace() → 获取并缓存 stack_frames
  │     └─ jump_to_stack_frame() → 打开文件 + 设置选区 + 居中视图
  └─ set_status() → 状态栏显示停止原因
```

[jump_to_stack_frame](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L60-L88) 的关键步骤：

1. 从 StackFrame 提取 source path
2. `editor.open(&path, Action::Replace)` 打开文件
3. 将 DAP 的 1-indexed 位置转换为 Helix 的 0-indexed 字符偏移
4. 设置 Selection 并居中视图

### 4.7 线程 Picker

[thread_picker](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L24-L78) 展示所有线程的名称和状态（running/stopped 等），支持预览（定位到线程的第一帧位置），选择后执行回调。

---

## 五、关键数据流总结

### 5.1 完整的调试会话生命周期

```
1. 用户按 Space G l → dap_launch
   ↓
2. 选择 DebugTemplate → dap_start_impl
   ↓
3. Registry::start_client → 创建 Client → Transport 启动 → block_on(initialize) ← 同步握手
   ↓
4. Client::launch/attach → 异步通过 dap_callback 提交 → dap_start_impl 返回
   ↓
5. Adapter 发送 Initialized 事件（通过 Registry.incoming 流）
   ↓
6. handle_debugger_message: 同步断点 → configurationDone → set_active_client
   ↓
7. 程序运行中...（用户可设置断点、步进等）
   ↓
8. Adapter 发送 Stopped 事件
   ↓
9. handle_debugger_message: 获取线程/栈帧 → 跳转位置 → 显示暂停指示器
   ↓
10. 用户按 Space G c → dap_continue → resume_application → 清除暂停指示器
   ↓
... 循环 7-10 ...
   ↓
11. Adapter 发送 Terminated 事件 或 用户按 Space G t → dap_terminate
   ↓
12. 断开连接 → 清除断点验证状态 → unset_active_client
```

### 5.2 Registry 的多客户端管理

[Registry](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/registry.rs#L12-L21) 使用 SlotMap 管理多个 Client：

- `inner: SlotMap<DebugAdapterId, Client>` — 所有客户端的存储
- `current_client_id: Option<DebugAdapterId>` — 当前活跃客户端
- `incoming: SelectAll<...>` — 合并所有 Client 的消息流为统一的事件源

`SelectAll` 使得 Editor 的事件循环只需监听一个 `incoming` 流，即可接收所有调试器的消息，每条消息携带 `DebugAdapterId` 以区分来源。

### 5.3 Client 内部状态

[Client](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L30-L50) 维护的关键运行时状态：

| 字段 | 类型 | 用途 |
|------|------|------|
| `id` | `DebugAdapterId` | Client 的唯一标识（SlotMap 键） |
| `_process` | `Option<Child>` | Adapter 子进程（stdio/tcp_process 路径有，tcp 远程连接路径没有） |
| `server_tx` | `UnboundedSender<Payload>` | 向 Adapter 发送消息的通道 |
| `request_counter` | `AtomicU64` | 请求序列号计数器 |
| `caps` | `Option<DebuggerCapabilities>` | Adapter 能力（initialize 后填充） |
| `socket` | `Option<SocketAddr>` | TCP 端口地址（tcp_process 路径设置，用于子调试会话复用） |
| `stack_frames` | `HashMap<ThreadId, Vec<StackFrame>>` | 每个线程的栈帧缓存 |
| `thread_states` | `ThreadStates` (= `HashMap<ThreadId, String>`) | 线程运行状态 |
| `thread_id` | `Option<ThreadId>` | 当前活跃线程 |
| `active_frame` | `Option<usize>` | 当前线程的活跃栈帧索引 |
| `progress` | `ProgressMap` | 进度报告状态 |
| `connection_type` | `Option<ConnectionType>` | Launch 或 Attach |
| `starting_request_args` | `Option<Value>` | 启动参数（用于 restart） |
| `quirks` | `DebuggerQuirks` | Adapter 的特殊行为适配 |
| `config` | `Option<DebugAdapterConfig>` | 启动时使用的配置（用于子调试会话复用） |

### 5.4 反向请求处理

Adapter 可以向 Client 发送反向请求，详细处理逻辑见 2.6 节。总结两种反向请求：

- **RunInTerminal**：Adapter 请求在外部终端运行被调试程序。触发条件：调试模板设置了 `runInTerminal: true`，且 Helix 配置了外部终端。
- **StartDebugging**：Adapter 请求启动子调试会话。触发条件：Adapter 检测到需要调试子进程（如 fork 场景）。限制：父调试器必须通过 tcp_process 路径创建（`client.socket` 不为 None，说明 Adapter 在监听端口 accept 连接）。子调试器连接到同一监听端口，建立**独立的新 TCP 连接**，各自独立做 initialize 握手。

---

## 六、模块间依赖关系图

```
┌───────────────────────────────────────────────────────────┐
│                       helix-term                            │
│  ┌────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │ commands/dap.rs│  │ commands/typed.rs│  │ ui/editor.rs│ │
│  │ (Space G 命令) │  │ (:debug-start,   │  │ (鼠标设断点) │ │
│  │                │  │  :debug-remote,  │  │             │ │
│  │                │  │  :debug-eval)    │  │             │ │
│  └───────┬────────┘  └────────┬─────────┘  └──────┬──────┘ │
│          │                   │                    │        │
│  ┌───────┴───────────────────┴────────────────────┴──────┐ │
│  │           keymap/default.rs (Space G 子菜单)           │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────┴──────────────────────────────┐
│                        helix-view                             │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │ handlers/dap.rs  │  │          editor.rs               │ │
│  │ (事件处理/断点同步)│  │ (breakpoints/Registry/事件循环)  │ │
│  └────────┬─────────┘  └──────────────┬───────────────────┘ │
│           │                           │                     │
│  ┌────────┴───────────────────────────┴───────────────────┐ │
│  │           gutter.rs (断点/暂停指示器渲染)                │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────┴──────────────────────────────┐
│                        helix-dap                              │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────────────┐  │
│  │  client.rs   │  │ transport.rs│  │   registry.rs     │  │
│  │ (DAP 客户端)  │  │ (消息编解码) │  │  (多客户端管理)   │  │
│  └──────┬───────┘  └──────┬──────┘  └───────────────────┘  │
└─────────┼─────────────────┼────────────────────────────────┘
          │                 │
┌─────────┴─────────────────┴────────────────────────────────┐
│                   helix-dap-types                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  lib.rs (Request/Event/Response 类型、Breakpoint 等)    ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 七、配置来源

调试器配置来自 `languages.toml` 中的 `[language.debugger]` 段，结构对应 [DebugAdapterConfig](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-core/src/syntax/config.rs#L486-L497)：

```toml
[[language]]
name = "rust"

[language.debugger]
name = "lldb-dap"
transport = "stdio"
command = "lldb-dap"

[[language.debugger.templates]]
name = "binary"
request = "launch"
completion = [ { name = "binary", completion = "filename" } ]
args = { program = "{0}" }
```

配置字段说明：
- `name`: Adapter 标识符（传递给 initialize 请求的 adapter_id）
- `transport`: `"stdio"` 或 `"tcp"`
- `command`: Adapter 可执行文件
- `args`: Adapter 启动参数（不是被调试程序的参数）
- `port_arg`: TCP 模式的端口参数格式（如 `"--port {}"`）
- `templates`: 调试模板列表，每个模板定义：
  - `name`: 模板名（用于 `:debug-start <name>` 选择）
  - `request`: `"launch"` 或 `"attach"`
  - `completion`: 参数补全配置列表（Named 或 Advanced）
  - `args`: 启动参数（支持 `{0}`、`{1}` 等占位符）
- `quirks`: Adapter 行为适配（如 `absolute_paths`）
