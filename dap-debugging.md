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
| `:debug-start` | `dbg` | [debug_start](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2051-L2062) | 使用 stdio 启动本地调试会话，可指定模板名和参数 |
| `:debug-remote` | `dbg-tcp` | [debug_remote](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2064-L2083) | 通过 TCP 地址连接远程 Adapter，再指定模板名和参数 |
| `:debug-eval` | （无别名） | [debug_eval](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs#L2029-L2049) | 在当前调试上下文（栈帧）中求值表达式，**非启动命令** |

命令参数格式：
- `:debug-start [模板名] [参数1] [参数2] ...`
- `:debug-remote [host:port] [模板名] [参数1] [参数2] ...`

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

### 1.4 Client 创建与连接方式

[Client::process](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L55-L70) 根据 transport 类型和 socket 参数分发到不同的连接方式：

| 场景 | 方式 | 方法 | 说明 |
|------|------|------|------|
| `:debug-start` 或 `Space G l` | `stdio` | [Client::stdio](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L114-L145) | 启动 Adapter 子进程，通过 stdin/stdout 通信，stderr 单独输出到日志 |
| config.transport="tcp" + port_arg | `tcp` 自启动 | [Client::tcp_process](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L165-L203) | 启动 Adapter 子进程并传入端口参数（如 `--port 12345`），等待 500ms 后 TCP 连接 |
| `:debug-remote` | `tcp` 远程连接 | [Client::tcp](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L105-L112) | 直接连接远程 Adapter 的 TCP 地址，不启动子进程 |

所有方式最终都汇聚到 [Client::streams](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L72-L103)，它：
- 创建 `Transport` 层处理底层的消息编解码
- 启动 `recv` 协程转发 Adapter 消息到上层
- 返回 `(Client, UnboundedReceiver<(DebugAdapterId, Payload)>)` 对

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
| `caps` | `Option<DebuggerCapabilities>` | Adapter 能力（initialize 后填充） |
| `stack_frames` | `HashMap<ThreadId, Vec<StackFrame>>` | 每个线程的栈帧缓存 |
| `thread_states` | `ThreadStates` (= `HashMap<ThreadId, String>`) | 线程运行状态 |
| `thread_id` | `Option<ThreadId>` | 当前活跃线程 |
| `active_frame` | `Option<usize>` | 当前线程的活跃栈帧索引 |
| `progress` | `ProgressMap` | 进度报告状态 |
| `connection_type` | `Option<ConnectionType>` | Launch 或 Attach |
| `starting_request_args` | `Option<Value>` | 启动参数（用于 restart） |
| `quirks` | `DebuggerQuirks` | Adapter 的特殊行为适配 |

### 5.4 反向请求处理

Adapter 可以向 Client 发送反向请求：

- **RunInTerminal**：Adapter 请求在终端运行程序，Helix 使用配置的外部终端启动进程，返回 process_id
- **StartDebugging**：Adapter 请求启动子调试会话，Helix 在同一 TCP socket 上创建新的 Client，根据 request 类型执行 launch 或 attach

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
