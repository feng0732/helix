# Helix 文本对象与移动实现规则

## 一、核心数据结构：Range 与 Selection

### Range（[selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/selection.rs#L55-L63)）

Range 是 Helix 中最基础的选区单元，由三个字段构成：

```rust
pub struct Range {
    pub anchor: usize,           // 锚点：扩展选区时不移动的一端
    pub head: usize,             // 头部：扩展选区时移动的一端
    pub old_visual_position: Option<(u32, u32)>,  // 上一次的视觉偏移（软换行+列号）
}
```

**关键语义**：

- **Gap indexing**：anchor 和 head 的索引代表字符之间的"间隙"位置，而非字符本身。例如索引 1 表示第 1 个和第 2 个字符之间的位置。
- **方向由大小决定**：`head >= anchor` → Forward 方向；`head < anchor` → Backward 方向。
- **左闭右开**：Range 在左侧包含，在右侧排他。两个相邻的 Range（共享一条边）不算重叠，但零宽 Range 会与另一个 Range 的左边界重叠。

### Block Cursor 可见光标语义（[selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/selection.rs#L332-L395)）

Helix 的光标是 **1 字素宽的方块光标（block cursor）**，始终位于 Range 的 **head 侧**，向 Range 内部延伸一个字素。`cursor()` 方法返回此方块**左边界**的位置：

| Range 方向 | head 位置 | cursor() 结果 | 可见光标覆盖范围 |
|-----------|-----------|--------------|----------------|
| Forward (anchor < head) | head 在右端 | `prev_grapheme_boundary(head)` | `[cursor, head]` — 右端最后一个字素 |
| Backward (head < anchor) | head 在左端 | `head` | `[head, next_grapheme_boundary(head)]` — 左端第一个字素 |
| 零宽 (head == anchor) | head=anchor | `head` | `[head, next_grapheme_boundary(head)]` — 从 head 起向右一个字素 |

> **关键理解**：head 是选区边界的间隙位置，而可见光标是覆盖一个字素的方块。`cursor()` 返回的是方块的左边缘位置。对于 Forward 选区，光标贴在选区右边缘上、朝左看；对于 Backward 选区，光标贴在选区左边缘上、朝右看。

**put_cursor 方法**（[selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/selection.rs#L353-L395)）：

```rust
pub fn put_cursor(self, text: RopeSlice, char_idx: usize, extend: bool) -> Range
```

将 block cursor 的左边缘移到 `char_idx`：

- `extend=false`：返回 `Range::point(char_idx)`（零宽 Range），丢弃原有选区。
- `extend=true`：保留 anchor，只移动 head；如果新位置跨过了 anchor，anchor 会向内收缩一个字素（模拟 1-宽度 block cursor 的内侧边）。

### Selection（[selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/selection.rs#L416-L420)）

Selection 是 Range 的有序集合，永远至少包含一个 Range：

```rust
pub struct Selection {
    ranges: SmallVec<[Range; 1]>,
    primary_index: usize,       // 主选区索引
}
```

Selection 构造时自动归一化（normalize）：按 `Range::from()` 排序，合并重叠的 Range。

### Range 的方向操作

| 方法 | 作用 |
|------|------|
| `from()` / `to()` | 返回 min(anchor,head) / max(anchor,head)，丢弃方向 |
| `direction()` | head >= anchor → Forward，否则 Backward |
| `flip()` | 交换 anchor 和 head |
| `with_direction(dir)` | 如果当前方向与 dir 不同则 flip |
| `put_cursor(text, idx, extend)` | 将 block cursor 左边缘移到 idx |
| `cursor(text)` | 返回 head 侧 block cursor 的左边界位置 |

---

## 二、TextObject 枚举：三种选区模式（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L52-L68)）

```rust
pub enum TextObject {
    Around,    // 选中对象及其边界（括号、空白等）
    Inside,    // 仅选中对象内容，不含边界
    Movement,  // 用于在对象之间跳转（goto treesitter object）
}
```

- **Around** 和 **Inside** 用于 `ma` / `mi` 选区命令，返回覆盖整个对象的 Range。
- **Movement** 仅在 `goto_treesitter_object` 中作为备选捕获名使用，优先级低于 Around/Inside。

---

## 三、TextObject 的四种实现

### 3.1 单词选区：textobject_word（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L71-L112)）

**流程**：
1. 取 cursor 位置。
2. 向后找词起始边界 `word_start`，向前找词结束边界 `word_end`。
3. `find_word_boundary` 根据 `long` 参数决定是否在字符类别变化时停下。
4. 特殊情况：若 `word_start == word_end`（光标在空白上），返回零宽 Range。

**Inside vs Around 的差异**：

| 模式 | 行为 |
|------|------|
| Inside | `Range::new(word_start, word_end)` — 纯词内容 |
| Around | 优先向右扩展连续空白；若右侧无空白则向左扩展；结果包含一侧空白 |

示例（光标在 "middle" 的 'd' 上）：
- `"cursor at middle of word"` → Inside: `(10, 16)`, Around: `(10, 17)`（右侧 1 个空格）

### 3.2 段落选区：textobject_paragraph（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L114-L199)）

段落以空行分隔。`count` 控制跨多少个非空段落。

**流程**：
1. 判断当前行与前后行的空/非空状态。
2. 向后搜索段落边界：先跳空行，再跳非空行。
3. 向前搜索 `count` 个非空段落：交替跳非空行和空行。
4. 若已到文末，回溯寻找上一个段落起始。

**Inside vs Around 的差异**：

| 模式 | 行为 |
|------|------|
| Inside | 尾部不含最后的空白段落（空行组） |
| Around | 尾部包含空白段落（空行组） |

### 3.3 括号对选区：textobject_pair_surround（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L201-L253)）

两个入口：
- `textobject_pair_surround` — 指定括号字符 `ch`
- `textobject_pair_surround_closest` — 自动查找最近的括号对

底层调用 `surround::find_nth_pairs_pos` / `find_nth_closest_pairs_pos`，返回 `(anchor, head)` 括号对位置（无序）。

**Inside vs Around 的差异**（当 anchor < head 时）：

| 模式 | 范围 |
|------|------|
| Inside | `(next_grapheme(anchor), head)` — 跳过左括号，不含右括号 |
| Around | `(anchor, next_grapheme(head))` — 含左右括号 |

当 anchor > head 时对称处理。`count` 参数控制向外跳几层嵌套括号。

### 3.4 Tree-sitter 语法对象选区：textobject_treesitter（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L258-L293)）

**流程**：
1. 将 cursor 的 char 位置转为 byte 位置。
2. 通过 `syntax.layer_for_byte_range` 确定当前注入层（支持嵌入语言）。
3. 用 `loader.textobject_query` 获取当前语言的 textobject 查询。
4. 构造捕获名 `"{object_name}.{textobject}"`（如 `"function.inside"`、`"class.around"`）。
5. 从所有捕获节点中筛选包含当前 byte 位置的节点，选最小节点（最内层匹配）。
6. 将节点的 byte 范围转为 char 范围，返回 `Range::new(start_char, end_char)`（始终 Forward 方向）。

> **注意**：`textobject_treesitter` 返回的 Range 始终是 Forward 方向（anchor 在对象起点，head 在对象终点）。调用方负责根据需要调整方向。

---

## 四、Motion 系统

### 4.1 Direction 与 Movement 枚举（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L22-L32)）

```rust
pub enum Direction { Forward, Backward }  // 方向
pub enum Movement { Extend, Move }        // 行为：扩展选区 / 移动
```

- **Move**：移动到新位置，结果通常是零宽或 1 字素宽的 Range。
- **Extend**：保留 anchor，移动 head 扩展选区。

### 4.2 水平移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L34-L53)）

```rust
pub fn move_horizontally(slice, range, dir, count, behaviour, ...) -> Range
```

计算新位置后调用 `range.put_cursor(slice, new_pos, behaviour == Movement::Extend)`。

- **Move 模式**：返回零宽 Range（`Range::point(new_pos)`）。
- **Extend 模式**：保留 anchor，扩展 head 到新位置。

### 4.3 垂直移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L55-L166)）

分为 `move_vertically_visual`（支持软换行）和 `move_vertically`（逻辑行移动）。

核心逻辑：
1. 获取当前光标的视觉坐标 `(row, col)`。
2. 保留 `old_visual_position` 用于记忆列号（短行不丢失目标列号）。
3. 移动到目标行，用 `char_idx_at_visual_block_offset` 计算新位置。
4. 特殊处理：Extend 模式下如果目标行是空行则不移动。

### 4.4 单词移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L168-L262)）

12 种单词移动目标（[WordMotionTarget](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L391-L410)）：

| 类别 | 目标 | 说明 |
|------|------|------|
| Word | NextWordStart / NextWordEnd / PrevWordStart / PrevWordEnd | 标准单词（字符类别变化为边界） |
| Long Word | NextLongWordStart / NextLongWordEnd / PrevLongWordStart / PrevLongWordEnd | WORD（仅空白为边界，标点与字母相连） |
| Sub Word | NextSubWordStart / NextSubWordEnd / PrevSubWordStart / PrevSubWordEnd | 子词（驼峰、下划线处也分割） |

**word_move 函数的关键步骤**（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L216-L262)）：

1. **方向预处理**：根据移动方向和当前 Range 方向，构造 1 字素宽的 `start_range`（代表 block cursor）：
   - 向前移动且当前是 Forward 选区 → `Range::new(prev(head), head)`
   - 向前移动且当前是 Backward 选区 → `Range::new(head, next(head))`
   - 向后移动对称处理
2. **循环 count 次**：调用 `chars.range_to_target(target, range)` 计算每次移动。
3. **提前退出**：如果移动结果与上一次相同，说明已到边界。

> **注意**：`word_move` 返回的 Range **不是零宽的**。它返回从起始边界到目标边界的完整范围，相当于"从当前光标位置到目标位置之间的选区"。

**range_to_target 核心逻辑**（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L416-L490)）：

1. 若向后移动，反转迭代器。
2. 跳过起始的换行符，若经过换行则更新 anchor。
3. 遍历字符，调用 `reached_target` 判断是否到达目标边界。
4. 首次到达时如果 head 未移动（`head == head_start`），将 anchor 更新到 head 位置（表示"从一个边界开始移动"需要前进到下一个边界）。
5. 第二次到达时停止。

**三种边界判断函数**：

| 函数 | 规则 |
|------|------|
| `is_word_boundary(a, b)` | `categorize_char(a) != categorize_char(b)` |
| `is_long_word_boundary(a, b)` | Word 与 Punctuation 不算边界，其余类别变化算 |
| `is_sub_word_boundary(a, b, dir)` | Word 内部：下划线变更是边界；正向时小写→大写是边界，反向时大写→小写是边界 |

**reached_target 的条件**（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L524-L559)）：

到达目标 = 满足边界条件 **且** 满足额外约束：

- `NextWordStart` / `PrevWordEnd`：边界 + next_ch 非空白（或 EOL）
- `NextWordEnd` / `PrevWordStart`：边界 + prev_ch 非空白（或 next_ch 是 EOL）
- Long Word 版本同理，换用 `is_long_word_boundary`
- Sub Word 版本增加下划线处理

### 4.5 段落移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L264-L354)）

`move_prev_paragraph` 和 `move_next_paragraph` 接受 `behavior: Movement` 参数：

- 向后移动：先跳空行，再跳非空行，到达段落起始。
- 向前移动：先跳非空行，再跳空行，到达段落结束。
- **Move 模式**：anchor 设为 cursor 位置，head 设为段落边界。
- **Extend 模式**：anchor 取 `put_cursor(head, true).anchor`（保留扩展语义）。

### 4.6 Tree-sitter 对象跳转：goto_treesitter_object（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L561-L624)）

`goto_treesitter_object` 用于 `]f` / `[f` 等跳转命令。函数签名：

```rust
pub fn goto_treesitter_object(
    slice: RopeSlice, range: Range, object_name: &str,
    dir: Direction, syntax: &Syntax, loader: &syntax::Loader, count: usize,
) -> Range
```

**执行流程**：

1. 依次尝试匹配 `{name}.movement`、`{name}.around`、`{name}.inside` 三种捕获名（`capture_nodes_any`）。
2. **Forward 方向**：找 `start_byte > cursor_byte` 的节点，按 `(start_byte, Reverse(end_byte))` 排序取最小（最近、最短的对象）。
3. **Backward 方向**：找 `end_byte < cursor_byte` 的节点，按 `(end_byte, Reverse(start_byte))` 排序取最大（最近、最短的对象）。
4. 循环 count 次逐步跳转，直到无法继续。

**返回值**：始终返回 Forward 方向的完整对象 Range：`Range::new(start_char, end_char)`。

> **关键校正**：函数内注释 "head of range should be at beginning" 与代码不符。实际代码 `Range::new(start_char, end_char)` 中 head 在 `end_char`（对象终点），anchor 在 `start_char`（对象起点）。返回值是覆盖整个对象的 Forward Range。

### 4.7 父节点端点移动：move_parent_node_end（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L637-L699)）

用于 `]e` / `[e` 跳到语法节点端点，是理解光标语义的**标杆函数**：

```rust
pub fn move_parent_node_end(syntax, text, selection, dir, movement) -> Selection
```

**Forward 方向**：移到当前节点的 `end_byte` 位置（节点末尾的间隙）。

**Backward 方向**：移到当前节点的 `start_byte` 位置；如果已在起点，则向上到父节点的起点。

**Move 模式（Normal 模式的典型行为）**：
- 原 Range 是 Forward → `Range::new(end_head, end_head + 1)`（1 字素宽的 Forward Range）
- 原 Range 是 Backward → `Range::new(end_head + 1, end_head)`（1 字素宽的 Backward Range）
- **效果**：保留原方向，可见光标落在目标字符上。

**Extend 模式（Select 模式的典型行为）**：
- 如果目标位置 >= anchor → `end_head += 1`（向右侧多扩展一个字素，使最后一个字符被包含）
- `Range::new(range.anchor, end_head)` — 保留原 anchor，扩展 head

> **设计模式**：Move 模式产生 1 字素宽的"光标选区"，方向与原选区一致。Extend 模式则保留 anchor 并扩展 head，且在向前扩展时 head 需 +1 以包含目标字符。

---

## 五、命令层组合方式

### 5.1 两种命令组合模式

Helix 的命令层有两种不同的 Normal/Select 模式组合模式：

**模式 A：独立函数对**（单词移动风格）

为 Normal 模式和 Select 模式分别提供独立的命令函数，绑定到不同模式的 keymap 中。

- Normal 模式 `w` → `move_next_word_start` → `move_word_impl`
- Select 模式 `w` → `extend_next_word_start` → `extend_word_impl`

实现见 [commands.rs:1195-1256](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L1195-L1256) 和 [commands.rs:1588-1649](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L1588-L1649)。

**模式 B：模式检测内部分支**（段落/跳转风格）

同一个命令函数内部通过 `editor.mode == Mode::Select` 判断行为。

```rust
let behavior = if editor.mode == Mode::Select {
    Movement::Extend
} else {
    Movement::Move
};
```

示例：`goto_para_impl`（[commands.rs:1258-1279](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L1258-L1279)）、`goto_ts_object_impl`。

### 5.2 apply_motion 机制（[editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-view/src/editor.rs#L1398-L1410)）

```rust
pub fn apply_motion<F: Fn(&mut Self) + 'static>(&mut self, motion: F) {
    motion(self);
    self.last_motion = Some(Box::new(motion));  // 保存用于重复
}
```

所有移动/选区命令都通过 `apply_motion` 执行，自动记录到 `last_motion` 供 `repeat_last_motion`（`.` 命令）使用。

### 5.3 TextObject 选区命令（[commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L6231-L6329)）

入口：`ma`（Around）/ `mi`（Inside），然后等待第二个按键选择对象类型：

| 按键 | 对象 | 实现函数 |
|------|------|----------|
| `w` | word | `textobject_word` (long=false) |
| `W` | WORD | `textobject_word` (long=true) |
| `p` | paragraph | `textobject_paragraph` |
| `m` | closest pair | `textobject_pair_surround_closest` |
| `t` | class (TS) | `textobject_treesitter("class")` |
| `f` | function (TS) | `textobject_treesitter("function")` |
| `a` | parameter (TS) | `textobject_treesitter("parameter")` |
| `c` | comment (TS) | `textobject_treesitter("comment")` |
| `T` | test (TS) | `textobject_treesitter("test")` |
| `e` | entry (TS) | `textobject_treesitter("entry")` |
| `x` | xml-element (TS) | `textobject_treesitter("xml-element")` |
| `g` | change (diff hunk) | `textobject_change` |
| 其他非字母数字字符 | 括号对 | `textobject_pair_surround(ch)` |

> **模式说明**：`ma`/`mi` 是"选择文本对象"命令，本身就是选区操作，与 Normal/Select 模式无关 — 无论在哪种模式下，它都会用对象范围替换当前选区。

### 5.4 Goto TS Object 跳转命令的光标定位（[commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L6132-L6165)）

`]f` / `[f` 等跳转命令通过 `goto_ts_object_impl` 实现。这是理解**前后光标定位**和 **Normal/Select 模式交互**的关键函数：

```rust
let selection = doc.selection(view.id).clone().transform(|range| {
    let new_range = movement::goto_treesitter_object(
        text, range, object, direction, syntax, &loader, count,
    );

    if editor.mode == Mode::Select {
        let head = if new_range.head < range.anchor {
            new_range.anchor   // 对象完全在原 anchor 之前：head 取对象起点
        } else {
            new_range.head     // 对象在原 anchor 之后或包含它：head 取对象终点
        };
        Range::new(range.anchor, head)
    } else {
        new_range.with_direction(direction)
    }
});
```

#### Normal 模式：光标定位详解

`goto_treesitter_object` 返回的是覆盖整个对象的 Forward Range（anchor=对象起点，head=对象终点）。Normal 模式下通过 `with_direction(direction)` 调整方向：

| 跳转方向 | with_direction 后 | head 位置 | 可见光标 |
|---------|-----------------|-----------|----------|
| Forward (`]f`) | Forward（不变） | 对象终点 | 对象的最后一个字符（head 向左一个字素） |
| Backward (`[f`) | Backward（翻转） | 对象起点 | 对象的第一个字符（head 向右一个字素） |

> **直观理解**：向前跳 → 光标落在对象的**远边（末尾）**；向后跳 → 光标落在对象的**远边（开头）**。光标始终在对象的外侧边缘上，朝向跳转的反方向。

#### Select 模式：选区扩展详解

Select 模式下保留原 anchor，只扩展 head。扩展的目标位置由对象相对于原 anchor 的位置决定：

- **对象完全在原 anchor 之前**（`new_range.head < range.anchor`）：head 取 `new_range.anchor`（对象起点），选区向左扩展到对象开头。
- **对象在原 anchor 之后或包含它**（`new_range.head >= range.anchor`）：head 取 `new_range.head`（对象终点），选区向右扩展到对象末尾。

> **设计逻辑**：方向隐含在 `goto_treesitter_object` 返回对象的位置中 — 向前跳转找到的对象在光标之后，向后跳转找到的对象在光标之前。Select 模式无需显式判断 direction，只需根据对象与原 anchor 的相对位置选择扩展到对象的哪一端。

### 5.5 单词移动的 Normal/Select 组合

单词移动采用"独立函数对"模式，是另一种命令组合风格：

**Normal 模式**（`move_word_impl`，[commands.rs:1195-1208](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L1195-L1208)）：
```rust
let selection = doc.selection(view.id).clone()
    .transform(|range| move_fn(text, range, count));
```
直接使用 `move_fn`（如 `move_next_word_start`）的返回值替换选区。`move_fn` 返回从起始位置到目标位置的完整范围。

**Select 模式**（`extend_word_impl`，[commands.rs:1588-1602](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L1588-L1602)）：
```rust
let selection = doc.selection(view.id).clone().transform(|range| {
    let word = extend_fn(text, range, count);
    let pos = word.cursor(text);
    range.put_cursor(text, pos, true)  // extend=true
});
```
先用 `extend_fn` 获取目标 word Range，取其 `cursor()` 位置，再用 `put_cursor(..., true)` 扩展原选区。

> **关键区别**：Normal 模式下单词移动的结果是一个**非零宽**的选区（从原位置到目标词的范围），而字符移动/`move_parent_node_end` 的 Move 模式结果是**零宽或 1 字素宽**的"光标选区"。这是 Helix 中不同运动类型的不一致之处。

### 5.6 Object 层选区操作（[object.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/object.rs)）

Tree-sitter 节点级别的选区操作，用于 `]p` / `[p`、expand/shrink 等：

| 函数 | 操作 |
|------|------|
| `expand_selection` | 选区扩展到当前节点的父节点 |
| `shrink_selection` | 选区收缩到当前节点的第一个子节点 |
| `select_next_sibling` | 选区移到下一个兄弟节点 |
| `select_prev_sibling` | 选区移到上一个兄弟节点 |
| `select_all_siblings` | 选中父节点下所有命名子节点 |
| `select_all_children` | 选中当前节点下所有命名子节点 |

这些操作直接操作 tree-sitter cursor，将节点的 byte 范围转为 char 范围构建 Range。

---

## 六、可见光标与命令组合的深层关系

### 6.1 三种"光标"概念的对照

在 Helix 代码中存在三个层次的"光标"概念，容易混淆：

| 概念 | 类型 | 含义 | 相关方法 |
|------|------|------|----------|
| head 位置 | `usize` | Range 的活动端点（间隙位置） | `range.head` |
| cursor 位置 | `usize` | block cursor 左边缘的位置（间隙位置） | `range.cursor(text)` |
| 可见方块光标 | 视觉概念 | 覆盖 1 个字素的方块 | 由 cursor 位置向右延伸 1 字素 |

三者的关系：
- Forward Range: `cursor = prev_grapheme_boundary(head)` → 光标贴在 head 内侧
- Backward Range: `cursor = head` → 光标贴在 head 内侧（向右延伸）
- 零宽 Range: `cursor = head` → 光标从 head 开始向右延伸

### 6.2 命令组合的四种模式

根据底层运动函数的设计和命令层的组合方式，Helix 的运动命令可分为四类：

| 类型 | 代表命令 | Move 模式结果 | Extend 模式实现 | 组合方式 |
|------|----------|--------------|----------------|----------|
| 字符级移动 | `h`/`l` | 零宽 Range | `put_cursor(..., true)` | `Movement` 参数 |
| 节点端点移动 | `]e`/`[e` | 1 字素宽 Range（保留方向） | anchor 不变，head 扩展 | `Movement` 参数 |
| 单词移动 | `w`/`b`/`e` | 非零宽 Range（从原位置到目标） | `word.cursor() + put_cursor(true)` | 独立函数对 |
| 对象跳转 | `]f`/`[f` | 覆盖整个对象的 Range（方向由跳转方向决定） | 根据对象相对位置选端扩展 | 模式检测分支 |

### 6.3 语法对象跳转的完整执行路径

以 `]f`（跳到下一个函数）为例，完整执行路径：

1. **keymap 查找**：Normal 模式下 `]f` → `goto_next_function`
2. **命令入口**：`goto_next_function` → `goto_ts_object_impl("function", Direction::Forward)`
3. **获取对象**：调用 `goto_treesitter_object` 找到下一个 function 节点，返回 `Range::new(start, end)`（Forward 方向，覆盖整个函数）
4. **模式处理**：
   - **Normal 模式**：`new_range.with_direction(Forward)` → 仍是 Forward Range，head 在函数末尾。可见光标落在函数的最后一个字符上。
   - **Select 模式**：原 anchor 保留，head 设为 `new_range.head`（函数末尾）。选区从原 anchor 扩展到函数末尾。
5. **应用与记录**：通过 `apply_motion` 执行，保存到 `last_motion` 供重复。

### 6.4 选区方向与光标的朝向

Range 的方向决定了可见光标的"朝向"，这对理解 `with_direction` 的作用很重要：

- **Forward Range**：光标在右端，朝左看（看向 anchor 方向）
- **Backward Range**：光标在左端，朝右看（看向 anchor 方向）

`goto_ts_object_impl` 中 `new_range.with_direction(direction)` 的作用是让光标朝向跳转的**反方向**：
- 向前跳 → Forward 方向 → 光标在右端朝左（回望来时路）
- 向后跳 → Backward 方向 → 光标在左端朝右（回望来时路）

这种设计与"选区的活动端在外侧"的直觉一致。

---

## 七、TextObject 选区 vs Motion 跳转的区别总结

| 维度 | TextObject 选区 (ma/mi) | Motion 跳转 (]f/[f) |
|------|------------------------|---------------------|
| 目标 | 当前光标所在的对象 | 下一个/上一个对象 |
| 输出 | 覆盖整个对象的 Range（Forward） | 覆盖整个对象的 Range（方向由跳转方向决定） |
| 方向 | 始终 Forward（由调用方决定是否调整） | Normal 模式下由 `with_direction(direction)` 决定 |
| Select 模式 | 不适用（本身就是选区替换） | 保留原 anchor，扩展 head 到对象的近端或远端 |
| count 语义 | 嵌套层级（括号）/ 段落数 | 跳过的对象个数 |
| 光标位置 | 对象内部（覆盖整个对象） | 对象边缘（Forward 在末尾，Backward 在开头） |

---

## 八、Char 分类体系（[chars.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/chars.rs#L6-L12)）

```
CharCategory::Whitespace   — 空白字符
CharCategory::Eol          — 换行符
CharCategory::Word         — 字母、数字、下划线
CharCategory::Punctuation  — 标点符号
CharCategory::Unknown      — 其他
```

- **word boundary**：任意两类字符的交界处
- **long word boundary**：排除 Word ↔ Punctuation 的交界处（标点与单词视为一体）
- **sub word boundary**：Word 内部的驼峰大小写变化处和下划线处
