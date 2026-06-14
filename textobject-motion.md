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
- **Block cursor 语义**：用户可见的光标是一个 1 字素宽的方块，位于 head 一侧。`cursor()` 方法返回此方块左边界位置：
  - Forward 时 `cursor = prev_grapheme_boundary(head)`
  - Backward 时 `cursor = head`

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
| `put_cursor(text, idx, extend)` | 将 block cursor 移到 idx；extend=true 保留 anchor 并扩展选区，extend=false 创建零宽 Range |
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

- **Around** 和 **Inside** 用于 `ma` / `mi` 选区命令。
- **Movement** 仅在 `goto_treesitter_object` 中使用，不用于 `select_textobject`。

---

## 三、TextObject 的四种实现

### 3.1 单词选区：textobject_word（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L71-L112)）

**流程**：
1. 取 cursor 位置。
2. 向后找词起始边界 `word_start`，向前找词结束边界 `word_end`。
3. `find_word_boundary` 根据 `long` 参数决定是否在字符类别变化时停下（`long=false` 时 Word 与 Punctuation 之间是边界；`long=true` 时只有空白/EOL 是边界）。
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
1. 判断当前行与前后行的空/非空状态（`prev_empty_to_line`、`curr_empty_to_line`）。
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
- `textobject_pair_surround_closest` — 自动查找最近的括号对（`ch=None`）

底层调用 `surround::find_nth_pairs_pos` / `find_nth_closest_pairs_pos`，返回 `(anchor, head)` 括号对的位置（无序）。

**Inside vs Around 的差异**（当 anchor < head 时）：

| 模式 | 范围 |
|------|------|
| Inside | `(next_grapheme(anchor), head)` — 跳过左括号，不含右括号 |
| Around | `(anchor, next_grapheme(head))` — 含左右括号 |

当 anchor > head 时对称处理。

`count` 参数控制向外跳几层嵌套括号。

### 3.4 Tree-sitter 语法对象选区：textobject_treesitter（[textobject.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/textobject.rs#L258-L293)）

**流程**：
1. 将 cursor 的 char 位置转为 byte 位置。
2. 通过 `syntax.layer_for_byte_range` 确定当前注入层（支持嵌入语言）。
3. 用 `loader.textobject_query` 获取当前语言的 textobject 查询。
4. 构造捕获名 `"{object_name}.{textobject}"`（如 `"function.inside"`、`"class.around"`）。
5. 从所有捕获节点中筛选包含当前 byte 位置的节点，选最小节点（`min_by_key(|n| node.byte_range().len())`）。
6. 将节点的 byte 范围转为 char 范围构建 Range。

---

## 四、Motion 系统

### 4.1 Direction 与 Movement 枚举（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L22-L32)）

```rust
pub enum Direction { Forward, Backward }  // 方向
pub enum Movement { Extend, Move }        // 行为：扩展选区 / 仅移动光标
```

- **Move**：光标移到新位置，丢弃原有选区（创建零宽或 1 字素宽 Range）。
- **Extend**：光标移到新位置，保留 anchor 不变，选区扩展。

### 4.2 水平移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L34-L53)）

```rust
pub fn move_horizontally(slice, range, dir, count, behaviour, ...) -> Range
```

计算新位置后调用 `range.put_cursor(slice, new_pos, behaviour == Movement::Extend)`。

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

1. **方向预处理**：根据移动方向调整起始 Range，解决 block cursor 1-宽度语义问题：
   - 向前移动且当前是 Forward 选区 → `Range::new(prev(head), head)`
   - 向前移动且当前是 Backward 选区 → `Range::new(head, next(head))`
   - 向后移动对称处理
2. **循环 count 次**：调用 `chars.range_to_target(target, range)` 计算每次移动。
3. **提前退出**：如果移动结果与上一次相同，说明已到边界。

**range_to_target 核心逻辑**（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L416-L490)）：

1. 若向后移动，反转迭代器。
2. 跳过起始的换行符，若经过换行则更新 anchor。
3. 遍历字符，调用 `reached_target` 判断是否到达目标边界。
4. 首次到达时如果 head 未移动，将 anchor 更新到 head 位置（表示"从一个边界开始移动"需要前进到下一个边界）。
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

`move_prev_paragraph` 和 `move_next_paragraph`：

- 向后移动：先跳空行，再跳非空行，到达段落起始。
- 向前移动：先跳非空行，再跳空行，到达段落结束。
- Move 模式：anchor 设为 cursor 位置（不保留原选区），head 设为段落边界。
- Extend 模式：anchor 取 `put_cursor(head, true).anchor`（保留扩展语义）。

### 4.6 Tree-sitter 对象跳转（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L562-L624)）

`goto_treesitter_object` 用于 `]f` / `[f` 等跳转命令：

1. 依次尝试匹配 `{name}.movement`、`{name}.around`、`{name}.inside` 三种捕获名（`capture_nodes_any`）。
2. Forward 方向：找 `start_byte > cursor_byte` 的最小节点（最近、最短）。
3. Backward 方向：找 `end_byte < cursor_byte` 的最大节点（最近、最短）。
4. 循环 count 次逐步跳转，直到无法继续。

### 4.7 父节点移动（[movement.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/movement.rs#L637-L699)）

`move_parent_node_end` 用于 `]e` / `[e` 跳到语法节点端点：

- Forward：移到当前节点的 end_byte 位置。
- Backward：移到当前节点的 start_byte；若已在 start 则向上到父节点。
- Move 模式：保留原 Range 方向，创建 1-宽度光标。
- Extend 模式：anchor 不变，head 移到节点端点。

---

## 五、命令层组合方式

### 5.1 apply_motion 机制（[editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-view/src/editor.rs#L1398-L1410)）

```rust
pub fn apply_motion<F: Fn(&mut Self) + 'static>(&mut self, motion: F) {
    motion(self);
    self.last_motion = Some(Box::new(motion));  // 保存用于重复
}
```

所有移动/选区命令都通过 `apply_motion` 执行，自动记录到 `last_motion` 供 `repeat_last_motion`（`.` 命令）使用。

### 5.2 TextObject 选区命令（[commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L6231-L6329)）

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

### 5.3 Goto TS Object 跳转命令（[commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-term/src/commands.rs#L6132-L6165)）

`]f` / `[f` 等跳转命令通过 `goto_ts_object_impl` 实现：

- 调用 `movement::goto_treesitter_object` 获取新 Range。
- **Normal 模式**：`new_range.with_direction(direction)` — 光标放在目标对象起始（Forward）或末尾（Backward）。
- **Select 模式**：`Range::new(range.anchor, head)` — 保留原 anchor，扩展到目标对象。

### 5.4 Object 层选区操作（[object.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/object.rs)）

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

## 六、选区范围与光标移动的交互规则总结

### TextObject 选区 vs Motion 跳转的区别

| 维度 | TextObject 选区 (ma/mi) | Motion 跳转 (]f/[f 等) |
|------|------------------------|------------------------|
| 目的 | 选中当前所在的对象 | 跳转到下一个/上一个对象 |
| 输出 | 一个覆盖整个对象的 Range | 一个光标位于目标对象起始/末尾的 Range |
| 保留方向 | 保留原选区方向 | Forward → head 在对象起始，Backward → head 在对象末尾 |
| Select 模式 | 不适用（已经是选区操作） | 保留 anchor，扩展 head 到目标 |
| count 语义 | 嵌套层级（括号）/ 段落数 | 跳过的对象个数 |

### Block Cursor 1-宽度语义

Helix 的光标是 1 字素宽的 block cursor，而非零宽插入点。这意味着：

- **Forward Range (anchor < head)**：head 指向 block cursor 右侧间隙，`cursor()` 返回 `prev_grapheme_boundary(head)`，即 block cursor 左边界。
- **Backward Range (head < anchor)**：head 本身就是 block cursor 左边界，`cursor()` 直接返回 head。
- **put_cursor 的 extend 逻辑**：当新位置跨过 anchor 时，anchor 会向内收缩一个字素（模拟 1-宽度 block cursor 的内侧边）。

### Char 分类体系（[chars.rs](file:///d:/fz/0601/solo-dogfeeding/code/278-helix/helix-core/src/chars.rs#L6-L12)）

```
CharCategory::Whitespace   — 空白字符
CharCategory::Eol          — 换行符
CharCategory::Word         — 字母、数字、下划线
CharCategory::Punctuation  — 标点符号
CharCategory::Unknown      — 其他
```

Word 和 Punctuation 之间的边界是普通 word boundary；Long word 仅以 Whitespace/EOL 为边界；Sub word 还在 camelCase 和 snake_case 处分割。
