# Helix 命令模式（Command Mode）解析流程

本文档详细解析 Helix 编辑器中，用户从按下 `:` 进入命令模式，到输入文本被解析、校验、补全、展示文档提示并最终执行为命令动作的完整流程。

---

## 总览：完整数据流转路径

```
用户按键 `:` 
  → 静态命令 command_mode() 被触发 [keymap/default.rs#L64]
    → 创建 Prompt UI 组件并推入 compositor [commands/typed.rs#L4058-L4074]
      ┌───────────────────────────────────────────────────────────────┐
      │                        Prompt 事件循环                        │
      │                                                               │
      │  ═════════════ handle_event 阶段（按键后立即执行）════════════ │
      │  按键事件 → Prompt::handle_event [ui/prompt.rs#L606-L766]     │
      │         │                                                     │
      │         ├─ 字符/删除键 → 修改 self.line                       │
      │         │       ↓                                             │
      │         │  1. recalculate_completion [ui/prompt.rs#L157-L160] │
      │         │     调用 completion_fn → 计算补全候选，保存到 self.completion │
      │         │  2. callback_fn(Update) → execute_command_line(Update) │
      │         │     无验证、无展开，仅让命令感知输入变化             │
      │         │                                                     │
      │         ├─ Tab/S-Tab → 切换 self.selection + 替换 self.line  │
      │         │       ↓                                             │
      │         │  ❌ 不重算补全候选（仅切换索引+填充）              │
      │         │  ✅ callback_fn(Update)（Tab 时，S-Tab 同理）       │
      │         │  例外：唯一候选是目录 → recalculate_completion      │
      │         │                                                     │
      │         ├─ Enter 键 → callback_fn(Validate) → close_fn       │
      │         │       ↓                                             │
      │         │  1. callback_fn(Validate) → execute_command_line(Validate) │
      │         │     完整解析 + 验证 + 展开 + 执行                   │
      │         │  2. 返回 close_fn → compositor.pop() 移除 Prompt   │
      │         │  3. should_redraw=true                              │
      │         │                                                     │
      │         └─ Esc/C-c → callback_fn(Abort) → close_fn           │
      │                 ↓                                             │
      │                 callback_fn(Abort) → 命令函数可恢复状态       │
      │                 返回 close_fn → compositor.pop() 移除 Prompt  │
      │                                                               │
      │  ═════════════════ render 阶段（红屏时执行）═════════════════ │
      │  输入变化时：Prompt 仍在 layers 中 → Prompt::render_prompt   │
      │         ├─ 渲染补全列表（使用 handle_event 阶段计算好的 completion）│
      │         └─ doc_fn(&self.line) → 计算文档提示 → 渲染帮助浮窗  │
      │                                                               │
      │  Enter/Esc 时：Prompt 已被 pop 移除 → 只渲染底层 EditorView │
      │         └─ EditorView::render → 编辑区 + 状态栏（含错误信息） │
      └───────────────────────────────────────────────────────────────┘
```

> **重要纠正**：`doc_fn` 文档提示计算 **不在 handle_event/Update 回调阶段**执行，而是在随后的 `render` 阶段执行。两者是完全分离的两个阶段。

---

## 第一层：进入命令模式

### 1. 按键映射触发

在默认 keymap 中，Normal 模式下的 `:` 键映射到静态命令 `command_mode`：

- 定义位置：[keymap/default.rs](./helix-term/src/keymap/default.rs#L64-L64)
- 注册位置：[commands.rs](./helix-term/src/commands.rs#L402-L402)

```rust
":" => command_mode,
```

`command_mode` 是一个 **Static 命令**（`MappableCommand::Static`），当被 `MappableCommand::execute` 调用时直接执行函数指针。

### 2. command_mode() 函数：Prompt 的创建

函数定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4058-L4074)

```rust
pub(super) fn command_mode(cx: &mut Context) {
    let mut prompt = Prompt::new(
        ":".into(),                              // 提示符
        Some(':'),                               // 历史记录寄存器
        complete_command_line,                   // 补全计算函数
        move |cx, input, event| {                // 事件回调闭包
            if let Err(err) = execute_command_line(cx, input, event) {
                cx.editor.set_error(err.to_string());
            }
        },
    );
    prompt.doc_fn = Box::new(command_line_doc);  // 文档提示函数
    prompt.recalculate_completion(cx.editor);    // 初始补全计算
    cx.push_layer(Box::new(prompt));             // 推入 UI 层栈
}
```

关键点：
- `Prompt` 是一个模态 UI 组件，接管所有键盘输入
- 绑定了三个核心函数：`complete_command_line`（补全）、`command_line_doc`（文档提示）、回调闭包（执行命令）
- 错误通过 `cx.editor.set_error()` 反馈给用户

---

## 第二层：Prompt UI 组件的输入处理

`Prompt` 组件定义在 [ui/prompt.rs](./helix-term/src/ui/prompt.rs)

### 1. 核心数据结构

```rust
pub struct Prompt {
    prompt: Cow<'static, str>,    // 提示符（如 ":"）
    line: String,                 // 当前输入的完整文本
    cursor: usize,                // 光标字节位置
    completion: Vec<Completion>,  // 补全候选项
    selection: Option<usize>,     // 当前选中的补全索引
    completion_fn: CompletionFn,  // 补全计算函数
    callback_fn: CallbackFn,      // 事件回调函数
    doc_fn: DocFn,                // 文档提示函数
    // ... 历史记录、光标滚动等字段
}
```

### 2. 键盘事件处理

`Prompt::handle_event` 在 [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L606-L766) 处理所有按键：

| 按键 | 行为 | 是否重算补全 | 触发回调事件 |
|------|------|-------------|-------------|
| 普通字符 | 插入到 `self.line` | ✅ `insert_char` 内部调用 `recalculate_completion` | `PromptEvent::Update` |
| `Enter` | 执行命令，关闭 Prompt | ❌ 特殊：目录补全时调用 `recalculate_completion`，否则不调用 | `PromptEvent::Validate` |
| `Esc` / `C-c` | 关闭 Prompt，丢弃输入 | ❌ | `PromptEvent::Abort` |
| `Tab` | 切换补全选项，自动填充 `self.line` | ❌ 不重算候选，仅切换 `self.selection` 索引；**例外**：唯一候选是目录时调用 `recalculate_completion` | `PromptEvent::Update` |
| `S-Tab` | 反向切换补全选项，自动填充 `self.line` | ❌ 不重算候选，仅切换 `self.selection` 索引 | `PromptEvent::Update` |
| `Backspace` / `C-h` | 删除前一个字符 | ✅ `delete_char_backwards` 内部调用 `recalculate_completion` | `PromptEvent::Update` |
| `C-w` / `A-Backspace` | 删除前一个词 | ✅ `delete_word_backwards` 内部调用 `recalculate_completion` | `PromptEvent::Update` |
| `Up` / `C-p` | 浏览历史记录 | ✅ `change_history` 内部调用 `recalculate_completion` | `PromptEvent::Update` |
| `Down` / `C-n` | 浏览历史记录 | ✅ `change_history` 内部调用 `recalculate_completion` | `PromptEvent::Update` |
| `C-q` | 退出补全选择状态 | ❌ 仅 `exit_selection()` | 无 |

### 3. Enter 键的特殊处理

当用户按下 `Enter` 时 [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L679-L709)：

```rust
key!(Enter) => {
    // 如果正在补全目录且输入以路径分隔符结尾，就不执行，而是继续补全
    if self.selection.is_some() && self.line.ends_with(std::path::MAIN_SEPARATOR) {
        self.recalculate_completion(cx.editor);
    } else {
        // 如果输入为空，使用历史记录中最近的一条
        let input = if self.line.is_empty() {
            &last_item  // 从历史寄存器读取
        } else {
            // 保存到历史寄存器（如果和上一条不同）
            if let Some(register) = self.history_register {
                cx.editor.registers.push(register, self.line.clone());
            }
            &self.line
        };

        // 关键：调用回调，传递 Validate 事件
        (self.callback_fn)(cx, input, PromptEvent::Validate);

        return close_fn;  // 从 compositor 栈中移除 Prompt
    }
}
```

---

## 第三层：两个阶段的动作划分：handle_event 阶段 vs render 阶段

> **纠正**：之前文档将文档提示描述为 Update 事件的并行分支是错误的。实际流程分为两个串行阶段：
> 1. **handle_event 阶段**（按键后立即执行）：补全计算 + Update 回调
> 2. **render 阶段**（随后红屏时执行）：文档提示计算 + 所有渲染

### 完整调用时序

#### 时序 A：输入字符（重算补全候选）

```
application::handle_terminal_events() [application.rs#L685-L760]
  ↓
compositor::handle_event() [compositor.rs#L144-L175]
  ↓
Prompt::handle_event() [ui/prompt.rs#L754-L761]
  ├─ 步骤 1：self.insert_char(c, cx) → 修改 self.line
  │   └─ 内部调用 self.recalculate_completion(cx.editor) [ui/prompt.rs#L268]
  │       └─ 调用 completion_fn → 计算补全，保存到 self.completion
  ├─ 步骤 2：(self.callback_fn)(cx, &self.line, PromptEvent::Update)
  │   └─ execute_command_line(Update) → 无验证无展开的命令解析
  └─ 返回 EventResult::Consumed(None) → consumed=true → should_redraw=true
  ↓
application::render() [application.rs#L255-L286]
  ↓
compositor::render() → Prompt 仍在 layers 中
  ↓
Prompt::render() → render_prompt() [ui/prompt.rs#L404-L511]
  ├─ 渲染补全列表（使用 self.completion）
  └─ doc_fn(&self.line) [ui/prompt.rs#L479] → 文档提示 → 渲染帮助浮窗
```

#### 时序 B：Tab 切换补全选项（不重算候选）

```
Prompt::handle_event() [ui/prompt.rs#L721-L728]
  ├─ 步骤 1：self.change_completion_selection(Forward) [ui/prompt.rs#L375-L394]
  │   ├─ 切换 self.selection 索引（不调用 recalculate_completion）
  │   └─ self.line.replace_range(range, &item.content) → 替换输入文本
  ├─ 步骤 2：唯一候选是目录？
  │   └─ 是 → self.recalculate_completion()（列出目录内容）
  ├─ 步骤 3：(self.callback_fn)(cx, &self.line, PromptEvent::Update)
  └─ 返回 EventResult::Consumed(None) → should_redraw=true
  ↓
render 阶段：同上，渲染更新后的补全列表和文档提示
```

#### 时序 C：Enter 执行命令（Prompt 被移除，底层重绘）

```
Prompt::handle_event() [ui/prompt.rs#L679-L709]
  ├─ 步骤 1：(self.callback_fn)(cx, input, PromptEvent::Validate)
  │   └─ execute_command_line(Validate) → 完整解析+验证+展开+执行
  └─ 返回 close_fn = EventResult::Consumed(Some(callback))
      ↓
compositor::handle_event() [compositor.rs#L161-L164]
  ├─ 收集 callback 到 callbacks 列表
  └─ 循环结束后执行 callback → compositor.pop() → Prompt 从 layers 移除
  返回 consumed=true → should_redraw=true
  ↓
application::render() [application.rs#L255-L286]
  ↓
compositor::render() → 遍历剩余 layers（只有 EditorView）
  ↓
EditorView::render() → 渲染编辑区 + 状态栏（含错误信息）
```

| 阶段 | 执行时机 | 调用的函数 | 作用 |
|------|---------|-----------|------|
| **handle_event 阶段** | 按键后立即同步执行 | `recalculate_completion` → `completion_fn` | 计算补全候选，保存到 `self.completion`（仅字符/删除/历史键触发，Tab 不触发） |
| **handle_event 阶段** | 按键后立即同步执行 | `callback_fn(Update/Validate/Abort)` | Update：命令感知输入变化；Validate：完整执行命令；Abort：恢复状态 |
| **render 阶段** | should_redraw=true 时执行 | Prompt 仍在：`doc_fn` → 文档提示 + 补全渲染 | Prompt 已移除：EditorView 渲染编辑区+状态栏 |

---

## 第三层分支 A：文档提示 command_line_doc（render 阶段）

定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4076-L4146)

**调用时机**：在 `Prompt::render_prompt` 中调用 [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L479-L479)，属于 render 阶段，**与 Update 回调无关**。每次重绘时都会重新计算，用于在命令行上方显示当前命令的帮助文档。

```rust
fn command_line_doc(input: &str) -> Option<Cow<'_, str>> {
    // 步骤 1：分离命令名
    let (command, _, _) = command_line::split(input);
    let command = TYPABLE_COMMAND_MAP.get(command)?;

    // 步骤 2：简单情况（无别名无 flag）直接返回 doc
    if command.aliases.is_empty() && command.signature.flags.is_empty() {
        return Some(Cow::Borrowed(command.doc));
    }

    // 步骤 3：组装完整文档
    let mut doc = command.doc.to_string();

    // 分支 3a：有别名，追加 Aliases 行
    if !command.aliases.is_empty() {
        write!(doc, "\nAliases: {}", command.aliases.join(", ")).unwrap();
    }

    // 分支 3b：有 flag，追加 Flags 表
    if !command.signature.flags.is_empty() {
        doc.push_str("\nFlags:");

        // 计算所有 flag 的最大长度，用于对齐
        let max_flag_len = command.signature.flags.iter().map(flag_len).max().unwrap();

        for flag in command.signature.flags {
            // 格式化每个 flag：--name/-n <arg>    文档
            write!(
                doc,
                "\n  --{flag_text}{spacer:spacing$}  {doc}",
                doc = flag.doc,
                flag_text = format_args!(
                    "{}{}{}{}",
                    flag.name,
                    if flag.alias.is_some() { "/-" } else { "" },
                    if let Some(alias) = flag.alias { alias.encode_utf8(&mut buf) } else { "" },
                    if flag.completions.is_some() { " <arg>" } else { "" }
                ),
            ).unwrap();
        }
    }

    Some(Cow::Owned(doc))
}
```

**文档提示的完整分支逻辑：**

```
command_line_doc(input)
  ├─ split() 分离命令名
  ├─ TYPABLE_COMMAND_MAP 查找
  │   └─ 找不到 → 返回 None（不显示文档）
  └─ 找到命令
      ├─ 无别名且无 flag → 返回 Borrowed(doc)
      ├─ 有别名 → 追加 "\nAliases: a, b"
      └─ 有 flag → 追加 "\nFlags:" + 对齐的 flag 列表
           ├─ 每个 flag 格式：--name/-n <arg>    文档
           └─ 计算最大 flag 长度用于列对齐
```

---

## 第三层分支 B：命令补全（handle_event 阶段）

### recalculate_completion 触发时机

定义在 [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L157-L160)

**调用时机**：在 `handle_event` 阶段，每次修改 `self.line` 后立即调用。具体触发点包括：

| 操作 | 代码位置 |
|------|---------|
| 插入字符 | `insert_char()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L268-L268) |
| 插入字符串 | `insert_str()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L274-L274) |
| 删除字符（向前） | `delete_char_backwards()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L295-L295) |
| 删除字符（向后） | `delete_char_forwards()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L302-L302) |
| 删除单词（向前） | `delete_word_backwards()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L310-L310) |
| 删除单词（向后） | `delete_word_forwards()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L317-L317) |
| 删除到行尾 | `kill_to_end_of_line()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L324-L324) |
| 删除到行首 | `kill_to_start_of_line()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L331-L331) |
| 切换补全选项 | `change_completion_selection()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L385-L385) |
| 从寄存器粘贴 | `paste_from_register()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L372-L372) |
| 浏览历史记录 | `cycle_history()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L372-L372) |
| 设置输入行 | `set_line()` → [ui/prompt.rs](./helix-term/src/ui/prompt.rs#L124-L124) |
| Prompt 创建时 | `command_mode()` → [commands/typed.rs](./helix-term/src/commands/typed.rs#L4072-L4072) |

```rust
pub fn recalculate_completion(&mut self, editor: &Editor) {
    self.exit_selection();
    self.completion = (self.completion_fn)(editor, &self.line);
}
```

### complete_command_line 补全逻辑

定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4148-L4168)

每次输入变化时调用，根据当前输入状态返回补全候选项。

```rust
fn complete_command_line(editor: &Editor, input: &str) -> Vec<ui::prompt::Completion> {
    let (command, rest, complete_command) = command_line::split(input);

    if complete_command {
        // 分支 B1：正在输入命令名 → 模糊匹配所有命令名
        fuzzy_match(
            input,
            TYPABLE_COMMAND_LIST.iter().map(|command| command.name),
            false,
        )
        .into_iter()
        .map(|(name, _)| (0.., name.into()))
        .collect()
    } else {
        // 分支 B2：命令名已完成 → 进入参数补全
        TYPABLE_COMMAND_MAP
            .get(command)
            .map_or_else(Vec::new, |cmd| {
                let args_offset = command.len() + 1;  // +1 跳过空格
                complete_command_args(editor, cmd.signature, &cmd.completer, rest, args_offset)
            })
    }
}
```

### 分支 B2 深度：complete_command_args 参数补全

定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4170-L4278)

参数补全的核心是：用 Tokenizer 模拟解析参数，根据最后一个 token 的类型和 Args 的解析状态决定补全内容。

```rust
pub fn complete_command_args(
    editor: &Editor,
    signature: Signature,
    completer: &CommandCompleter,
    input: &str,
    offset: usize,
) -> Vec<ui::prompt::Completion> {
    let cursor = input.len();
    let prefix = &input[..cursor];

    // 步骤 1：模拟解析（validate=false，不报错）
    let mut tokenizer = Tokenizer::new(prefix, false);
    let mut args = Args::new(signature, false);
    let mut final_token = None;
    let mut is_last_token = true;

    while let Some(token) = args.read_token(&mut tokenizer).unwrap() {
        final_token = Some(token.clone());
        args.push(token.content).unwrap();
        if tokenizer.pos() >= cursor {
            is_last_token = false;
        }
    }

    // 步骤 2：输入以空格结尾时，构造空 token 模拟新参数
    let token = if is_last_token {
        let token = Token::empty_at(prefix.len());
        args.push(token.content.clone()).unwrap();
        token
    } else {
        final_token.unwrap()
    };

    // 步骤 3：token 已闭合（如引号配对）不补全
    if token.is_terminated {
        return Vec::new();
    }

    // 步骤 4：根据 token.kind 分 8 种补全分支
    match token.kind {
        // 分支 B2a：普通文本或引号 → 看解析状态
        TokenKind::Unquoted | TokenKind::Quoted(_) => {
            match args.completion_state() {
                // 状态 A：位置参数 → 调用命令的 completer
                CompletionState::Positional => {
                    let n = args.len().checked_sub(1).unwrap();
                    let completer = completer.for_argument_number(n);
                    completer(editor, &token.content)
                        .into_iter()
                        .map(|(range, span)| quote_completion(&token, range, span, offset))
                        .collect()
                }
                // 状态 B：正在输入 flag → 模糊匹配所有 flag 名
                CompletionState::Flag(_) => fuzzy_match(
                    token.content.trim_start_matches('-'),
                    signature.flags.iter().map(|flag| flag.name),
                    false,
                )
                .into_iter()
                .map(|(name, _)| ((offset + token.content_start).., format!("--{name}").into()))
                .collect(),
                // 状态 C：等待 flag 参数值 → 模糊匹配该 flag 的 completions
                CompletionState::FlagArgument(flag) => fuzzy_match(
                    &token.content,
                    flag.completions.unwrap(),
                    false,
                )
                .into_iter()
                .map(|(value, _)| ((offset + token.content_start).., (*value).into()))
                .collect(),
            }
        }
        // 分支 B2b：双引号内部 或 Shell 展开 → 递归补全
        TokenKind::Expand | TokenKind::Expansion(ExpansionKind::Shell) => {
            let arg_completer = matches!(args.completion_state(), CompletionState::Positional)
                .then(|| {
                    let n = args.len().checked_sub(1).unwrap();
                    completer.for_argument_number(n)
                });
            complete_expand(editor, &token, arg_completer, offset + token.content_start)
        }
        // 分支 B2c：变量展开 %{...} → 补全变量名
        TokenKind::Expansion(ExpansionKind::Variable) => {
            complete_variable_expansion(&token.content, offset + token.content_start)
        }
        // 分支 B2d：Unicode 展开 → 不补全
        TokenKind::Expansion(ExpansionKind::Unicode) => Vec::new(),
        // 分支 B2e：寄存器展开 %reg{...} → 补全寄存器名
        TokenKind::Expansion(ExpansionKind::Register) => {
            complete_register_expansion(editor, &token.content, offset + token.content_start)
        }
        // 分支 B2f：不完整的展开前缀 %foo → 补全展开类型（sh/reg/u/变量名）
        TokenKind::ExpansionKind => {
            complete_expansion_kind(&token.content, offset + token.content_start)
        }
    }
}
```

**参数补全完整分支树：**

```
complete_command_args
  ├─ Tokenizer 模拟解析（validate=false）
  ├─ 空格结尾 → 构造空 token
  ├─ token.is_terminated → 返回空（如闭合引号后不补全）
  └─ match token.kind
      ├─ Unquoted/Quoted
      │    ├─ CompletionState::Positional → 调用命令的 Completer
      │    │     ├─ completers::filename     → 文件路径补全 [ui/mod.rs#L528-L680]
      │    │     ├─ completers::directory    → 目录补全 [ui/mod.rs#L577-L593]
      │    │     ├─ completers::buffer       → 已打开 buffer 补全 [ui/mod.rs#L444-L455]
      │    │     ├─ completers::theme        → 主题名补全 [ui/mod.rs#L457-L471]
      │    │     ├─ completers::language     → 语言名补全 [ui/mod.rs#L546-L559]
      │    │     ├─ completers::setting      → 配置项名补全 [ui/mod.rs#L514-L526]
      │    │     └─ quote_completion         → 含空格时自动加引号转义
      │    ├─ CompletionState::Flag → fuzzy_match 所有 flag 名
      │    └─ CompletionState::FlagArgument → fuzzy_match 该 flag 的候选值
      ├─ Expand / Expansion(Shell) → complete_expand 递归补全
      ├─ Expansion(Variable) → complete_variable_expansion 补全变量名
      ├─ Expansion(Unicode) → 空
      ├─ Expansion(Register) → complete_register_expansion 补全寄存器名
      └─ ExpansionKind → complete_expansion_kind 补全展开类型（sh/reg/u/变量）
```

### 分支 C：命令解析 execute_command_line（handle_event 阶段）

定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4015-L4036)

**调用时机**：在 `handle_event` 阶段通过 `callback_fn` 调用。**Update 事件** 和 **Validate 事件** 都会走这个分支，但行为完全不同。**render 阶段不会调用此函数**。

```rust
fn execute_command_line(
    cx: &mut compositor::Context,
    input: &str,
    event: PromptEvent,
) -> anyhow::Result<()> {
    // 步骤 1：分离命令名和参数
    let (command, rest, _) = command_line::split(input);

    if command.is_empty() {
        return Ok(());  // 空命令，忽略
    }

    // 特殊分支：纯数字命令 → 跳转到指定行
    if command.parse::<usize>().is_ok() && rest.trim().is_empty() {
        let cmd = TYPABLE_COMMAND_MAP.get("goto").unwrap();
        return execute_command(cx, cmd, command, event);
    }

    // 步骤 2：在命令注册表中查找
    match typed::TYPABLE_COMMAND_MAP.get(command) {
        Some(cmd) => execute_command(cx, cmd, rest, event),
        // 关键：只有 Validate 事件找不到命令才报错
        None if event == PromptEvent::Validate => {
            Err(anyhow!("no such command: '{command}'"))
        }
        // Update 事件找不到命令不报错，允许用户继续输入
        None => Ok(()),
    }
}
```

### split() 函数：命令与参数的分离

定义在 [command_line.rs](./helix-core/src/command_line.rs#L35-L44)

```rust
pub fn split(line: &str) -> (&str, &str, bool) {
    const SEPARATOR_PATTERN: [char; 2] = [' ', '\t'];

    // 按第一个空白字符分割
    let (command, rest) = line.split_once(SEPARATOR_PATTERN).unwrap_or((line, ""));

    // complete_command 标记：是否仍在输入命令名
    let complete_command =
        command.is_empty() || (rest.trim().is_empty() && !line.ends_with(SEPARATOR_PATTERN));

    (command, rest, complete_command)
}
```

例如：
- `"w"` → `("w", "", true)` — 正在输入命令名，补全命令
- `"write "` → `("write", "", false)` — 命令名已完成，补全参数
- `"write foo.txt"` → `("write", "foo.txt", false)` — 已有参数，补全参数

---

## 第四层：TypableCommand 命令注册表

### 1. TYPABLE_COMMAND_LIST 与 TYPABLE_COMMAND_MAP

- 命令列表定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs)（约 3000-4000 行的大数组）
- 命令映射构建在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4004-L4013)

```rust
pub static TYPABLE_COMMAND_MAP: Lazy<HashMap<&'static str, &'static TypableCommand>> =
    Lazy::new(|| {
        TYPABLE_COMMAND_LIST
            .iter()
            .flat_map(|cmd| {
                // 每个命令名作为 key
                std::iter::once((cmd.name, cmd))
                    // 所有别名也作为 key，指向同一个命令
                    .chain(cmd.aliases.iter().map(move |&alias| (alias, cmd)))
            })
            .collect()
    });
```

### 2. TypableCommand 结构体

定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L20-L30)

```rust
pub struct TypableCommand {
    pub name: &'static str,                    // 命令名，如 "write"
    pub aliases: &'static [&'static str],      // 别名，如 ["w"]
    pub doc: &'static str,                     // 帮助文档
    pub fun: fn(&mut compositor::Context, Args, PromptEvent) -> anyhow::Result<()>,
    pub completer: CommandCompleter,           // 参数补全逻辑
    pub signature: Signature,                  // 参数签名（位置参数数量、支持的标志等）
}
```

### 3. Signature 参数签名

定义在 [command_line.rs](./helix-core/src/command_line.rs#L93-L142)

每个命令声明它接受什么样的参数：

```rust
pub struct Signature {
    pub positionals: (usize, Option<usize>),   // (最少, 最多) 位置参数
    pub raw_after: Option<u8>,                 // N 个位置参数后，剩余内容原样作为一个参数
    pub flags: &'static [Flag],                // 支持的标志位
    pub _dummy: (),
}
```

示例（`:write` 命令）：
- `positionals: (0, Some(1))` —— 0 或 1 个位置参数（文件名）
- `flags` 包含 `--no-format` 标志

---

## 第五层：execute_command — Update vs Validate 的核心差异

函数定义在 [commands/typed.rs](./helix-term/src/commands/typed.rs#L4038-L4055)

**这是两种事件的关键分叉点：**

```rust
pub(super) fn execute_command(
    cx: &mut compositor::Context,
    cmd: &TypableCommand,
    args: &str,
    event: PromptEvent,
) -> anyhow::Result<()> {
    let args = if event == PromptEvent::Validate {
        // ============== Validate 模式（按回车） ==============
        // ✅ 开启验证
        // ✅ 执行全部展开（变量/Shell/寄存器/Unicode）
        // ❌ 解析失败会报错，命令终止
        Args::parse(args, cmd.signature, true, |token| {
            expansion::expand(cx.editor, token).map_err(|err| err.into())
        })
        .map_err(|err| anyhow!("'{}': {err}", cmd.name))?
    } else {
        // ============== Update 模式（输入变化） ==============
        // ❌ 关闭验证（避免打断用户输入）
        // ❌ 不执行展开（避免 Shell 执行等副作用）
        // ✅ 解析永远不会失败（expect 断言）
        Args::parse(args, cmd.signature, false, |token| Ok(token.content))
            .expect("arg parsing cannot fail when validation is turned off")
    };

    // 调用命令的实际实现函数
    (cmd.fun)(cx, args, event).map_err(|err| anyhow!("'{}': {err}", cmd.name))
}
```

### 输入变化 vs 回车执行：触发动作总览

| 事件类型 | 触发时机 | 触发的动作 |
|---------|---------|-----------|
| **输入变化**（普通字符、删除、Tab、方向键等） | `handle_event` 阶段 + `render` 阶段 | `handle_event` 阶段：<br>1. 修改 `self.line`（可能重算补全候选，见上表）<br>2. `callback_fn(Update)` → `execute_command_line(Update)`<br><br>`render` 阶段：<br>3. `Prompt::render_prompt()` → 渲染补全列表 + 文档提示 |
| **回车执行**（Enter 键） | `handle_event` 阶段 + `render` 阶段 | `handle_event` 阶段：<br>1. 保存输入到历史寄存器<br>2. `callback_fn(Validate)` → `execute_command_line(Validate)`<br>3. 返回 `close_fn` → compositor 执行回调 `compositor.pop()` 移除 Prompt 层<br>4. `compositor.handle_event` 返回 `consumed=true` → `should_redraw=true`<br><br>`render` 阶段：<br>5. **仍然执行** `application::render()`<br>6. `compositor.render()` 遍历剩余层 → 渲染底层 `EditorView`（显示命令执行结果或错误信息）<br>7. **不再渲染 Prompt**（已从 layers 移除） |

> **纠正**：之前文档描述"回车后没有 render 阶段"是错误的。回车后 `should_redraw=true` 仍然触发 `application::render()`，但此时 Prompt 已从 compositor 的 layers 中移除，所以 render 只渲染底层 EditorView，不会渲染 Prompt 自身。

### Update vs Validate 解析动作完整对比

| 解析阶段 | Update 事件（输入变化） | Validate 事件（按回车） |
|---------|------------------------|------------------------|
| **Tokenizer 分词** | ✅ 完整分词，validate=false | ✅ 完整分词，validate=true |
| **Tokenizer validate** | `false` — unterminated token 不报错 | `true` — 引号未闭合报 `UnterminatedToken` |
| **参数数量校验** | ❌ 跳过，`finish()` 直接返回 Ok | ✅ 校验，数量不符报 `WrongPositionalCount` |
| **Flag 重复校验** | ❌ 跳过 | ✅ 重复报 `DuplicatedFlag` |
| **未知 Flag 处理** | ✅ 当作位置参数（方便补全） | ❌ 报 `UnknownFlag` |
| **Flag 参数缺失校验** | ❌ 跳过 | ✅ 报 `FlagMissingArgument` |
| **Expansion 展开** | ❌ 不展开，原样返回 token.content | ✅ 全部展开（变量/Shell/寄存器/Unicode） |
| **副作用风险** | 无（不执行 Shell，不访问寄存器） | 有（执行 `%sh{cmd}`，读取寄存器） |
| **解析失败处理** | `expect()` 断言，理论上永不失败 | 报 `anyhow::Error`，显示给用户 |
| **命令函数调用** | ✅ 调用，但多数命令检测到 event != Validate 直接返回 Ok | ✅ 调用，真正执行操作 |

---

## 第六层：Tokenizer 词法分析（分词）

`Tokenizer` 定义在 [command_line.rs](./helix-core/src/command_line.rs#L387-L700)

它是一个迭代器，将输入字符串切分为一个个 `Token`。

### 1. Token 结构体

```rust
pub struct Token<'a> {
    pub kind: TokenKind,         // 令牌类型
    pub content_start: usize,    // 内容起始字节位置（用于补全替换）
    pub content: Cow<'a, str>,   // 令牌内容
    pub is_terminated: bool,     // 分隔符是否已闭合（如引号是否配对）
}
```

### 2. TokenKind 令牌类型

| 类型 | 示例输入 | 含义 |
|------|---------|------|
| `Unquoted` | `hello` | 普通未引用文本 |
| `Quoted(Single)` | `'hello world'` | 单引号引用，字面量，不展开 |
| `Quoted(Backtick)` | `` `echo hi` `` | 反引号引用，字面量，不展开 |
| `Expand` | `"line: %{cursor_line}"` | 双引号引用，内部可递归展开 |
| `Expansion(Variable)` | `%{cursor_line}` | 变量展开 |
| `Expansion(Unicode)` | `%u{25CF}` | Unicode 码点转字符 |
| `Expansion(Shell)` | `%sh{date}` | Shell 命令输出 |
| `Expansion(Register)` | `%reg{a}` | 寄存器内容 |
| `ExpansionKind` | `%foo` | 不完整的展开（用于补全） |

### 3. Tokenizer::next() 分词主逻辑

```rust
impl<'a> Iterator for Tokenizer<'a> {
    type Item = Result<Token<'a>, ParseArgsError<'a>>;

    fn next(&mut self) -> Option<Self::Item> {
        self.skip_blanks();  // 跳过空格和制表符

        let byte = self.byte()?;
        match byte {
            // 引号字符：调用 parse_quoted
            b'"' | b'\'' | b'`' => { ... }
            // 百分号：调用 parse_percent_token 解析扩展
            b'%' => self.parse_percent_token(),
            // 其他字符：解析为未引用 token，处理反斜杠转义
            _ => {
                // Unix 下支持反斜杠转义：\ 空格、\引号、\%
                Some(Ok(Token {
                    kind: TokenKind::Unquoted,
                    content_start: self.pos,
                    content: self.parse_unquoted(),
                    is_terminated: false,
                }))
            }
        }
    }
}
```

### 4. 引号转义规则

使用 **双写转义**：
- `'it''s'` → 内容为 `it's`
- `"say ""hello"""` → 内容为 `say "hello"`

平衡嵌套（用于 `%{...}`、`%sh{...}` 等）：
- `%sh{echo {hello}}` → 配对最外层的 `{}`，内容为 `echo {hello}`

---

## 第七层：Args 参数解析

`Args` 定义在 [command_line.rs](./helix-core/src/command_line.rs#L738-L1014)

`Args` 将 tokens 解释为 **位置参数（positionals）** 和 **标志位（flags）**。

### 1. Args::parse() 主流程

```rust
pub fn parse<M>(
    line: &'a str,
    signature: Signature,
    validate: bool,
    mut try_map_fn: M,     // token → 内容的映射函数（用于展开）
) -> Result<Self, Box<dyn Error>>
where
    M: FnMut(Token<'a>) -> Result<Cow<'a, str>, Box<dyn Error>>,
{
    let mut tokenizer = Tokenizer::new(line, validate);
    let mut args = Self::new(signature, validate);

    while let Some(token) = args.read_token(&mut tokenizer)? {
        let arg = try_map_fn(token)?;  // 在这里执行展开（Validate 模式）
        args.push(arg)?;               // 分类为 positional 或 flag
    }

    args.finish()?;  // 最终校验（Validate 模式）
    Ok(args)
}
```

### 2. read_token() — raw_after 的特殊处理

```rust
pub fn read_token<'p>(
    &mut self,
    parser: &mut Tokenizer<'p>,
) -> Result<Option<Token<'p>>, ParseArgsError<'p>> {
    // 如果已解析的 positional 数量 >= raw_after
    if self.signature.raw_after.is_some_and(|max| self.len() >= max as usize) {
        self.only_positionals = true;
        Ok(parser.rest())  // 剩余输入全部作为一个 Expand token，不再分词
    } else {
        parser.next().transpose()
    }
}
```

例如 `:set-option rulers [20, 30]`：
- `raw_after = Some(1)` → 第一个 positional 是 `"rulers"`
- 之后调用 `parser.rest()` → 剩余的 `" [20, 30]"` 作为单个字符串
- 后续 `:set-option` 内部自行解析 JSON

### 3. push() — 参数分类逻辑

```rust
pub fn push(&mut self, arg: Cow<'a, str>) -> Result<(), ParseArgsError<'a>> {
    // 1. "--" 标记：之后全部视为 positional
    if !self.only_positionals && arg == "--" {
        self.only_positionals = true;
        self.state = CompletionState::Flag(None);
    }
    // 2. 上一个是需要参数的 flag → 当前 token 作为该 flag 的参数值
    else if let Some(flag) = self.flag_awaiting_argument() {
        self.flags.insert(flag.name, arg);
        self.state = CompletionState::FlagArgument(flag);
    }
    // 3. 以 "-" 开头，且未进入 only_positionals → 视为 flag
    else if !self.only_positionals && arg.starts_with('-') {
        let flag = /* 在 signature.flags 中查找长名 --foo 或短名 -f */;

        let Some(flag) = flag else {
            // 验证模式下未知 flag 报错；否则当作 positional（方便补全）
            if self.validate {
                return Err(ParseArgsError::UnknownFlag { text: arg });
            }
            self.positionals.push(arg);
            return Ok(());
        };

        // 重复 flag 检测
        if self.validate && self.flags.contains_key(flag.name) {
            return Err(ParseArgsError::DuplicatedFlag { flag: flag.name });
        }

        self.flags.insert(flag.name, Cow::Borrowed(""));
        self.state = CompletionState::Flag(Some(*flag));
    }
    // 4. 否则就是位置参数
    else {
        self.positionals.push(arg);
        self.state = CompletionState::Positional;
    }
    Ok(())
}
```

### 4. finish() — 最终校验

```rust
fn finish(&self) -> Result<(), ParseArgsError<'a>> {
    if !self.validate { return Ok(()); }

    // 需要参数的 flag 没有接收到参数
    if let Some(flag) = self.flag_awaiting_argument() {
        return Err(ParseArgsError::FlagMissingArgument { flag: flag.name });
    }
    // 位置参数数量不符合签名
    self.signature.check_positional_count(self.positionals.len())?;

    Ok(())
}
```

---

## 第八层：Expansion 展开机制

展开逻辑定义在 [expansion.rs](./helix-view/src/expansion.rs)

### 1. expand() 主分发函数

```rust
pub fn expand<'a>(editor: &Editor, token: Token<'a>) -> Result<Cow<'a, str>> {
    match token.kind {
        // 单引号/反引号/普通文本：原样返回
        TokenKind::Unquoted | TokenKind::Quoted(_) => Ok(token.content),

        // 变量展开：%{cursor_line}
        TokenKind::Expansion(ExpansionKind::Variable) => {
            let var = Variable::from_name(&token.content)?;
            expand_variable(editor, var)
        }

        // Unicode 展开：%u{25CF} → ●
        TokenKind::Expansion(ExpansionKind::Unicode) => {
            let ch = char::from_u32(u32::from_str_radix(&token.content, 16)?)?;
            Ok(Cow::Owned(ch.to_string()))
        }

        // 双引号内部：递归展开
        TokenKind::Expand => expand_inner(editor, token.content),

        // Shell 命令：%sh{date}
        TokenKind::Expansion(ExpansionKind::Shell) => expand_shell(editor, token.content),

        // 寄存器：%reg{a}
        TokenKind::Expansion(ExpansionKind::Register) => expand_register(editor, token.content),

        TokenKind::ExpansionKind => unreachable!(),
    }
}
```

### 2. expand_inner() — 双引号内的递归展开

```rust
fn expand_inner<'a>(editor: &Editor, content: Cow<'a, str>) -> Result<Cow<'a, str>> {
    let mut escaped = String::new();
    let mut start = 0;

    while let Some(offset) = content[start..].find('%') {
        let idx = start + offset;

        // %% → 转义为单个 %
        if content.as_bytes().get(idx + 1) == Some(&b'%') {
            escaped.push_str(&content[start..=idx]);
            start = idx + 2;
        } else {
            escaped.push_str(&content[start..idx]);
            // 用 Tokenizer 递归解析这个 %... 展开
            let mut tokenizer = Tokenizer::new(&content[idx..], true);
            let token = tokenizer.parse_percent_token().unwrap()?;
            // 递归展开（可能嵌套多层）
            let expanded = expand(editor, token)?;
            escaped.push_str(expanded.as_ref());
            start = idx + tokenizer.pos();
        }
    }
    // ...
}
```

### 3. 支持的变量（Variable 枚举）

| 变量名 | 含义 |
|--------|------|
| `%{cursor_line}` | 主光标所在行（1-indexed） |
| `%{cursor_column}` | 主光标所在列（1-indexed） |
| `%{buffer_name}` | 当前 buffer 显示名 |
| `%{file_path_absolute}` | 当前文件绝对路径 |
| `%{line_ending}` | 当前文档换行符 |
| `%{current_working_directory}` | 工作目录 |
| `%{workspace_directory}` | 包含 .git/.svn 等的最近祖先目录 |
| `%{language}` | 当前语言 |
| `%{selection}` | 主选区内容 |
| `%{selection_line_start}` | 选区起始行 |
| `%{selection_line_end}` | 选区结束行 |

---

## 第九层：错误反馈完整路径

所有错误最终都会被 `execute_command_line` 的闭包捕获，并通过完整路径展示到用户界面。

### 1. 错误设置

```rust
move |cx: &mut compositor::Context, input: &str, event: PromptEvent| {
    if let Err(err) = execute_command_line(cx, input, event) {
        cx.editor.set_error(err.to_string());
    }
}
```

### 2. Editor::set_error 实现

定义在 [editor.rs](./helix-view/src/editor.rs#L1460-L1464)

```rust
pub fn set_error<T: Into<Cow<'static, str>>>(&mut self, error: T) {
    let error = error.into();
    log::debug!("editor error: {}", error);
    self.status_msg = Some((error, Severity::Error));
}
```

同时还有：
- `set_status()` → `Severity::Info` [editor.rs](./helix-view/src/editor.rs#L1453-L1456)
- `set_warning()` → `Severity::Warning` [editor.rs](./helix-view/src/editor.rs#L1467-L1471)

### 3. 错误清除时机

错误消息会在以下时机被清除：
- 用户按下任何按键时 [ui/editor.rs](./helix-term/src/ui/editor.rs#L1480-L1480)
- 处理非键盘输入时 [ui/editor.rs](./helix-term/src/ui/editor.rs#L1172-L1172)

### 4. 状态栏渲染

最终在 `EditorView::render` 中渲染到屏幕 [ui/editor.rs](./helix-term/src/ui/editor.rs#L1644-L1660)

```rust
// render status msg
if let Some((status_msg, severity)) = &cx.editor.status_msg {
    status_msg_width = status_msg.width();
    use helix_view::editor::Severity;
    let style = if *severity == Severity::Error {
        cx.editor.theme.get("error")    // 红色样式
    } else {
        cx.editor.theme.get("ui.text")  // 普通文本样式
    };

    // 绘制在屏幕最左下角
    surface.set_string(
        area.x,
        area.y + area.height.saturating_sub(1),
        status_msg,
        style,
    );
}
```

### 错误类型 ParseArgsError

定义在 [command_line.rs](./helix-core/src/command_line.rs#L164-L189)

| 错误变体 | 触发场景 | 用户看到的消息 |
|---------|---------|--------------|
| `WrongPositionalCount` | 参数数量不符合 signature | "expected exactly 1 argument, got 2" |
| `UnterminatedToken` | 引号或括号未闭合 | "unterminated token ..." |
| `DuplicatedFlag` | 同一个 flag 指定多次 | "flag '--foo' specified more than once" |
| `UnknownFlag` | signature 中未定义的 flag | "unknown flag '--bar'" |
| `FlagMissingArgument` | 需要参数的 flag 未给值 | "flag '--foo' missing an argument" |
| `MissingExpansionDelimiter` | `%foo` 后没有分隔符 | "'%' was not properly escaped..." |
| `UnknownExpansion` | 未知的展开类型 | "unknown expansion 'xxx'" |

### 命令自身的错误

命令函数返回的 `anyhow::Result<()>` 中的错误会被加上命令名前缀：

```rust
(cmd.fun)(cx, args, event).map_err(|err| anyhow!("'{}': {err}", cmd.name))
```

例如 `:write` 在权限不足时会显示：`'write': Permission denied (os error 13)`

---

## 第十层：命令函数内部的 PromptEvent 处理

每个 TypableCommand 的实现函数都会首先检查事件类型：

```rust
fn write(cx: &mut compositor::Context, args: Args, event: PromptEvent) -> anyhow::Result<()> {
    // 只有 Validate 事件才真正执行操作
    if event != PromptEvent::Validate {
        return Ok(());
    }
    // ... 实际写文件逻辑
}
```

有些命令（如 `:theme`）会在 `Update` 事件时做预览：

```rust
fn theme(cx: &mut compositor::Context, args: Args, event: PromptEvent) -> anyhow::Result<()> {
    match event {
        PromptEvent::Abort => {
            cx.editor.unset_theme_preview()?;  // 取消时恢复原主题
        }
        PromptEvent::Update => {
            if let Some(theme_name) = args.first() {
                if let Ok(theme) = cx.editor.theme_loader.load(theme_name) {
                    cx.editor.set_theme_preview(theme)?;  // 实时预览
                }
            }
        }
        PromptEvent::Validate => {
            // 回车时真正应用主题
            if let Some(theme_name) = args.first() {
                let theme = cx.editor.theme_loader.load(theme_name)?;
                cx.editor.set_theme(theme)?;
            }
        }
    }
    Ok(())
}
```

---

## 完整示例对比：输入变化（Update）与回车执行（Validate）的实际差异

### 场景：用户输入 `:write --no-format test.txt` 过程

#### 阶段 1：输入过程（每次按键 → handle_event 阶段 + render 阶段）

每次按键分为 **两个串行阶段** 执行：

**handle_event 阶段（按键后立即执行）：**

| 步骤 | 执行动作 |
|------|---------|
| 1 | 修改 `self.line`，如插入字符 `t` → `self.line` 变为 `"write --no-format test.t"` |
| 2 | `self.recalculate_completion(cx.editor)` → 调用 `complete_command_line("write --no-format test.t")` → `complete_command=false` → 进入参数补全 → 根据光标位置补全文件名，结果保存到 `self.completion` |
| 3 | `callback_fn(Update)` → `execute_command_line(..., Update)` → `Args::parse(validate=false, no_expand)` → 不验证、不展开 → `write(cx, args, Update)` → 函数检测到 event != Validate，直接返回 Ok |
| 4 | 返回 `EventResult::Consumed(None)` → `should_redraw = true` |

**render 阶段（handle_event 之后，should_redraw=true 时执行）：**

| 步骤 | 执行动作 |
|------|---------|
| 5 | `Prompt::render_prompt()` → 渲染补全列表（使用步骤 2 计算好的 `self.completion`） |
| 6 | `doc_fn("write --no-format test.t")` → `command_line_doc()` → 计算 write 命令的帮助文档，包括 Aliases 和 Flags 列表 → 渲染帮助浮窗（在命令行上方） |

---

#### 阶段 2：按下 Enter（触发 Validate）

触发 **handle_event 阶段 + render 阶段**，但 render 阶段不渲染 Prompt（已移除）：

**handle_event 阶段：**

| 步骤 | 执行动作 |
|------|---------|
| 1 | `callback_fn(Validate)` → `execute_command_line(..., Validate)` |
| 1a | `split()` → `("write", "--no-format test.txt", false)` |
| 1b | 查找到 `write` 命令 |
| 1c | `execute_command(..., validate=true)` |
| 1d | `Tokenizer::new("--no-format test.txt", true)` → 分词 → 两个 Unquoted token |
| 1e | 每个 token 调用 `expansion::expand()` → 都是 Unquoted，原样返回 |
| 1f | `Args::push("--no-format")` → 匹配 flag `no-format` |
| 1g | `Args::push("test.txt")` → positionals = `["test.txt"]` |
| 1h | `Args::finish()` → 验证通过 ✓ |
| 1i | `write(cx, args, Validate)` → 检测到 Validate → 实际保存文件 |
| 2 | 返回 `close_fn`（`EventResult::Consumed(Some(callback))`） |
| 3 | `compositor.handle_event` 收集 callback，循环结束后执行 `compositor.pop()` → Prompt 从 layers 移除 |
| 4 | `compositor.handle_event` 返回 `consumed=true` → `should_redraw=true` |

**render 阶段（should_redraw=true 触发）：**

| 步骤 | 执行动作 |
|------|---------|
| 5 | `application::render()` → `compositor.render()` |
| 6 | 遍历剩余 layers（此时只有 EditorView，Prompt 已被 pop 移除） |
| 7 | `EditorView::render()` → 渲染编辑区 + 状态栏<br>• 命令成功 → 状态栏可能显示成功信息<br>• 命令失败 → `editor.status_msg` 包含 `Severity::Error` → 红色错误文本显示在状态栏左下角 |

---

## 涉及的核心文件清单

| 文件 | 作用 |
|------|------|
| [helix-core/src/command_line.rs](./helix-core/src/command_line.rs) | Tokenizer、Args、Signature、Flag、ParseArgsError 等核心解析逻辑 |
| [helix-term/src/commands/typed.rs](./helix-term/src/commands/typed.rs) | TypableCommand 定义、TYPABLE_COMMAND_LIST/MAP、execute_command_line/execute_command、complete_command_line/complete_command_args、command_line_doc、各命令实现 |
| [helix-term/src/ui/prompt.rs](./helix-term/src/ui/prompt.rs) | Prompt UI 组件，handle_event 阶段的输入处理、recalculate_completion、callback_fn 调用；render 阶段的 doc_fn 调用、文档/补全渲染 |
| [helix-term/src/ui/mod.rs](./helix-term/src/ui/mod.rs#L424-L700) | completers 子模块：filename、directory、buffer、theme、language、setting 等补全函数 |
| [helix-term/src/application.rs](./helix-term/src/application.rs#L685-L760) | 主循环：handle_terminal_events 中的事件分发，handle_event 与 render 两个阶段的调度 |
| [helix-term/src/compositor.rs](./helix-term/src/compositor.rs#L144-L175) | UI 层事件分发：从顶层 UI 层（Prompt）向下传递键盘事件 |
| [helix-term/src/ui/editor.rs](./helix-term/src/ui/editor.rs#L1644-L1660) | EditorView::render 中 status_msg 的状态栏渲染逻辑 |
| [helix-term/src/commands.rs](./helix-term/src/commands.rs) | MappableCommand 定义，Static/Typable/Macro 命令分发 |
| [helix-term/src/keymap/default.rs](./helix-term/src/keymap/default.rs#L64) | 按键映射，`:` → command_mode |
| [helix-view/src/expansion.rs](./helix-view/src/expansion.rs) | Token 展开：变量、Shell、寄存器、Unicode、递归展开 |
| [helix-view/src/editor.rs](./helix-view/src/editor.rs#L1460-L1464) | set_error/set_status/set_warning 实现，status_msg 字段 |
