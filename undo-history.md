# Helix 撤销历史机制源码分析

> 代码引用统一采用仓库相对路径（以仓库根目录为基准），如 `helix-core/src/history.rs#L51-L54`

## 一、核心数据结构

### 1. History（撤销树）

定义于 `helix-core/src/history.rs#L51-L54`：

```rust
pub struct History {
    revisions: Vec<Revision>,  // 所有修订版本，线性存储
    current: usize,            // 当前修订版本的索引
}
```

History 并非简单的栈，而是一棵**线性化存储的树**。每个 Revision 知道自己的父节点（`parent`）和最后一个子节点（`last_child`），支持分支式的 undo/redo 路径。

### 2. Revision（单个历史节点）

定义于 `helix-core/src/history.rs#L57-L66`：

```rust
struct Revision {
    parent: usize,                   // 父修订索引（根节点指向自身 0）
    last_child: Option<NonZeroUsize>,// 最后一个子修订索引
    transaction: Transaction,        // 从父状态到本状态的正向事务
    inversion: Transaction,          // 从本状态回退到父状态的反转事务
    timestamp: Instant,              // 提交时间戳（用于 :earlier/:later 时间导航）
}
```

关键设计：每个 Revision 同时存储 **transaction**（正向变更）和 **inversion**（反转变更）。反转事务是通过 `Transaction::invert(&original.doc)` 生成的，其中 Delete 操作变为 Insert（恢复被删文本），Insert 操作变为 Delete。inversion 还通过 `.with_selection()` 保存了撤销时需要恢复的光标位置。

### 3. 根节点

History 初始化时创建一个虚拟根 Revision（`helix-core/src/history.rs#L68-L82`），其 transaction 和 inversion 均为空 ChangeSet，parent 指向自身（0），无子节点。`current` 初始为 0。

---

## 二、编辑事务进入历史栈的流程

### 阶段一：事务应用时 —— 累积到 `changes` ChangeSet

当编辑操作发生时（如 `insert_char`、`delete_char_backward`），事务通过 `Document::apply()` → `apply_inner()` 应用到文档（`helix-view/src/document.rs#L1628-L1651`）：

```rust
fn apply_inner(&mut self, transaction: &Transaction, view_id: ViewId, ...) -> bool {
    // 首次变更时，保存旧状态快照
    if self.changes.is_empty() && !transaction.changes().is_empty() {
        self.old_state = Some(State {
            doc: self.text.clone(),
            selection: self.selection(view_id).clone(),
        });
    }
    self.apply_impl(transaction, view_id, ...);
    // 将本次变更 compose 到 pending changes 中
    if !transaction.changes().is_empty() {
        take_with(&mut self.changes, |changes| {
            changes.compose(transaction.changes().clone())
        });
    }
}
```

核心机制：Document 维护两个关键字段：
- `changes: ChangeSet` —— **待提交的累积变更**，每次 apply 都会 compose 进去
- `old_state: Option<State>` —— **本次累积变更之前的状态快照**，仅在 `changes` 从空变非空时设置一次

这意味着在 Insert 模式下，连续的按键操作不会逐个进入历史栈，而是通过 ChangeSet 的 `compose` 操作合并成一个大的 ChangeSet。例如连续输入 "hello" 五次 insert 事务，compose 后等价于一个 `Insert("hello")` 操作（见 `helix-core/src/transaction.rs#L1116-L1154` 的 optimized_composition 测试）。

### 阶段二：提交到历史栈 —— `append_changes_to_history`

当满足"合并边界"条件时，调用 `helix-view/src/document.rs#L1787-L1808` 的 `append_changes_to_history`：

```rust
pub fn append_changes_to_history(&mut self, view: &mut View) {
    if self.changes.is_empty() { return; }
    let new_changeset = ChangeSet::new(self.text().slice(..));
    let changes = std::mem::replace(&mut self.changes, new_changeset);
    let transaction = Transaction::from(changes).with_selection(self.selection(view.id).clone());
    let old_state = self.old_state.take().expect("no old_state available");
    let mut history = self.history.take();
    history.commit_revision(&transaction, &old_state);
    self.history.set(history);
    view.apply(&transaction, self);
}
```

流程：
1. 检查 `changes` 是否为空，空则跳过
2. 取出并替换 `changes`（重置为新的空 ChangeSet）
3. 用累积的 changes 构造 Transaction，附带当前选区
4. 取出 `old_state`（变更前快照）
5. 调用 `History::commit_revision` 提交

### 阶段三：History 内部提交

`helix-core/src/history.rs#L85-L110` 的 `commit_revision`：

```rust
pub fn commit_revision(&mut self, transaction: &Transaction, original: &State) {
    self.commit_revision_at_timestamp(transaction, original, Instant::now());
}

pub fn commit_revision_at_timestamp(&mut self, transaction: &Transaction, original: &State, timestamp: Instant) {
    let inversion = transaction.invert(&original.doc).with_selection(original.selection.clone());
    let new_current = self.revisions.len();
    self.revisions[self.current].last_child = NonZeroUsize::new(new_current);
    self.revisions.push(Revision {
        parent: self.current,
        last_child: None,
        transaction: transaction.clone(),
        inversion,
        timestamp,
    });
    self.current = new_current;
}
```

关键操作：
1. 根据原始文档生成 inversion（反转事务），并保存原始选区
2. 将当前 Revision 的 `last_child` 更新为新节点的索引
3. 新 Revision 的 `parent` 指向当前节点
4. 推入 `revisions` 向量，`current` 前进到新节点

**注意**：`last_child` 只保留最后一个子节点。如果 undo 后再做新编辑，原来的 redo 分支不会被删除（仍在 revisions 向量中），但 `last_child` 被更新为新分支，所以 redo 只能走最新分支。

---

## 三、合并边界（何时提交到历史栈）

合并边界决定了哪些连续编辑会被合并为一个 undo 单元。核心规则是：**Insert 模式下的连续操作延迟合并，退出 Insert 模式或其他触发条件时提交**。

### 主要触发点汇总

| 触发场景 | 位置 | 说明 |
|---|---|---|
| **键事件处理后（非 Insert 模式）** | `helix-term/src/ui/editor.rs#L1561-L1565` | 每次按键命令执行后，若不在 Insert 模式，立即提交。**这是退出 Insert 模式时提交变化的关键路径**——ESC 触发 `normal_mode`，模式切换后下一行判断 `mode != Insert`，立即提交。 |
| **Paste 事件（非 Insert 模式）** | `helix-term/src/ui/editor.rs#L1462-L1466` | 粘贴后若不在 Insert 模式则提交 |
| **Undo 操作前** | `helix-view/src/document.rs#L1667` | `undo_redo_impl` 中，undo 前先提交 pending changes |
| **Earlier 操作前** | `helix-view/src/document.rs#L1752` | `earlier_later_impl` 中，earlier 前先提交 |
| **Redo / Later 操作前（有 pending changes）** | `helix-view/src/document.rs#L1668-L1669` | 若有 pending changes 则 redo/later 直接返回 false，拒绝操作 |
| **Paste（Insert 模式）** | `helix-term/src/commands.rs#L4922-L4924` | Insert 模式下粘贴**前**先提交，粘贴**后**再次提交 |
| **补全（Completion）确认** | `helix-term/src/ui/completion.rs#L214` | 补全确认**前**先提交 pending changes，使补全前后分离为独立 undo 单元 |
| **补全（Completion）确认后** | `helix-term/src/ui/editor.rs#L1561-L1565` | 补全 transaction 应用后，由于此时仍在 Insert 模式，**键事件后不会自动提交**，而是留在 changes 中，后续继续打字会 compose 到同一个 undo 单元 |
| **Jump / Push jump** | `helix-term/src/commands.rs#L3958` | 跳转前提交 |
| **保存文件前** | `helix-term/src/commands/typed.rs#L398` | `:write` 前提交 |
| **格式化后** | `helix-term/src/commands.rs#L3781` | 格式化应用后提交 |
| **手动 checkpoint** | `helix-term/src/commands.rs#L4785-L4788` | `commit_undo_checkpoint` 命令 |
| **Picker 回调后（非 Insert 模式）** | `helix-term/src/commands.rs#L3648-L3650` | Picker 选择执行后若非 Insert 模式则提交 |
| **替换命令后** | `helix-term/src/commands.rs#L6697` | `:%s/x/y/g` 替换执行后提交 |
| **Reload 文件后** | `helix-view/src/document.rs#L1295` | 文件 reload 差异应用后提交 |

### 合并边界：插入模式粘贴详解

插入模式下的粘贴（Bracketed Paste 或剪贴板 `p`/`P`）会**先提交再粘贴再提交**，形成两个明确的 undo 边界。

#### 路径一：Bracketed Paste（终端粘贴事件）

入口在 `helix-term/src/ui/editor.rs#L1450-L1468`：

```
Event::Paste(contents)
    └── paste_bracketed_value(commands.rs#L4993-L5002)
            │
            ├── mode == Insert → Paste::Cursor
            └── paste_impl(commands.rs#L4910-L4991)
                    ├── [边界A] mode == Insert → append_changes_to_history  ← 粘贴前提交
                    ├── 构造粘贴 Transaction
                    ├── doc.apply(&transaction, view.id)  → 变更 compose 到 changes
                    └── [边界B] append_changes_to_history  ← 粘贴后立即提交
```

**关键代码** `helix-term/src/commands.rs#L4918-L4991`：

```rust
fn paste_impl(values, doc, view, action, count, mode) {
    if values.is_empty() { return; }

    if mode == Mode::Insert {                    // ← 边界A
        doc.append_changes_to_history(view);     // 粘贴前：先将之前的打字提交
    }
    // ... 构造粘贴 transaction ...
    doc.apply(&transaction, view.id);            // 粘贴内容应用到文档
    doc.append_changes_to_history(view);         // ← 边界B：粘贴后立即提交
}
```

**Bracketed Paste 完成后**，回到 `Event::Paste` 处理流程，由于 Insert 模式判断（`mode != Insert` 为 false），**不会**再次触发 `append_changes_to_history`（`helix-term/src/ui/editor.rs#L1464`）。

**因此，Bracketed Paste 在 Insert 模式下形成 3 个独立 undo 单元：**

| 阶段 | undo 单元 | 包含内容 | 触发提交点 |
|---|---|---|---|
| 1 | 粘贴前的打字 | 从进入 Insert 模式或上次边界到粘贴前的所有编辑 | `paste_impl` 内边界A |
| 2 | 粘贴本身 | 粘贴的全部内容 | `paste_impl` 内边界B |
| 3 | 粘贴后的打字 | 粘贴后继续输入的字符，直到退出 Insert 模式或下次边界 | 退出 Insert 时由按键后检查提交 |

#### 路径二：Normal 模式下 `p`/`P` 剪贴板粘贴

入口在 `helix-term/src/commands.rs#L5087-L5095`，由键命令触发：

```
Key: p (Normal模式)
    └── paste_after / paste_before
            └── paste(editor, register, pos, count)
                    └── paste_impl(mode = Normal)
                            ├── mode != Insert → 跳过边界A
                            ├── doc.apply(&transaction)
                            └── append_changes_to_history (边界B)
    └── 键事件后检查 mode != Insert → 再次调用 append_changes_to_history
                                    （此时 changes 已空，无实际作用）
```

**Normal 模式粘贴**只有 1 个 undo 单元（粘贴本身），且由 `paste_impl` 内部的边界B 提交。

---

### 合并边界：补全（Completion）确认详解

补全确认过程中会在**精确的时机**插入 undo 边界，同时使用 SavePoint 机制处理"幽灵变更"。

#### 补全触发与 SavePoint

当用户输入触发字符（如 `.` 或单词字符）时，补全被触发，CompletionHandler 存入 `active_completions`，每个 provider 对应一个 `ResponseContext`，其中包含：

`helix-view/src/handlers/completion.rs#L30-L38`：
```rust
pub struct ResponseContext {
    pub is_incomplete: bool,
    pub priority: i8,
    pub savepoint: Arc<SavePoint>,   // ← 补全触发瞬间的文档快照
}
```

**savepoint 的作用**：保存补全触发时文档的完整状态（文本+选区），供后续"幽灵预览"和"确认时回滚"使用。

#### 幽灵预览（Preview Completion Insert）

如果启用了 `preview_completion_insert`，用户在补全列表中移动光标（C-n/C-p/Tab）时，会**临时应用**当前选中项的 transaction 到文档中，形成"幽灵变更"：

`helix-term/src/ui/completion.rs#L161-L199`：
```rust
PromptEvent::Update if preview_completion_insert => {
    if matches!(editor.last_completion, Some(CompleteAction::Triggered)) {
        editor.last_completion = Some(CompleteAction::Selected {
            savepoint: doc.savepoint(view),   // ← 首次预览时也保存 savepoint
        })
    }
    let item = item.unwrap();
    let context = &editor.handlers.completions.active_completions[&item.provider()];
    doc.restore(view, &context.savepoint, false);  // ← 先回滚到补全触发前状态
    // ...
    doc.apply_temporary(&transaction, view.id)      // ← 临时应用补全 transaction（不通知 LSP）
}
```

**关键点**：
- `restore` 使用 savepoint 回滚，会使用 `apply_inner` 路径 → 变更会 compose 到 `changes`
- 但 `apply_temporary` 也走 `apply_inner` 路径 → 变更也会 compose 到 `changes`
- 回滚和应用在 `changes` 层面是成对的，净效果为补全触发后的新输入被"抹掉"替换为补全内容

#### 确认时的边界与提交

用户按 Tab 或 Enter 确认补全项时，流程如下（`helix-term/src/ui/completion.rs#L202-L282`）：

```
PromptEvent::Validate (确认补全)
    │
    ├── [1] 若有 CompleteAction::Selected savepoint → restore (emit=false)
    │       ← 回滚幽灵预览，不通知 LSP
    │
    ├── [2] doc.restore(view, &context.savepoint, true)
    │       ← 回滚到补全触发瞬间的状态，本次 emit=true
    │       ← restore 的变更 compose 到 changes
    │
    ├── [边界C] doc.append_changes_to_history(view)
    │       ← ★关键边界：提交补全触发后到确认前的所有 pending changes
    │       ← 此时 changes 被重置为空，old_state 被消费
    │
    ├── [3] 构造补全确认 transaction（含 LSP resolve）
    │
    ├── [4] doc.apply(&transaction, view.id)   ← emit LSP 通知
    │       ← changes 从空变非空 → 设置新的 old_state
    │       ← 补全内容 compose 到 changes
    │
    ├── [5] 如有 snippet → 激活 ActiveSnippet
    │
    ├── [6] 如有 additional_edits → doc.apply(&transaction, view.id)
    │       ← 额外变更 compose 到 changes（与补全在同一 undo 单元）
    │
    └── trigger_auto_completion(editor, true)
            ← 补全结束后，若仍在 Insert 模式，继续打字会 compose 到 changes
```

**边界C 的代码定位** `helix-term/src/ui/completion.rs#L209-L214`：
```rust
let item = item.unwrap();
let context = &editor.handlers.completions.active_completions[&item.provider()];
doc.restore(view, &context.savepoint, true);  // 回滚到补全触发时
// save an undo checkpoint before the completion
doc.append_changes_to_history(view);          // ← 边界C
```

#### 补全确认后的后续行为

补全确认后，事件仍然是同一个按键（Tab 或 Enter）触发的。让我们追踪其完整路径：

```
用户按 Tab（Insert 模式下补全列表开启）
    │
    ├── handle_event: Completion.handle_event(Event::Key(Tab))
    │       → Menu 触发 PromptEvent::Validate
    │       → 执行上述补全确认流程 [1]-[6]
    │       → 返回 EventResult::Consumed(callback=Some(close_fn))
    │
    ├── consumed=true → 跳过 insert_mode (不进入普通打字逻辑)
    │
    └── 键事件后检查: mode != Insert ?
            仍为 Insert 模式 → condition=false
            → 不触发 append_changes_to_history！
```

**这意味着：补全确认本身不会立即提交为一个独立的 Revision！**

补全确认后的 `changes` 状态：
- `changes` = 补全 transaction（可能含 additional_edits）
- 后续继续打字（如输入参数、空格等）→ 继续 compose 到同一个 `changes`
- 只有当**退出 Insert 模式**或**触发其他边界**（如再次粘贴、再次补全确认、undo 等）时，补全内容 + 后续打字才一起提交

#### 补全完整 undo 单元划分示例

**场景**：Insert 模式输入 `pri` → 触发补全 → Tab 确认 `println!` → 继续输入 `("hello")` → Esc

| undo 单元 | 包含内容 | 提交触发点 |
|---|---|---|
| 1 | 进入 Insert 到补全触发前的编辑（如之前的打字） | 边界C：补全确认前的 `append_changes_to_history` |
| 2 | `println!("hello")`（补全内容 + 后续继续输入的参数和括号） | 退出 Insert 模式时的键事件后检查（`mode != Insert`） |

**场景**：Insert 模式输入 `pri` → Tab 确认补全 → 再次粘贴 `args` → 继续输入 `;` → Esc

| undo 单元 | 包含内容 | 提交触发点 |
|---|---|---|
| 1 | 补全触发前的编辑 | 边界C |
| 2 | `println!`（补全本身） | 粘贴前边界A（`paste_impl` 内的 `append_changes_to_history`） |
| 3 | 粘贴内容 `args` | 粘贴后边界B（`paste_impl` 末尾） |
| 4 | `;`（粘贴后继续输入） | 退出 Insert 模式 |

#### 多补全连续确认（Tab 链式补全）

如果补全后立即触发新补全并再次确认（如输入 `std::` 后确认，紧跟着又有补全）：

```
第一次补全确认(边界C)
    └── 提交"补全前的打字"
第一次补全 transaction 应用 → changes 非空
    ↓
第二次补全被触发 → 新的 context.savepoint 建立（此时 changes 仍非空）
    ↓
第二次补全确认(边界C):
    ├── restore(第二次 savepoint) → compose 到 changes
    └── append_changes_to_history → 提交：(第一次补全 + 两次补全间的输入 + 回滚) 的合成
```

每次补全确认的边界C 都会提交当前 changes，将补全之间的编辑分离为独立单元。

---

### 合并逻辑总结

1. **Insert 模式内**：所有编辑（打字、退格等）仅 compose 到 `changes` 中，不提交历史。整个 Insert 会话（从 `i`/`a` 到 `Esc`）的所有变更**默认**合并为一个 undo 单元。
2. **Normal 模式下**：每个命令执行后立即提交（`helix-term/src/ui/editor.rs#L1561-L1565` 的键事件后检查），每个命令是独立的 undo 单元。
3. **Insert 模式粘贴**：粘贴**前**提交（边界A）使粘贴成为**新的独立起点**；粘贴**后**立即提交（边界B）使粘贴本身成为独立 undo 单元。粘贴前后各切一刀。
4. **补全确认**：确认**前**提交（边界C）将"补全触发前的打字"与"补全本身"切开。但补全本身**不立即提交**，而是留在 `changes` 中与后续输入继续合并，直到退出 Insert 或遇到其他边界。
5. **savepoint/restore 与 changes 的关系**：`Document::restore` 内部使用 `apply_inner` 路径（`helix-view/src/document.rs#L1730-L1747`），其 revert transaction 会正常 compose 到 `changes` 中。因此 savepoint 回滚不会破坏合并逻辑——回滚与后续编辑在 changes 层面自然合并（净效果可能为零）。

---

## 四、回退恢复（Undo / Redo）

### 基础 Undo / Redo

`helix-core/src/history.rs#L138-L155` 的 `undo` 和 `redo`：

```rust
pub fn undo(&mut self) -> Option<&Transaction> {
    if self.at_root() { return None; }
    let current_revision = &self.revisions[self.current];
    self.current = current_revision.parent;
    Some(&current_revision.inversion)  // 返回反转事务
}

pub fn redo(&mut self) -> Option<&Transaction> {
    let current_revision = &self.revisions[self.current];
    let last_child = current_revision.last_child?;
    self.current = last_child.get();
    Some(&self.revisions[last_child.get()].transaction)  // 返回正向事务
}
```

- **Undo**：移动 `current` 到父节点，返回 inversion 事务。调用方将 inversion 应用到文档即可恢复到父状态。
- **Redo**：移动 `current` 到 `last_child`，返回正向 transaction。应用后前进到子状态。

Document 层的 `undo_redo_impl`（`helix-view/src/document.rs#L1665-L1687`）：

```rust
fn undo_redo_impl(&mut self, view: &mut View, undo: bool) -> bool {
    if undo {
        self.append_changes_to_history(view);  // undo 前先提交 pending
    } else if !self.changes.is_empty() {
        return false;  // 有 pending changes 时 redo 被拒绝
    }
    let mut history = self.history.take();
    let txn = if undo { history.undo() } else { history.redo() };
    let success = if let Some(txn) = txn {
        self.apply_impl(txn, view.id, true)    // ← 注意：用 apply_impl，不走 apply_inner
    } else { false };
    self.history.set(history);
    if success {
        self.changes = ChangeSet::new(self.text().slice(..));  // 重置 pending
        view.sync_changes(self);  // 同步跳转列表
    }
    success
}
```

**注意**：undo/redo 使用 `apply_impl` 而非 `apply`，跳过了 `apply_inner` 中的 changes 累积和 old_state 保存逻辑。undo/redo 后直接重置 `changes`。这意味着 undo/redo 产生的文档变更**不会**被 compose 到 `changes` 中，也不会形成新的历史分支（除非用户紧接着又做了新编辑）。

### Redo 的保护机制

当有 pending changes（`changes` 非空）时执行 Redo/Later，会直接返回 false（`helix-view/src/document.rs#L1668-L1669`）。这是为了避免以下不一致：

- pending changes 代表"当前 Insert 会话中未提交的编辑"
- 如果 redo 了某个历史 revision，文档状态跳转到未来，pending changes 的 `len` 与新文档长度不匹配
- 此时继续编辑会导致 ChangeSet compose 失败或状态错乱

用户必须先手动 Undo（Undo 操作内部会先 `append_changes_to_history` 提交 pending），或者 Esc 退出 Insert 提交后才能 Redo。

### 分支式 Undo（树形结构）

Undo 后进行新编辑，会创建新分支。例如：

```
Rev0 → Rev1 → Rev2 → Rev3
              ↘ Rev4（undo 到 Rev1 后编辑产生）
```

- Rev1 的 `last_child` 从 Rev2 更新为 Rev4
- Rev2 仍在 revisions 中，但不再被 `last_child` 引用
- Redo 只能沿 `last_child` 走到 Rev4，无法回到 Rev2
- 要回到 Rev2 需要通过 `:earlier` / `:later` 或 `changes_since` 等机制

### 时间导航：Earlier / Later

`helix-core/src/history.rs#L287-L302` 的 `earlier` 和 `later` 支持两种模式：

```rust
pub enum UndoKind {
    Steps(usize),              // 按步数跳转
    TimePeriod(Duration),      // 按时间跳转
}
```

#### 按步数跳转

`jump_backward(delta)` / `jump_forward(delta)` 直接通过索引偏移在 revisions 向量中移动，**不依赖树结构**，而是利用 revisions 的线性排列顺序。

#### 按时间跳转

`jump_duration_backward` / `jump_duration_forward` 计算目标时间戳，然后通过二分查找（`binary_search_by` 比较时间戳）定位最近的修订版本。

#### 跨分支跳转：jump_to

`helix-core/src/history.rs#L213-L224` 的 `jump_to` 是底层跳转机制：

```rust
fn jump_to(&mut self, to: usize) -> Vec<Transaction> {
    let lca = self.lowest_common_ancestor(self.current, to);
    let up = self.path_up(self.current, lca);   // 当前 → LCA 的路径
    let down = self.path_up(to, lca);            // 目标 → LCA 的路径
    self.current = to;
    let up_txns = up.iter().map(|&n| self.revisions[n].inversion.clone());
    let down_txns = down.iter().rev().map(|&n| self.revisions[n].transaction.clone());
    up_txns.chain(down_txns).collect()
}
```

1. 找到 current 和 target 的**最近公共祖先（LCA）**
2. 从 current 向上走到 LCA（使用 inversion 事务）
3. 从 LCA 向下走到 target（使用正向 transaction，逆序）
4. 返回所有需要依次应用的事务列表

LCA 查找算法（`helix-core/src/history.rs#L183-L199`）使用双路径集合交替上升法，类似链表找交点。

与 undo/redo 类似，`earlier_later_impl`（`helix-view/src/document.rs#L1750-L1773`）中的 `earlier` 操作前也会先 `append_changes_to_history` 提交 pending changes，而 `later` 操作前若有 pending changes 则直接返回 false。

### changes_since

`helix-core/src/history.rs#L124-L135` 的 `changes_since` 返回从指定修订到当前修订的合成事务，通过 compose 将所有变更合并为单个 Transaction。用于 LSP 等需要获取增量变更的场景。

---

## 五、ChangeSet 的 compose 与 invert

### compose（合并变更）

`helix-core/src/transaction.rs#L163-L293` 的 `ChangeSet::compose` 将两个连续的 ChangeSet 合并为一个。前提：`self.len_after == other.len`（第一个的输出长度等于第二个的输入长度）。

compose 通过双指针逐一消费两个 ChangeSet 的 Operation 序列，处理 Retain/Delete/Insert 的各种组合情况，生成等价的单一 ChangeSet。这是实现"Insert 模式下连续编辑合并为一个 undo 单元"的基础。

典型合并效果（`optimized_composition` 测试 `helix-core/src/transaction.rs#L1116-L1154`）：
- `Insert("h")` ∘ `Retain(1), Insert("e")` ∘ `Retain(2), Insert("l")` ∘ ... → 最终等价于 `Insert("hello")`
- 多个相邻的 Retain/Delete 会合并计数（`ChangeSet::retain` 和 `delete` 内部直接累加 `self.changes.last_mut()`）

### invert（生成反转）

`helix-core/src/transaction.rs#L312-L339` 的 `ChangeSet::invert` 根据**原始文档**生成反转 ChangeSet：

- `Retain(n)` → `Retain(n)`（保留不变）
- `Delete(n)` → `Insert(原始文档对应位置的文本)`（恢复被删内容）
- `Insert(s)` → `Delete(s.chars().count())`（删除插入的内容）

由于 Delete 操作不存储被删文本，invert 必须访问原始文档才能恢复。这也是为什么 History 的 Revision 需要存储 inversion 而非每次重新计算。

---

## 六、完整流程示例

### 场景：Insert 模式输入 "abc" 后 Esc

1. **按 `i`**：进入 Insert 模式，不提交历史
2. **按 `a`**：
   - `insert_char`（`helix-term/src/commands.rs#L4315-L4344`）生成 Transaction `Insert("a")`
   - `apply_inner` 中：`changes` 从空变非空 → 保存 `old_state`
   - compose：`changes = Insert("a")`
3. **按 `b`**：
   - 生成 Transaction `Insert("b")`（位置偏移 +1）
   - compose：`changes = Insert("a") ∘ (Retain(1), Insert("b"))` = `Insert("ab")`
4. **按 `c`**：类似，`changes` 变为 `Insert("abc")`
5. **按 `Esc`**：
   - `normal_mode` 将 mode 设为 Normal（`helix-term/src/commands.rs#L3952-L3954`）
   - 键事件处理后，判断 `mode != Insert` → 调用 `append_changes_to_history`（`helix-term/src/ui/editor.rs#L1563-L1564`）
   - 从 `changes` 构造 Transaction，用 `old_state` 生成 inversion
   - `History::commit_revision` 创建新 Revision，`current` 前进
6. **按 `u`（undo）**：
   - 先 `append_changes_to_history`（此时 changes 为空，跳过）
   - `history.undo()` 返回 inversion（`Delete(3)`）
   - 应用 inversion，文档恢复为 "abc" 之前的状态
   - 重置 `changes`

### 场景：Insert 模式输入 "hel" → 粘贴 "lo wo" → 输入 "rld" → Esc

1. **输入 "hel"**：`changes` 累积为 `Insert("hel")`，`old_state` = 空文档快照
2. **终端粘贴 "lo wo"（Bracketed Paste）**：
   - `paste_impl` 内边界A：`append_changes_to_history`
     - 提交 Rev1 = Insert("hel")，`changes` 重置为空
   - 构造粘贴 transaction，`doc.apply`
     - `changes` 变为 Insert("lo wo")，新的 `old_state` = "hel" 快照
   - `paste_impl` 内边界B：`append_changes_to_history`
     - 提交 Rev2 = Insert("lo wo")，`changes` 重置为空
3. **输入 "rld"**：`changes` 累积为 Insert("rld")，新的 `old_state` = "hello wo" 快照
4. **Esc**：键事件后检查提交 Rev3 = Insert("rld")
5. **Undo 结果**：
   - 1st u: "hello wo rld" → "hello wo"（撤销 Rev3 "rld"）
   - 2nd u: → "hel"（撤销 Rev2 粘贴 "lo wo"）
   - 3rd u: → ""（撤销 Rev1 "hel"）

### 场景：Insert 模式输入 "pri" → 补全 → Tab 确认 println! → 输入 "(x)" → Esc

1. **输入 "pri"**：`changes` = Insert("pri")，`old_state` = 初始快照
2. **补全触发**：建立 `context.savepoint`（"pri" 状态快照）
3. **可能的幽灵预览**：用户按 C-n 浏览，不断 restore 再 apply_temporary，`changes` 在 compose 层面净效果为切换预览内容
4. **Tab 确认 println!**：
   - 回滚到 context.savepoint（emit=true）
   - **边界C**：`append_changes_to_history` → 提交 Rev1 = Insert("pri")
   - 构造 println! transaction 并 `doc.apply`
     - `changes` = Insert("println!")，新 `old_state` = "pri" 快照
   - 仍在 Insert 模式，键事件后**不提交**
5. **输入 "(x)"**：compose 到 changes，`changes` = Insert("println!(" + "(x)") 的合成
6. **Esc**：提交 Rev2 = Insert("println!(x)")
7. **Undo 结果**：
   - 1st u: → "pri"（撤销补全 + 后续输入的合并体）
   - 2nd u: → ""（撤销之前的打字）

---

## 七、已知限制

源码注释中提到的局限（`helix-core/src/history.rs#L40-L48`）：

1. **选区变更不产生历史记录**：仅 buffer 内容变更会提交历史，纯选区移动不记录
2. **历史无上限**：revisions 向量无限增长，长时间编辑会消耗大量内存
3. **必须存储 inversion**：因为 Delete 操作不保存被删文本，undo 需要 inversion 来恢复
4. **分支丢失**：`last_child` 只保留最新分支，旧分支只能通过 earlier/later 的线性索引访问，无法通过 redo 回到
5. **ChangeSet 追加式合并**：Insert 模式下所有连续编辑（包括错误输入和删除）都被 compose 为一个"等效"ChangeSet。撤销时只能整体撤销，无法回到中间态（例如撤销打字但保留粘贴内容，若粘贴和打字没被边界切开）。这是边界划分机制的设计取舍。
