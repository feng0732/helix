# Helix 撤销历史机制源码分析

## 一、核心数据结构

### 1. History（撤销树）

定义于 [history.rs](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L51-L54)：

```rust
pub struct History {
    revisions: Vec<Revision>,  // 所有修订版本，线性存储
    current: usize,            // 当前修订版本的索引
}
```

History 并非简单的栈，而是一棵**线性化存储的树**。每个 Revision 知道自己的父节点（`parent`）和最后一个子节点（`last_child`），支持分支式的 undo/redo 路径。

### 2. Revision（单个历史节点）

定义于 [history.rs](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L57-L66)：

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

History 初始化时创建一个虚拟根 Revision（[history.rs](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L68-L82)），其 transaction 和 inversion 均为空 ChangeSet，parent 指向自身（0），无子节点。`current` 初始为 0。

---

## 二、编辑事务进入历史栈的流程

### 阶段一：事务应用时 —— 累积到 `changes` ChangeSet

当编辑操作发生时（如 `insert_char`、`delete_char_backward`），事务通过 `Document::apply()` → `apply_inner()` 应用到文档：

```rust
// document.rs L1628-L1651
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

这意味着在 Insert 模式下，连续的按键操作不会逐个进入历史栈，而是通过 ChangeSet 的 `compose` 操作合并成一个大的 ChangeSet。例如连续输入 "hello" 五次 insert 事务，compose 后等价于一个 `Insert("hello")` 操作（见 [transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/transaction.rs#L1116-L1154) 的 optimized_composition 测试）。

### 阶段二：提交到历史栈 —— `append_changes_to_history`

当满足"合并边界"条件时，调用 [append_changes_to_history](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-view/src/document.rs#L1787-L1808)：

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

[commit_revision](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L85-L110)：

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

### 主要触发点

| 触发场景 | 位置 | 说明 |
|---|---|---|
| **键事件处理后（非 Insert 模式）** | [editor.rs L1561-L1565](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/ui/editor.rs#L1561-L1565) | 每次按键命令执行后，若不在 Insert 模式，立即提交。**这是退出 Insert 模式时提交变化的关键路径**——ESC 触发 `normal_mode`，模式切换后下一行判断 `mode != Insert`，立即提交。 |
| **Paste 事件（非 Insert 模式）** | [editor.rs L1462-L1466](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/ui/editor.rs#L1462-L1466) | 粘贴后若不在 Insert 模式则提交 |
| **Undo 操作前** | [document.rs L1667](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-view/src/document.rs#L1667) | `undo_redo_impl` 中，undo 前先提交 pending changes |
| **Earlier 操作前** | [document.rs L1752](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-view/src/document.rs#L1752) | `earlier_later_impl` 中，earlier 前先提交 |
| **Redo / Later 操作前（有 pending changes）** | [document.rs L1668-L1669](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-view/src/document.rs#L1668-L1669) | 若有 pending changes 则 redo 直接返回 false，避免不一致 |
| **Paste（Insert 模式）** | [commands.rs L4922-L4924](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L4922-L4924) | Insert 模式下粘贴前先提交，使粘贴成为独立 undo 单元 |
| **补全（Completion）确认** | [completion.rs L214](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/ui/completion.rs#L214) | 补全确认前先提交，使补全成为独立 undo 单元 |
| **Jump / Push jump** | [commands.rs L3958](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L3958) | 跳转前提交 |
| **保存文件前** | [typed.rs L398](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands/typed.rs#L398) | `:write` 前提交 |
| **格式化** | [commands.rs L3781](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L3781) | 格式化应用后提交 |
| **手动 checkpoint** | [commands.rs L4785-L4788](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L4785-L4788) | `commit_undo_checkpoint` 命令 |
| **Picker 回调后（非 Insert 模式）** | [commands.rs L3648-L3650](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L3648-L3650) | Picker 选择执行后若非 Insert 模式则提交 |
| **替换命令后** | [commands.rs L6697](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-term/src/commands.rs#L6697) | `:` 替换执行后提交 |

### 合并逻辑总结

1. **Insert 模式内**：所有编辑（打字、退格等）仅 compose 到 `changes` 中，不提交历史。整个 Insert 会话（从 `i`/`a` 到 `Esc`）的所有变更合并为一个 undo 单元。
2. **Normal 模式下**：每个命令执行后立即提交，每个命令是独立的 undo 单元。
3. **特殊情况打断合并**：如 Insert 模式下的粘贴、补全确认会先提交再执行，形成独立的 undo 单元。

---

## 四、回退恢复（Undo / Redo）

### 基础 Undo / Redo

[undo](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L138-L146) 和 [redo](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L148-L155)：

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

Document 层的 [undo_redo_impl](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-view/src/document.rs#L1665-L1687)：

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
        self.apply_impl(txn, view.id, true)
    } else { false };
    self.history.set(history);
    if success {
        self.changes = ChangeSet::new(self.text().slice(..));  // 重置 pending
        view.sync_changes(self);  // 同步跳转列表
    }
    success
}
```

**注意**：undo/redo 使用 `apply_impl` 而非 `apply`，跳过了 `apply_inner` 中的 changes 累积和 old_state 保存逻辑。undo/redo 后直接重置 `changes`。

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

[earlier](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L287-L302) 和 [later](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L296-L302) 支持两种模式：

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

[jump_to](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L213-L224) 是底层跳转机制：

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

LCA 查找算法（[lowest_common_ancestor](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L183-L199)）使用双路径集合交替上升法，类似链表找交点。

### changes_since

[changes_since](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L124-L135) 返回从指定修订到当前修订的合成事务，通过 compose 将所有变更合并为单个 Transaction。用于 LSP 等需要获取增量变更的场景。

---

## 五、ChangeSet 的 compose 与 invert

### compose（合并变更）

[ChangeSet::compose](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/transaction.rs#L163-L293) 将两个连续的 ChangeSet 合并为一个。前提：`self.len_after == other.len`（第一个的输出长度等于第二个的输入长度）。

compose 通过双指针逐一消费两个 ChangeSet 的 Operation 序列，处理 Retain/Delete/Insert 的各种组合情况，生成等价的单一 ChangeSet。这是实现"Insert 模式下连续编辑合并为一个 undo 单元"的基础。

### invert（生成反转）

[ChangeSet::invert](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/transaction.rs#L312-L339) 根据**原始文档**生成反转 ChangeSet：

- `Retain(n)` → `Retain(n)`（保留不变）
- `Delete(n)` → `Insert(原始文档对应位置的文本)`（恢复被删内容）
- `Insert(s)` → `Delete(s.chars().count())`（删除插入的内容）

由于 Delete 操作不存储被删文本，invert 必须访问原始文档才能恢复。这也是为什么 History 的 Revision 需要存储 inversion 而非每次重新计算。

---

## 六、完整流程示例

### 场景：Insert 模式输入 "abc" 后 Esc

1. **按 `i`**：进入 Insert 模式，不提交历史
2. **按 `a`**：
   - `insert_char` 生成 Transaction `Insert("a")`
   - `apply_inner` 中：`changes` 从空变非空 → 保存 `old_state`
   - compose：`changes = Insert("a")`
3. **按 `b`**：
   - `insert_char` 生成 Transaction `Insert("b")`（位置偏移 +1）
   - compose：`changes = Insert("a") ∘ (Retain(1), Insert("b"))` = `Insert("ab")`
4. **按 `c`**：类似，`changes` 变为 `Insert("abc")`
5. **按 `Esc`**：
   - `normal_mode` 将 mode 设为 Normal
   - 键事件处理后，判断 `mode != Insert` → 调用 `append_changes_to_history`
   - 从 `changes` 构造 Transaction，用 `old_state` 生成 inversion
   - `History::commit_revision` 创建新 Revision，`current` 前进
6. **按 `u`（undo）**：
   - 先 `append_changes_to_history`（此时 changes 为空，跳过）
   - `history.undo()` 返回 inversion（`Delete(3)`）
   - 应用 inversion，文档恢复为 "abc" 之前的状态
   - 重置 `changes`

---

## 七、已知限制

源码注释中提到的局限（[history.rs](file:///d:/fz/0601/solo-dogfeeding/code/274-helix/helix-core/src/history.rs#L40-L48)）：

1. **选区变更不产生历史记录**：仅 buffer 内容变更会提交历史，纯选区移动不记录
2. **历史无上限**：revisions 向量无限增长，长时间编辑会消耗大量内存
3. **必须存储 inversion**：因为 Delete 操作不保存被删文本，undo 需要 inversion 来恢复
4. **分支丢失**：`last_child` 只保留最新分支，旧分支只能通过 earlier/later 的线性索引访问，无法通过 redo 回到
