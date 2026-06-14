# Helix 命令模式（Command Mode）解析流程

本文档详细解析 Helix 编辑器中，用户从按下 `:` 进入命令模式，到输入文本被解析、校验并最终执行为命令动作的完整流程。

## 总览：数据流转路径

```
用户按键 `:` 
  → 静态命令 command_mode() 被触发
    → 创建 Prompt UI 组件并推入 compositor
      → 用户输入字符，Prompt 捕获并回调
        → execute_command_line() 解析 "命令名 + 参数"
          → split() 分离命令名与剩余参数
          → TYPABLE_COMMAND_MAP 查找 TypableCommand
            → execute_command() 执行：
              → Tokenizer 词法分析（分词）
              → Args 参数解析（标志位 + 位置参数）
              → expansion::expand() 变量/Shell/寄存器/Unicode 展开
              → cmd.fun() 最终调用命令函数
                → 成功：命令生效
                → 失败：cx.editor.set_error() 显示错误
```

---

## 第一层：进入命令模式

### 1. 按键映射触发

在默认 keymap 中，Normal 模式下的 `:` 键映射到静态命令 `command_mode`：

- 定义位置：[keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/keymap/default.rs#L64)
- 注册位置：[commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands.rs#L402)

```
":" => command_mode,
```

`command_mode` 是一个 **Static 命令**（MappableCommand::Static），当被 `MappableCommand::execute` 调用时直接执行函数指针。

### 2. command_mode() 函数：Prompt 的创建

函数定义在 [commands/typed.rs:4058-4074](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs#L4058-L4074)

```rust
pub(super) fn command_mode(cx: &mut Context) {
    let mut prompt = Prompt::new(
        ":".into(),                              // 提示符
        Some(':'),                               // 历史记录寄存器
        complete_command_line,                   // 补全函数
        move |cx, input, event| {                // 回调闭包
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
- 回调闭包会在三种事件下触发：`Update`（输入变化）、`Validate`（按回车）、`Abort`（按 Esc/C-c）
- 错误通过 `cx.editor.set_error()` 反馈给用户

---

## 第二层：Prompt UI 组件的输入处理

`Prompt` 组件定义在 [ui/prompt.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/ui/prompt.rs)

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

`Prompt::handle_event` 在 [ui/prompt.rs:606-766](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/ui/prompt.rs#L606-L766) 处理所有按键：

| 按键 | 行为 | 回调事件 |
|------|------|----------|
| 普通字符 | 插入到 `self.line`，更新补全 | `PromptEvent::Update` |
| `Enter` | 执行命令（详见下文） | `PromptEvent::Validate` |
| `Esc` / `C-c` | 关闭 Prompt，丢弃输入 | `PromptEvent::Abort` |
| `Tab` / `S-Tab` | 切换补全选项 | `PromptEvent::Update` |
| `Backspace` / `C-h` | 删除前一个字符 | `PromptEvent::Update` |
| `C-w` / `A-Backspace` | 删除前一个词 | `PromptEvent::Update` |
| `Up` / `C-p` | 浏览历史记录 | `PromptEvent::Update` |
| `Down` / `C-n` | 浏览历史记录 | `PromptEvent::Update` |

### 3. Enter 键的特殊处理

当用户按下 `Enter` 时 [ui/prompt.rs:679-709](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/ui/prompt.rs#L679-L709)：

```rust
key!(Enter) => {
    // 如果正在补全目录，就不执行，而是进入目录继续补全
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

## 第三层：命令行解析总入口 execute_command_line

函数定义在 [commands/typed.rs:4015-4036](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs#L4015-L4036)

```rust
fn execute_command_line(
    cx: &mut compositor::Context,
    input: &str,
    event: PromptEvent,
) -> anyhow::Result<()> {
    // 第一步：分离命令名和参数
    let (command, rest, _) = command_line::split(input);

    if command.is_empty() {
        return Ok(());  // 空命令，忽略
    }

    // 特殊：纯数字命令 => 跳转到指定行
    if command.parse::<usize>().is_ok() && rest.trim().is_empty() {
        let cmd = TYPABLE_COMMAND_MAP.get("goto").unwrap();
        return execute_command(cx, cmd, command, event);
    }

    // 第二步：在命令注册表中查找
    match typed::TYPABLE_COMMAND_MAP.get(command) {
        Some(cmd) => execute_command(cx, cmd, rest, event),
        None if event == PromptEvent::Validate => {
            // Validate 阶段找不到命令才报错
            Err(anyhow!("no such command: '{command}'"))
        }
        None => Ok(()),  // Update 阶段不报错，允许用户继续输入
    }
}
```

### split() 函数：命令与参数的分离

定义在 [command_line.rs:35-44](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs#L35-L44)

```rust
pub fn split(line: &str) -> (&str, &str, bool) {
    const SEPARATOR_PATTERN: [char; 2] = [' ', '\t'];

    // 按第一个空白字符分割
    let (command, rest) = line.split_once(SEPARATOR_PATTERN).unwrap_or((line, ""));

    // 是否仍在输入命令名（用于决定补全命令名还是参数）
    let complete_command =
        command.is_empty() || (rest.trim().is_empty() && !line.ends_with(SEPARATOR_PATTERN));

    (command, rest, complete_command)
}
```

例如：
- `"w"` → `("w", "", true)` — 正在输入命令名
- `"write "` → `("write", "", false)` — 命令名已完成，等待参数
- `"write foo.txt"` → `("write", "foo.txt", false)` — 已有参数

---

## 第四层：TypableCommand 命令注册表

### 1. TYPABLE_COMMAND_LIST 与 TYPABLE_COMMAND_MAP

- 命令列表定义在 [commands/typed.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs)（约 3000-4000 行的大数组）
- 命令映射构建在 [commands/typed.rs:4004-4013](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs#L4004-L4013)

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

定义在 [commands/typed.rs:20-30](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs#L20-L30)

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

定义在 [command_line.rs:93-142](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs#L93-L142)

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

## 第五层：execute_command — 参数解析与展开

函数定义在 [commands/typed.rs:4038-4055](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs#L4038-L4055)

```rust
pub(super) fn execute_command(
    cx: &mut compositor::Context,
    cmd: &TypableCommand,
    args: &str,
    event: PromptEvent,
) -> anyhow::Result<()> {
    let args = if event == PromptEvent::Validate {
        // Validate 阶段：完整解析 + 验证 + 展开
        Args::parse(args, cmd.signature, true, |token| {
            expansion::expand(cx.editor, token).map_err(|err| err.into())
        })
        .map_err(|err| anyhow!("'{}': {err}", cmd.name))?
    } else {
        // Update 阶段：仅解析，不验证，不展开（避免副作用）
        Args::parse(args, cmd.signature, false, |token| Ok(token.content))
            .expect("arg parsing cannot fail when validation is turned off")
    };

    // 调用命令的实际实现函数
    (cmd.fun)(cx, args, event).map_err(|err| anyhow!("'{}': {err}", cmd.name))
}
```

**关键区分：**
- `PromptEvent::Validate`（回车）：**开启验证**，执行展开，失败报错
- `PromptEvent::Update`（输入中）：**关闭验证**，不展开（避免执行 shell 命令等副作用），仅用于补全
- `PromptEvent::Abort`（取消）：命令函数自行决定是否处理

---

## 第六层：Tokenizer 词法分析（分词）

`Tokenizer` 定义在 [command_line.rs:387-700](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs#L387-L700)

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

`Args` 定义在 [command_line.rs:738-1014](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs#L738-L1014)

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
        let arg = try_map_fn(token)?;  // 在这里执行展开
        args.push(arg)?;               // 分类为 positional 或 flag
    }

    args.finish()?;  // 最终校验
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

展开逻辑定义在 [helix-view/src/expansion.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-view/src/expansion.rs)

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

## 第九层：错误反馈机制

所有错误最终都会被 `execute_command_line` 的闭包捕获：

```rust
move |cx: &mut compositor::Context, input: &str, event: PromptEvent| {
    if let Err(err) = execute_command_line(cx, input, event) {
        cx.editor.set_error(err.to_string());
    }
}
```

### 错误类型 ParseArgsError

定义在 [command_line.rs:164-189](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs#L164-L189)

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

## 完整示例：执行 `:write --no-format test.txt`

让我们走一遍完整流程：

### 步骤 1：用户按 `:`
→ `command_mode()` 被调用，创建 Prompt，推入 compositor。

### 步骤 2：用户逐字符输入 `write --no-format test.txt`
→ 每次输入触发 Prompt 的 Update 事件：
  - `execute_command_line("write --no-format test.txt", Update)`
  - `split()` 得到 `("write", "--no-format test.txt", false)`
  - 查找到 `write` 命令
  - `execute_command(..., validate=false)` → 不做验证和展开
  - 命令函数检测到 `event != Validate`，直接返回 Ok
  - 补全系统根据 Tokenizer 状态提供参数补全

### 步骤 3：用户按 Enter
→ `execute_command_line("write --no-format test.txt", Validate)`

子步骤 3a：`split()` → `("write", "--no-format test.txt", false)`

子步骤 3b：在 `TYPABLE_COMMAND_MAP` 查到 `write` 命令，其 signature 为：
```rust
positionals: (0, Some(1)),
flags: &[Flag { name: "no-format", alias: None, doc: "...", completions: None }],
```

子步骤 3c：`Tokenizer::new("--no-format test.txt", true)` 分词：
- Token 1: `kind=Unquoted, content="--no-format"`
- Token 2: `kind=Unquoted, content="test.txt"`

子步骤 3d：每个 token 调用 `expansion::expand()`：
- 都是 `Unquoted`，原样返回

子步骤 3e：`Args::push()` 逐个分类：
- `"--no-format"` → 匹配到 flag `no-format`，`flags.insert("no-format", "")`
- `"test.txt"` → positionals = `["test.txt"]`

子步骤 3f：`Args::finish()` 验证：
- positionals 数量 1，在 `(0, Some(1))` 范围内 ✓
- 没有等待参数的 flag ✓

子步骤 3g：调用 `write(cx, args, Validate)`：
- `event == Validate`，进入执行逻辑
- `args.first()` → `Some("test.txt")`
- `args.has_flag("no-format")` → `true`
- 执行保存操作

子步骤 3h：成功 → Prompt 关闭，回到 Normal 模式

---

## 涉及的核心文件清单

| 文件 | 作用 |
|------|------|
| [helix-core/src/command_line.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-core/src/command_line.rs) | Tokenizer、Args、Signature、Flag、错误类型等核心解析逻辑 |
| [helix-term/src/commands/typed.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands/typed.rs) | TypableCommand 定义、TYPABLE_COMMAND_LIST/MAP、execute_command_line/execute_command、各命令实现 |
| [helix-term/src/ui/prompt.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/ui/prompt.rs) | Prompt UI 组件，处理键盘输入、光标移动、历史记录、补全渲染 |
| [helix-term/src/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/commands.rs) | MappableCommand 定义，Static/Typable/Macro 命令分发 |
| [helix-term/src/keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-term/src/keymap/default.rs) | 按键映射，`:` → command_mode |
| [helix-view/src/expansion.rs](file:///d:/fz/0601/solo-dogfeeding/code/269-helix/helix-view/src/expansion.rs) | Token 展开：变量、Shell、寄存器、Unicode、递归展开 |
