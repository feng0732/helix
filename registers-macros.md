# 寄存器与宏代码处理路径分析

## 概述

本文档梳理 Helix 编辑器中寄存器（Registers）与宏（Macros）的代码处理路径，聚焦**录制**、**存储**、**回放**三种核心操作，以及它们之间的**组合边界**（录制中触发回放、回放按键不写回录制、嵌套与递归保护）。特别深入分析**嵌套宏回放的触发条件**、**寄存器选择机制**、**默认寄存器**以及**递归栈如何确定回放目标**。

---

## 一、核心数据结构

### 1.1 寄存器 Registers

**定义位置**: [register.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/register.rs#L24-L32)

```rust
pub struct Registers {
    inner: HashMap<char, Vec<String>>,      // 寄存器名 -> 值列表（反向存储）
    clipboard_provider: Box<dyn DynAccess<ClipboardProvider>>,
    pub last_search_register: char,
}
```

**特殊寄存器**:
| 寄存器 | 读行为 | 写行为 |
|--------|--------|--------|
| `_` 黑洞 | 返回空迭代器 | 丢弃 |
| `#` 选区索引 | 动态生成选区编号 | 拒绝写入 |
| `.` 选区内容 | 动态返回当前选区文本 | 拒绝写入 |
| `%` 文档路径 | 动态返回当前文件名 | 拒绝写入 |
| `*` 系统剪贴板 | 优先读剪贴板，回退到缓存 | 同时写剪贴板和缓存 |
| `+` 主剪贴板 | 同上 | 同上 |

**存储策略**: `write` 时 `values.reverse()` 反向存储，`read` 时 `.rev()` 再反转回来。目的是让 `push` 操作（向末尾追加）在反向后的向量头部插入，保持 O(1)。

### 1.2 宏状态字段

**定义位置**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/editor.rs#L1203-L1207)

```rust
pub struct Editor {
    pub count: Option<std::num::NonZeroUsize>,
    pub selected_register: Option<char>,         // ⭐ 当前选中的寄存器（前缀状态）
    pub registers: Registers,
    pub macro_recording: Option<(char, Vec<KeyEvent>)>,  // (寄存器名, 已捕获的按键序列)
    pub macro_replaying: Vec<char>,                      // 正在回放的寄存器名栈
}
```

**关键理解**: 
- `selected_register` 是**前缀状态**，表示用户刚刚按了 `"` 键正在等待选择寄存器
- `macro_recording` 和 `macro_replaying` 是**独立**的，不是互斥的，可以同时活跃

### 1.3 命令上下文 Context

**定义位置**: [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L104-L112)

```rust
pub struct Context<'a> {
    pub register: Option<char>,                // ⭐ 当前命令使用的寄存器
    pub count: Option<NonZeroUsize>,
    pub editor: &'a mut Editor,
    pub callback: Vec<crate::compositor::Callback>,
    pub on_next_key_callback: Option<(OnKeyCallback, OnKeyCallbackKind)>,
    pub jobs: &'a mut Jobs,
}
```

**关键理解**: `cx.register` 是**单次命令的**寄存器参数，由 `selected_register` 在命令执行前传递而来。

### 1.4 编辑模式 Mode

**定义位置**: [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/document.rs#L65-L69)

```rust
pub enum Mode {
    Normal = 0,
    Select = 1,
    Insert = 2,
}
```

---

## 二、按键事件的统一入口与录制捕获

### 2.1 Compositor.handle_event — 所有按键必经之路

**定义位置**: [compositor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/compositor.rs#L144-L182)

```rust
pub fn handle_event(&mut self, event: &Event, cx: &mut Context) -> bool {
    // ① 录制捕获：在事件分发之前
    if let (Event::Key(key), Some((_, keys))) = (event, &mut cx.editor.macro_recording) {
        if cx.editor.macro_replaying.is_empty() {
            keys.push(*key);
        }
    }

    // ② 事件分发到各组件层（冒泡）
    for layer in self.layers.iter_mut().rev() {
        match layer.handle_event(event, cx) {
            EventResult::Consumed(Some(callback)) => {
                callbacks.push(callback);
                consumed = true;
                break;
            }
            // ...
        }
    }

    // ③ 执行回调
    for callback in callbacks {
        callback(self, cx)
    }

    consumed
}
```

**三个阶段**:
1. **阶段①** — 录制捕获：按键**先于**命令执行被记录
2. **阶段②** — 事件分发：按键被传递给 EditorView 执行命令
3. **阶段③** — 回调执行：命令产生的回调在此执行（包括宏回放）

### 2.2 录制捕获的守卫条件

阶段①的核心判断逻辑：

```
if macro_recording.is_some()         // 正在录制？
    && macro_replaying.is_empty()    // 且没有在回放？
    → keys.push(key)                 // 才记录按键
```

这意味着：
- **不在录制** → 不捕获（显然）
- **在录制 + 不在回放** → 捕获按键到 `macro_recording`
- **在录制 + 在回放** → **不捕获**，回放产生的按键不会污染录制内容

---

## 三、寄存器选择机制

### 3.1 寄存器选择命令 select_register

**定义位置**: [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6006-L6017)

```rust
fn select_register(cx: &mut Context) {
    cx.editor.autoinfo = Some(Info::from_registers(
        "Select register",
        &cx.editor.registers,
    ));
    cx.on_next_key(move |cx, event| {      // ⭐ 注册下一键回调
        cx.editor.autoinfo = None;
        if let Some(ch) = event.char() {
            cx.editor.selected_register = Some(ch);  // ⭐ 设置前缀状态
        }
    })
}
```

**工作原理**:
1. 用户按 `"` 键 → 触发 `select_register`
2. `select_register` 注册 `on_next_key` 回调，等待下一个按键
3. 下一个按键（如 `a`）到来时，回调被执行 → 设置 `selected_register = Some('a')`
4. 再下一个按键（如 `q` 或 `Q`）到来时，`selected_register` 被传递给命令

### 3.2 selected_register 到 cx.register 的传递

**定义位置**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/ui/editor.rs#L1077-L1095)

```rust
_ => {
    cxt.count = cxt.editor.count;

    // ⭐ 关键点：命令执行前从 selected_register 取出
    cxt.register = cxt.editor.selected_register.take();

    let res = self.handle_keymap_event(mode, cxt, event);
    // ...

    if self.keymaps.pending().is_empty() {
        cxt.editor.count = None
        // selected_register 已被 take()，不再恢复
    } else {
        // ⭐ 如果按键还有 pending（如组合键），归还 selected_register
        cxt.editor.selected_register = cxt.register.take();
    }
}
```

**传递路径**:
```
用户按 "a 选择寄存器 a
  ↓
selected_register = Some('a')  ← on_next_key 回调设置

用户按 q 触发回放
  ↓
cx.register = selected_register.take() = Some('a')  ← 命令执行前取出
  ↓
replay_macro(cx) 中使用 cx.register.unwrap_or('@') = 'a'
```

**关键细节**:
- `selected_register.take()` 会把 `selected_register` 置为 `None`
- 如果按键序列还有 pending（例如 `g` 等待第二个键），`selected_register` 会被归还
- 单键命令（如 `q`、`Q`、`y`、`p`）执行后 `selected_register` 保持 `None`

### 3.3 on_next_key 的执行时序

**定义位置**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/ui/editor.rs#L1475-L1485)

```rust
Event::Key(mut key) => {
    // ...
    let mode = cx.editor.mode();

    // ⭐ on_next_key 在常规命令处理之前执行
    if !self.on_next_key(OnKeyCallbackKind::PseudoPending, &mut cx, key) {
        match mode {
            Mode::Insert => { /* ... */ }
            mode => self.command_mode(mode, &mut cx, key),
        }
    }
    // ...
}
```

**时序**:
1. 按键到来 → 先检查 `on_next_key` 回调
2. 如果有回调且类型匹配 → 执行回调（设置 `selected_register`）→ 不再执行常规命令
3. 如果没有回调 → 执行常规命令（此时 `cx.register` 已被设置）

**重要结论**: `on_next_key` 回调在回放中**完全正常工作**。回调是 EditorView 的状态，与事件来源（终端还是宏回放）无关。这是嵌套宏回放能够选择不同寄存器的基础。

---

## 四、默认寄存器

### 4.1 两套不同的默认寄存器

Helix 有**两套独立**的默认寄存器：

| 用途 | 默认值 | 代码位置 |
|------|--------|----------|
| **宏操作**（record/replay） | `'@'` | [commands.rs:6915](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6915)、[commands.rs:6923](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6923) |
| **普通 yank/paste** | `'"'` | [editor.rs:1114](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/editor.rs#L1114) |

### 4.2 宏操作的默认寄存器

```rust
// record_macro 开始录制时
let reg = cx.register.take().unwrap_or('@');  // ⭐ 默认 '@'

// replay_macro 回放时
let reg = cx.register.unwrap_or('@');          // ⭐ 默认 '@'
```

### 4.3 普通 yank/paste 的默认寄存器

```rust
// yank 时
let reg_name = cx.register
    .unwrap_or_else(|| cx.editor.config.load().default_yank_register);  // ⭐ 默认 '"'

// paste 时
cx.register.unwrap_or(cx.editor.config().default_yank_register)        // ⭐ 默认 '"'
```

### 4.4 为什么有两套默认值

这是 Vim 传统的延续：
- 无名寄存器 `"` 用于普通的删除、复制、粘贴
- 宏寄存器 `@` 专门用于宏录制和回放
- 用户可以通过 `"` 前缀显式选择任意寄存器覆盖默认值

---

## 五、三种核心操作

### 5.1 录制 (record_macro)

**定义位置**: [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6893-L6920)

```rust
fn record_macro(cx: &mut Context) {
    if let Some((reg, mut keys)) = cx.editor.macro_recording.take() {
        // ── 停止录制 ──
        keys.pop();                    // 移除本次 Q 键本身
        let s = keys.into_iter()
            .map(|key| { /* 序列化为字符串 */ })
            .collect::<String>();
        cx.editor.registers.write(reg, vec![s]);
    } else {
        // ── 开始录制 ──
        let reg = cx.register.take().unwrap_or('@');  // ⭐ 目标寄存器
        cx.editor.macro_recording = Some((reg, Vec::new()));
    }
}
```

**目标寄存器确定**:
- 如果用户之前按了 `"a` → `cx.register = Some('a')` → 录制到寄存器 `a`
- 如果没有前缀 → `cx.register = None` → `unwrap_or('@')` → 录制到默认寄存器 `@`

**toggle 语义**: 同一个命令，第一次调用开始录制，第二次调用停止录制。

**停止时的 `keys.pop()`**: compositor 阶段①已经把停止键 `Q` push 进了 keys，这里 pop 掉它，确保宏内容不包含终止键。

### 5.2 存储 (registers.write)

**定义位置**: [register.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/register.rs#L80-L103)

宏内容以**单个字符串**存入寄存器：
- 普通字符直接拼接（如 `ihello`）
- 特殊按键用 `<>` 包裹（如 `<esc>`, `<C-a>`）
- 完整示例：`"ihello<esc>"` = 进入插入模式 → 输入 hello → 回到普通模式

### 5.3 回放 (replay_macro)

**定义位置**: [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6922-L6968)

```rust
fn replay_macro(cx: &mut Context) {
    let reg = cx.register.unwrap_or('@');    // ⭐ 1. 确定目标寄存器

    // ⭐ 2. 防递归检查：目标寄存器是否在回放栈中？
    if cx.editor.macro_replaying.contains(&reg) {
        cx.editor.set_error(...);
        return;
    }

    // 3. 从寄存器读取并解析按键序列
    let keys: Vec<KeyEvent> = /* registers.read → parse_macro */;

    // 4. 立即标记为正在回放
    cx.editor.macro_replaying.push(reg);

    // 5. 注册延迟回调
    let count = cx.count();
    cx.callback.push(Box::new(move |compositor, cx| {
        for _ in 0..count {
            for &key in keys.iter() {
                compositor.handle_event(&Event::Key(key), cx);  // 递归调用！
            }
        }
        cx.editor.macro_replaying.pop();
    }));
}
```

**目标寄存器确定顺序**:
1. 取 `cx.register`（来自 `selected_register.take()`）
2. 如果 `None` → 使用默认值 `'@'`

**递归保护检查**（第2步）:
- 检查**目标寄存器** `reg` 是否在 `macro_replaying` 栈中
- 这是按**寄存器名**检查，不是检查栈顶
- 间接递归（A→B→A）也会被检测到

**时序关键点**:
- `macro_replaying.push(reg)` 在步骤4 **立即执行**
- 回调在步骤5被注册到 `cx.callback`，**不是立即执行**
- 回调实际执行时机：compositor 阶段③
- `macro_replaying.pop()` 在回调内部末尾，回放完成后执行

**回调中递归调用 `compositor.handle_event`**: 每个模拟按键都会完整走一遍 compositor 的 ①②③ 三个阶段。

---

## 六、嵌套宏回放的触发条件

### 6.1 什么是嵌套宏回放

嵌套宏回放指：**一个宏的内容中包含了触发另一个宏回放的按键序列**。

**典型场景**（来自测试用例 [commands.rs:883-907](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/tests/test/commands.rs#L883-L907)）:
```
寄存器 'a' 内容: "ihello<esc>"       ← 插入 "hello"
寄存器 '@' 内容: "\"aqi<space>world<esc>"  ← 选择寄存器 a，回放，再插入 " world"

用户按 q 回放 @ → 输出 "hello world"
```

### 6.2 触发嵌套回放的必要条件

嵌套回放的触发需要**三个按键**在宏内容中连续出现：

| 按键 | 作用 | 代码 |
|------|------|------|
| `"` | 触发 `select_register`，注册 on_next_key 回调 | [commands.rs:6006](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6006) |
| `{reg}` | on_next_key 回调执行，设置 `selected_register = Some(reg)` | [commands.rs:6014](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6014) |
| `q` | 触发 `replay_macro`，使用 `cx.register.unwrap_or('@')` = `reg` | [commands.rs:6923](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6923) |

### 6.3 嵌套回放的完整执行流程

以测试用例为例：寄存器 `@` 内容为 `"aqi world<esc>`，用户按 `q` 回放。

```
用户按 q
│
│ [外层 compositor.handle_event — 处理用户的 q 键]
│
├─ 阶段①: macro_recording=None → 不捕获
│
├─ 阶段②:
│   EditorView.handle_event
│   ├─ on_next_key = None → 跳过
│   └─ command_mode:
│       ├─ cx.register = selected_register.take() = None
│       ├─ handle_keymap_event → replay_macro()
│       │   ├─ reg = None.unwrap_or('@') = '@'
│       │   ├─ macro_replaying.contains('@')? → [] → 否
│       │   ├─ 从 @ 读取内容: "\"aqi world<esc>"
│       │   ├─ parse_macro → [Key('"'), Key('a'), Key('q'), Key('i'), ...]
│       │   ├─ macro_replaying.push('@')  →  macro_replaying = ['@']
│       │   └─ cx.callback.push(回放回调)
│       └─ keymaps.pending() = true? → 否，count = None
│
├─ 阶段③: 执行回放回调
│   │
│   ├─ 第1个键: Key('"')  ← 触发 select_register
│   │   │
│   │   └─ compositor.handle_event(Key('"'))
│   │       ├─ 阶段①: macro_replaying=['@'] → 非空 → 不捕获
│   │       ├─ 阶段②:
│   │       │   EditorView.handle_event
│   │       │   ├─ on_next_key = None → 跳过
│   │       │   └─ command_mode → select_register()
│   │       │       └─ cx.on_next_key(closure)  ← 注册回调
│   │       └─ 阶段③: 无回调
│   │
│   ├─ 第2个键: Key('a')  ← on_next_key 回调执行
│   │   │
│   │   └─ compositor.handle_event(Key('a'))
│   │       ├─ 阶段①: macro_replaying=['@'] → 非空 → 不捕获
│   │       ├─ 阶段②:
│   │       │   EditorView.handle_event
│   │       │   ├─ on_next_key(PseudoPending) → 执行回调！
│   │       │   │   └─ selected_register = Some('a')  ⭐ 目标寄存器确定
│   │       │   └─ 返回 true → 跳过 command_mode
│   │       └─ 阶段③: 无回调
│   │
│   ├─ 第3个键: Key('q')  ← 触发 replay_macro('a')
│   │   │
│   │   └─ compositor.handle_event(Key('q'))
│   │       ├─ 阶段①: macro_replaying=['@'] → 非空 → 不捕获
│   │       ├─ 阶段②:
│   │       │   EditorView.handle_event
│   │       │   ├─ on_next_key = None → 跳过
│   │       │   └─ command_mode:
│   │       │       ├─ cx.register = selected_register.take() = Some('a')  ⭐ 取出
│   │       │       ├─ handle_keymap_event → replay_macro()
│   │       │       │   ├─ reg = Some('a').unwrap_or('@') = 'a'  ⭐ 目标是 'a'
│   │       │       │   ├─ macro_replaying.contains('a')? → ['@'] → 否 ✅ 通过
│   │       │       │   ├─ 从 'a' 读取内容: "ihello<esc>"
│   │       │       │   ├─ parse_macro → [Key('i'), Key('h'), ...]
│   │       │       │   ├─ macro_replaying.push('a')  →  ['@', 'a']  ⭐ 嵌套入栈
│   │       │       │   └─ cx.callback.push(宏a回放回调)
│   │       │       └─ keymaps.pending() = true? → 否
│   │       └─ 阶段③: 执行宏a回放回调
│   │           ├─ 依次回放 i, h, e, l, l, o, <esc>
│   │           │   (每个键都走完整 handle_event，macro_replaying=['@','a'] 非空 → 都不捕获)
│   │           └─ macro_replaying.pop()  →  ['@']  ⭐ 嵌套出栈
│   │
│   ├─ 第4个键: Key('i')  ← 进入插入模式
│   ├─ ... 后续按键 ...
│   └─ macro_replaying.pop()  →  []  ⭐ 外层回放结束
```

### 6.4 关键点总结

1. **on_next_key 在回放中正常工作**: 回调是 EditorView 的状态，与事件来源无关
2. **目标寄存器由 selected_register 传递**: `"` + `a` → `selected_register='a'` → `q` → `cx.register='a'`
3. **递归检查是按寄存器名**: `.contains('a')` 检查 `'a'` 是否在栈中，不是检查栈顶
4. **嵌套回放时栈增长**: `['@']` → `['@', 'a']` → `['@']` → `[]`
5. **整个嵌套过程录制捕获都被跳过**: 只要 `macro_replaying` 非空，阶段①就跳过捕获

---

## 七、递归栈与回放目标的交互

### 7.1 递归保护的精确逻辑

```rust
fn replay_macro(cx: &mut Context) {
    let reg = cx.register.unwrap_or('@');    // ① 确定目标寄存器

    // ② 检查目标寄存器是否正在回放栈中
    if cx.editor.macro_replaying.contains(&reg) {
        cx.editor.set_error(...);
        return;
    }

    // ③ 通过检查，push 到栈中
    cx.editor.macro_replaying.push(reg);

    // ④ 注册回调
    cx.callback.push(Box::new(move |compositor, cx| {
        // ... 回放 ...
        cx.editor.macro_replaying.pop();  // ⑤ 回放结束后 pop
    }));
}
```

### 7.2 允许 vs 禁止的场景

| 场景 | macro_replaying 栈 | 目标 reg | contains(&reg) | 结果 |
|------|-------------------|----------|----------------|------|
| 直接回放 `@` | `[]` | `'@'` | `[]` → false | ✅ 允许 |
| `@` 回放中触发 `a` | `['@']` | `'a'` | `['@']` → false | ✅ 允许（嵌套） |
| `@` 回放中触发 `@` | `['@']` | `'@'` | `['@']` → true | ❌ 禁止（直接递归） |
| `@`→`a` 中触发 `@` | `['@', 'a']` | `'@'` | `['@', 'a']` → true | ❌ 禁止（间接递归） |
| `@`→`a` 中触发 `b` | `['@', 'a']` | `'b'` | `['@', 'a']` → false | ✅ 允许（多层嵌套） |

### 7.3 间接递归的检测

**场景**: 宏 `@` 调用 `a`，宏 `a` 调用 `@`

```
用户按 q 回放 @
  ↓
replay_macro('@')
  macro_replaying = []
  contains('@')? → false
  push('@') → ['@']
  注册回调

回调执行:
  回放宏 @ 的内容
    ...
    遇到 "a  → selected_register = Some('a')
    遇到 q   → replay_macro('a')
                 contains('a')? → ['@'] → false
                 push('a') → ['@', 'a']
                 注册回调
                 
                 回调执行:
                   回放宏 a 的内容
                     ...
                     遇到 "a  → selected_register = Some('a')
                     遇到 q   → replay_macro('a')
                                  contains('a')? → ['@', 'a'] → true ⛔
                                  set_error + return
                                  ❌ 间接递归被阻止
                   pop('a') → ['@']
  pop('@') → []
```

**结论**: `.contains(&reg)` 是全栈搜索，不是只检查栈顶。这确保了任何形式的递归（直接或间接）都被阻止。

### 7.4 递归保护的代价

保守的全栈检查意味着：**即使不是真正的无限递归，只要目标寄存器在栈中某处，就会被拒绝**。

**示例**: 宏 `@` 调用 `a`，想在 `a` 中再次调用 `@` —— 即使这不会导致无限循环（例如有条件终止），也会被拒绝。

这是一个保守但安全的设计选择。

---

## 八、边界场景分析

### 8.1 录制中触发回放（Q 录制 → q 回放）

**场景**: 用户按 `Q` 开始录制，录制过程中按 `q` 触发回放。

```
用户按 q（macro_recording=Some(('@',keys)), macro_replaying=[]）
│
├─ 阶段①: 录制捕获
│   macro_recording=Some, macro_replaying=[]  →  keys.push('q')  ✅ 捕获
│
├─ 阶段②: 事件分发 → replay_macro()
│   ├─ reg = None.unwrap_or('@') = '@'
│   ├─ macro_replaying.contains('@')? → [] → 否
│   ├─ macro_replaying.push('@')          → ['@']
│   └─ cx.callback.push(回放回调)
│
├─ 阶段③: 回调执行
│   for key in keys:
│     compositor.handle_event(key)
│       ├─ 阶段①: macro_replaying=['@'] → 非空 → 不捕获 ⭐
│       ├─ 阶段②: 正常执行命令
│       └─ 阶段③: 回调（如有）
│   macro_replaying.pop()                 → []
│
继续录制，后续按键恢复捕获
```

**结论**: 录制中触发回放是允许的。`q` 键本身被录制，但回放产生的按键不被录制。

### 8.2 录制中触发嵌套回放

**场景**: 录制时按 `"aq` —— 选择寄存器 `a` 并回放它。

```
用户按 "（录制中）
  ↓
阶段①: macro_replaying=[] → 捕获 '"' ✅
阶段②: select_register() → 注册 on_next_key 回调

用户按 a（录制中）
  ↓
阶段①: macro_replaying=[] → 捕获 'a' ✅
阶段②: on_next_key 回调 → selected_register = Some('a')

用户按 q（录制中）
  ↓
阶段①: macro_replaying=[] → 捕获 'q' ✅
阶段②: command_mode
  cx.register = selected_register.take() = Some('a')
  replay_macro('a')
    contains('a')? → [] → 否
    push('a') → ['a']
    cx.callback.push(回放回调)

阶段③: 执行回调（回放 a 的内容）
  for key in a_keys:
    compositor.handle_event(key)
      阶段①: macro_replaying=['a'] → 非空 → 不捕获 ⭐
  pop('a') → []
```

**结论**: 触发嵌套回放的按键（`"`, `a`, `q`）被录制，但嵌套回放产生的按键不被录制。

### 8.3 回放按键不写回录制内容的机制

**守卫代码**: [compositor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/compositor.rs#L147-L151)

```rust
if let (Event::Key(key), Some((_, keys))) = (event, &mut cx.editor.macro_recording) {
    if cx.editor.macro_replaying.is_empty() {   // ⭐ 守卫条件
        keys.push(*key);
    }
}
```

**双重条件**:
1. `macro_recording.is_some()` — 必须正在录制
2. `macro_replaying.is_empty()` — 必须不在回放

只有同时满足两个条件，按键才被捕获。回放中条件2不满足，因此所有回放产生的按键都被跳过。

**注意**: 这不是"录制和回放互斥"的设计，而是"回放期间暂停录制捕获"。`macro_recording` 本身仍然是 `Some`，回放结束后自动恢复捕获。

### 8.4 callback 注册与执行的时序

**关键理解**: `cx.callback` 不是立即执行的。

```
replay_macro() 函数体内:
  ├── macro_replaying.push(reg)     ← 立即
  └── cx.callback.push(closure)     ← 注册延迟回调

← 返回到 EditorView.handle_event

EditorView.handle_event:
  └── let callbacks = take(&mut cx.callback)    ← 取出回调
      └── EventResult::Consumed(Some(callback))  ← 返回给 compositor

compositor 阶段③:
  └── callback(self, cx)                        ← 在此执行
      └── 回放回调:
          for key in keys:
              compositor.handle_event(...)       ← 递归！
          macro_replaying.pop()                  ← 回放结束后
```

**为什么 push 必须在 callback 之前**: 如果 push 也在 callback 内部，那么在回调执行之前，`macro_replaying` 仍为空。此时如果有其他机制触发 `replay_macro`，就无法检测到递归。代码注释（[commands.rs:6963-L6965](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6963-L6965)）也确认了这一点：

> The macro under replay is cleared at the end of the callback, not in the macro replay context, or it will not correctly protect the user from replaying recursively.

**为什么 pop 必须在 callback 内部**: pop 标志着回放的真正结束。如果 pop 在 callback 外部（即 `replay_macro` 函数末尾），则在回调执行期间 `macro_replaying` 已经被清除，递归保护就失效了。

---

## 九、完整状态矩阵

| 场景 | `macro_recording` | `macro_replaying` | `selected_register` | 录制捕获 | 命令执行 |
|------|-------------------|-------------------|---------------------|----------|----------|
| 普通编辑 | `None` | `[]` | `None` | 否 | 是 |
| 按 `"` 等待选择寄存器 | `None` | `[]` | `None` (on_next_key 待触发) | 否 | 是 |
| 按 `"a` 已选择寄存器 | `None` | `[]` | `Some('a')` | 否 | 是 |
| 仅录制 | `Some(('@',keys))` | `[]` | `None/Some` | **是** | 是 |
| 仅回放 | `None` | `['@']` | `None/Some` | 否 | 是 |
| 录制+回放 | `Some(('@',keys))` | `['@']` | `None/Some` | **否**（守卫跳过） | 是 |
| 录制+嵌套回放 | `Some(('@',keys))` | `['@','a']` | `None/Some` | **否** | 是 |

---

## 十、关键调用链图

### 10.1 嵌套宏回放的完整调用链

```
用户按 q（回放宏 @）
│
│ [外层 compositor.handle_event]
│
├─ 阶段①: macro_replaying=[] → 不捕获
│
├─ 阶段②: command_mode → replay_macro('@')
│   ├─ reg = cx.register.unwrap_or('@') = '@'
│   ├─ contains('@')? → [] → false
│   ├─ push('@') → ['@']
│   └─ cx.callback.push(回放回调)
│
├─ 阶段③: 执行回放回调
│   │
│   ├─ Key('"') → compositor.handle_event
│   │   └─ 阶段②: select_register() → cx.on_next_key(closure)
│   │
│   ├─ Key('a') → compositor.handle_event
│   │   └─ 阶段②: on_next_key 执行 → selected_register = Some('a')
│   │
│   ├─ Key('q') → compositor.handle_event
│   │   └─ 阶段②: command_mode
│   │       ├─ cx.register = selected_register.take() = Some('a')
│   │       └─ replay_macro('a')
│   │           ├─ reg = 'a'
│   │           ├─ contains('a')? → ['@'] → false
│   │           ├─ push('a') → ['@', 'a']
│   │           └─ cx.callback.push(宏a回调)
│   │
│   ├─ 阶段③（嵌套）: 执行宏a回调
│   │   ├─ for key in a_keys: compositor.handle_event(key)
│   │   │   └─ 阶段①: macro_replaying=['@','a'] → 不捕获
│   │   └─ pop('a') → ['@']
│   │
│   └─ pop('@') → []
```

### 10.2 递归保护的调用链

```
replay_macro(reg)
│
├─ let reg = cx.register.unwrap_or('@')  ⭐ 目标寄存器确定
│
├─ if macro_replaying.contains(&reg):   ⭐ 全栈检查
│   ├─ set_error
│   └─ return  ❌
│
├─ macro_replaying.push(reg)            ⭐ 入栈
├─ cx.callback.push(closure)
│
└─ 回调执行:
    for key in keys:
      compositor.handle_event(key)
      │
      ├─ 阶段①: macro_replaying 非空 → 不捕获
      │
      └─ 阶段②: 如果 key 映射到 replay_macro
          │
          ├─ replay_macro(reg2):
          │   ├─ reg2 确定
          │   └─ macro_replaying.contains(&reg2)?  ⭐ 再次检查
          │       ├─ true → ❌ 阻止
          │       └─ false → 继续嵌套
          │
    macro_replaying.pop()               ⭐ 出栈
```

---

## 十一、设计要点总结

### 11.1 录制与回放不互斥

`macro_recording` 和 `macro_replaying` 是**独立字段**，可以同时活跃。录制中按 `q` 触发回放完全合法，回放结束后自动恢复录制。

### 11.2 回放按键不写回录制是守卫条件的效果

不是通过状态互斥实现的，而是通过 compositor 的双重守卫条件：

```rust
if macro_recording.is_some() && macro_replaying.is_empty()
```

回放期间第二个条件为假，所有按键（包括回放产生的）都不会进入 `macro_recording` 的 keys。

### 11.3 嵌套通过栈结构自然支持

`macro_replaying` 是 `Vec<char>`，每层回放 push 自己的寄存器名，结束后 pop。外层回放不受内层影响。

### 11.4 寄存器选择在回放中正常工作

`on_next_key` 回调是 EditorView 的状态，与事件来源无关。宏内容中的 `"` + `reg` + `q` 序列可以正确选择并回放不同的寄存器。

### 11.5 递归保护是按寄存器名全栈检查

使用 `.contains(&reg)` 而非只检查栈顶，因此**间接递归**（A→B→A）也能被检测到。代价是无法在宏 B 中回放正在外层执行的宏 A，即使这不是真正的无限递归。

### 11.6 push/pop 的时序是递归保护生效的关键

- `push` 在 callback 注册**之前**（立即执行），确保回调执行时标记已就位
- `pop` 在 callback **内部末尾**（延迟执行），确保整个回放期间标记持续有效

### 11.7 两套默认寄存器

- 宏操作（record/replay）默认寄存器是 `'@'`
- 普通 yank/paste 默认寄存器是 `'"'`
- 都可以通过 `"` 前缀显式选择覆盖

---

## 十二、默认键绑定

**定义位置**: [default.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/keymap/default.rs#L160-L161)、[default.rs:L331](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/keymap/default.rs#L331)

| 按键 | 命令 | 说明 |
|------|------|------|
| `"` | `select_register` | 选择寄存器前缀 |
| `Q` | `record_macro` | 开始/停止录制宏 |
| `q` | `replay_macro` | 回放宏 |
| `"` + `{reg}` + `Q` | `select_register` + `record_macro` | 录制到指定寄存器（如 `"aQ`） |
| `"` + `{reg}` + `q` | `select_register` + `replay_macro` | 回放指定寄存器（如 `"aq`） |

---

## 十三、相关文件索引

| 文件 | 作用 |
|------|------|
| [helix-view/src/register.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/register.rs) | 寄存器核心实现（读写、特殊寄存器） |
| [helix-view/src/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/editor.rs) | Editor 结构体，含 `selected_register`、`macro_recording`、`macro_replaying` 字段 |
| [helix-view/src/input.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/input.rs) | KeyEvent、`parse_macro` 宏解析 |
| [helix-term/src/compositor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/compositor.rs) | 事件分发、**录制捕获的唯一点** |
| [helix-term/src/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs) | `record_macro`、`replay_macro`、`select_register` 命令实现 |
| [helix-term/src/ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/ui/editor.rs) | EditorView 按键处理、`on_next_key` 执行、`selected_register` 传递、回调注册 |
| [helix-term/src/keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/keymap/default.rs) | 默认键绑定（Q=录制, q=回放, "=选择寄存器） |
| [helix-view/src/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/document.rs) | Mode 枚举定义 |
| [helix-term/tests/test/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/tests/test/commands.rs#L883-L907) | 嵌套宏回放测试用例 |
