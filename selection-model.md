# Selection Model & Zero-Width Insertion

## 1. Zero-Width Insertion (零宽插入) 的定义与形成

**零宽插入**即 `from == to` 的变更：在某个位置插入文本，但不删除任何原有字符。

### 1.1 构造入口

```rust
// transaction.rs#L866-L870
pub fn insert(doc: &Rope, selection: &Selection, text: Tendril) -> Self {
    Self::change_by_selection(doc, selection, |range| {
        (range.head, range.head, Some(text.clone()))
    })
}
```

[Transaction::insert](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L866-L870)
为每个选区范围生成 `Change = (range.head, range.head, Some(text))`，即 `from == to == range.head`。

### 1.2 ChangeSet 构建时的 span=0 处理

[ChangeSet::from_changes](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L531-L568) 逐个处理 `Change`：

```rust
for (from, to, tendril) in changes {
    changeset.retain(from - last);
    let span = to - from;   // 零宽插入时 span = 0
    match tendril {
        Some(text) => {
            changeset.insert(text);  // Insert("text")
            changeset.delete(span);  // span = 0 → delete(0)：**不写入任何 Delete Operation**
        }
        None => changeset.delete(span),
    }
    last = to;
}
```

[ChangeSet::delete](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L122) 内部当 `len == 0` 时直接 return，不 push `Operation::Delete(0)`。因此零宽插入生成的 Operation 序列只有 `Insert(text)`，没有尾随 `Delete`。

### 1.3 Operation 存储形式对比

| 场景 | Change (from, to, tendril) | span | Operations 序列 |
|---|---|---|---|
| 纯插入（零宽） | (10, 10, Some("abc")) | 0 | `Retain(10) → Insert("abc") → Retain(…)` |
| 选区替换 | (10, 15, Some("abc")) | 5 | `Retain(10) → Insert("abc") → Delete(5) → Retain(…)` |
| 纯删除 | (10, 15, None) | 5 | `Retain(10) → Delete(5) → Retain(…)` |

## 2. ChangeIterator：读取变更时的两条分支

[ChangeIterator::next](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L898-L929) 反向遍历 Operation 还原 `Change`：

```rust
match self.iter.next()? {
    Retain(len) => self.pos += len,
    Delete(len) => {
        let start = self.pos;
        self.pos += len;
        return Some((start, self.pos, None));  // 纯删除
    }
    Insert(s) => {
        let start = self.pos;
        if let Some(Delete(len)) = self.iter.peek() {
            self.iter.next();
            self.pos += len;
            return Some((start, self.pos, Some(s.clone()))); // 替换分支
        } else {
            return Some((start, start, Some(s.clone())));    // 纯插入分支
        }
    }
}
```

**关键判定**：`Insert` 后面是否跟随 `Delete`。
- 有 `Delete` → `to = pos + len > start` → **替换分支**（from < to）
- 无 `Delete`（peek 为空或不是 Delete）→ `to = start` → **纯插入分支**（from == to）

**零宽插入走纯插入分支的必然原因**：构造 ChangeSet 时 `delete(0)` 不产生 Operation，所以 `Insert` 后面没有 `Delete`，`peek()` 得到的是下一个 `Retain` 或其他非 Delete Operation，触发纯插入分支。即使非零宽插入场景（如 span 非 0）只要 `delete(span)` 没写 Delete（例如 span 刚好 0 的等价情况）同样走纯插入分支。

## 3. 纯插入 vs 选区替换：对文本变更的影响

### 3.1 文本结果差异

| 场景 | 原文档位置 10-15 | 变更 | 新位置 10 起内容 | 净长度变化 |
|---|---|---|---|---|
| 纯插入 | "hello" | (10,10,Some("abc")) | "abchello" | +3 |
| 选区替换 | "hello" | (10,15,Some("abc")) | "abc" | -2（删除 5 + 插入 3） |
| 纯删除 | "hello" | (10,15,None) | ""（空） | -5 |

### 3.2 选区映射差异

文本映射通过 `ChangeSet::map_pos` 处理，`Assoc` 策略影响光标落点：

- **纯插入**：`from == to == start`，插入点之前（`pos < start`）不变，插入点及之后（`pos >= start`）按 `Assoc` 偏移 +inserted_len
- **选区替换**：`from < to`，区间内的位置按 `Assoc` 映射到新区间端点；区间外的位置按净长度差（inserted_len - deleted_len）偏移

### 3.3 LSP 通知差异

- [apply](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1654)（emit_lsp_notification=true）：向 LSP 发送 `textDocument/didChange`，服务端感知到完整变更（含删除范围 + 插入文本）
- [apply_temporary](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1661)（emit_lsp_notification=false）：不通知 LSP，用于补全弹层预览等临时场景

## 4. 纯插入 vs 选区替换：对撤销记录的影响

### 4.1 撤销记录写入链路

[Document::apply_inner](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1628-L1652) 积累变更到 `self.changes`：

```rust
if self.changes.is_empty() && !transaction.changes().is_empty() {
    self.old_state = Some(State { doc, selection });  // 首次变更时记录原始状态快照
}
if !transaction.changes().is_empty() {
    self.changes.compose(transaction.changes().clone()); // 多次编辑合并为一个 ChangeSet
}
```

[Document::append_changes_to_history](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1787-L1808) 提交到 History：

```rust
if self.changes.is_empty() { return; }  // 无实际文本变更则不写历史
let transaction = Transaction::from(changes).with_selection(current_selection);
history.commit_revision(&transaction, &old_state);  // 同时存正向 + 反向 transaction
```

### 4.2 空 ChangeSet 不产生撤销记录

当 `ChangeSet::is_empty()` 返回 true 时（例如 Transaction 只有 selection 更新、没有文本 Operation）：
- `apply_inner` 跳过 `old_state` 记录和 `self.changes.compose`
- `append_changes_to_history` 直接 return，不写入 History

### 4.3 纯插入与选区替换在撤销中的不同行为

| 维度 | 纯插入 (from == to, Some(text)) | 选区替换 (from < to, Some(text)) | 纯删除 (from < to, None) |
|---|---|---|---|
| Change 正向 | Insert(text) | Insert(text) + Delete(span) | Delete(span) |
| Invert 反向（撤销时） | Delete(inserted_len) | Delete(inserted_len) + Insert(original_text) | Insert(original_text) |
| History.inversion 存储内容 | 只需要删除插入文本，无需存原文（未删字符） | 必须同时存被删除的原文，撤销时恢复 | 必须存被删除的原文 |
| 撤销后选区落点 | 回到插入点（from == to） | 回到原选区范围（from < to），恢复选区宽度 | 回到删除区间起点 |
| 反向 transaction.changes() 判空 | 非空（必有 Delete） | 非空 | 非空 |

**正向与反向的具体构建**（通过 [Transaction::invert](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L608-L616) → [ChangeSet::invert](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/transaction.rs#L261)）：

```
正向操作（纯插入）: Retain(10) + Insert("abc")
    ↓ invert
反向操作（撤销）  : Retain(10) + Delete(3)         // 长度差：-3

正向操作（选区替换）: Retain(10) + Insert("abc") + Delete(5)
    ↓ invert
反向操作（撤销）    : Retain(10) + Delete(3) + Insert("hello")  // 需保留原文 "hello"
```

### 4.4 连续编辑合并为单个撤销单元

在 `apply_inner` 中，只要 `self.changes` 未被 `append_changes_to_history` 清空，连续的 apply 会被 `compose` 合并：
- 先纯插入 `(10,10,Some("ab"))`，再纯插入 `(12,12,Some("cd"))`，被合并为 `(10,10,Some("abcd"))` 作为一个撤销单元
- 先选区替换 `(10,15,Some("abc"))`，再选区替换 `(10,13,Some("xyz"))`，被合并为 `(10,15,Some("xyz"))` 作为一个撤销单元

合并后撤销只需 undo 一次即回到合并前的状态。

## 5. Savepoint 机制与 apply 模式

### 5.1 Savepoint 创建与恢复

[Document::savepoint](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1704-L1727) 记录当前文档快照：
- `revert` 初始化为 `Transaction::new(doc).with_selection(selection)`（空 transaction）
- 后续 `apply_inner` 时（[document.rs#L1481-L1494](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1481-L1494)），每次 transaction 都会被 **invert 后 compose 进 savepoint.revert**，实现累积回滚

[Document::restore](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/document.rs#L1730-L1765) 应用 revert transaction，同时也将 revert 合入 `self.changes`（用于后续 `append_changes_to_history` 提交到撤销栈作为一个单元）。

### 5.2 Apply vs ApplyTemporary

| 维度 | apply (emit_lsp=true) | apply_temporary (emit_lsp=false) |
|---|---|---|
| LSP 通知 | 发送 didChange | 不发送 |
| 文本变更 | 实际修改 doc | 实际修改 doc |
| self.changes 累积 | 是 | 是 |
| savepoint.revert 累积 | 是 | 是 |
| 撤销记录（append_changes_to_history） | 正常写入 | 正常写入（取决于后续是否 append） |
| 典型场景 | 确认补全、普通编辑 | 弹层预览（ghost text 预览） |

补全流程中的 savepoint + apply_temporary 组合：
1. `savepoint = doc.savepoint(view)`
2. `doc.apply_temporary(insert_transaction)` → 插入预览文本，不通知 LSP
3. 用户确认 → `doc.restore(view, savepoint)` 撤销预览 → `doc.apply(confirm_transaction)` 正式确认并通知 LSP
4. 用户取消 → `doc.restore(view, savepoint)` 撤销预览，无 LSP 通知
