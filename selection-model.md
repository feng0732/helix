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

## 二、选择集如何驱动文本变更

### 2.1 变更的底层表示：ChangeSet 与 Operation

[transaction.rs:12-20](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L12-L20)

```rust
pub enum Operation {
    Retain(usize),   // 保留 n 个字符
    Delete(usize),   // 删除 n 个字符
    Insert(Tendril), // 插入文本
}
```

`ChangeSet` 是 `Operation` 的序列，描述从文档 A 到文档 B 的完整变换。它采用类似 OT（Operational Transformation）的线性表示方式。

#### Assoc —— 位置关联策略

[transaction.rs:32-48](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L32-L48)

编辑时，位置如何随插入/删除而漂移？`Assoc` 定义了 6 种策略：

| 策略 | 含义 | 典型用途 |
|------|------|----------|
| `Before` | 粘附到插入点之前 | 普通光标尾部 |
| `After` | 粘附到插入点之后 | 普通光标头部 |
| `BeforeWord` | 词边界前粘附 | 诊断起点 |
| `AfterWord` | 词边界后粘附 | 诊断终点 |
| `BeforeSticky` | 等长替换时保持偏移 | 选择锚点 |
| `AfterSticky` | 等长替换时保持偏移 | 选择头部 |

**Selection 映射的 Assoc 选择逻辑**（见 [selection.rs:490-506](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L490-L506)）：

```
anchor < head 时（正向选择）:
  anchor → AfterSticky   （锚点跟随插入起点之后）
  head   → BeforeSticky  （头部跟随删除终点之前）

anchor > head 时（反向选择）:
  head   → AfterSticky
  anchor → BeforeSticky

anchor == head 时（零宽光标）:
  两者都 → AfterSticky
```

这种策略确保了：在选择两端插入文本时，选择范围会"包住"新文本；而删除时选择会正确收缩。

### 2.2 Transaction —— 可撤销的变更单元

[transaction.rs:573-577](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L573-L577)

```rust
pub struct Transaction {
    changes: ChangeSet,
    selection: Option<Selection>, // 可选：显式指定编辑后的选择
}
```

`Transaction` = `ChangeSet` + 可选的 Selection 覆盖。如果提供了 `selection`，编辑后直接使用该选择；否则通过 `ChangeSet.map` 自动推导。

### 2.3 面向选择集的 Transaction 构造器

这是选择集驱动文本变更的核心 API。

#### change_by_selection：逐范围生成变更

[transaction.rs:706-711](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L706-L711)

```rust
pub fn change_by_selection<F>(doc: &Rope, selection: &Selection, f: F) -> Self
where
    F: FnMut(&Range) -> Change,
```

**工作原理**：对 Selection 中的每个 Range 调用 `f`，生成一个 `(from, to, replacement)` 三元组，再将这些变更按序合成为 `ChangeSet`。

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

与 `change_by_selection` 类似，但闭包可返回 `Option<Range>` 来显式指定每个范围编辑后的新位置。适用于编辑后光标位置不能简单推导的场景（如插入后光标要跳到文本末尾）。

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

### 2.4 Document::apply —— 事务应用的完整流程

[document.rs:1435-1626](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1435-L1626)

调用链：`Document::apply` → `apply_inner` → `apply_impl`

**apply_impl 执行步骤（Selection 相关部分）：**

```
1. ChangeSet.apply(&mut self.text)     // 实际修改 Rope 文本

2. 若 changes 为空且 transaction 带 selection：
     直接用提供的 selection 更新（纯光标移动类事务）

3. 若 changes 非空：
   a. 遍历所有视图的 Selection：
        selection = selection.map(transaction.changes())
                         .ensure_invariants(...)
      （自动更新每个视图的光标位置）
   b. 更新 view_position（滚动锚点）
   c. 更新 savepoint 回滚事务
   d. 更新 tree-sitter 语法树
   e. 更新 diagnostics 位置
   f. 更新 inlay hints 位置
   g. 更新 document highlights 位置
   h. 派发 DocumentDidChange 事件

4. 若 transaction 显式指定了 selection：
     覆盖当前视图的 selection（优先级最高）
     派发 SelectionDidChange 事件
```

**关键机制**：`Selection::map(ChangeSet)` 内部调用 `ChangeSet::update_positions`，后者通过单次线性扫描 `ChangeSet`，批量映射所有 Range 的 anchor 和 head 位置，时间复杂度 O(N+M)（N 为操作数，M 为位置数）。见 [transaction.rs:388-510](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs#L388-L510)。

---

## 三、选择集与撤销/重做机制

### 3.1 History —— 修订树

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

**设计**：每个修订同时存储正向和反向事务。反向事务（inversion）不仅还原文本，还保存了**撤销时要恢复的 Selection**。

### 3.2 State —— （文本 + 选择集）快照

[history.rs:7-11](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L7-L11)

```rust
pub struct State {
    pub doc: Rope,
    pub selection: Selection,
}
```

### 3.3 提交修订：Selection 的保存

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

**inversion 事务的生成**：
- 文本还原：通过 `ChangeSet::invert()` 生成（Delete ↔ Insert 互换）
- 选择还原：直接使用 `original.selection`（编辑操作前的选择集）

这意味着：**撤销不仅还原文本，还还原到编辑前的光标/选区位置**。

### 3.4 Document 层的累积与提交

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

    let old_state = self.old_state.take().expect("no old_state available");

    let mut history = self.history.take();
    history.commit_revision(&transaction, &old_state);  // old_state 包含编辑前选择
    self.history.set(history);
}
```

### 3.5 Undo/Redo 的完整流程

**Undo**（[document.rs:1665-1687](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs#L1665-L1687)）：
```
1. append_changes_to_history()   // 先把当前累积的变更提交掉
2. history.undo() → 返回 revision.inversion（含旧文本+旧选择）
3. apply_impl(txn, view.id)      // 应用反向事务，恢复文本和选择
```

**Redo**：
```
1. 检查 self.changes 是否为空（有未提交的变更则拒绝 redo）
2. history.redo() → 返回 revision.transaction（含新文本）
3. apply_impl(txn, view.id)      // 应用正向事务
```

---

## 四、选择集如何驱动命令行为

### 4.1 命令的标准模式

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

### 4.2 命令分类与 Selection 的使用

#### A. 纯 Selection 变换类（不修改文本）

这类命令通过 `Selection::transform` 或直接构造新 Selection，调用 `set_selection`。

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

### 4.3 多光标（多 Range）的行为保证

**Selection 归一化**（[selection.rs:560-587](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs#L560-L587)）确保：

1. **自动排序**：Range 始终按 `from()` 升序排列，因此从后向前处理不会产生偏移错乱
2. **重叠合并**：创建或变换后重叠的 Range 会被合并，避免重复编辑
3. **主索引追踪**：合并时会重新计算 `primary_index`，确保主光标不丢失

这使得命令无需关心多光标是否重叠——只需逐 Range 处理，Selection 自身保证一致性。

### 4.4 纯光标移动与事务的关系

普通的移动（h/j/k/l、w/b 等）不创建 Transaction，直接调用 `set_selection`。这些操作不产生撤销记录（History 的设计限制，见 [history.rs:41-42](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs#L41-L42) 的注释："Changes in selections currently don't commit history changes"）。

---

## 五、数据流总览

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
        │    │    ├─ ChangeSet.apply(&mut text)        ── 修改文本
        │    │    │
        │    │    ├─ 对每个视图的 Selection:
        │    │    │    selection.map(changes)          ── 推导新选择位置
        │    │    │
        │    │    ├─ 更新 diagnostics / inlay hints
        │    │    ├─ 派发 DocumentDidChange 事件
        │    │    │
        │    │    └─ [txn 含 selection] 覆盖当前视图选择
        │    │
        │    └─ changes.compose(txn.changes)          ── 累积到未提交变更
        │
        └─ [稍后] append_changes_to_history
             ├─ 用 old_state 构造 Revision.inversion（含旧选择）
             └─ history.commit_revision(...)            ── 写入撤销树
```

---

## 六、关键文件索引

| 文件 | 职责 | 路径 |
|------|------|------|
| **selection.rs** | Range/Selection 定义、映射、归一化 | [helix-core/src/selection.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/selection.rs) |
| **transaction.rs** | ChangeSet/Operation/Transaction、位置映射 Assoc | [helix-core/src/transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/transaction.rs) |
| **history.rs** | History/Revision 撤销树、State 快照 | [helix-core/src/history.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-core/src/history.rs) |
| **document.rs** | Document 存储多视图 Selection、apply 流程、历史提交 | [helix-view/src/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-view/src/document.rs) |
| **commands.rs** | 各类编辑命令，展示 Selection→Transaction→apply 的典型用法 | [helix-term/src/commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/264-helix/helix-term/src/commands.rs) |
