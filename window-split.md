# Helix 多窗口 Split 协作机制分析

## 概述

Helix 编辑器的多窗口拆分系统采用**二叉树式布局结构**，通过视图树（Tree）管理多个编辑视图。文档（Document）与视图（View）是多对多关系：一个文档可以在多个视图中显示，一个视图同一时间显示一个文档。

---

## 核心数据结构

### 1. 视图树（Tree）

**文件**: [tree.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/tree.rs)

```rust
pub struct Tree {
    root: ViewId,           // 根节点（容器）
    pub focus: ViewId,      // 当前焦点视图
    area: Rect,             // 整个区域
    nodes: SlotMap<ViewId, Node>,  // 所有节点（SlotMap 提供稳定索引）
    stack: Vec<(ViewId, Rect)>,    // 遍历用栈（复用避免重复分配）
}
```

**节点类型**（Content 枚举）:
- `Content::View(Box<View>)` — 叶子节点，实际的编辑视图
- `Content::Container(Box<Container>)` — 容器节点，管理子节点布局

**容器（Container）**:
```rust
pub struct Container {
    layout: Layout,     // Horizontal（上下）或 Vertical（左右）
    children: Vec<ViewId>,
    area: Rect,
}
```

**布局方向**:
- `Layout::Horizontal` — 水平分割，子视图上下排列（高度分配）
- `Layout::Vertical` — 垂直分割，子视图左右排列（宽度分配，有 1px 间隔）

---

### 2. 视图（View）

**文件**: [view.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/view.rs)

```rust
pub struct View {
    pub id: ViewId,
    pub area: Rect,                    // 视图在屏幕上的区域
    pub doc: DocumentId,               // 当前显示的文档
    pub jumps: JumpList,               // 跳转列表
    pub docs_access_history: Vec<DocumentId>,  // 文档访问历史
    doc_revisions: HashMap<DocumentId, usize>, // 文档修订版本（延迟同步用）
    // ... 其他配置
}
```

关键点：
- 每个 View 有独立的 `doc` 字段，指向当前显示的文档
- `doc_revisions` 实现**延迟同步**：切换视图时才同步文档变更到该视图
- 每个视图有独立的跳转列表和访问历史

---

### 3. 文档（Document）

**文件**: [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/document.rs)

```rust
pub struct Document {
    pub(crate) id: DocumentId,
    text: Rope,                        // 文档文本（共享）
    selections: HashMap<ViewId, Selection>,  // 每个视图的光标选区
    view_data: HashMap<ViewId, ViewData>,    // 每个视图的滚动位置等
    inlay_hints: HashMap<ViewId, DocumentInlayHints>,
    jump_labels: HashMap<ViewId, Vec<Overlay>>,
    document_highlights: HashMap<ViewId, DocumentHighlights>,
    // ... 其他字段
}
```

关键点：
- 文档文本是**共享的**（单一实例）
- 每个视图有独立的 `Selection`（光标/选区），存储在 `selections: HashMap<ViewId, Selection>` 中
- 每个视图有独立的 `ViewData`（滚动偏移量等），存储在 `view_data: HashMap<ViewId, ViewData>` 中
- 内联提示、跳转标签等也是按视图隔离的

---

## 编辑区域拆分机制

### 拆分流程

拆分操作入口在 [commands.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-term/src/commands.rs#L5948-L5962) 的 `split` 函数：

```rust
fn split(editor: &mut Editor, action: Action) {
    let (view, doc) = current!(editor);
    let id = doc.id();
    let selection = doc.selection(view.id).clone();
    let offset = doc.view_offset(view.id);

    editor.switch(id, action);  // 执行拆分

    // 新视图匹配原视图的选区和滚动位置
    let (view, doc) = current!(editor);
    doc.set_selection(view.id, selection);
    doc.set_view_offset(view.id, offset);
}
```

### Tree::split 核心逻辑

**文件**: [tree.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/tree.rs#L144-L215)

两种情况：

**情况 A：父容器布局方向与拆分方向相同**
- 直接在父容器中插入新节点（焦点节点之后）
- 不需要创建新容器

**情况 B：父容器布局方向与拆分方向不同**
- 创建一个新的 Container（布局方向为拆分方向）
- 将原焦点节点和新节点都作为新容器的子节点
- 用新容器替换原焦点节点在父容器中的位置

拆分完成后调用 `self.recalculate()` 重新计算所有视图的区域。

---

## 布局刷新（recalculate）

**文件**: [tree.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/tree.rs#L355-L442)

布局计算使用**栈式深度优先遍历**，从根容器开始，自上而下为每个节点分配屏幕区域。

### 水平分割（Layout::Horizontal，上下排列）

```rust
Layout::Horizontal => {
    let len = container.children.len();
    let height = area.height / len as u16;
    let mut child_y = area.y;
    for (i, child) in container.children.iter().enumerate() {
        let mut area = Rect::new(container.area.x, child_y, container.area.width, height);
        child_y += height;   // 无间隔！直接累加高度
        if i == len - 1 {
            area.height = container.area.y + container.area.height - area.y;
        }
        self.stack.push((*child, area));
    }
}
```

**水平分割要点**：
- **无间隔**（没有 inner_gap）
- 高度 = 容器高度 / 子视图数（整除）
- 每个子视图 x 坐标相同，y 依次累加高度
- **最后一个子视图**：`height = 容器底部 - 当前 y**（补全整除余数）

例：容器高 80，3 个视图
- height = 80/3 = 26
- 视图0：y=0, h=26；视图1：y=26, h=26；视图2：y=52, h=80-52=**28**
- 总高度 = 26 + 26 + 28 = 80 ✓

---

### 垂直分割（Layout::Vertical，左右排列）

垂直分割的间隔计算比较**不直观**，需要仔细分析。核心代码如下：

```rust
Layout::Vertical => {
    let len = container.children.len();
    let len_u16 = len as u16;

    let inner_gap = 1u16;
    let total_gap = inner_gap * len_u16.saturating_sub(2);  // ⚠️ 关键：是 len-2，不是 len-1

    let used_area = area.width.saturating_sub(total_gap);
    let width = used_area / len_u16;

    let mut child_x = area.x;

    for (i, child) in container.children.iter().enumerate() {
        let mut area = Rect::new(child_x, container.area.y, width, container.area.height);
        child_x += width + inner_gap;   // 每个视图之后都 +1（包括最后一个！）

        if i == len - 1 {
            // ⚠️ 最后一个视图的 width 被强制修正
            area.width = container.area.x + container.area.width - area.x;
        }

        self.stack.push((*child, area));
    }
}
```

#### 关键设计解析

**1. 为什么是 `len-2` 而不是 `len-1`？**

直觉上 N 个视图左右相邻应该有 **N-1 个 gap**。但代码中 `total_gap = len-2`，这是因为：

- 循环中 `child_x += width + inner_gap` **对每个子视图（包括最后一个）都加了 `inner_gap`**
- 然而最后一个子视图的 width **之后会被强制修正**为"容器右端 - 当前x"
- 这意味着**最后一个 gap 的 1px 空间被最后一个视图"吃掉"了**
- 所以在计算基础宽度时，只需要扣除 `len-2` 个 gap 的空间

**2. 完整的宽度计算流程**

步骤说明（以容器宽度 W、N 个视图为例）：
1. `total_gap = 1 * (N - 2)` &nbsp;&nbsp;—— 预扣 N-2 个间隔
2. `used_area = W - (N-2)` &nbsp;&nbsp;—— 剩余可用宽度
3. `base_width = used_area / N` &nbsp;&nbsp;—— 每个视图的基础宽度（整除）
4. 循环分配：
   - 前 N-1 个视图：宽度 = base_width，下一个视图的 x = 当前x + base_width + **1**
   - 最后一个视图：**width = 容器右端 - 当前x**（覆盖掉原本的 gap 空间）
5. 结果：实际存在 **N-1 个间隔**（前 N-1 个视图之后各有 1px gap，最后一个没有），最后一个视图宽度 = base_width + 余数 + **1**（吸收掉最后一个 gap）

#### 数值验证（测试用例 1）

**测试名**：`all_vertical_views_have_same_width`
- 容器总宽：**180**，视图数：**3**
- total_gap = 1 × (3-2) = 1
- used_area = 180 - 1 = **179**
- base_width = 179 / 3 = **59**

循环过程：

- **i=0（第1个视图）**：
  - x = 0，width = 59
  - child_x = 0 + 59 + 1 = **60**
  - 范围：[0, 59]，gap 位于 x=59

- **i=1（第2个视图）**：
  - x = 60，width = 59
  - child_x = 60 + 59 + 1 = **120**
  - 范围：[60, 119]，gap 位于 x=119

- **i=2（第3个视图，最后一个）**：
  - x = 120
  - **强制修正**：width = 180 - 120 = **60**
  - 范围：[120, 179]

**最终宽度**：[59, 59, 60]
**总占用验证**：59（视图0）+ 1（gap0）+ 59（视图1）+ 1（gap1）+ 60（视图2）= 180 ✓
**实际 gap 数**：2 个 = 3-1 ✓（total_gap 只预扣了 1 个，因为最后 1 个 gap 被视图 2 的修正宽度吸收了）

与测试断言完全一致：
```rust
vec![180/3 - 1, 180/3 - 1, 180/3]  // [59, 59, 60] ✓
```

---

#### 数值验证（测试用例 2）

**测试名**：`vsplit_gap_rounding`
- 容器总宽：**80**，视图数：**10**
- total_gap = 1 × (10-2) = **8**
- used_area = 80 - 8 = **72**
- base_width = 72 / 10 = **7**

循环过程（前 9 个视图）：
- 每个视图 width = 7
- 每次 child_x 增加 7+1 = 8
- 9 次之后 child_x = 9 × 8 = **72**

第 10 个视图（i=9，最后一个）：
- x = 72
- **强制修正**：width = 80 - 72 = **8**

**最终宽度**：9 个 7 + 1 个 8 = [7,7,7,7,7,7,7,7,7, 8]
**总占用验证**：
- 视图宽度合计：9×7 + 8 = 71
- gap 合计：9 个 × 1 = 9
- 总计：71 + 9 = 80 ✓

与测试断言一致：
```rust
std::iter::repeat_n(7, 9).chain(Some(8))  // ✓
```

---

### 拆分后的尺寸变化（逐步场景）

以容器总宽 180px 为例，观察每次 vsplit（垂直拆分）后各视图宽度的变化：

| 阶段 | 视图数 N | total_gap(N-2) | used_area | base_width | 各视图实际宽度 | 实际 gap 数 |
|------|---------|---------------|-----------|-----------|----------------|------------|
| 初始 | 1 | 0（saturating_sub） | 180 | 180 | [180] | 0 |
| 第1次 vsplit | 2 | 0 | 180 | 90 | [90, 89] | 1 |
| 第2次 vsplit | 3 | 1 | 179 | 59 | [59, 59, 60] | 2 |
| 第3次 vsplit | 4 | 2 | 178 | 44 | [44, 44, 44, 45] | 3 |

**逐阶段说明**：

**N=1**（初始单视图）：
- total_gap = 1.saturating_sub(2) = 0
- base_width = 180 / 1 = 180
- 最后一个（也是唯一）视图被修正为 width = 180 - 0 = **180**
- 无 gap

**N=2**（第一次垂直拆分）：
- total_gap = 2 - 2 = 0
- base_width = 180 / 2 = 90
- 视图0：x=0, w=90 → child_x=91
- 视图1（最后）：x=91, w=180-91 = **89**
- 实际宽度：[90, 89]，1 个 gap（在 x=90）
- 验证：90 + 1 + 89 = 180 ✓
- ⚠️ 注意：这里不是 [90, 90]！因为第一个 gap（在视图0之后）的 1px 被视图1的修正"吃"掉了 1px，所以视图1是 89

**N=3**（第二次垂直拆分）：
- total_gap = 3 - 2 = 1
- base_width = (180-1)/3 = 59
- 视图0：w=59, x=0→60
- 视图1：w=59, x=60→120
- 视图2（最后）：x=120, w=60
- 实际宽度：[59, 59, 60]，2 个 gap
- 59+1+59+1+60 = 180 ✓

**N=4**（第三次垂直拆分）：
- total_gap = 4 - 2 = 2
- base_width = (180-2)/4 = 178/4 = **44**
- 视图0：w=44, x=0 → child_x = 45
- 视图1：w=44, x=45 → child_x = 90
- 视图2：w=44, x=90 → child_x = 135
- 视图3（最后）：x=135, w = 180-135 = **45**
- 实际宽度：[44, 44, 44, 45]，3 个 gap
- 44+1+44+1+44+1+45 = 180 ✓

---

### 两种布局的对比

| 特性 | 水平分割 (Horizontal) | 垂直分割 (Vertical) |
|------|----------------------|---------------------|
| 排列方向 | 上下堆叠 | 左右并排 |
| 分配维度 | 高度 | 宽度 |
| 间隔 inner_gap | **无**（0） | **有**（1px） |
| gap 预扣公式 | 无 | N-2 个 |
| 基础尺寸公式 | height / N | (W - (N-2)) / N |
| 最后一个处理 | 补全高度余数 | 补全宽度余数 + 吸收最后 1 个 gap |

---

### 触发 recalculate 的时机

1. `insert` — 在当前焦点后插入新视图
2. `split` — 拆分视图（水平或垂直）
3. `remove` — 移除视图（可能触发容器合并）
4. `resize` — 窗口大小变化
5. `transpose` — 转置父容器布局方向（Horizontal ↔ Vertical）
6. `swap_split_in_direction` — 与相邻视图交换位置（交换 area）

---

## 焦点切换

### 焦点切换方式

| 函数 | 说明 |
|------|------|
| `focus_next()` | 按遍历顺序下一个 |
| `focus_prev()` | 按遍历顺序上一个 |
| `focus_direction(dir)` | 按方向（上下左右）跳转 |

### 方向查找（find_split_in_direction）

**文件**: [tree.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/tree.rs#L449-L546)

算法采用**递归向上查找**：
1. 从当前节点开始，检查父容器的布局方向
2. 如果目标方向与父容器布局方向**一致**（如向右查找且父容器是 Vertical），则在兄弟节点中查找
3. 如果目标方向与父容器布局方向**垂直**（如向右查找但父容器是 Horizontal），则递归向上查找
4. 找到目标子节点后，如果是容器，需要在容器中找到视觉上最近的视图

### focus() 完整流程

**文件**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/editor.rs#L2190-L2214)

```rust
pub fn focus(&mut self, view_id: ViewId) {
    if self.tree.focus == view_id { return; }

    self.enter_normal_mode();                 // 切换到普通模式
    let (view, doc) = current!(self);
    doc.append_changes_to_history(view);      // 提交旧文档的未提交变更
    self.ensure_cursor_in_view(view_id);      // 确保光标在视图内

    // 同步所有视图的变更
    for (view, _focused) in self.tree.views_mut() {
        let doc = doc_mut!(self, &view.doc);
        view.sync_changes(doc);
    }

    self.tree.focus = view_id;                // 更新焦点
    doc_mut!(self).mark_as_focused();         // 标记新文档为焦点

    dispatch(DocumentFocusLost { ... });      // 触发焦点丢失事件
}
```

---

## 文档共享与多视图协作

### 文档变更同步机制

#### 1. 主动视图实时更新

当在焦点视图中编辑时，`Document::apply_impl` 会**立即更新所有视图**的选区和滚动位置：

**文件**: [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/document.rs#L1434-L1479)

```rust
fn apply_impl(&mut self, transaction: &Transaction, view_id: ViewId, ...) -> bool {
    // 1. 更新文本（共享）
    changes.apply(&mut self.text);

    // 2. 更新所有视图的选区
    for selection in self.selections.values_mut() {
        *selection = selection
            .clone()
            .map(transaction.changes())      // 通过变更映射选区
            .ensure_invariants(self.text.slice(..));
    }

    // 3. 更新所有视图的滚动锚点
    for view_data in self.view_data.values_mut() {
        view_data.view_position.anchor = transaction
            .changes()
            .map_pos(view_data.view_position.anchor, Assoc::Before);
    }
}
```

这意味着：**一个视图中的编辑会立即反映在所有显示同一文档的视图中**（选区自动跟随文本变化移动）。

#### 2. 非活动视图延迟同步

虽然选区和滚动锚点是实时更新的，但视图的 `doc_revisions` 机制提供了**延迟同步**优化：

**文件**: [view.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/view.rs#L688-L700)

```rust
pub(crate) fn changes_to_sync(&mut self, doc: &mut Document) -> Option<Transaction> {
    let latest_revision = doc.get_current_revision();
    let current_revision = *self.doc_revisions
        .entry(doc.id())
        .or_insert(latest_revision);

    if current_revision == latest_revision {
        return None;
    }

    doc.history.get_mut().changes_since(current_revision)
}
```

`view.sync_changes(doc)` 在以下时机调用：
- `focus()` — 切换焦点时同步所有视图
- `_refresh()` — 刷新时同步所有视图
- `replace_document_in_view()` — 替换视图文档时
- `switch(... Action::Load)` — 加载文档时

延迟同步主要用于跳转列表（jumplist）等非关键状态的更新。

### 同一文档多视图的数据隔离

| 数据 | 共享/隔离 | 存储位置 |
|------|-----------|----------|
| 文本内容 | 共享 | `Document.text` |
| 语法树 | 共享 | `Document.syntax` |
| 诊断信息 | 共享 | `Document.diagnostics` |
| 光标选区 | 按视图隔离 | `Document.selections: HashMap<ViewId, Selection>` |
| 滚动位置 | 按视图隔离 | `Document.view_data: HashMap<ViewId, ViewData>` |
| 内联提示 | 按视图隔离 | `Document.inlay_hints` |
| 跳转列表 | 按视图隔离 | `View.jumps` |
| 文档访问历史 | 按视图隔离 | `View.docs_access_history` |

### 视图初始化

新视图打开文档时调用 `Document::ensure_view_init`：

**文件**: [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/document.rs#L1411-L1423)

```rust
pub fn ensure_view_init(&mut self, view_id: ViewId) {
    if !self.selections.contains_key(&view_id) {
        self.reset_selection(view_id);  // 初始化选区
        // ... 初始化其他视图相关数据
    }
}
```

### 关闭视图清理

关闭视图时会清理文档中该视图的所有数据：

**文件**: [editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-view/src/editor.rs#L2059-L2066)

```rust
pub fn close(&mut self, id: ViewId) {
    // 从所有文档中移除该视图的数据
    for doc in self.documents_mut() {
        doc.remove_view(id);
    }
    self.tree.remove(id);
    self._refresh();
}
```

---

## 渲染流程

**文件**: [ui/editor.rs](file:///d:/fz/0601/solo-dogfeeding/code/273-helix/helix-term/src/ui/editor.rs#L1603-L1632)

```rust
fn render(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    // ... bufferline、命令行等处理

    cx.editor.resize(editor_area);  // 检查窗口大小变化，触发 recalculate

    // 遍历所有视图进行渲染
    for (view, is_focused) in cx.editor.tree.views() {
        let doc = cx.editor.document(view.doc).unwrap();
        self.render_view(cx.editor, doc, view, area, surface, is_focused);
    }

    // ... statusline、命令行等
}
```

`render_view` 会根据 `view.area` 在正确的位置渲染每个视图的内容。

---

## 关键交互总结

### 拆分视图
```
用户触发 vsplit
    ↓
commands::split()
    ↓ 复制当前选区和滚动位置
editor::switch(id, VerticalSplit)
    ↓
tree::split(view, Vertical)
    ↓ 调整树结构
tree::recalculate()
    ↓ 重新计算所有视图区域
document::ensure_view_init(view_id)
    ↓ 初始化新视图的选区等数据
设置新视图选区和滚动位置（与原视图一致）
```

### 切换焦点
```
用户触发 jump_view_right
    ↓
editor::focus_direction(Right)
    ↓
tree::find_split_in_direction(current, Right)
    ↓ 递归查找目标视图
editor::focus(view_id)
    ↓
  1. 提交旧文档的未提交变更
  2. 确保光标在视图内
  3. 同步所有视图的变更
  4. 更新 tree.focus
  5. 标记新文档为焦点
  6. 发送 DocumentFocusLost 事件
```

### 编辑文档（多视图共享同一文档）
```
在焦点视图中输入文本
    ↓
document::apply_impl(transaction, view_id)
    ↓
  1. 更改共享文本
  2. 遍历更新 ALL 视图的选区（通过变更映射）
  3. 遍历更新 ALL 视图的滚动锚点
  4. 更新语法树、诊断等
```

### 窗口大小变化
```
检测到终端大小变化
    ↓
editor::resize(area)
    ↓
tree::resize(area) → tree::recalculate()
    ↓ 重新计算所有视图区域
editor::_refresh()
    ↓
  1. 遍历所有视图
  2. 同步文档变更
  3. 确保光标在视图内
```

---

## 架构层次

```
┌─────────────────────────────────────────┐
│         helix-term (UI 层)              │
│  commands / ui/editor.rs / compositor   │
├─────────────────────────────────────────┤
│         helix-view (视图层)             │
│  Editor / Tree / View / Document        │
├─────────────────────────────────────────┤
│         helix-core (核心层)             │
│  Rope / Selection / Transaction         │
└─────────────────────────────────────────┘
```

- **helix-term**: 处理用户输入、渲染、命令绑定
- **helix-view**: 编辑器状态管理（文档、视图、拆分树）
- **helix-core**: 文本操作核心数据结构

多窗口拆分的核心逻辑位于 `helix-view` 层，由 `Tree` 统一管理布局，`Document` 管理共享数据和视图隔离数据，`View` 管理视图特有的状态。
