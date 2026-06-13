# Helix 模态键映射分派机制详解

本文档详细解析 Helix 编辑器中从按键输入到编辑动作落地的完整链路，重点阐明**按键匹配、配置覆盖、命令执行**三者之间的关联和实现方式。

所有源码路径均为相对于仓库根目录的相对路径。

---

## 1. 整体架构概览

按键输入到编辑动作的完整处理链路如下：

```
终端输入
  ↓
[helix-term/src/application.rs] handle_terminal_events()
  ↓  转换为内部 Event 类型
[helix-term/src/compositor.rs] Compositor::handle_event()
  ↓  从顶层向底层逐层冒泡
[helix-term/src/ui/editor.rs] EditorView::handle_event()
  ├─ Insert 模式 → insert_mode()
  └─ Normal/Select 模式 → command_mode()
       ↓
[helix-term/src/ui/editor.rs] handle_keymap_event()
       ↓
[helix-term/src/keymap.rs] Keymaps::get()  →  Trie 树查找
       ↓
KeymapResult::{Matched, Pending, MatchedSequence, NotFound, Cancelled}
       ↓
MappableCommand::execute()  →  实际编辑动作
```

---

## 2. 核心数据结构

### 2.1 KeyEvent —— 按键事件

定义位置：[helix-view/src/input.rs](helix-view/src/input.rs#L65-L69)

```rust
pub struct KeyEvent {
    pub code: KeyCode,
    pub modifiers: KeyModifiers,
}
```

- `KeyCode`：物理按键（字符、功能键、方向键等），见 [helix-view/src/keyboard.rs](helix-view/src/keyboard.rs#L362-L419)
- `KeyModifiers`：修饰键组合（SHIFT / CONTROL / ALT / SUPER），位标志实现

**字符串解析与规范化**：通过 `FromStr` trait 将 `"C-w"`、`"S-A-F12"` 等字符串解析为 `KeyEvent`。解析时会对字符键进行规范化：如 `"C-S-r"` 与 `"C-R"` 等价（小写+SHIFT 转为大写），见 [helix-view/src/input.rs](helix-view/src/input.rs#L437-L445)。

### 2.2 Mode —— 编辑模式

定义位置：[helix-view/src/document.rs](helix-view/src/document.rs#L65-L69)

```rust
pub enum Mode {
    Normal = 0,
    Select = 1,
    Insert = 2,
}
```

每个模式对应**独立**的键映射树，是连接配置覆盖、按键匹配、命令执行的关键维度。

### 2.3 KeyTrie —— 键映射前缀树

定义位置：[helix-term/src/keymap.rs](helix-term/src/keymap.rs#L109-L114)

```rust
pub enum KeyTrie {
    MappableCommand(MappableCommand),  // 叶子：单个命令
    Sequence(Vec<MappableCommand>),    // 叶子：命令序列
    Node(KeyTrieNode),                 // 中间节点：子键映射
}
```

`KeyTrieNode` 内部使用 `IndexMap<KeyEvent, KeyTrie>` 存储子键，构成一棵多叉树。例如默认 Normal 模式下：

```
root (Node)
  ├─ 'i' → MappableCommand(insert_mode)
  ├─ 'g' → Node("Goto")
  │     ├─ 'g' → MappableCommand(goto_file_start)
  │     ├─ 'e' → MappableCommand(goto_last_line)
  │     └─ ...
  ├─ ' ' (space) → Node("Space")
  │     └─ ...
  └─ ...
```

### 2.4 Keymaps —— 状态保持的映射查找器

定义位置：[helix-term/src/keymap.rs](helix-term/src/keymap.rs#L264-L271)

```rust
pub struct Keymaps {
    pub map: Box<dyn DynAccess<HashMap<Mode, KeyTrie>>>,  // 各模式的根 Trie
    state: Vec<KeyEvent>,         // 当前已输入但未决的按键序列
    pub sticky: Option<KeyTrieNode>,  // 粘滞节点（激活后可复用）
}
```

- `map` 通过 `ArcSwap` 实现热更新配置 —— **配置覆盖**的结果存储于此
- `state` 保存多键组合的中间状态 —— **按键匹配**的运行时上下文
- `sticky` 支持"粘滞模式"，改变后续按键匹配的起点

### 2.5 KeymapResult —— 查找结果

定义位置：[helix-term/src/keymap.rs](helix-term/src/keymap.rs#L248-L259)

```rust
pub enum KeymapResult {
    Pending(KeyTrieNode),        // 需要更多按键，附带候选项
    Matched(MappableCommand),    // 匹配到单个命令
    MatchedSequence(Vec<MappableCommand>),  // 匹配到命令序列
    NotFound,                    // 根级别未找到
    Cancelled(Vec<KeyEvent>),    // 中途无效按键，附带已输入的序列
}
```

这是**按键匹配**输出给**命令执行**的核心数据结构，定义了五种标准处理分支。

### 2.6 MappableCommand —— 可映射命令

定义位置：[helix-term/src/commands.rs](helix-term/src/commands.rs#L214-L230)

```rust
pub enum MappableCommand {
    Typable { name: String, args: String, doc: String },
    Static { name: &'static str, fun: fn(cx: &mut Context), doc: &'static str },
    Macro { name: String, keys: Vec<KeyEvent> },  // 注意：无 doc 字段
}
```

`execute()` 方法是**命令执行**阶段的最终入口，三种变体有完全不同的落地方式（详见第 5 章）。

Static 变体通过 `static_commands!` 宏在编译期生成 `const` 常量，例如 `MappableCommand::move_char_left`，其函数指针指向同名的 Rust 函数。

---

## 3. 配置覆盖：静态构建 Trie 树

**配置覆盖发生在编辑器启动/配置刷新时**，负责将多层配置（默认 → 全局 → 工作区）合并为最终的 `HashMap<Mode, KeyTrie>`。

### 3.1 默认映射的构建

[helix-term/src/keymap/default.rs](helix-term/src/keymap/default.rs) 的 `default()` 函数在**运行时**被调用（编辑器启动或配置刷新时），通过 `keymap!` 宏展开为运行时代码构建 Trie 树：

```rust
keymap!({ "Normal mode"
    "i" => insert_mode,
    "g" => { "Goto"
        "g" => goto_file_start,
        "e" => goto_last_line,
    },
    "j" | "down" => move_line_down,  // 多键同义绑定
})
```

`keymap!` 宏的展开代码（见 [helix-term/src/keymap/macros.rs](helix-term/src/keymap/macros.rs#L96-L117)）在运行时执行以下操作：

1. 创建 `IndexMap::with_capacity(_cap)` 存储子键
2. 对每个绑定执行 `"key".parse::<KeyEvent>().unwrap()` 解析按键字符串
3. 将 `(KeyEvent, KeyTrie)` 对插入 `IndexMap`
4. 用 `assert!(_duplicate.is_none())` **运行时**检查重复键
5. 包装为 `KeyTrie::Node(KeyTrieNode { ... })`

注意：虽然键字符串解析和 Map 构建在运行时完成，但 `MappableCommand::insert_mode`、`MappableCommand::goto_file_start` 等 Static 命令本身由 `static_commands!` 宏在**编译期**生成为 `const` 常量。

### 3.2 合并算法：KeyTrieNode::merge()

[helix-term/src/keymap.rs](helix-term/src/keymap.rs#L45-L55)

```rust
pub fn merge(&mut self, mut other: Self) {
    for (key, trie) in std::mem::take(&mut other.map) {
        if let Some(KeyTrie::Node(node)) = self.map.get_mut(&key) {
            if let KeyTrie::Node(other_node) = trie {
                node.merge(other_node);  // 双方都是 Node → 递归合并
                continue;
            }
        }
        self.map.insert(key, trie);  // 否则直接覆盖
    }
}
```

**覆盖规则**（只有 Node-Node 会合并，其余一律覆盖）：

| 原节点（默认） | 用户配置 | 合并结果 |
|---|---|---|
| 叶子（命令） | 叶子（命令） | 用户覆盖默认 |
| 叶子（命令） | Node（子菜单） | 用户 Node 覆盖默认叶子 |
| Node（子菜单） | 叶子（命令） | 用户叶子覆盖默认 Node |
| Node（子菜单） | Node（子菜单） | **递归合并**两者子键 |

### 3.3 配置加载流程

[helix-term/src/config.rs](helix-term/src/config.rs#L59-L118)

```
默认 keymap::default()
     ↓ merge_keys(全局 ~/.config/helix/config.toml)
     ↓ merge_keys(工作区 .helix/config.toml，需受信任)
最终 HashMap<Mode, KeyTrie>
```

关键点：
- 后加载的配置优先级更高
- 用户可以向已有的子菜单（如 `g` 跳转菜单）**追加**新绑定，而非必须完全替换
- 合并结果存入 `Config.keys`，通过 `ArcSwap` 包装后注入 `Keymaps.map`

---

## 4. 按键匹配：动态消费 Trie 树

**按键匹配发生在每个按键事件到达时**，使用配置覆盖阶段构建的 Trie 树，将按键序列转换为 `KeymapResult`。

核心算法：[Keymaps::get()](helix-term/src/keymap.rs#L307-L356)

### 4.1 完整算法流程

```rust
pub fn get(&mut self, mode: Mode, key: KeyEvent) -> KeymapResult {
    let keymaps = &*self.map();       // ← 读取配置覆盖的结果
    let keymap = &keymaps[&mode];     // ← 按当前模式选择 Trie 树

    // 1) Esc 键特殊处理
    if key!(Esc) == key {
        if !self.state.is_empty() {
            return KeymapResult::Cancelled(self.state.drain(..).collect());
        }
        self.sticky = None;
    }

    // 2) 确定查找起点（受 sticky 影响）
    let first = self.state.first().unwrap_or(&key);
    let trie_node = match self.sticky {
        Some(ref trie) => Cow::Owned(KeyTrie::Node(trie.clone())),
        None => Cow::Borrowed(keymap),
    };

    // 3) 首键快速匹配
    let trie = match trie_node.search(&[*first]) {
        Some(KeyTrie::MappableCommand(ref cmd)) => return KeymapResult::Matched(cmd.clone()),
        Some(KeyTrie::Sequence(ref cmds)) => return KeymapResult::MatchedSequence(cmds.clone()),
        None => return KeymapResult::NotFound,
        Some(t) => t,
    };

    // 4) 多键组合逐级下探
    self.state.push(key);
    match trie.search(&self.state[1..]) {
        Some(KeyTrie::Node(map)) => {
            if map.is_sticky {
                self.state.clear();
                self.sticky = Some(map.clone());  // ← 改变后续匹配起点
            }
            KeymapResult::Pending(map.clone())
        }
        Some(KeyTrie::MappableCommand(cmd)) => {
            self.state.clear();
            KeymapResult::Matched(cmd.clone())
        }
        Some(KeyTrie::Sequence(cmds)) => {
            self.state.clear();
            KeymapResult::MatchedSequence(cmds.clone())
        }
        None => KeymapResult::Cancelled(self.state.drain(..).collect()),
    }
}
```

### 4.2 关键设计要点

1. **`mode` 参数**：直接决定使用哪棵 Trie 树，建立了"模式 → 键映射"的关联
2. **`state` 字段**：累积多键组合，实现 `g g`、`C-w v` 等多键命令
3. **`sticky` 字段**：激活后改变后续查找起点，影响后续按键匹配
4. **`search()` 方法**：沿着 `state` 累积的按键路径在 Trie 中逐级查找

---

## 5. 命令执行：KeymapResult 落地

**命令执行发生在按键匹配之后**，由 `EditorView` 消费 `KeymapResult`，产生实际编辑效果。

### 5.1 事件分派入口

[EditorView::handle_event()](helix-term/src/ui/editor.rs#L1436-L1601) 是所有按键事件的处理入口：

```rust
Event::Key(mut key) => {
    canonicalize_key(&mut key);

    // on_next_key 优先截获（如 f 命令等待目标字符）
    if !self.on_next_key(OnKeyCallbackKind::PseudoPending, &mut cx, key) {
        match mode {
            Mode::Insert => self.insert_mode(&mut cx, key),
            mode => self.command_mode(mode, &mut cx, key),
        }
    }
}
```

### 5.2 Insert 模式

[insert_mode()](helix-term/src/ui/editor.rs#L984-L1011)

```rust
fn insert_mode(&mut self, cx: &mut commands::Context, event: KeyEvent) {
    if let Some(keyresult) = self.handle_keymap_event(Mode::Insert, cx, event) {
        match keyresult {
            KeymapResult::NotFound => {
                // 未绑定的字符直接插入文档
                if let Some(ch) = event.char() {
                    commands::insert::insert_char(cx, ch);
                }
            }
            KeymapResult::Cancelled(pending) => {
                // 多键组合取消时，已输入字符逐个落盘
                for ev in pending { ... }
            }
            _ => unreachable!(),
        }
    }
}
```

### 5.3 Normal / Select 模式

[command_mode()](helix-term/src/ui/editor.rs#L1013-L1098) 有三类特殊处理：

**1) 计数前缀（Count）**
```rust
// 数字键只有在未被当前模式映射占用时才作为计数前缀
(key!(i @ '1'..='9'), None) if !self.keymaps.contains_key(mode, event) => {
    cxt.editor.count = NonZeroUsize::new(i as usize);
}
```

这里调用 `self.keymaps.contains_key()` 查询 Trie 树，体现了**按键匹配**结果会**影响控制流**。

**2) 重复操作（`.`）**
```rust
(key!('.'), _) if self.keymaps.pending().is_empty() => {
    for _ in 0..cxt.editor.count.map_or(1, NonZeroUsize::into) {
        self.last_insert.0.execute(cx);  // 重放进入 Insert 的命令
        for key in self.last_insert.1.clone() {
            self.insert_mode(cx, key);   // 重放 Insert 期间的输入
        }
    }
}
```

**3) 常规命令查找**
```rust
_ => {
    cxt.count = cxt.editor.count;
    cxt.register = cxt.editor.selected_register.take();
    let res = self.handle_keymap_event(mode, cxt, event);
    if matches!(&res, Some(KeymapResult::NotFound)) {
        self.on_next_key(OnKeyCallbackKind::Fallback, cxt, event);
    }
}
```

### 5.4 handle_keymap_event：核心执行封装

[handle_keymap_event()](helix-term/src/ui/editor.rs#L933-L982)

```rust
fn handle_keymap_event(
    &mut self, mode: Mode, cxt: &mut commands::Context, event: KeyEvent
) -> Option<KeymapResult> {
    let key_result = self.keymaps.get(mode, event);  // ← 调用按键匹配

    let mut execute_command = |command: &commands::MappableCommand| {
        command.execute(cxt);  // ← 命令执行落地
        helix_event::dispatch(PostCommand { command, cx: cxt });

        // 模式切换会改变后续按键匹配使用的 Trie 树
        let current_mode = cxt.editor.mode();
        if current_mode != last_mode {
            helix_event::dispatch(OnModeSwitch {
                old_mode: last_mode, new_mode: current_mode, cx: cxt
            });
            if current_mode == Mode::Insert {
                self.last_insert.0 = command.clone();  // 记录用于 . 重复
                self.last_insert.1.clear();
            }
        }
        last_mode = current_mode;
    };

    match &key_result {
        KeymapResult::Matched(command) => execute_command(command),
        KeymapResult::Pending(node) => cxt.editor.autoinfo = Some(node.infobox()),
        KeymapResult::MatchedSequence(commands) => {
            for command in commands { execute_command(command); }
        }
        KeymapResult::NotFound | KeymapResult::Cancelled(_) => return Some(key_result),
    }
    None
}
```

### 5.5 三种命令的落地方式详解

`MappableCommand::execute()` 定义位置：[helix-term/src/commands.rs](helix-term/src/commands.rs#L248-L286)

三种变体有完全不同的落地路径，下表先做对比，再分别展开：

| | Static | Typable | Macro |
|---|---|---|---|
| **分发方式** | 直接调用函数指针 | 按名称查表再分发 | 封装按键回调注入事件循环 |
| **执行时机** | 同步，当前函数栈 | 同步，当前函数栈 | 延迟，下一轮事件循环 |
| **execute() 接收** | `commands::Context`（含 count/register/callback） | `commands::Context`（含 count/register/callback） | `commands::Context`（含 count/register/callback） |
| **回调接收** | 无回调 | 无回调 | `compositor::Context`（仅 editor/jobs） |
| **命令来源** | `static_commands!` 宏生成的 `const` 常量 | 用户 TOML 配置中的冒号命令 | `@` 寄存器录制的按键序列 |
| **典型示例** | `move_char_left`、`insert_mode` | `:write`、`:buffer-close` | `@miw` |

---

#### Static —— 直接函数调用

```rust
Self::Static { fun, .. } => (fun)(cx),
```

- **执行方式**：直接调用函数指针 `fn(cx: &mut Context)`
- **典型场景**：绝大多数编辑命令（移动、删除、插入模式切换等）
- **命令来源**：由 `static_commands!` 宏在编译期生成的 `const` 常量
- **运行时开销**：分发本身仅一次函数指针调用，但被调函数内部的逻辑开销因命令而异（如 `move_char_left` 只移动光标，而 `global_search` 会启动异步搜索）

#### Typable —— 命令行命令分发

```rust
Self::Typable { name, args, doc: _ } => {
    if let Some(command) = typed::TYPABLE_COMMAND_MAP.get(name.as_str()) {
        let mut cx = compositor::Context { editor: cx.editor, jobs: cx.jobs, scroll: None };
        if let Err(e) = typed::execute_command(&mut cx, command, args, PromptEvent::Validate) {
            cx.editor.set_error(format!("{}", e));
        }
    } else {
        cx.editor.set_error(format!("no such command: '{name}'"));
    }
}
```

- **执行方式**：从 `TYPABLE_COMMAND_MAP` 按名称查找，再调用 `typed::execute_command`
- **典型场景**：用户在 TOML 配置中通过 `:write`、`:buffer-close` 等冒号命令绑定的键
- **命令来源**：用户 TOML 配置，或 `keymap!` 宏中的字符串命令名（如 `"buffer-close"`）
- **Context 转换**：`execute()` 接收的是 `commands::Context`，但在内部创建了一个新的 `compositor::Context`（仅保留 `editor` 和 `jobs`，丢失 `count`、`register`、`callback`）
- **容错**：若 `name` 在 `TYPABLE_COMMAND_MAP` 中查不到，设置错误提示而非 panic

#### Macro —— 按键序列重放

```rust
Self::Macro { keys, .. } => {
    // execute() 接收 commands::Context
    if cx.editor.macro_replaying.contains(&'@') {
        cx.editor.set_error(
            "Cannot execute macro because the [@] register is already playing a macro",
        );
        return;
    }
    cx.editor.macro_replaying.push('@');
    let keys = keys.clone();
    // 回调签名使用 compositor::Context
    cx.callback.push(Box::new(move |compositor, cx| {
        for key in keys.into_iter() {
            compositor.handle_event(&compositor::Event::Key(key), cx);
        }
        cx.editor.macro_replaying.pop();
    }));
}
```

**两种 Context 的边界：**

1. **execute() 阶段（commands::Context）**：
   - `MappableCommand::execute(&self, cx: &mut commands::Context)` 接收完整的 `commands::Context`，包含 `count`、`register`、`callback`、`on_next_key_callback`
   - 这一阶段检查 `macro_replaying` 防止递归，标记 `'@'` 进入重放状态
   - 将按键重放逻辑封装为 `compositor::Callback`，push 到 `cx.callback`

2. **回调执行阶段（compositor::Context）**：
   - 回调签名是 `Box<dyn FnOnce(&mut Compositor, &mut compositor::Context)>`，仅能访问 `editor`、`jobs`、`scroll`
   - `count`、`register` 等 commands::Context 特有的字段在此边界**丢失**

**回调传递全流程：**

```
EditorView::handle_event(compositor::Context)
  ↓ 创建
commands::Context { callback: Vec::new(), ... }
  ↓ 调用
MappableCommand::execute(cx) → push callback 到 cx.callback
  ↓ 收集（见 ui/editor.rs#L1547-L1575）
let callbacks = take(&mut cx.callback);
EventResult::Consumed(Some(Box::new(|compositor, cx| {
    for cb in callbacks { cb(compositor, cx); }
})))
  ↓ 返回给 Compositor
Compositor::handle_event() → 执行所有 callback（见 compositor.rs#L153-L179）
  ↓ 回调中调用
compositor.handle_event(Event::Key(key), cx)  → 完整事件分派
```

**重放按键的执行边界：**
- 重放的每个按键都走完整的事件分派链路：Compositor → EditorView → Keymaps::get → MappableCommand::execute
- 每个被重放的按键都会触发 `EditorView::handle_event` 创建**新的** `commands::Context`，count/register 等上下文会重置
- 递归保护通过 `editor.macro_replaying` 标志在整个重放过程中生效

- **执行方式总结**：不直接修改文档，而是将按键序列封装为 Compositor 回调，**将按键重新注入事件循环**
- **典型场景**：`@` 寄存器回放录制的宏
- **关键机制**：
  1. 递归保护：检查 `macro_replaying` 防止 `@` 寄存器递归调用
  2. 延迟执行：通过 `cx.callback.push` 将回调交给 Compositor 处理
  3. 事件重放：逐个按键调用 `compositor.handle_event(Event::Key(key))`，经过完整的事件分派链路
- **重要区别**：Static/Typable 命令同步执行在当前函数栈上，Macro 命令则是**延迟**在下一轮事件循环中重放按键

---

## 6. 三者关联剖析：配置覆盖 → 按键匹配 → 命令执行

三者是**层层递进、相互影响**的闭环关系：

```
┌───────────────────────────────────────────────────────────────────┐
│                    配置覆盖（静态阶段）                            │
│  [config.rs] Config::load()                                        │
│      default() → merge(global) → merge(workspace)                  │
│      生成 HashMap<Mode, KeyTrie>                                   │
│          ↓ 注入                                                    │
│  [keymap.rs] Keymaps.map (ArcSwap)  ←──────────────────────────┐   │
│          ↓ 被消费                                               │   │
└───────────────────────────────────────────────────────────────────┘   │
            ↓                                                         │
┌───────────────────────────────────────────────────────────────────┐ │
│                   按键匹配（动态阶段）                             │ │
│  [keymap.rs] Keymaps::get(mode, key)                               │ │
│      1. 根据 mode 选择 Trie 树（取决于上次命令执行的结果）         │ │
│      2. 沿 state 路径查找                                         │ │
│      3. 返回 KeymapResult                                         │ │
│          ↓ 被消费                                                 │ │
└───────────────────────────────────────────────────────────────────┘ │
            ↓                                                         │
┌───────────────────────────────────────────────────────────────────┐ │
│                   命令执行（落地阶段）                             │ │
│  [ui/editor.rs] handle_keymap_event()                             │ │
│      Matched → command.execute() → 模式可能改变                   │ │
│      Pending → 显示 infobox                                       │ │
│      NotFound → 插入字符 / on_next_key 回退                       │ │
│          ↓ 副作用                                                 │ │
│      模式切换 → 改变后续 get() 调用的 mode 参数 ───────────────────┘ │
│      sticky 设置 → 改变后续 get() 调用的查找起点                    │
└───────────────────────────────────────────────────────────────────┘
```

### 6.1 配置覆盖 → 按键匹配：数据流向

- **输入**：配置覆盖的产物 `HashMap<Mode, KeyTrie>` 存储在 `Keymaps.map` 中
- **消费**：`Keymaps::get(mode, key)` 每次调用都从 `map` 中读取当前模式的 Trie 树
- **热更新**：通过 `ArcSwap` 支持配置刷新后按键匹配行为即时变更

### 6.2 按键匹配 → 命令执行：控制流向

- **输入**：`KeymapResult` 定义了 5 种标准分支
- **消费**：
  - `Matched` / `MatchedSequence` → 执行命令
  - `Pending` → 显示帮助弹窗，等待下一键
  - `NotFound` → Insert 模式插入字符，Normal 模式尝试 on_next_key 回退
  - `Cancelled` → 多键组合取消，Insert 模式下将已输入字符落盘

### 6.3 命令执行 → 按键匹配：反馈闭环

命令执行产生的副作用会**改变后续按键匹配的行为**：

1. **模式切换**：`insert_mode` 等命令执行后，`editor.mode()` 返回新值，下次 `Keymaps::get()` 将使用不同模式的 Trie 树
2. **Sticky 节点激活**：命中标记为 sticky 的子菜单后，`Keymaps.sticky` 被设置，后续查找起点改变
3. **on_next_key 回调**：某些命令（如 `f` 查找）设置回调后，下次按键将绕过正常的 Trie 查找
4. **计数与寄存器**：数字键和 `"` 键的处理依赖 `keymaps.contains_key()` 查询结果

### 6.4 代码中的具体关联点

| 关联类型 | 代码位置 | 关联方式 |
|---|---|---|
| 配置→匹配 | [keymap.rs#L309-L310](helix-term/src/keymap.rs#L309-L310) | `get()` 读取 `self.map()` 获取 Trie |
| 匹配→执行 | [ui/editor.rs#L941](helix-term/src/ui/editor.rs#L941) | `handle_keymap_event()` 调用 `self.keymaps.get()` |
| 匹配→执行（分支） | [commands.rs#L248-L286](helix-term/src/commands.rs#L248-L286) | `execute()` 中 Static/Typable/Macro 三种落地分支 |
| 执行→匹配（模式） | [ui/editor.rs#L948-L963](helix-term/src/ui/editor.rs#L948-L963) | 模式切换后下次 `get()` 用新 mode |
| 执行→匹配（sticky） | [keymap.rs#L340-L343](helix-term/src/keymap.rs#L340-L343) | sticky 节点改变查找起点 |
| 执行→匹配（count） | [ui/editor.rs#L1025](helix-term/src/ui/editor.rs#L1025) | `contains_key()` 查询影响计数逻辑 |
| 执行→匹配（macro） | [commands.rs#L268-L283](helix-term/src/commands.rs#L268-L283) | Macro 将按键重新注入 compositor 事件循环 |

---

## 7. 辅助机制

### 7.1 on_next_key 回调

某些命令（如 `f` 查找字符、`m` 环绕操作）需要等待**下一个按键**作为参数。它们通过设置 `on_next_key_callback` 截获后续按键，绕过正常的键映射查找。

```rust
// 命令中设置回调
cx.on_next_key(|cx, key| { /* 处理下一个按键 */ });

// 事件入口优先检查
if !self.on_next_key(OnKeyCallbackKind::PseudoPending, &mut cx, key) {
    // 只有 on_next_key 不消费事件时，才走正常映射逻辑
}
```

`pseudo_pending` 字段用于在状态栏显示这些临时等待的按键。

### 7.2 寄存器（Register）

`"` 前缀键用于选择寄存器，选中的寄存器暂存于 `editor.selected_register`，在命令执行时取出注入 `Context::register`。

### 7.3 粘滞节点（Sticky Node）

某些子菜单（如 Window 模式）可标记 `sticky=true`，进入后无需重复按前缀键即可连续执行子命令。例如按 `C-w` 后可连续按 `v`/`s`/`q` 进行窗口操作。

---

## 8. 完整示例：`gg` 执行流程

用户在 Normal 模式按下 `g` 再按 `g`（跳转文件开头），贯穿三个阶段：

### 阶段 1：配置覆盖（已在启动时完成）
`default()` 构建的 Trie 树中，`g` 键对应 `Node("Goto")`，其下 `g` 键对应 `goto_file_start` 命令。

### 阶段 2：按第一个 `g`（按键匹配 + 命令执行）
1. `Keymaps::get(Normal, 'g')` → 命中 `KeyTrie::Node("Goto")`
2. 返回 `KeymapResult::Pending(node)`
3. `state` 变为 `['g']`
4. `handle_keymap_event` 显示 Goto 子菜单的帮助弹窗

### 阶段 3：按第二个 `g`（按键匹配 + 命令执行）
1. `Keymaps::get(Normal, 'g')` 首键匹配到 Node
2. `state.push('g')` → `['g', 'g']`
3. `trie.search(&state[1..])` → 命中 `MappableCommand::Static { name: "goto_file_start", fun: goto_file_start, doc: "..." }`
4. 返回 `KeymapResult::Matched(goto_file_start)`
5. `state` 清空
6. `execute_command` 闭包调用 `command.execute(cx)`：
   - 匹配到 `MappableCommand::Static { fun, .. }` 分支
   - 直接调用函数指针 `goto_file_start(cx)`
   - 内部构建 `Transaction`（将光标移动到文档开头）应用到 `Document`
7. 派发 `PostCommand` 钩子
8. 模式未变化，无需特殊处理

---

## 9. 关键文件索引

| 文件 | 职责 |
|---|---|
| [helix-term/src/keymap.rs](helix-term/src/keymap.rs) | Trie 数据结构、查找算法、合并逻辑 |
| [helix-term/src/keymap/macros.rs](helix-term/src/keymap/macros.rs) | `keymap!` / `key!` 等宏定义 |
| [helix-term/src/keymap/default.rs](helix-term/src/keymap/default.rs) | 默认键映射 |
| [helix-term/src/ui/editor.rs](helix-term/src/ui/editor.rs) | EditorView 事件处理与命令执行 |
| [helix-term/src/compositor.rs](helix-term/src/compositor.rs) | 组件层事件冒泡机制 |
| [helix-term/src/config.rs](helix-term/src/config.rs) | 配置加载与键映射合并 |
| [helix-term/src/commands.rs](helix-term/src/commands.rs) | MappableCommand 定义与 Context |
| [helix-view/src/input.rs](helix-view/src/input.rs) | KeyEvent 定义与字符串解析 |
| [helix-view/src/keyboard.rs](helix-view/src/keyboard.rs) | KeyCode / KeyModifiers 定义 |
| [helix-view/src/document.rs](helix-view/src/document.rs) | Mode 定义 |
| [helix-term/src/application.rs](helix-term/src/application.rs) | 主事件循环与终端事件接入 |
