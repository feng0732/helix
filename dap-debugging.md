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
用户按键 → helix-term 命令 → helix-view/editor 状态变更 → helix-dap Client 发请求
                                                                        ↓
                                                            DAP Adapter 进程 (stdio/TCP)
                                                                        ↓
UI 刷新 ← helix-view 事件处理 ← helix-dap Transport 接收 ← Adapter 响应/事件
```

---

## 一、会话启动（Session Startup）

### 1.1 入口：用户触发

会话启动有两个入口：

- **键盘快捷键** `G l`（Space 下 `G` 进入 Debug 菜单，`l` 启动）：映射到 [dap_launch](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L234-L287)
- **命令行** `:debug` 或 `:debug-remote`：定义在 [typed.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/typed.rs)，最终都调用 [dap_start_impl](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L116-L187)

### 1.2 启动流程详解

以 `dap_launch` 为例，完整链路如下：

```
dap_launch()
  ├─ 1. 检查是否已有活跃调试器（同一时间只允许一个）
  ├─ 2. 从当前文档的 language_config 中获取 DebugAdapterConfig
  ├─ 3. 展示 Picker 让用户选择 DebugTemplate
  │     └─ 若 template 有 completion 字段 → 弹出 Prompt 逐项收集参数
  │         └─ debug_parameter_prompt() 循环收集，最后调用 dap_start_impl()
  └─ 4. 若无 completion → 直接调用 dap_start_impl()
```

### 1.3 dap_start_impl 核心逻辑

[dap_start_impl](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L116-L187) 做了以下关键步骤：

1. **启动 Client**：调用 `editor.debug_adapters.start_client(socket, config)`
   - 内部通过 [Registry::start_client](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/registry.rs#L33-L59) 创建 Client
   - 根据 transport 类型选择 `stdio` 或 `tcp` 方式连接 Adapter 进程
   - 立即发送 `initialize` 请求，获取 Adapter 的 `DebuggerCapabilities`
   - 将 Client 的 incoming receiver 注册到 Registry 的 `SelectAll` 流中

2. **组装 launch/attach 参数**：将 DebugTemplate 的 args 与用户输入的参数合并，插入 `cwd`

3. **发送 launch 或 attach 请求**：根据 template.request 判断是 `"launch"` 还是 `"attach"`，通过 `dap_callback` 异步执行

### 1.4 Client 创建与连接方式

[Client::process](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L55-L70) 根据 transport 类型分发：

| 方式 | 方法 | 说明 |
|------|------|------|
| `stdio` | [Client::stdio](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L114-L145) | 启动 Adapter 子进程，通过 stdin/stdout 通信 |
| `tcp` + port_arg | [Client::tcp_process](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L165-L203) | 启动 Adapter 子进程并传入端口参数，再 TCP 连接 |
| `tcp` (远程) | [Client::tcp](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L105-L112) | 直接连接远程 Adapter 地址（`:debug-remote` 命令） |

所有方式最终都汇聚到 [Client::streams](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L72-L103)，它：
- 创建 Transport 层处理底层的消息编解码
- 启动 `recv` 协程转发 Adapter 消息到上层
- 返回 `(Client, UnboundedReceiver<(DebugAdapterId, Payload)>)` 对

### 1.5 初始化握手协议

DAP 规范要求在 launch/attach 之前先完成 initialize。在 [Registry::start_client](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/registry.rs#L53-L54) 中：

```rust
block_on(client.initialize(config.name.clone()))?;
client.quirks = config.quirks.clone();
```

[Client::initialize](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-dap/src/client.rs#L371-L392) 发送 `InitializeArguments`，声明 Helix 的客户端能力（如 `supports_run_in_terminal_request`、`supports_progress_reporting` 等），Adapter 返回 `DebuggerCapabilities` 存入 `client.caps`。

### 1.6 会话启动后的 "Initialized" 事件流

Adapter 在准备好接收配置时会发送 `Initialized` 事件。[handle_debugger_message](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-view/src/handlers/dap.rs#L375-L396) 处理此事件时：

1. 遍历 `editor.breakpoints` 中已有的断点，逐一调用 `breakpoints_changed()` 同步给 Adapter
2. 调用 `debugger.configuration_done()` 通知 Adapter 配置完成
3. 调用 `debug_adapters.set_active_client(id)` 标记当前活跃调试器

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
    Event(Event),      // Adapter → Client 的单向通知（如 stopped、continued）
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

1. **键盘命令** `G b`：[dap_toggle_breakpoint](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/commands/dap.rs#L390-L402) 在当前光标行切换断点
2. **鼠标点击 gutter**：[ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-term/src/ui/editor.rs#L1273) 中，左键点击 gutter 区域时调用 `dap_toggle_breakpoint_impl`

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

### 4.5 调试操作与界面联动

| 操作 | 快捷键 | 函数 | 界面效果 |
|------|--------|------|----------|
| 启动调试 | `G l` | dap_launch | 弹出模板选择 Picker |
| 切换断点 | `G b` | dap_toggle_breakpoint | gutter 显示/隐藏断点标记 |
| 继续执行 | `G c` | dap_continue | 清除暂停指示器 |
| 暂停 | `G h` | dap_pause | 弹出线程 Picker |
| 单步进入 | `G i` | dap_step_in | 清除暂停指示器，等待 Stopped 事件 |
| 单步跳出 | `G o` | dap_step_out | 同上 |
| 单步跳过 | `G n` | dap_next | 同上 |
| 查看变量 | `G v` | dap_variables | 弹出变量 Popup |
| 终止调试 | `G t` | dap_terminate | 清除断点验证状态 |
| 编辑条件 | `G C-c` | dap_edit_condition | 弹出条件输入 Prompt |
| 编辑日志 | `G C-l` | dap_edit_log | 弹出日志消息 Prompt |
| 切换线程 | `G s t` | dap_switch_thread | 弹出线程 Picker |
| 切换栈帧 | `G s f` | dap_switch_stack_frame | 弹出栈帧 Picker |
| 启用异常 | `G e` | dap_enable_exceptions | 设置异常断点过滤器 |
| 禁用异常 | `G E` | dap_disable_exceptions | 清除异常断点过滤器 |
| 重启调试 | `G r` | dap_restart | 重启当前调试会话 |

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
1. 用户按 G l → dap_launch
   ↓
2. 选择 DebugTemplate → dap_start_impl
   ↓
3. Registry::start_client → 创建 Client → Transport 启动 → initialize 请求
   ↓
4. Client::launch/attach → 异步等待响应
   ↓
5. Adapter 发送 Initialized 事件
   ↓
6. handle_debugger_message: 同步断点 → configurationDone → set_active_client
   ↓
7. 程序运行中...（用户可设置断点、步进等）
   ↓
8. Adapter 发送 Stopped 事件
   ↓
9. handle_debugger_message: 获取线程/栈帧 → 跳转位置 → 显示暂停指示器
   ↓
10. 用户按 G c → dap_continue → resume_application → 清除暂停指示器
   ↓
... 循环 7-10 ...
   ↓
11. Adapter 发送 Terminated 事件 或 用户按 G t → dap_terminate
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
┌─────────────────────────────────────────────────────────┐
│                    helix-term                            │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │ commands/dap │  │ commands/typed│  │  ui/editor.rs  │ │
│  │ (DAP 命令)   │  │ (:debug 命令) │  │ (鼠标设断点)    │ │
│  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘ │
│         │                 │                   │          │
│  ┌──────┴─────────────────┴───────────────────┴────────┐ │
│  │              keymap/default.rs (G 快捷键)            │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────┐
│                     helix-view                           │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │ handlers/dap.rs  │  │       editor.rs               │ │
│  │ (事件处理/断点同步) │  │ (breakpoints/Registry/事件循环)│ │
│  └────────┬─────────┘  └──────────────┬───────────────┘ │
│           │                           │                  │
│  ┌────────┴───────────────────────────┴───────────────┐ │
│  │              gutter.rs (断点/暂停指示器渲染)          │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────┐
│                      helix-dap                           │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────────┐ │
│  │  client.rs   │  │transport.rs│  │  registry.rs     │ │
│  │ (DAP 客户端)  │  │ (消息编解码) │  │ (多客户端管理)    │ │
│  └──────┬───────┘  └─────┬─────┘  └──────────────────┘ │
└─────────┼────────────────┼─────────────────────────────┘
          │                │
┌─────────┴────────────────┴─────────────────────────────┐
│                   helix-dap-types                        │
│  ┌─────────────────────────────────────────────────────┐│
│  │  lib.rs (Request/Event/Response 类型、Breakpoint 等) ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

---

## 七、配置来源

调试器配置来自 `languages.toml` 中的 `[language.debugger]` 段，结构对应 [DebugAdapterConfig](file:///d:/fz/0601/solo-dogfeeding/code/277-helix/helix-core/src/syntax/config.rs#L486-L497)：

```toml
[[language]]
name = "rust"
debugger = { name = "lldb-vscode", transport = "stdio", command = "lldb-vscode",
             templates = [
               { name = "binary", request = "launch", completion = [{ name = "binary", completion = "filename" }], args = { program = "{0}" } }
             ]}
```

- `name`: Adapter 标识符（传递给 initialize 请求的 adapter_id）
- `transport`: `"stdio"` 或 `"tcp"`
- `command`: Adapter 可执行文件
- `args`: Adapter 启动参数
- `port_arg`: TCP 模式的端口参数格式（如 `"--port {}"`）
- `templates`: 调试模板列表，每个模板定义名称、请求类型、参数补全和启动参数
- `quirks`: Adapter 行为适配（如 `absolute_paths`）
