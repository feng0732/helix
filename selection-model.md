# Helix Selection 模型深度分析

## 一、核心数据结构

### 1.1 Range —— 单个选择范围

[selection.rs:54-63](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L54-L63)

```rust
pub struct Range {
    pub anchor: usize,       // 锚点：扩展时不移动的一端
    pub head: usize,         // 头部：扩展时移动的一端
    pub old_visual_position: Option<(u32, u32)>, // 软换行时的视觉偏移
}
```

**关键设计要点：**

- **间隙索引（Gap Indexing）**：所有位置使用 `char` 偏移量，表示字符之间的间隙。例如位置 `1` 表示第 1 和第 2 个字符之间。
- **方向无关性**：`anchor` 和 `head` 无固定大小关系。`head < anchor` 表示反向选择。
- **不变量**：左闭右开区间 `[from, to)`，其中 `from = min(anchor, head)`，`to = max(anchor, head)`。
- **块光标语义**：用户看到的块光标位于 `head` 侧，向内覆盖一个 grapheme。

**Range 核心方法：**

| 方法 | 作用 | 位置 |
|------|------|------|
| `from()/to()` | 获取规范化的起止位置（无视方向） | [selection.rs:86-96](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L86-L96) |
| `direction()` | 返回 `Forward`/`Backward` | [selection.rs:128-135](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L128-L135) |
| `cursor(text)` | 块光标所在字符位置 | [selection.rs:333-341](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L333-L341) |
| `map(changes)` | 通过 ChangeSet 映射位置（用于编辑后更新） | [selection.rs:180-203](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L180-L203) |
| `grapheme_aligned()` | 对齐到 grapheme 边界 | [selection.rs:276-301](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L276-L301) |

### 1.2 Selection —— 选择集（多光标容器）

[selection.rs:416-420](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L416-L420)

```rust
pub struct Selection {
    ranges: SmallVec<[Range; 1]>,  // 内联存储至少 1 个 Range
    primary_index: usize,          // 主光标索引
}
```

**核心不变量：**
1. 永远非空（至少含一个 Range）
2. 所有 Range 按 `from()` 排序
3. 相邻/重叠 Range 会被自动合并（`normalize()`）
4. 所有 Range 对齐到 grapheme 边界

**Selection 核心方法：**

| 方法 | 作用 | 位置 |
|------|------|------|
| `map(changes)` | 通过 ChangeSet 更新所有 Range 位置并重新归一化 | [selection.rs:479-481](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L479-L481) |
| `transform(F)` | 对每个 Range 应用闭包后归一化 | [selection.rs:636-644](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L636-L644) |
| `ensure_invariants(text)` | 确保 grapheme 对齐、最小宽度、不重叠、排序 | [selection.rs:662-665](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L662-L665) |
| `fragments(text)` | 遍历所有 Range 对应的文本片段 | [selection.rs:673-679](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L673-L679) |

### 1.3 Document 中的 Selection 存储

[document.rs:141-145](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L141-L145)

```rust
pub struct Document {
    selections: HashMap<ViewId, Selection>, // 每个视图独立的选择集
    // ...
}
```

**设计特点：** 一个文档可被多个视图打开，每个视图维护独立的选择集。这使得分屏编辑时两个视图可以有不同的光标位置。

---

## 二、位置粘附策略 Assoc 详解

这是理解选择集位置映射的核心。

### 2.1 Assoc 的 6 种策略

[transaction.rs:32-48](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L32-L48)

```rust
pub enum Assoc {
    Before,
    After,
    AfterWord,
    BeforeWord,
    BeforeSticky,
    AfterSticky,
}
```

**基础语义（间隙索引视角）：**

位置 `p` 是第 `p` 个字符之前的间隙。一个位置有"前侧"（第 `p-1` 个字符之后）和"后侧"（第 `p` 个字符之前）。

- **`Before`**：位置粘附在**前侧**的字符上。前侧字符移动时，位置跟着移动。
- **`After`**：位置粘附在**后侧**的字符上。后侧字符移动时，位置跟着移动。
- **`BeforeSticky` / `AfterSticky`**：粘性版本，在**等长替换**时行为不同（见下文）。
- **`BeforeWord` / `AfterWord`**：词边界粘附，主要用于 diagnostics。

### 2.2 纯插入（Insert）时的位置处理机制

纯插入的核心代码在 [transaction.rs:491-499](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L491-L499)：

```rust
} else {
    // at insert point
    map!(
        |pos, assoc: Assoc| (old_pos == pos).then(|| {
            new_pos + assoc.insert_offset(s)
        }),
        i
    );
}
```

**触发条件**：Insert 分支只处理 `old_pos == pos`，即位置**精确等于**插入点的位置才会在这里被特殊映射。其他位置（`pos < old_pos` 或 `pos > old_pos`）由前后的 Retain 分支处理，此时 Assoc 不参与计算（Assoc 被忽略）。

**insert_offset 的实现**（[transaction.rs:56-65](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L56-L65)）：

```rust
fn insert_offset(self, s: &str) -> usize {
    let chars = s.chars().count();
    match self {
        Assoc::After | Assoc::AfterSticky => chars,
        Assoc::AfterWord => s.chars().take_while(|&c| char_is_word(c)).count(),
        Assoc::Before | Assoc::BeforeSticky => 0,
        Assoc::BeforeWord => chars - s.chars().rev().take_while(|&c| char_is_word(c)).count(),
    }
}
```

**落在插入点上的位置映射**：

| 策略 | 插入后新位置 | 直观理解 |
|------|-------------|----------|
| `Before` / `BeforeSticky` | `new_pos + 0 = new_pos` | 停在插入文本之前 |
| `After` / `AfterSticky` | `new_pos + s.len()` | 跳到插入文本之后 |
| `BeforeWord` / `AfterWord` | 停在词边界处 | 用于 diagnostics |

**不在插入点上的位置**：
- `pos < old_pos`：由前面的 Retain 处理，位置不变
- `pos > old_pos`：由后面的 Retain 处理，整体后移插入文本的长度（`s.len()`）

### 2.3 纯删除时的行为

[transaction.rs:462-464](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L462-L464)

```rust
Delete(_) => {
    map!(|pos, _| (old_end > pos).then_some(new_pos), i);
}
```

**纯删除场景**：所有落在删除范围内的位置，**无论 Assoc 是什么**，都坍缩到删除起始位置 `new_pos`。

原因：删除后，删除范围内的所有"字符"都消失了，位置只能落在删除后留下的单个间隙上。

### 2.4 替换（Insert + Delete）时的行为

[transaction.rs:466-503](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L466-L503)

在 ChangeSet 中，"替换"由 `Insert(new_text)` + `Delete(old_len)` 两个操作连续表示（先构造 Insert，再构造 Delete）。位置映射时将两者合并处理。

替换区域：旧文档 `[old_pos, old_pos + old_len)` → 新文档 `s`（长度 `s_len`）。

对于落在替换区域内的位置 `pos`（`old_pos <= pos < old_pos + old_len`）：

```
if pos == old_pos 且 assoc.stay_at_gaps():
    → new_pos （替换起始位置，不走 sticky 逻辑）
else:
    ins = assoc.insert_offset(s)
    if old_len == ins 且 assoc.sticky():
        → new_pos + (pos - old_pos)  // 保持相对偏移（sticky 行为）
    else:
        → new_pos + ins               // 跳到插入偏移位置
```

#### stay_at_gaps 在替换起始边界的作用

`stay_at_gaps()` 定义在 [transaction.rs:52-54](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L52-L54)：

```rust
fn stay_at_gaps(self) -> bool {
    !matches!(self, Self::BeforeWord | Self::AfterWord)
}
```

- 对 Selection 映射使用的 `AfterSticky` 和 `BeforeSticky`，`stay_at_gaps()` 都返回 `true`
- 结果：**落在替换起始点（pos == old_pos）的所有位置，无论 Assoc 是什么（只要不是 Word 类），都固定映射为 new_pos**

这个规则的意义：确保替换区域的起始边界永远不会被"吞入"替换区域内部，即起点永远粘在替换的最开头间隙上。

#### AfterSticky 的替换行为

`AfterSticky` 的 `insert_offset` = `s.len()`（插入文本总长度），因此：

- **落在替换起始点（pos == old_pos）**：stay_at_gaps 为 true → 固定为 new_pos
- **落在替换区域内部（pos > old_pos）**：
  - **等长替换**（`old_len == s.len()`）：`old_len == ins` 条件成立 → **保持相对偏移**（sticky）
  - **不等长替换**：条件不成立 → 跳到**替换区域末尾**（`new_pos + s.len()`）

直观理解：AfterSticky 粘在替换区域后边界上，起点被固定在起始位置，等长时内部位置"跟随"文本保持相对位置，不等长时内部位置被"吸"到末尾。

#### BeforeSticky 的替换行为

`BeforeSticky` 的 `insert_offset` = `0`，因此：

- **落在替换起始点（pos == old_pos）**：stay_at_gaps 为 true → 固定为 new_pos
- **落在替换区域内部（pos > old_pos）**：
  - 条件 `old_len == ins` 即 `old_len == 0`（删除长度为 0）
  - 但 `delete(0)` 会被优化掉，不会产生 Delete 操作，也就不会进入替换分支
  - **所以在实际的替换场景中，BeforeSticky 的 sticky 行为从不触发**
  - 结果：所有落在替换区域内部（pos > old_pos）的 BeforeSticky 位置，都跳到**替换区域开头**（`new_pos + 0`）

> **注意**：这是代码中的不对称性。`AfterSticky` 的 sticky 在等长替换时对内部位置生效，`BeforeSticky` 的 sticky 在替换场景中永不生效。两者的注释描述是对称的，但实际实现不对称。原因是 sticky 触发条件要求 `old_len == ins`，而 BeforeSticky 的 `ins = 0`，只有 `old_len = 0` 才满足，这在实际替换中不可能发生。

### 2.5 Selection 映射的 Assoc 选择

[selection.rs:490-506](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L490-L506)

对于每个 Range：

```
anchor < head （正向选择）:
    anchor = from   → AfterSticky  （范围起点）
    head   = to     → BeforeSticky （范围终点）

anchor > head （反向选择）:
    head   = from   → AfterSticky  （范围起点）
    anchor = to     → BeforeSticky （范围终点）

anchor == head （零宽光标）:
    两者都是 → AfterSticky
```

**规律：范围的起始端（from 侧）用 AfterSticky，范围的结束端（to 侧）用 BeforeSticky。**

### 2.6 插入边界的选择集漂移：完整场景推导

以下所有场景以一个正向选择 `[2, 5)` 为例（from=2 → AfterSticky，to=5 → BeforeSticky），插入文本 "ABC"（长度 3）。

推导原则：
1. 端点精确等于插入点 → 进入 Insert 分支，按 insert_offset 映射
2. 端点在插入点之前 → 由前面的 Retain 处理，位置不变
3. 端点在插入点之后 → 由后面的 Retain 处理，后移插入长度（+3）

#### 场景 A：在选择起点（插入点 = 2 = from）插入

- **from=2**：精确等于插入点 → Insert 分支，AfterSticky 的 insert_offset = 3 → 新位置 = 2 + 3 = 5
- **to=5**：在插入点之后（5 > 2）→ 后续 Retain 处理，后移 3 → 新位置 = 5 + 3 = 8
- **新选择**：`[5, 8)`
- **新文本位置**：插入的 "ABC" 在新文档位置 2..5
- **是否留在选区内**：**不包含**。新选择从 5 开始，"ABC" 在 2..5，在选区之前。
- **直观效果**：在选区开头前插入文字，选区整体后移，不包含新文本。

#### 场景 B：在选择终点（插入点 = 5 = to）插入

- **from=2**：在插入点之前（2 < 5）→ 前面的 Retain 处理，不变 → 2
- **to=5**：精确等于插入点 → Insert 分支，BeforeSticky 的 insert_offset = 0 → 新位置 = 5 + 0 = 5
- **新选择**：`[2, 5)`
- **新文本位置**：插入的 "ABC" 在新文档位置 5..8
- **是否留在选区内**：**不包含**。新选择到 5 结束，"ABC" 在 5..8，在选区之后。
- **直观效果**：在选区末尾后插入文字，选区保持不变，不包含新文本。

#### 场景 C：在选区内部（插入点 = 3，from < 3 < to）插入

- **from=2**：在插入点之前（2 < 3）→ 前面的 Retain 处理，不变 → 2
- **to=5**：在插入点之后（5 > 3）→ 后续 Retain 处理，后移 3 → 新位置 = 5 + 3 = 8
- **新选择**：`[2, 8)`
- **新文本位置**：插入的 "ABC" 在新文档位置 3..6
- **是否留在选区内**：**完全包含**。新选择 [2, 8) 覆盖 [3, 6)。
- **直观效果**：在选区中间插入文字，选区自动扩大以包住新文本。

#### 场景 D：在选择之前（插入点 = 0，< from）插入

- from 和 to 都 > 0，都由后续 Retain 处理，整体后移 3。
- **新选择**：`[5, 8)`。选区整体后移，新文本不进入选区。

#### 场景 E：在选择之后（插入点 = 6，> to）插入

- from 和 to 都 < 6，都由前面的 Retain 处理，不变。
- **新选择**：`[2, 5)`。选区完全不变，新文本不进入选区。

#### 场景 F：零宽光标（选择 = [3, 3)）在光标处插入

- anchor 和 head 都 = 3，都是 AfterSticky。
- 插入点 = 3，精确等于光标位置。
- 两者都按 AfterSticky 映射：insert_offset = 3 → 新位置 = 3 + 3 = 6
- **新光标**：`[6, 6)`，位于插入文本之后。
- **直观效果**：输入字符后光标跳到新字符之后，符合用户预期。

#### 三种边界场景的总结表

| 插入位置 | 新选择区间 | "ABC"的位置 | 是否在选区内 |
|----------|-----------|------------|-------------|
| 起点 from（2） | `[5, 8)` | 2..5 | ❌ 否 |
| 终点 to（5） | `[2, 5)` | 5..8 | ❌ 否 |
| 选区内部（3） | `[2, 8)` | 3..6 | ✅ 是 |

### 2.7 替换与删除的漂移示例

**场景 1：删除位置 [1, 6)（选择完全在删除区内）**
- from=2 和 to=5 都在删除范围内 → 都坍缩到 1
- 新选择：`[1, 1)`（零宽光标落在删除起始处）

**场景 2：替换 [3, 6) 为 "XY"（2 字符，不等长），选择 [2, 5) 跨替换边界**
- from=2（AfterSticky）：在替换区外（前），pos < old_pos=3 → 不变 → 2
- to=5（BeforeSticky）：在替换区内（pos=5 在 [3,6) 内且 > old_pos=3），sticky 不生效 → 跳到替换开头 → 3
- 新选择：`[2, 3)`（终点被"吸"到替换起始处）

**场景 3：替换 [1, 6) 为 "abcde"（5 字符，等长），选择 [2, 4) 完全在替换内**
- from=2（AfterSticky）：pos > old_pos=1，等长替换（5==5，ins=5）→ 保持偏移 → 1 + (2-1) = 2
- to=4（BeforeSticky）：pos > old_pos=1，在替换区内，sticky 不生效 → 跳到开头 → 1
- 新选择：`[2, 1)`（即反向选择 `[1, 2)`）
- 不对称性：等长替换时起点保持偏移，但终点跳到开头。

**场景 4：替换 [1, 6) 为 "abc"（3 字符，变短），选择 [2, 4) 完全在替换内**
- from=2（AfterSticky）：pos > old_pos=1，不等长替换 → 跳到末尾 → 1 + 3 = 4
- to=4（BeforeSticky）：在替换区内，sticky 不生效 → 跳到开头 → 1
- 新选择：`[4, 1)`（即反向选择 `[1, 4)`，覆盖整个替换区域）

---

## 三、选择集如何驱动文本变更

### 3.1 变更的底层表示：ChangeSet 与 Operation

[transaction.rs:12-20](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L12-L20)

```rust
pub enum Operation {
    Retain(usize),   // 保留 n 个字符
    Delete(usize),   // 删除 n 个字符
    Insert(Tendril), // 插入文本
}
```

`ChangeSet` 是 `Operation` 的序列，描述从文档 A 到文档 B 的完整变换。它采用类似 OT（Operational Transformation）的线性表示方式。

**ChangeSet 的构造**：替换操作按 `Insert` → `Delete` 顺序构造。

[transaction.rs:556-558](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L556-L558)

```rust
Some(text) => {
    changeset.insert(text);
    changeset.delete(span);
}
```

### 3.2 Transaction —— 可撤销的变更单元

[transaction.rs:573-577](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L573-L577)

```rust
pub struct Transaction {
    changes: ChangeSet,
    selection: Option<Selection>, // 可选：显式指定编辑后的选择
}
```

`Transaction` = `ChangeSet` + 可选的 Selection 覆盖。如果提供了 `selection`，编辑后直接使用该选择；否则通过 `ChangeSet.map` 自动推导。

### 3.3 面向选择集的 Transaction 构造器

这是选择集驱动文本变更的核心 API。

#### change_by_selection：逐范围生成变更

[transaction.rs:706-711](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L706-L711)

```rust
pub fn change_by_selection<F>(doc: &Rope, selection: &Selection, f: F) -> Self
where
    F: FnMut(&Range) -> Change,
```

**工作原理**：对 Selection 中的每个 Range 调用 `f`，生成一个 `(from, to, replacement)` 三元组，再将这些变更按序合成为 `ChangeSet`。因为 Range 已排序且不重叠，合成时不会产生偏移错乱。

**典型用例：大小写转换**

[commands.rs:1855-1869](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L1855-L1869)

```rust
fn switch_case_impl<F>(cx: &mut Context, change_fn: F) {
    let selection = doc.selection(view.id);
    let transaction = Transaction::change_by_selection(doc.text(), selection, |range| {
        let text: Tendril = change_fn(range.slice(doc.text().slice(..)));
        (range.from(), range.to(), Some(text))  // 用转换后的文本替换
    });
    doc.apply(&transaction, view.id);
}
```

#### change_by_and_with_selection：同时生成变更与新选择

[transaction.rs:756-800](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L756-L800)

与 `change_by_selection` 类似，但闭包可返回 `Option<Range>` 来显式指定每个范围编辑后的新位置。适用于编辑后光标位置不能通过 map 推导的场景（如插入后光标要跳到文本末尾，或自动配对时需要精确定位）。

**典型用例：插入字符**

[commands.rs:4315-4344](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L4315-L4344)

```rust
pub fn insert_char(cx: &mut Context, c: char) {
    let transaction = Transaction::change_by_and_with_selection(text, selection, |range| {
        let cursor = range.cursor(text.slice(..));
        let t = Tendril::from_iter([c]);
        ((cursor, cursor, Some(t)), None)  // None → 自动推导位置
        // 自动配对时返回 Some(new_range) 来精确定位光标
    });
    doc.apply(&transaction, view.id);
}
```

#### delete_by_selection：删除（支持重叠合并）

[transaction.rs:667-698](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L667-L698)

删除操作特殊处理：重叠的删除范围会被合并，避免重复删除导致偏移错乱。

**典型用例：删除选区**

[commands.rs:3002-3005](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L3002-L3005)

```rust
let transaction =
    Transaction::delete_by_selection(doc.text(), selection, |range| (range.from(), range.to()));
```

#### insert：在每个 head 处插入文本

[transaction.rs:866-870](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L866-L870)

```rust
pub fn insert(doc: &Rope, selection: &Selection, text: Tendril) -> Self {
    Self::change_by_selection(doc, selection, |range| {
        (range.head, range.head, Some(text.clone()))
    })
}
```

### 3.4 Document::apply —— 事务应用的完整流程

[document.rs:1435-1626](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1435-L1626)

调用链：`Document::apply` → `apply_inner` → `apply_impl`

**apply_impl 中 Selection 相关的执行步骤：**

```
1. ChangeSet.apply(&mut self.text)     // 实际修改 Rope 文本

2. 若 changes 为空（纯光标移动类事务）:
     若 transaction 带 selection → 直接设置，派发 SelectionDidChange
     返回

3. 若 changes 非空:
   a. 对所有视图的 Selection 调用 .map(transaction.changes())
      → 自动推导每个视图的新光标位置
   b. 更新 view_position（滚动锚点，Assoc::Before）
   c. 更新 savepoint 回滚事务
   d. 更新 tree-sitter 语法树
   e. 更新 diagnostics 位置（起点 After/AfterWord，终点 Before/BeforeWord）
   f. 更新 inlay hints 位置（Assoc::After）
   g. 更新 document highlights 位置（两端都是 After）
   h. 派发 DocumentDidChange 事件

4. 若 transaction 显式指定了 selection：
     覆盖当前视图的 selection（优先级高于自动推导）
     派发 SelectionDidChange 事件
```

**关键机制**：`Selection::map(ChangeSet)` 内部调用 `ChangeSet::update_positions`，后者通过单次线性扫描 `ChangeSet`，批量映射所有 Range 的 anchor 和 head 位置，时间复杂度 O(N+M)（N 为操作数，M 为位置数）。见 [transaction.rs:388-510](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L388-L510)。

### 3.5 自动推导 vs 显式指定

| 方式 | 时机 | 适用场景 |
|------|------|----------|
| 自动推导（Selection::map） | 每次编辑，对所有视图执行 | 普通编辑操作，选择随文本自然漂移 |
| 显式指定（Transaction::selection） | 编辑后覆盖当前视图 | 光标跳转、模式切换、撤销/重做等需要精确定位的场景 |

**设计权衡**：自动推导保证了所有视图的选择都能正确跟随文本变化，但无法表达语义级别的光标移动（如"删除后光标移到行首"）。显式指定提供了精确控制，但只影响当前视图。

### 3.6 插入边界规则对文本变更的语义影响

插入边界的选择集漂移规则直接决定了编辑器的交互语义，以下是典型场景的行为对应：

#### 1. 选区边界粘贴不污染选区

在选择起点或终点粘贴时，插入的文本落在选区之外：
- **在起点前粘贴**：选区整体后移，粘贴文本在选区之前（场景 A）
- **在终点后粘贴**：选区保持不变，粘贴文本在选区之后（场景 B）

符合用户预期：粘贴操作不会无意中扩大选区范围。后续对选区执行「删除」或「变换」操作时，粘贴的内容不会被误操作。

#### 2. 选区内部输入自动扩展选区

在选区中间（非端点位置）插入字符时，新文本进入选区：
- 例如：选中 "hello"，在 "ll" 中间插入 "xx"，新选择变为 "hellxxo"

这种行为使得：
- 用户可以在选中的文本中间继续输入，选区自然跟随扩展
- 多光标输入时，每个光标处插入的字符都留在对应光标所在的选区内
- 自动缩进、括号补平等编辑操作产生的插入也会自然留在选区内

#### 3. 零宽光标处的输入语义

光标（零宽选择）在插入点处输入：
- AfterSticky 保证光标跳到新字符之后
- 连续输入字符时光标始终在最新字符之后，符合阅读顺序

#### 4. 多编辑操作的顺序敏感性

因为 change_by_selection 构造 ChangeSet 时按 Range 排序（from 升序），当多个 Range 的编辑操作相互影响时，靠前的插入会改变靠后范围的位置偏移。

但由于 Selection 的规范化（不重叠、排序），逐 Range 编辑不会交叉干扰，合成的 ChangeSet 天然正确。

#### 5. 替换操作的选择行为

当编辑操作本身是由选择集驱动的「选区替换」时（如 change_by_selection 产生的 Replace），插入点正好是每个 Range 的 from 位置，即「选择起点边界插入」场景。根据场景 A，新文本不进入选区——但实际上替换完成后，选区通常需要被精确设置为新文本的范围。

这就是为什么：
- **纯 map 推导不够用**：如果仅依赖自动推导，替换后选区可能坍缩或跳过新文本
- **需要 change_by_and_with_selection**：显式返回新 Range，指定编辑后光标位置
- **或使用 with_selection**：Transaction 携带最终选择覆盖自动推导结果

---

## 四、选择集与撤销/重做机制

### 4.1 History —— 修订树

[history.rs:50-66](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L50-L66)

```rust
pub struct History {
    revisions: Vec<Revision>,
    current: usize,
}

struct Revision {
    parent: usize,
    last_child: Option<NonZeroUsize>, // 用于 redo 分支
    transaction: Transaction,         // 正向：parent → self
    inversion: Transaction,           // 反向：self → parent
    timestamp: Instant,
}
```

**设计**：每个修订同时存储正向和反向事务。

### 4.2 State —— （文本 + 选择集）快照

[history.rs:7-11](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L7-L11)

```rust
pub struct State {
    pub doc: Rope,
    pub selection: Selection,
}
```

### 4.3 提交修订：Selection 的保存

[history.rs:89-110](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L89-L110)

```rust
pub fn commit_revision_at_timestamp(
    &mut self, transaction: &Transaction, original: &State, timestamp: Instant
) {
    let inversion = transaction
        .invert(&original.doc)
        .with_selection(original.selection.clone());  // ★ 关键：保存旧选择
    // ...
}
```

**关键点**：
- 文本还原：通过 `ChangeSet::invert()` 生成（Delete ↔ Insert 互换）
- 选择还原：**直接保存编辑前的 Selection**，而不是通过反向 ChangeSet 推导

**为什么不用反向 map 推导？**
因为 Assoc 策略的映射不是完美可逆的。例如：
- 选择完全落在替换区域内时，两端会坍缩到替换边界
- 纯删除时，范围内所有位置都坍缩到同一点
- 这些信息丢失的操作无法通过反向 map 精确恢复

保存原始 Selection 确保了撤销后选择状态与编辑前完全一致（多光标、方向、所有细节）。

### 4.4 Document 层的累积与提交

Document 并不为每次按键都创建修订。相反，它将连续的编辑操作累积在 `self.changes` 中，在合适的时机（退出插入模式、切换文档、超时等）才一次性提交到 History。

**apply_inner：累积变更**

[document.rs:1628-1652](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1628-L1652)

```rust
fn apply_inner(&mut self, transaction: &Transaction, view_id: ViewId, ...) -> bool {
    // 首次变更时记录"编辑前状态"
    if self.changes.is_empty() && !transaction.changes().is_empty() {
        self.old_state = Some(State {
            doc: self.text.clone(),
            selection: self.selection(view_id).clone(),  // 记录此刻的选择
        });
    }

    self.apply_impl(...);

    if !transaction.changes().is_empty() {
        // 组合（compose）到累积的 changeset 中
        take_with(&mut self.changes, |changes| {
            changes.compose(transaction.changes().clone())
        });
    }
}
```

**append_changes_to_history：提交到撤销树**

[document.rs:1787-1808](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1787-L1808)

```rust
pub fn append_changes_to_history(&mut self, view: &mut View) {
    if self.changes.is_empty() { return; }

    let changes = std::mem::replace(&mut self.changes, new_changeset);
    let transaction =
        Transaction::from(changes).with_selection(self.selection(view.id).clone());
        // ★ 正向事务携带"编辑后"的选择

    let old_state = self.old_state.take().expect("no old_state available");

    let mut history = self.history.take();
    history.commit_revision(&transaction, &old_state);  // old_state 含编辑前选择
    self.history.set(history);
}
```

**提交时两个方向的 Selection：**
- 正向事务（`transaction`）：携带**编辑后**的 Selection → 用于 redo
- 反向事务（`inversion`）：携带**编辑前**的 Selection → 用于 undo

### 4.5 Undo/Redo 的完整流程

**Undo**（[document.rs:1665-1687](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1665-L1687)）：
```
1. append_changes_to_history()   // 先把当前累积的变更提交掉
2. history.undo() → 返回 revision.inversion（含旧文本+旧选择）
3. apply_impl(txn, view.id)      // 应用反向事务
   a. 文本被还原
   b. 所有视图的 selection 先通过 map 自动推导
   c. 但 inversion 显式携带了旧 selection，覆盖当前视图
   d. 结果：当前视图的选择精确回到编辑前状态
```

**Redo**：
```
1. 检查 self.changes 是否为空（有未提交的变更则拒绝 redo）
2. history.redo() → 返回 revision.transaction（含新文本+新选择）
3. apply_impl(txn, view.id)      // 应用正向事务
   a. 文本被更新
   b. 所有视图的 selection 先通过 map 自动推导
   c. 正向事务显式携带了 selection，覆盖当前视图
   d. 结果：当前视图的选择精确回到编辑后状态
```

> **注意**：undo/redo 只影响**当前视图**的 selection（通过显式指定覆盖）。其他视图的 selection 仍通过 map 自动推导，可能产生漂移。这是因为 State 只存了当前视图的 selection。

### 4.6 边界漂移规则对撤销记录的影响

#### 1. 为什么撤销不使用反向 map 推导

这是插入边界规则直接决定的架构设计。以下场景展示了反向 map 的不可逆性：

**不可逆场景 A：选区内部插入**
- 编辑前选择 `[2, 5)`，在位置 3 插入 "ABC"（选区内部）
- 编辑后选择变为 `[2, 8)`（扩大，包含新文本）
- 文本层面反向操作是删除位置 `[3, 6)` 的 "ABC"
- 若用反向 map：from=2（<3）不变，to=8（>6）后移 -3 → 5 → 得到 `[2, 5)` ✓

**不可逆场景 B：选择起点插入**
- 编辑前选择 `[2, 5)`，在位置 2（起点）插入 "ABC"
- 编辑后选择变为 `[5, 8)`（整体后移，不包含新文本）
- 文本层面反向操作是删除位置 `[2, 5)` 的 "ABC"
- 若用反向 map：from=5（在 `[2,5)` 删除区内）→ 坍缩到 2；to=8（>5）后移 -3 → 5 → 得到 `[2, 5)` ✓

**不可逆场景 C：完全落入删除区**
- 编辑前选择 `[2, 5)`，删除 `[1, 8)`（选择完全在删除区内）
- 编辑后选择变为 `[1, 1)`（坍缩为零宽光标）
- 文本层面反向操作是在位置 1 插入原 7 个字符
- 若用反向 map：anchor=head=1，AfterSticky → 都跳到 1 + 7 = 8 → 得到 `[8, 8)` ✗
- **错误结果**：原始选择应该是 `[2, 5)`，但反向 map 只能得到光标位置

**不可逆场景 D：完全落入替换区**
- 编辑前选择 `[2, 4)`，替换 `[1, 6)` 为 "abc"（3 字符）
- 编辑后 from=4, to=1 → 归一化为反向选择 `[1, 4)`
- 反向操作是替换 `[1, 4)` 为原 5 个字符
- 若用反向 map：from=1（替换起始点，stay_at_gaps=true）→ 固定为 1；to=4（替换区内，BeforeSticky）→ 跳到开头 → 1 → 得到 `[1, 1)` ✗
- **错误结果**：原始选择是 `[2, 4)`，但反向 map 得到 `[1, 1)`

**结论**：删除坍缩、替换坍缩、不等长替换的不对称漂移，都会导致反向 map 无法恢复原始选择。这就是撤销记录必须**直接保存原始 Selection 快照**的根本原因。

#### 2. 累积变更过程中的选择漂移

在插入模式下，用户连续按键产生多次编辑，这些编辑被 compose 到一个累积的 ChangeSet 中：

**每次按键的 apply_impl 都会更新选择：**
```
第 1 次插入（位置 2 输入 'a'）：
  选择 [2, 2) → AfterSticky → 跳到 3 → 新选择 [3, 3)

第 2 次插入（位置 3 输入 'b'）：
  选择 [3, 3) → AfterSticky → 跳到 4 → 新选择 [4, 4)

第 N 次插入：
  选择持续后移...
```

**对撤销记录的影响：**
- `old_state.selection` 只记录**第一次变更前**的选择（如插入模式进入时的光标位置）
- 最终提交到撤销树的 `inversion` 携带的是这个**首次变更前**的选择
- 正向事务携带的是提交时的选择（经过所有 map 漂移后的最终光标）
- 这样撤销时一次性回到插入前的位置，redo 时跳到插入结束的位置，符合用户直觉

#### 3. 多视图下撤销记录的选择不一致性

撤销时：
- 当前视图的 Selection 由 `with_selection(original.selection)` 显式覆盖 → **精确还原**
- 其他视图的 Selection 仅通过 `.map(inversion.changes())` 自动推导 → **可能漂移**

漂移示例：
- 视图 A（当前）：选择 `[2, 4)` 被完全删除
- 视图 B（另一个分屏）：同样的选择 `[2, 4)`
- 撤销删除后，视图 A 的选择精确还原为 `[2, 4)`，但视图 B 通过反向 map 可能得到不同结果（如坍缩到 `[2, 2)`）

这是多视图架构的设计权衡：State 结构只存一个 Selection，属于「当前视图」。

---

## 五、选择集如何驱动命令行为

### 5.1 命令的标准模式

绝大多数编辑命令遵循以下模式：

```
1. 获取当前视图的 Selection：
   let (view, doc) = current!(cx.editor);
   let selection = doc.selection(view.id);

2. 基于 Selection 创建 Transaction：
   let transaction = Transaction::xxx_by_selection(
       doc.text(), selection, |range| { ... }
   );

3. 应用事务：
   doc.apply(&transaction, view.id);

4. （可选）切换模式、派发事件等
```

### 5.2 命令分类与 Selection 的使用

#### A. 纯 Selection 变换类（不修改文本）

这类命令通过 `Selection::transform` 或直接构造新 Selection，调用 `set_selection`。不产生撤销记录。

**示例：翻转选区方向** [commands.rs:3094-3102](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L3094-L3102)
```rust
fn flip_selections(cx: &mut Context) {
    let selection = doc.selection(view.id).clone()
        .transform(|range| range.flip());
    doc.set_selection(view.id, selection);
}
```

**示例：进入插入模式（调整光标位置）** [commands.rs:3120-3136](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L3120-L3136)
```rust
fn insert_mode(cx: &mut Context) {
    enter_insert_mode(cx);
    let selection = doc.selection(view.id).clone()
        .transform(|range| Range::new(range.to(), range.from()));
    doc.set_selection(view.id, selection);
}
```

#### B. 删除类命令

**delete_selection_impl** [commands.rs:2983-3020](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L2983-L3020)：

```
对于 selection 中的每个 range：
  1. 若需要 yank：收集 range.fragment(text) 到寄存器
  2. Transaction::delete_by_selection(...) → (range.from, range.to)
  3. doc.apply(...)
  4. 根据操作类型切换模式（Delete→Normal，Change→Insert）
```

**插入模式下的删除** `delete_by_selection_insert_mode` [commands.rs:3023-3065](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L3023-L3065)：

与普通删除的区别：
- 对 forward 删除，先将 `range.head += 1`（避免视觉上文字向左移动）
- 删除文档末尾时自动插入换行符
- 手动调整 selection，再 apply transaction

#### C. 插入类命令

**insert_char**（单字符插入，见 [commands.rs:4315-4344](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L4315-L4344)）：

核心逻辑：对每个 range，在 `range.cursor()` 位置（块光标位置）插入字符。若启用了自动配对，则返回自定义的新 range 来精确定位光标。

**insert_newline**（换行，见 [commands.rs:4445-](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs#L4445)）：

复杂之处：
- 删除行尾空白（自动缩进）
- 计算下一行的缩进量
- 若在注释中，延续注释标记
- 每个 range 独立计算自己的新光标偏移

#### D. 变换类命令（基于选区内容）

**switch_case** / **replace** 等模式的核心：

```rust
Transaction::change_by_selection(doc.text(), selection, |range| {
    let new_text = transform(range.slice(doc.text()));
    (range.from(), range.to(), Some(new_text))
})
```

每个 range 独立处理，天然支持多光标。例如将选中的"hello"和"world"分别替换为"HELLO"和"WORLD"。

### 5.3 多光标（多 Range）的行为保证

**Selection 归一化**（[selection.rs:560-587](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L560-L587)）确保：

1. **自动排序**：Range 始终按 `from()` 升序排列，因此从后向前处理不会产生偏移错乱
2. **重叠合并**：创建或变换后重叠的 Range 会被合并，避免重复编辑
3. **主索引追踪**：合并时会重新计算 `primary_index`，确保主光标不丢失

这使得命令无需关心多光标是否重叠——只需逐 Range 处理，Selection 自身保证一致性。

### 5.4 纯光标移动与事务的关系

普通的移动（h/j/k/l、w/b 等）不创建 Transaction，直接调用 `set_selection`。这些操作不产生撤销记录（History 的设计限制，见 [history.rs:41-42](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L41-L42) 的注释："Changes in selections currently don't commit history changes"）。

---

## 六、数据流总览

```
用户按键
  │
  ▼
Command handler
  ├─ doc.selection(view.id)          ── 读取当前选择集
  │
  ├─ 构造 Transaction
  │    ├─ change_by_selection
  │    ├─ delete_by_selection
  │    ├─ change_by_and_with_selection
  │    └─ insert
  │
  └─ doc.apply(txn, view.id)
        │
        ├─ apply_inner
        │    ├─ [首次变更] old_state = (text, selection) 快照
        │    ├─ apply_impl
        │    │    ├─ ChangeSet.apply(&mut text)         ── 修改文本
        │    │    │
        │    │    ├─ 对每个视图的 Selection:
        │    │    │    selection.map(changes)           ── 自动推导新选择位置
        │    │    │    （AfterSticky / BeforeSticky 粘附策略）
        │    │    │
        │    │    ├─ 更新 diagnostics / inlay hints
        │    │    ├─ 派发 DocumentDidChange 事件
        │    │    │
        │    │    └─ [txn 含 selection] 覆盖当前视图选择  ── 优先级最高
        │    │
        │    └─ changes.compose(txn.changes)           ── 累积到未提交变更
        │
        └─ [稍后] append_changes_to_history
             ├─ 正向事务携带"编辑后" selection
             ├─ 用 old_state 构造 Revision.inversion（含编辑前 selection）
             └─ history.commit_revision(...)            ── 写入撤销树
```

---

## 七、关键文件索引

| 文件 | 职责 | 路径 |
|------|------|------|
| **selection.rs** | Range/Selection 定义、映射、归一化 | [helix-core/src/selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs) |
| **transaction.rs** | ChangeSet/Operation/Transaction、Assoc 粘附策略、位置映射 | [helix-core/src/transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs) |
| **history.rs** | History/Revision 撤销树、State 快照 | [helix-core/src/history.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs) |
| **document.rs** | Document 存储多视图 Selection、apply 流程、历史提交 | [helix-view/src/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs) |
| **commands.rs** | 各类编辑命令，展示 Selection→Transaction→apply 的典型用法 | [helix-term/src/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs) |
