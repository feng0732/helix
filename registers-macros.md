# 寄存器与宏代码处理路径分析

## 概述

本文档梳理 Helix 编辑器中寄存器（Registers）与宏（Macros）的代码处理路径，聚焦**录制**、**存储**、**回放**三种核心操作，以及它们之间的**组合边界**（录制中触发回放、回放按键不写回录制、嵌套与递归保护）。

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

**定义位置**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/editor.rs#L1206-L1207)

```rust
pub struct Editor {
    pub macro_recording: Option<(char, Vec<KeyEvent>)>,  // (寄存器名, 已捕获的按键序列)
    pub macro_replaying: Vec<char>,                      // 正在回放的寄存器名栈
}
```

**关键理解**: 这两个字段是**独立**的，不是互斥的。`macro_recording` 和 `macro_replaying` 可以同时处于活跃状态。

### 1.3 编辑模式 Mode

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

## 三、三种核心操作

### 3.1 录制 (record_macro)

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
        let reg = cx.register.take().unwrap_or('@');
        cx.editor.macro_recording = Some((reg, Vec::new()));
    }
}
```

**toggle 语义**: 同一个命令，第一次调用开始录制，第二次调用停止录制。

**停止时的 `keys.pop()`**: compositor 阶段①已经把停止键 `Q` push 进了 keys，这里 pop 掉它，确保宏内容不包含终止键。

### 3.2 存储 (registers.write)

**定义位置**: [register.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/register.rs#L80-L103)

宏内容以**单个字符串**存入寄存器：
- 普通字符直接拼接（如 `ihello`）
- 特殊按键用 `<>` 包裹（如 `<esc>`, `<C-a>`）
- 完整示例：`"ihello<esc>"` = 进入插入模式 → 输入 hello → 回到普通模式

寄存器值的反向存储是内部实现细节，宏写入时 `values.reverse()` 把只有一个元素的 vec 反转（无实际影响），读取时 `.rev()` 再反转回来。

### 3.3 回放 (replay_macro)

**定义位置**: [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6922-L6968)

```rust
fn replay_macro(cx: &mut Context) {
    let reg = cx.register.unwrap_or('@');

    // 防递归检查
    if cx.editor.macro_replaying.contains(&reg) {
        cx.editor.set_error(...);
        return;
    }

    // 从寄存器读取并解析按键序列
    let keys: Vec<KeyEvent> = /* registers.read → parse_macro */;

    // ① 立即标记为正在回放
    cx.editor.macro_replaying.push(reg);

    // ② 注册延迟回调
    let count = cx.count();
    cx.callback.push(Box::new(move |compositor, cx| {
        for _ in 0..count {
            for &key in keys.iter() {
                compositor.handle_event(&Event::Key(key), cx);  // 递归调用！
            }
        }
        cx.editor.macro_replaying.pop();  // ③ 回放结束后移除标记
    }));
}
```

**时序关键点**:
- `macro_replaying.push(reg)` 在步骤①**立即执行**
- 回调在步骤②被注册到 `cx.callback`，**不是立即执行**
- 回调实际执行时机：compositor 阶段③（见 2.1 节）
- `macro_replaying.pop()` 在步骤③，位于回调内部，回放完成后执行

**回调中递归调用 `compositor.handle_event`**: 每个模拟按键都会完整走一遍 compositor 的 ①②③ 三个阶段。

---

## 四、边界场景分析

### 4.1 录制中触发回放（Q 录制 → q 回放）

**场景**: 用户按 `Q` 开始录制，录制过程中按 `q` 触发回放。

**完整执行流程**:

```
用户按 q（此时 macro_recording=Some(('@',keys)), macro_replaying=[]）
│
├─ compositor 阶段①: 录制捕获
│   macro_recording=Some, macro_replaying=[]  →  keys.push('q')  ✅ 捕获
│
├─ compositor 阶段②: 事件分发
│   EditorView.handle_event → command_mode → replay_macro()
│   ├─ macro_replaying.contains('@')? → 否，通过
│   ├─ 从寄存器读取并解析按键
│   ├─ macro_replaying.push('@')          ← 立即标记
│   └─ cx.callback.push(回放回调)         ← 延迟注册
│
├─ compositor 阶段③: 回调执行
│   回放回调:
│   │
│   for key in keys:
│   │
│   ├─ compositor.handle_event(Event::Key(key))   ← 递归调用
│   │   │
│   │   ├─ 阶段①: 录制捕获
│   │   │   macro_recording=Some, macro_replaying=['@']  → 非空！
│   │   │   → 不捕获  ⭐ 回放按键不写回录制内容
│   │   │
│   │   ├─ 阶段②: 事件分发 → 正常执行命令
│   │   └─ 阶段③: 回调（如有）
│   │
│   macro_replaying.pop()                 ← 回放结束，移除标记
│
│ 此时状态: macro_recording=Some(('@',keys)), macro_replaying=[]
│ 继续录制...
```

**结论**: 录制中触发回放是允许的。回放期间 `macro_replaying` 非空，compositor 阶段①的守卫条件使得回放按键不会被写入录制内容。回放结束后 `macro_replaying.pop()` 清空，后续用户按键恢复被录制。

### 4.2 回放按键不写回录制内容的机制

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

### 4.3 嵌套回放：宏 A 回放中触发宏 B

**场景**: 寄存器 `@` 存有宏 A，寄存器 `a` 存有宏 B。宏 A 的内容包含 `q`（触发回放宏 B）。

**执行流程**:

```
用户按 q（回放宏 @）
│
├─ replay_macro('@')
│   ├─ macro_replaying.push('@')          → macro_replaying = ['@']
│   └─ cx.callback.push(回放回调)
│
├─ 回调执行:
│   for key in macro_A_keys:
│   │
│   ├─ ... 正常按键 ...
│   │
│   ├─ 遇到 'q' 键（触发 replay_macro('a')）
│   │   ├─ macro_replaying.contains('a')? → 否，通过
│   │   ├─ macro_replaying.push('a')      → macro_replaying = ['@', 'a']  ⭐ 嵌套入栈
│   │   └─ cx.callback.push(宏B回放回调)
│   │
│   ├─ 宏B回调执行（在当前 compositor.handle_event 的阶段③内）
│   │   for key in macro_B_keys:
│   │       compositor.handle_event(...)
│   │           → macro_replaying = ['@', 'a'] → 不捕获
│   │   macro_replaying.pop()              → macro_replaying = ['@']  ⭐ 嵌套出栈
│   │
│   ├─ ... 继续宏A的剩余按键 ...
│   │   → macro_replaying = ['@'] → 仍然不捕获
│   │
│   macro_replaying.pop()                  → macro_replaying = []  ⭐ 完全结束
```

**结论**: 嵌套回放通过栈结构自然支持。内层回放 push，结束后 pop，不影响外层。整个过程中 `macro_replaying` 始终非空，录制捕获始终被守卫条件阻止。

### 4.4 递归回放保护：宏 A 回放中再次触发宏 A

**场景**: 寄存器 `@` 存有宏 A，宏 A 的内容包含 `q`（触发回放同一寄存器 `@`）。

**执行流程**:

```
用户按 q（回放宏 @）
│
├─ replay_macro('@')
│   ├─ macro_replaying.contains('@')? → 否，通过
│   ├─ macro_replaying.push('@')      → macro_replaying = ['@']
│   └─ cx.callback.push(回放回调)
│
├─ 回调执行:
│   for key in macro_A_keys:
│   │
│   ├─ 遇到 'q' 键（触发 replay_macro('@')）
│   │   ├─ macro_replaying.contains('@')? → 是！  ⭐ 递归保护触发
│   │   ├─ set_error("Cannot replay from register [@]...")
│   │   └─ return  ← 直接返回，不执行回放
│   │
│   macro_replaying.pop()
```

**结论**: `macro_replaying.contains(&reg)` 检查阻止了同寄存器的递归回放。注意这是 **按寄存器名** 检查的——如果宏 A 调用宏 B，宏 B 调用宏 A，则宏 B 回放中触发 `replay_macro('@')` 时 `macro_replaying = ['@', 'b']`，`.contains('@')` 为真，也会被阻止。**间接递归同样被保护**。

### 4.5 callback 注册与执行的时序

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

**为什么 push 必须在 callback 之前**: 如果 push 也在 callback 内部，那么在回调执行之前，`macro_replaying` 仍为空。此时如果有其他机制（如事件循环的下一轮）触发 `replay_macro`，就无法检测到递归。代码注释（[commands.rs:6963-L6965](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs#L6963-L6965)）也确认了这一点：

> The macro under replay is cleared at the end of the callback, not in the macro replay context, or it will not correctly protect the user from replaying recursively.

**为什么 pop 必须在 callback 内部**: pop 标志着回放的真正结束。如果 pop 在 callback 外部（即 `replay_macro` 函数末尾），则在回调执行期间 `macro_replaying` 已经被清除，递归保护就失效了。

---

## 五、完整状态矩阵

| 场景 | `macro_recording` | `macro_replaying` | 录制捕获 | 命令执行 |
|------|-------------------|-------------------|----------|----------|
| 普通编辑 | `None` | `[]` | 否 | 是 |
| 仅录制 | `Some(('@',keys))` | `[]` | **是** | 是 |
| 仅回放 | `None` | `['@']` | 否 | 是 |
| 录制+回放 | `Some(('@',keys))` | `['@']` | **否**（守卫跳过） | 是 |
| 录制+嵌套回放 | `Some(('@',keys))` | `['@','a']` | **否** | 是 |

---

## 六、关键调用链图

### 6.1 录制中触发回放的完整调用链

```
用户按 q（录制进行中）
│
│ [compositor.handle_event — 外层调用]
│
├─ 阶段① 录制捕获:
│   macro_recording=Some, macro_replaying=[]  →  keys.push('q') ✅
│
├─ 阶段② 事件分发:
│   EditorView.handle_event → command_mode → replay_macro()
│   ├─ macro_replaying.push(reg)          → 状态变更
│   └─ cx.callback.push(回放闭包)
│
├─ 阶段③ 回调执行:
│   回放闭包(compositor, cx):
│   │
│   │ [compositor.handle_event — 递归调用，回放每个按键]
│   │
│   ├─ 阶段① 录制捕获:
│   │   macro_recording=Some, macro_replaying=[reg]  → 非空 → 不捕获 ⭐
│   │
│   ├─ 阶段② 事件分发: 命令正常执行
│   └─ 阶段③ 回调: （如有）
│   │
│   macro_replaying.pop()
│
│ [回到外层 compositor.handle_event]
│ 后续用户按键恢复录制捕获: macro_recording=Some, macro_replaying=[]
```

### 6.2 递归保护的调用链

```
replay_macro('@')
│
├─ macro_replaying.push('@')     → ['@']
└─ cx.callback.push(回放闭包)

回放闭包:
  for key in keys:
    compositor.handle_event(key)
    │
    ├─ 阶段①: macro_replaying=['@'] → 不捕获
    │
    ├─ 阶段②: 如果 key 映射到 replay_macro
    │   │
    │   ├─ replay_macro('@'):
    │   │   macro_replaying.contains('@')? → 真 ⛔
    │   │   set_error + return
    │   │
    │   └─ replay_macro('a'):  （间接递归）
    │       macro_replaying.contains('a')? → 否
    │       macro_replaying.push('a')     → ['@', 'a']
    │       cx.callback.push(宏B闭包)
    │       │
    │       宏B闭包:
    │         for key in b_keys:
    │           compositor.handle_event(key)
    │           │
    │           ├─ replay_macro('@'):  ← 间接递归
    │           │   macro_replaying.contains('@')? → 真 ⛔
    │           │   set_error + return
    │           │
    │         macro_replaying.pop()    → ['@']
    │
    macro_replaying.pop()          → []
```

---

## 七、设计要点总结

### 7.1 录制与回放不互斥

`macro_recording` 和 `macro_replaying` 是**独立字段**，可以同时活跃。录制中按 `q` 触发回放完全合法，回放结束后自动恢复录制。代码中没有任何地方在开始回放时清除录制状态，或在开始录制时检查回放状态。

### 7.2 回放按键不写回录制是守卫条件的效果

不是通过状态互斥实现的，而是通过 compositor 的双重守卫条件：

```rust
if macro_recording.is_some() && macro_replaying.is_empty()
```

回放期间第二个条件为假，所有按键（包括回放产生的）都不会进入 `macro_recording` 的 keys。

### 7.3 嵌套通过栈结构自然支持

`macro_replaying` 是 `Vec<char>`，每层回放 push 自己的寄存器名，结束后 pop。外层回放不受内层影响。

### 7.4 递归保护是按寄存器名检查

使用 `.contains(&reg)` 而非只检查栈顶，因此**间接递归**（A→B→A）也能被检测到。代价是无法在宏 B 中回放正在外层执行的宏 A，即使这不是真正的无限递归——这是保守但安全的选择。

### 7.5 push/pop 的时序是递归保护生效的关键

- `push` 在 callback 注册**之前**（立即执行），确保回调执行时标记已就位
- `pop` 在 callback **内部末尾**（延迟执行），确保整个回放期间标记持续有效

---

## 八、默认键绑定

**定义位置**: [default.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/keymap/default.rs#L160-L161)

| 按键 | 命令 | 说明 |
|------|------|------|
| `Q` | `record_macro` | 开始/停止录制宏 |
| `q` | `replay_macro` | 回放宏 |
| `"` + `{reg}` | 选择寄存器 | 如 `"aQ` 录制到寄存器 a，`"aq` 回放寄存器 a |

---

## 九、相关文件索引

| 文件 | 作用 |
|------|------|
| [helix-view/src/register.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/register.rs) | 寄存器核心实现（读写、特殊寄存器） |
| [helix-view/src/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/editor.rs) | Editor 结构体，含 `macro_recording` 和 `macro_replaying` 字段 |
| [helix-view/src/input.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/input.rs) | KeyEvent、`parse_macro` 宏解析 |
| [helix-term/src/compositor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/compositor.rs) | 事件分发、**录制捕获的唯一点** |
| [helix-term/src/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/commands.rs) | `record_macro`、`replay_macro` 命令实现 |
| [helix-term/src/ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/ui/editor.rs) | EditorView 按键处理、回调注册、录制指示器UI |
| [helix-term/src/keymap/default.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-term/src/keymap/default.rs) | 默认键绑定（Q=录制, q=回放） |
| [helix-view/src/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/270-helix/helix-view/src/document.rs) | Mode 枚举定义 |
