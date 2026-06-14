# LSP 增量同步机制深度解析

本文基于 Helix 编辑器源码，逐层拆解"文本变更如何转换为 LSP 增量协议格式、如何排队发送、如何保持版本一致"这一完整链路。

## 目录

1. [概述：三种同步模式](#1-概述三种同步模式)
2. [端到端数据流](#2-端到端数据流)
3. [第一步：文本变更的表示——ChangeSet 与 Operation](#3-第一步文本变更的表示changeset-与-operation)
4. [第二步：Document.apply() 应用事务并递增版本号](#4-第二步documentapply-应用事务并递增版本号)
5. [第三步：事件分发——DocumentDidChange](#5-第三步事件分发documentdidchange)
6. [第四步：LSP Handler 响应事件](#6-第四步lsp-handler-响应事件)
7. [第五步：增量转换——changeset_to_changes() 核心算法](#7-第五步增量转换changeset_to_changes-核心算法)
8. [第六步：发送通知——Client.notify() 与 Transport 排队](#8-第六步发送通知clientnotify-与-transport-排队)
9. [版本一致性保证](#9-版本一致性保证)
10. [幽灵事务——不通知 LSP 的临时变更](#10-幽灵事务不通知-lsp-的临时变更)
11. [反向链路：从 LSP 编辑回写到文档](#11-反向链路从-lsp-编辑回写到文档)
12. [关键代码引用索引](#12-关键代码引用索引)

---

## 1. 概述：三种同步模式

LSP 协议定义了 [TextDocumentSyncKind](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L1768-L1790) 枚举，控制编辑器如何将文本变更同步给语言服务器：

| 值 | 含义 | 传输内容 |
|----|------|----------|
| `NONE (0)` | 不同步 | 无 |
| `FULL (1)` | 全量同步 | 每次变更发送完整文档内容 |
| `INCREMENTAL (2)` | 增量同步 | 只发送变更的范围和新文本 |

Helix 同时支持 `FULL` 和 `INCREMENTAL`，由 [Client::text_document_did_change()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L1054-L1097) 根据服务器声明的能力自动选择。增量模式的优势：带宽小、服务端无需重新解析整篇文档、更适合高频编辑。

---

## 2. 端到端数据流

```
用户按键
   ↓
构造 Transaction（内含 ChangeSet + 可选 Selection）
   ↓
Document::apply()                         ← document.rs:1654
   ├─ apply_impl()                        ← document.rs:1435
   │   ├─ ChangeSet::apply(&mut self.text)  更新 Rope 文本
   │   ├─ self.version += 1                 递增版本号
   │   ├─ 映射 selection / diagnostic / inlay hint 位置
   │   └─ helix_event::dispatch(DocumentDidChange { doc, old_text, changes, ghost_transaction })
   ↓
LSP Handler 钩子响应                       ← handlers/lsp.rs:407
   ↓
Client::text_document_did_change()        ← client.rs:1054
   ├─ 检查服务器 sync capability
   ├─ INCREMENTAL → changeset_to_changes()  ← client.rs:944
   ├─ FULL → 整篇文档字符串
   └─ Client::notify::<DidChangeTextDocument>()  ← client.rs:1092
         ↓
Client::notify()                          ← client.rs:502
   └─ server_tx.send(Payload::Notification(...))  放入无界 channel
         ↓
Transport::send() 异步任务                  ← transport.rs:338
   ├─ 初始化前 → pending_messages 队列缓存
   └─ 初始化后 → JSON-RPC over stdio 发给语言服务器进程
         ↓
语言服务器
```

---

## 3. 第一步：文本变更的表示——ChangeSet 与 Operation

### Operation 枚举

[Operation](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L13-L20) 是变更的最小单元：

```rust
pub enum Operation {
    Retain(usize),     // 跳过 n 个字符（保持不变）
    Delete(usize),     // 删除 n 个字符
    Insert(Tendril),   // 在当前位置插入文本
}
```

三个操作按顺序排列，构成一个"编辑脚本"（edit script）。关键规则：
- `Retain` 和 `Delete` 都消耗旧文档的字符数
- `Insert` 不消耗旧文档字符，但会增加新文档位置
- `Insert` 后紧跟 `Delete` 表示"替换"

### ChangeSet 结构

[ChangeSet](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L25-L32) 是 Operation 的有序集合：

```rust
pub struct ChangeSet {
    pub(crate) changes: Vec<Operation>,
    len: usize,         // 旧文档长度（用于校验）
    len_after: usize,   // 新文档长度
}
```

`len` 字段是安全守卫：[apply()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L342-L366) 方法会先检查 `text.len_chars() != self.len`，长度不匹配则拒绝应用，防止错位。

### Transaction

[Transaction](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L578-L583) = ChangeSet + 可选的新 Selection：

```rust
pub struct Transaction {
    changes: ChangeSet,
    selection: Option<Selection>,
}
```

---

## 4. 第二步：Document.apply() 应用事务并递增版本号

[Document::apply()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1654-L1656) 是编辑器修改文本的唯一入口：

```rust
pub fn apply(&mut self, transaction: &Transaction, view_id: ViewId) -> bool {
    self.apply_inner(transaction, view_id, true)  // emit_lsp_notification = true
}
```

[apply_inner()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1628-L1652) 做两件事：
1. 调用 `apply_impl()` 实际修改文本
2. 将此次事务的 ChangeSet 与 `self.changes`（累积的未保存变更）做 compose 合并

[apply_impl()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1435-L1626) 的关键步骤：

```
1. old_doc = self.text().clone()           ← 保存变更前的文本快照
2. changes.apply(&mut self.text)           ← 将 ChangeSet 应用到 Rope
3. self.version += 1                       ← 版本号递增（仅当 changes 非空时）
4. 映射所有 selection 位置
5. 映射所有 diagnostic 位置
6. 映射所有 inlay hint 位置
7. helix_event::dispatch(DocumentDidChange { doc, old_text, changes, ghost_transaction })
```

**核心要点**：`version` 递增发生在 `dispatch` 之前，因此事件处理器拿到的 `doc.version()` 已经是新版本号。

---

## 5. 第三步：事件分发——DocumentDidChange

[DocumentDidChange](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/events.rs#L12-L18) 是通过 `helix_event::events!` 宏声明的事件：

```rust
DocumentDidChange<'a> {
    doc: &'a mut Document,
    view: ViewId,
    old_text: &'a Rope,       // 变更前的完整文本
    changes: &'a ChangeSet,   // 此次变更集
    ghost_transaction: bool   // 是否为"幽灵事务"（不通知 LSP）
}
```

`dispatch` 宏会遍历所有通过 `register_hook!` 注册的钩子函数，同步调用它们。这意味着事件处理是在 apply_impl 内部、主线程上同步完成的。

---

## 6. 第四步：LSP Handler 响应事件

[register_hooks()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L386-L431) 中注册了 `DocumentDidChange` 的钩子：

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    if !event.ghost_transaction {
        for language_server in event.doc.language_servers() {
            language_server.text_document_did_change(
                event.doc.versioned_identifier(),  // 包含 uri + version
                event.old_text,                     // 旧文本 Rope
                event.doc.text(),                   // 新文本 Rope
                event.changes,                      // ChangeSet
            );
        }
    }
    Ok(())
});
```

**关键点**：
- 遍历该文档关联的所有语言服务器，逐一发送通知
- `ghost_transaction` 为 true 时跳过整个通知（见第 10 节）
- `versioned_identifier()` 返回 `VersionedTextDocumentIdentifier { uri, version }`，其中 version 就是已经递增后的值

---

## 7. 第五步：增量转换——changeset_to_changes() 核心算法

[changeset_to_changes()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L944-L1052) 是整个增量同步的算法核心，负责将 Helix 内部的 `ChangeSet` 转换为 LSP 协议要求的 `Vec<TextDocumentContentChangeEvent>`。

### LSP 协议要求的输出格式

[TextDocumentContentChangeEvent](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L2279-L2292)：

```rust
pub struct TextDocumentContentChangeEvent {
    pub range: Option<Range>,        // 变更范围（None = 全文替换）
    pub range_length: Option<u32>,   // 已废弃
    pub text: String,                // 新文本
}
```

其中 `Range` 由两个 `Position { line, character }` 组成。

### 算法核心难点

源码中的注释一语道破（[client.rs:959-962](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L959-L962)）：

> TextEdit describes changes to the initial doc (concurrent), but TextDocumentContentChangeEvent describes a series of changes (sequential). So S -> S1 -> S2, meaning positioning depends on the previous edits.

即：LSP 的 `TextDocumentContentChangeEvent` 是**顺序的**——每个 change 的 range 基于前一个 change 应用后的文档状态。这和 Helix 内部的 ChangeSet（基于原始文档的并发描述）不同。

### 算法流程

```
输入：old_text, new_text, changeset, offset_encoding
输出：Vec<TextDocumentContentChangeEvent>

初始化：old_pos = 0, new_pos = 0

遍历 changeset.changes 中的每个 Operation：

  Retain(n):
    old_pos += n
    new_pos += n
    （不产生 change）

  Delete(n):
    start = pos_to_lsp_pos(new_text, new_pos)        ← 基于**新文本**计算起始位置
    end = traverse(start, old_text[old_pos..old_pos+n]) ← 从 start 沿被删文本遍历得到终点
    产出 change: { range: start..end, text: "" }
    old_pos += n

  Insert(s):
    start = pos_to_lsp_pos(new_text, new_pos)        ← 基于**新文本**计算起始位置
    new_pos += s.chars().count()
    如果下一个操作是 Delete(len):  ← 合并为"替换"
      end = traverse(start, old_text[old_pos..old_pos+len])
      产出 change: { range: start..end, text: s }
      old_pos += len
      消费掉 Delete
    否则:                         ← 纯"插入"
      产出 change: { range: start..start, text: s }
```

### traverse 辅助函数

[traverse()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L966-L995) 从给定的 LSP Position 出发，遍历一段 RopeSlice，逐字符更新 line/character：

```rust
fn traverse(pos: lsp::Position, text: RopeSlice, offset_encoding: OffsetEncoding) -> lsp::Position {
    let mut line = pos.line;
    let mut character = pos.character;
    for ch in text.chars() {
        if ch == '\n' || ch == '\r' {
            // 处理 \r\n
            line += 1;
            character = 0;
        } else {
            character += match offset_encoding {
                Utf8  => ch.len_utf8(),
                Utf16 => ch.len_utf16(),
                Utf32 => 1,
            };
        }
    }
    Position { line, character }
}
```

**为什么 range 的位置基于新文本计算？** 因为 LSP 的 `TextDocumentContentChangeEvent` 是顺序的：服务器会按顺序应用这些 change，所以每个 change 的 range 应该在前一个 change 应用后的文档上计算。而 Helix 传入的 `new_text` 正是所有变更应用后的最终文本，用 `new_pos` 索引进去就自然得到了"当前变更在最新文档中的位置"。这确保了顺序语义的正确性。

### pos_to_lsp_pos 位置编码

[pos_to_lsp_pos()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/lib.rs#L223-L251) 将文档中的字符偏移量转为 LSP Position，支持三种偏移编码：

| 编码 | character 含义 | 使用场景 |
|------|---------------|----------|
| UTF-8 | 字节偏移 | 现代服务器（如 rust-analyzer） |
| UTF-16 | UTF-16 码元偏移 | LSP 默认，VS Code 传统 |
| UTF-32 | 字符数偏移 | 等价于"字素"偏移 |

编码选择由服务器在初始化响应中的 `positionEncoding` 决定，客户端通过 [Client::offset_encoding()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L410-L424) 获取。

---

## 8. 第六步：发送通知——Client.notify() 与 Transport 排队

### Client.notify()

[notify()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L502-L533) 将参数序列化为 JSON-RPC Notification，通过无界 channel `server_tx` 发送：

```rust
pub fn notify<R: Notification>(&self, params: R::Params) {
    let notification = jsonrpc::Notification {
        jsonrpc: Some(Version::V2),
        method: R::METHOD.to_string(),
        params: Self::value_into_params(params),
    };
    server_tx.send(Payload::Notification(notification));
}
```

`server_tx` 是 `UnboundedSender<Payload>`——**无界通道**，意味着 notify 调用永远不会阻塞。消息立即入队，由 Transport 的异步发送任务取出处理。

### Transport 发送循环

[Transport::send()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/transport.rs#L338-L441) 是一个 tokio 异步任务，维护一个 `pending_messages: Vec<Payload>` 缓冲区：

```
loop {
    tokio::select! {
        // 分支 1：收到初始化完成信号
        notified => {
            is_pending = false;
            排空 pending_messages 队列，逐条发送给服务器进程
        }

        // 分支 2：从 client_rx 收到新消息
        msg = client_rx.recv() => {
            if is_pending {
                if 是 shutdown → 退出
                if 是 notification → 丢弃（初始化前的通知直接忽略）
                否则 → 放入 pending_messages 等待初始化后发送
            } else {
                直接 send_payload_to_server() 写入服务器 stdin
            }
        }
    }
}
```

**关键行为**：
- 服务器初始化完成前，所有 `didChange` 通知会被**直接丢弃**（notification 分支被 continue）
- 请求类消息（如 `textDocument/completion`）则被缓存在 `pending_messages` 中，初始化后统一发送
- 初始化完成后，所有消息通过 `send_payload_to_server()` 直接写入子进程 stdin，格式为 JSON-RPC over stdio（带 `Content-Length` 头）

### send_payload_to_server

[send_payload_to_server()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/transport.rs#L160-L179) 将 Payload 序列化为 JSON，写入子进程 stdin：

```
Content-Length: {len}\r\n\r\n{json}
```

写入后立即 `flush()`，确保消息及时送达。

---

## 9. 版本一致性保证

版本号是 LSP 增量同步的"一致性锚点"。

### 版本号递增

[Document](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L196) 中 `version: i32`，初始值为 0。在 [apply_impl()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1464) 中：

```rust
self.version += 1;  // 仅当 changes 非空时执行
```

### 版本号发送

[versioned_identifier()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L2110-L2112) 构造 `VersionedTextDocumentIdentifier`：

```rust
pub fn versioned_identifier(&self) -> lsp::VersionedTextDocumentIdentifier {
    lsp::VersionedTextDocumentIdentifier::new(self.url().unwrap(), self.version)
}
```

LSP 协议规定：`DidChangeTextDocumentParams.text_document.version` 指向**所有 content_changes 应用之后**的文档版本（参见 [DidChangeTextDocumentParams](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L2266-L2273) 的注释）。由于 `version` 递增发生在 dispatch 之前，handler 拿到的 version 正好就是"应用后"的版本号，语义正确。

### 诊断版本对齐

服务器发回的 `textDocument/publishDiagnostics` 通知也携带 version。在 [handle_lsp_diagnostics()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L279-L296) 中：

```rust
if let Some((version, doc)) = version.zip(doc.as_ref()) {
    if version != doc.version() {
        log::info!("Version ({version}) is out of date ... dropping PublishDiagnostic notification");
        return;
    }
}
```

如果服务器返回的诊断版本号与当前文档版本不一致，说明诊断已过时，直接丢弃。这是防止"旧诊断覆盖新状态"的关键保护。

### 反向编辑的版本校验

当服务器发来 `workspace/applyEdit` 请求时，[apply_text_edits()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L82-L131) 也会校验版本：

```rust
if let Some(version) = version {
    if version != doc.version() {
        return Err(ApplyEditErrorKind::DocumentChanged);
    }
}
```

如果版本不匹配（文档在请求发出后又被修改了），拒绝应用编辑，返回 `DocumentChanged` 错误。

### 为什么不用去抖或批量合并？

与某些编辑器（如 VS Code）会对 `didChange` 通知做去抖/合并不同，Helix 的做法是**每次事务立即发送**，不做合并。这得益于：

1. **同步 dispatch**：事件钩子在 `apply_impl` 内同步调用，保证顺序
2. **无界 channel**：`notify()` 不会阻塞编辑器主循环
3. **版本号单调递增**：即使多条通知在网络中乱序到达，服务器也能通过版本号判断先后

---

## 10. 幽灵事务——不通知 LSP 的临时变更

[Document::apply_temporary()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1658-L1663) 允许应用事务但不通知语言服务器：

```rust
pub fn apply_temporary(&mut self, transaction: &Transaction, view_id: ViewId) -> bool {
    self.apply_inner(transaction, view_id, false)  // emit_lsp_notification = false
}
```

当 `emit_lsp_notification = false` 时，dispatch 的事件中 `ghost_transaction = true`：

```rust
helix_event::dispatch(DocumentDidChange {
    ghost_transaction: !emit_lsp_notification,  // ← true
    ...
});
```

LSP Handler 中检查此标志：

```rust
if !event.ghost_transaction {
    // 只有非幽灵事务才发送 didChange
    for language_server in event.doc.language_servers() {
        language_server.text_document_did_change(...);
    }
}
```

**但注意**：即使幽灵事务不通知 LSP，`version` 仍然会递增！这意味着幽灵事务会产生"版本号跳跃"——服务器看到的版本号可能不连续。LSP 协议允许版本号不连续（参见 [VersionedTextDocumentIdentifier](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L970-L980) 的注释："The number doesn't need to be consecutive"），所以这不会导致协议错误。

---

## 11. 反向链路：从 LSP 编辑回写到文档

增量同步是双向的。当语言服务器通过 `workspace/applyEdit` 请求修改文档时：

1. [apply_text_edits()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L82-L131) 校验版本号
2. [generate_transaction_from_edits()](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/lib.rs#L410-L477) 将 LSP TextEdit 转换回 Transaction
3. `doc.apply(&transaction, view.id)` 应用事务
4. 应用后又会触发 `DocumentDidChange`，再次通过 `didChange` 通知服务器

为避免无限循环，服务器不应该在收到自己发起的编辑后再发回相同的编辑——这由 LSP 协议语义保证。

`generate_transaction_from_edits` 还有一个优化：如果只有一个编辑且覆盖整个文档（全文档替换），它会调用 `compare_ropes()` 做 diff，生成更精细的 ChangeSet 而非暴力替换整篇文本。

---

## 12. 关键代码引用索引

| 环节 | 文件 | 行号 | 函数/结构 |
|------|------|------|-----------|
| Operation 枚举 | [transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L13-L20) | L13-L20 | `enum Operation` |
| ChangeSet 结构 | [transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L25-L32) | L25-L32 | `struct ChangeSet` |
| ChangeSet::apply | [transaction.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-core/src/transaction.rs#L342-L366) | L342-L366 | `fn apply()` |
| Document::apply | [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1654-L1656) | L1654-L1656 | `fn apply()` |
| Document::apply_impl | [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1435-L1626) | L1435-L1626 | `fn apply_impl()` |
| version 递增 | [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1464) | L1464 | `self.version += 1` |
| version 字段 | [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L196) | L196 | `version: i32` |
| DocumentDidChange 事件 | [events.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/events.rs#L12-L18) | L12-L18 | `DocumentDidChange` |
| LSP Handler 钩子 | [handlers/lsp.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L407-L421) | L407-L421 | `register_hook!` |
| text_document_did_change | [client.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L1054-L1097) | L1054-L1097 | `fn text_document_did_change()` |
| changeset_to_changes | [client.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L944-L1052) | L944-L1052 | `fn changeset_to_changes()` |
| traverse 辅助函数 | [client.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L966-L995) | L966-L995 | `fn traverse()` |
| pos_to_lsp_pos | [lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/lib.rs#L223-L251) | L223-L251 | `fn pos_to_lsp_pos()` |
| Client::notify | [client.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/client.rs#L502-L533) | L502-L533 | `fn notify()` |
| Transport::send 循环 | [transport.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/transport.rs#L338-L441) | L338-L441 | `async fn send()` |
| handle_lsp_diagnostics | [handlers/lsp.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L279-L365) | L279-L296 | 版本校验 |
| apply_text_edits | [handlers/lsp.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/handlers/lsp.rs#L82-L131) | L82-L131 | 反向编辑+版本校验 |
| generate_transaction_from_edits | [lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp/src/lib.rs#L410-L477) | L410-L477 | LSP 编辑 → Transaction |
| apply_temporary | [document.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-view/src/document.rs#L1658-L1663) | L1658-L1663 | 幽灵事务入口 |
| TextDocumentSyncKind | [lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L1768-L1790) | L1768-L1790 | 同步模式枚举 |
| VersionedTextDocumentIdentifier | [lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L970-L980) | L970-L980 | 版本号+URI |
| TextDocumentContentChangeEvent | [lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/276-helix/helix-lsp-types/src/lib.rs#L2279-L2292) | L2279-L2292 | 增量变更事件 |
