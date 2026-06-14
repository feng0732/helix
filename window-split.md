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

布局计算使用**栈式深度优先遍历**：

```rust
pub fn recalculate(&mut self) {
    self.stack.push((self.root, self.area));

    while let Some((key, area)) = self.stack.pop() {
        match &mut node.content {
            Content::View(view) => {
                view.area = area;  // 叶子节点直接分配区域
            }
            Content::Container(container) => {
                container.area = area;
                // 根据布局方向计算子节点区域
                match container.layout {
                    Layout::Horizontal => { /* 上下均分高度 */ },
                    Layout::Vertical => { /* 左右均分宽度 */ },
                }
                // 将子节点压入栈继续处理
            }
        }
    }
}
```

**垂直分割细节**:
- 子视图之间有 1px 间隔（`inner_gap = 1`）
- 最后一个子视图占用剩余宽度（处理整除余数）
- 总间隔 = `inner_gap * (len - 1)`

**水平分割细节**:
- 子视图高度均分
- 最后一个子视图占用剩余高度

**触发 recalculate 的时机**:
1. `insert` — 插入新视图
2. `split` — 拆分视图
3. `remove` — 移除视图
4. `resize` — 窗口大小变化
5. `transpose` — 转置布局方向

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
