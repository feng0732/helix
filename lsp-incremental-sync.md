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
8. [消息排队与发送——两层队列机制](#8-消息排队与发送两层队列机制)
9. [版本一致性保证——三条链路三种对齐](#9-版本一致性保证三条链路三种对齐)
10. [幽灵事务——不通知 LSP 的临时变更](#10-幽灵事务不通知-lsp-的临时变更)
11. [反向链路：从 LSP 编辑回写到文档](#11-反向链路从-lsp-编辑回写到文档)
12. [关键代码引用索引](#12-关键代码引用索引)

---

## 1. 概述：三种同步模式

LSP 协议定义了 `TextDocumentSyncKind` 枚举（helix-lsp-types/src/lib.rs#L1768-L1790），控制编辑器如何将文本变更同步给语言服务器：

| 值 | 含义 | 传输内容 |
|----|------|----------|
| `NONE (0)` | 不同步 | 无 |
| `FULL (1)` | 全量同步 | 每次变更发送完整文档内容 |
| `INCREMENTAL (2)` | 增量同步 | 只发送变更的范围和新文本 |

Helix 同时支持 `FULL` 和 `INCREMENTAL`，由 `Client::text_document_did_change()`（helix-lsp/src/client.rs#L1054-L1097）根据服务器声明的能力自动选择。增量模式的优势：带宽小、服务端无需重新解析整篇文档、更适合高频编辑。

---

## 2. 端到端数据流

```
用户按键
   ↓
构造 Transaction（内含 ChangeSet + 可选 Selection）
   ↓
Document::apply()                                    ← helix-view/src/document.rs#L1654
   ├─ apply_impl()                                  ← helix-view/src/document.rs#L1435
   │   ├─ ChangeSet::apply(&mut self.text)          更新 Rope 文本
   │   ├─ self.version += 1                          递增版本号
   │   ├─ 映射 selection / diagnostic / inlay hint 位置
   │   └─ dispatch(DocumentDidChange)
   ↓
LSP Handler 钩子响应                                  ← helix-view/src/handlers/lsp.rs#L407
   ↓
Client::text_document_did_change()                   ← helix-lsp/src/client.rs#L1054
   ├─ 检查服务器 sync capability
   ├─ INCREMENTAL → changeset_to_changes()           ← helix-lsp/src/client.rs#L944
   ├─ FULL → 整篇文档字符串
   └─ Client::notify::<DidChangeTextDocument>()      ← helix-lsp/src/client.rs#L1092
         ↓
Client::notify()                                     ← helix-lsp/src/client.rs#L502
   └─ server_tx.send(Payload::Notification(...))     放入发送队列（无界 channel）
         ↓
Transport::send() 异步任务                             ← helix-lsp/src/transport.rs#L338
   ├─ 未初始化 → 通知丢弃 / 请求缓存在 pending_messages（初始化前缓存）
   └─ 已初始化 → send_payload_to_server() 写入 stdin  → 语言服务器
```

---

## 3. 第一步：文本变更的表示——ChangeSet 与 Operation

### Operation 枚举

`Operation`（helix-core/src/transaction.rs#L13-L20）是变更的最小单元：

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

`ChangeSet`（helix-core/src/transaction.rs#L25-L32）是 Operation 的有序集合：

```rust
pub struct ChangeSet {
    pub(crate) changes: Vec<Operation>,
    len: usize,         // 旧文档长度（用于校验）
    len_after: usize,   // 新文档长度
}
```

`len` 字段是安全守卫：`apply()`（helix-core/src/transaction.rs#L342-L366）方法会先检查 `text.len_chars() != self.len`，长度不匹配则拒绝应用，防止错位。

### Transaction

`Transaction`（helix-core/src/transaction.rs#L578-L583）= ChangeSet + 可选的新 Selection：

```rust
pub struct Transaction {
    changes: ChangeSet,
    selection: Option<Selection>,
}
```

---

## 4. 第二步：Document.apply() 应用事务并递增版本号

`Document::apply()`（helix-view/src/document.rs#L1654-L1656）是编辑器修改文本的唯一入口：

```rust
pub fn apply(&mut self, transaction: &Transaction, view_id: ViewId) -> bool {
    self.apply_inner(transaction, view_id, true)  // emit_lsp_notification = true
}
```

`apply_inner()`（helix-view/src/document.rs#L1628-L1652）做两件事：
1. 调用 `apply_impl()` 实际修改文本
2. 将此次事务的 ChangeSet 与 `self.changes`（累积的未保存变更）做 compose 合并

`apply_impl()`（helix-view/src/document.rs#L1435-L1626）的关键步骤：

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

`DocumentDidChange`（helix-view/src/events.rs#L12-L18）是通过 `helix_event::events!` 宏声明的事件：

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

`register_hooks()`（helix-view/src/handlers/lsp.rs#L386-L431）中注册了 `DocumentDidChange` 的钩子：

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

`changeset_to_changes()`（helix-lsp/src/client.rs#L944-L1052）是整个增量同步的算法核心，负责将 Helix 内部的 `ChangeSet` 转换为 LSP 协议要求的 `Vec<TextDocumentContentChangeEvent>`。

### LSP 协议要求的输出格式

`TextDocumentContentChangeEvent`（helix-lsp-types/src/lib.rs#L2279-L2292）：

```rust
pub struct TextDocumentContentChangeEvent {
    pub range: Option<Range>,        // 变更范围（None = 全文替换）
    pub range_length: Option<u32>,   // 已废弃
    pub text: String,                // 新文本
}
```

其中 `Range` 由两个 `Position { line, character }` 组成。

### 算法核心难点

源码中的注释一语道破（helix-lsp/src/client.rs#L959-L962）：

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

`traverse()`（helix-lsp/src/client.rs#L966-L995）从给定的 LSP Position 出发，遍历一段 RopeSlice，逐字符更新 line/character：

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

`pos_to_lsp_pos()`（helix-lsp/src/lib.rs#L223-L251）将文档中的字符偏移量转为 LSP Position，支持三种偏移编码：

| 编码 | character 含义 | 使用场景 |
|------|---------------|----------|
| UTF-8 | 字节偏移 | 现代服务器（如 rust-analyzer） |
| UTF-16 | UTF-16 码元偏移 | LSP 默认，VS Code 传统 |
| UTF-32 | 字符数偏移 | 等价于"字素"偏移 |

编码选择由服务器在初始化响应中的 `positionEncoding` 决定，客户端通过 `Client::offset_encoding()`（helix-lsp/src/client.rs#L410-L424）获取。

---

## 8. 消息排队与发送——两层队列机制

从 `Client.notify()` 到消息最终写入语言服务器的 stdin，中间经过**两层队列**，职责截然不同。

### 第一层：发送队列——Unbounded Channel

**位置**：`Client::start()` 中创建（helix-lsp/src/client.rs#L246），由 `server_tx` / `client_rx` 构成。

**类型**：`tokio::sync::mpsc::unbounded_channel()`——无界通道。

**职责**：在编辑器主线程和 Transport 异步发送任务之间传递消息。`Client::notify()`（helix-lsp/src/client.rs#L502-L533）将 JSON-RPC Notification 序列化后放入 `server_tx`，Transport 的 `send()` 任务从 `client_rx` 取出处理。

**特点**：
- 无界意味着 `notify()` 永远不会阻塞编辑器主循环，调用即返回
- 消息顺序与 `notify()` 调用顺序严格一致（FIFO）
- 所有类型的消息共用此通道：Notification（如 didChange）、Request（如 completion）、Response

```
编辑器主线程                      Transport 异步任务
    │                                  │
    │  server_tx.send(Payload)         │  client_rx.recv()
    │ ──────────────────────────────►  │
    │         Unbounded Channel        │
```

### 第二层：初始化前缓存——pending_messages Vec

**位置**：`Transport::send()` 异步任务内部（helix-lsp/src/transport.rs#L345），是一个局部变量 `pending_messages: Vec<Payload>`。

**职责**：在语言服务器完成初始化握手之前，暂存那些不能提前发送的消息。

**触发条件**：仅当 `is_pending == true`（即服务器尚未完成初始化）时生效。

**行为**：

```
Transport::send() 主循环（helix-lsp/src/transport.rs#L380-L440）

tokio::select! {
    // ── 分支 A：收到初始化完成信号 ──
    _ = &mut notified, if is_pending => {
        is_pending = false;
        // 注入 Initialized 通知给内部事件系统
        // 排空 pending_messages，逐条 send_payload_to_server()
    }

    // ── 分支 B：从发送队列收到消息 ──
    msg = client_rx.recv() => match msg {
        // 情况 1：初始化前收到 shutdown → 直接退出
        is_pending && is_shutdown  => break

        // 情况 2：初始化前收到非 initialize 消息
        is_pending && !is_initialize => {
            if 是 Notification → continue（直接丢弃，不缓存）
            if 是 Request     → pending_messages.push(msg)（缓存，等初始化后发送）
        }

        // 情况 3：初始化后 / 本身就是 initialize 消息
        _ => send_payload_to_server()（立即发送）
    }
}
```

**初始化前 Notification 被丢弃的原因**：在服务器完成 `initialize` 握手之前，它还不具备处理 `textDocument/didChange` 等通知的能力。此时缓存通知没有意义——文档在服务器初始化完成后会通过 `LanguageServerInitialized` 钩子重新发送 `textDocument/didOpen`（helix-view/src/handlers/lsp.rs#L387-L404），其中包含文档的最新全文，相当于一次全量同步，覆盖了之前丢弃的增量通知。

### 两层队列的协作关系

```
Client.notify()
    │
    ▼ 放入
┌──────────────────────┐
│  发送队列             │  Unbounded Channel
│  (server_tx/rx)      │  永不阻塞，保证主线程流畅
└──────────┬───────────┘
           │ client_rx.recv()
           ▼
┌──────────────────────┐
│  Transport::send()   │  异步任务
│                      │
│  is_pending?         │
│  ├─ YES:             │
│  │   Notification → 丢弃
│  │   Request → pending_messages（初始化前缓存）
│  └─ NO:              │
│      → send_payload_to_server()
│        写入 stdin     │
└──────────────────────┘
```

### send_payload_to_server

`send_payload_to_server()`（helix-lsp/src/transport.rs#L160-L179）将 Payload 序列化为 JSON，写入子进程 stdin：

```
Content-Length: {len}\r\n\r\n{json}
```

写入后立即 `flush()`，确保消息及时送达。

---

## 9. 版本一致性保证——三条链路三种对齐

版本号（`Document.version: i32`，helix-view/src/document.rs#L196）是 LSP 增量同步的一致性锚点。初始值为 0，每次非空变更后 +1（helix-view/src/document.rs#L1464）。三条链路各自有不同的版本对齐策略。

### 链路一：编辑通知（didChange）——版本号随通知发出

**方向**：编辑器 → 语言服务器

**版本携带方式**：`DidChangeTextDocumentParams.text_document` 是一个 `VersionedTextDocumentIdentifier { uri, version }`（helix-lsp-types/src/lib.rs#L970-L980）。

**版本语义**：LSP 协议规定此 version 指向**所有 content_changes 应用之后**的文档版本（helix-lsp-types/src/lib.rs#L2266-L2273 的注释）。在 Helix 中，`version` 递增发生在 `dispatch(DocumentDidChange)` 之前（helix-view/src/document.rs#L1464），因此 Handler 通过 `event.doc.versioned_identifier()`（helix-view/src/document.rs#L2110-L2112）拿到的 version 恰好是"应用后"的值，语义正确。

**对齐机制**：服务器收到 `didChange` 后，按 version 单调递增的顺序应用变更。如果服务器发现收到的 version 不大于它已知的最新版本，就知道这是一个过时的通知可以忽略。协议层面不要求版本号连续，只需单调递增——这对幽灵事务场景很重要（见第 10 节）。

**时序保证**：
1. `apply_impl()` 内部，version 递增和 dispatch 在同一个同步调用栈中完成
2. Handler 同步调用 `text_document_did_change()` → `notify()` → `server_tx.send()`
3. Transport 的 send 任务按 channel 接收顺序写入 stdin
4. 因此，写入 stdin 的 `didChange` 消息的 version 严格单调递增

```
Document.apply_impl()                  Transport::send()
       │                                    │
       ├─ version = N                        │
       ├─ dispatch → notify()               │
       │    └─ send(Payload{version=N})  ──► │  send_payload_to_server(version=N)
       │                                    │
       ├─ version = N+1                     │
       ├─ dispatch → notify()               │
       │    └─ send(Payload{version=N+1})──►│  send_payload_to_server(version=N+1)
```

### 链路二：诊断回包（publishDiagnostics）——版本比对，过时即丢弃

**方向**：语言服务器 → 编辑器

**版本携带方式**：`PublishDiagnosticsParams.version: Option<i32>`（helix-lsp-types/src/lib.rs#L2266-L2273）。服务器**可以**在诊断通知中附带 version，表示该诊断基于哪个文档版本计算得出。此字段可选——并非所有服务器都提供。

**对齐机制**：`handle_lsp_diagnostics()`（helix-view/src/handlers/lsp.rs#L279-L296）在处理诊断时：

```rust
if let Some((version, doc)) = version.zip(doc.as_ref()) {
    if version != doc.version() {
        log::info!("Version ({version}) is out of date ... dropping");
        return;
    }
}
```

- 如果服务器提供了 version，且与当前文档 version 不一致 → 整批诊断**直接丢弃**
- 如果服务器未提供 version（`None`） → `version.zip(doc)` 为 `None`，跳过校验，诊断照常处理
- 比对逻辑用 `version.zip(doc.as_ref())` 而非单独检查，巧妙处理了"服务器不发 version"和"文档未找到"两种情况

**为什么用严格相等而非 >=？** 因为诊断的 version 是"基于哪个版本计算"而非"发给哪个版本"。如果编辑器 version 是 5，诊断 version 是 3，说明诊断基于旧文档计算，内容可能已经不对；如果诊断 version 是 6，说明服务器计算诊断时用到了编辑器尚未发送的变更（不可能发生，但防御性处理）。两种情况都不应采用，因此严格相等。

**丢弃后的影响**：过时诊断被丢弃后，不会更新文档的 diagnostic 列表，界面继续显示上一批有效诊断。这比显示错误诊断更安全。等服务器基于最新版本重新计算并推送新诊断时，version 对齐，诊断自然更新。

### 链路三：反向编辑（applyEdit）——版本不匹配则拒绝

**方向**：语言服务器 → 编辑器

**版本携带方式**：`OptionalVersionedTextDocumentIdentifier.version: Option<i32>`（helix-lsp-types/src/lib.rs#L970-L980）。服务器在 `TextDocumentEdit` 中可以附带 version，表示"此编辑应精确应用到该版本的文档上"。

**对齐机制**：`apply_text_edits()`（helix-view/src/handlers/lsp.rs#L82-L131）在应用编辑前：

```rust
if let Some(version) = version {
    if version != doc.version() {
        let err = format!("outdated workspace edit for {path:?}");
        log::error!("{err}, expected {} but got {version}", doc.version());
        return Err(ApplyEditErrorKind::DocumentChanged);
    }
}
```

- 服务器提供 version 且不匹配 → **拒绝应用**，返回 `DocumentChanged` 错误
- 服务器未提供 version → 跳过校验，直接应用（向后兼容）

**拒绝后的处理**：`apply_workspace_edit()`（helix-view/src/handlers/lsp.rs#L134-L226）收集第一个失败编辑的索引和错误类型，返回给服务器。服务器可以据此决定是否重试。注意 Helix 采用的是 `failureHandling: Abort` 策略（helix-lsp/src/client.rs#L611）——一旦任何编辑失败，立即中止，后续编辑不再应用。

**与诊断回包的区别**：诊断丢弃是"静默忽略"（只打日志），而反向编辑拒绝是"显式报错"（返回错误给服务器）。因为诊断是通知语义（fire-and-forget），而 applyEdit 是请求语义（需要响应）。

### 三条链路的版本对齐对比

| 维度 | 编辑通知 didChange | 诊断回包 publishDiagnostics | 反向编辑 applyEdit |
|------|-------------------|---------------------------|-------------------|
| 方向 | 编辑器 → 服务器 | 服务器 → 编辑器 | 服务器 → 编辑器 |
| version 含义 | 应用后文档版本 | 诊断所基于的文档版本 | 编辑应精确匹配的文档版本 |
| version 字段 | 必选 `i32` | 可选 `Option<i32>` | 可选 `Option<i32>` |
| 不匹配时 | 服务器自行忽略旧版本 | 整批诊断丢弃 | 拒绝应用，返回错误 |
| 无 version 时 | 不存在此情况 | 跳过校验，正常处理 | 跳过校验，直接应用 |
| 消息语义 | 通知（无需响应） | 通知（无需响应） | 请求（必须响应） |

---

## 10. 幽灵事务——不通知 LSP 的临时变更

`Document::apply_temporary()`（helix-view/src/document.rs#L1658-L1663）允许应用事务但不通知语言服务器：

```rust
pub fn apply_temporary(&mut self, transaction: &Transaction, view_id: ViewId) -> bool {
    self.apply_inner(transaction, view_id, false)  // emit_lsp_notification = false
}
```

当 `emit_lsp_notification = false` 时，dispatch 的事件中 `ghost_transaction = true`（helix-view/src/document.rs#L1610）：

```rust
helix_event::dispatch(DocumentDidChange {
    ghost_transaction: !emit_lsp_notification,  // ← true
    ...
});
```

LSP Handler 中检查此标志（helix-view/src/handlers/lsp.rs#L409）：

```rust
if !event.ghost_transaction {
    // 只有非幽灵事务才发送 didChange
    for language_server in event.doc.language_servers() {
        language_server.text_document_did_change(...);
    }
}
```

**版本号跳跃**：即使幽灵事务不通知 LSP，`version` 仍然会递增（递增在 dispatch 之前，不受 `ghost_transaction` 影响）。这意味着服务器看到的版本号可能不连续——比如从 5 直接跳到 7（6 被幽灵事务消费了）。LSP 协议允许版本号不连续（helix-lsp-types/src/lib.rs#L978："The number doesn't need to be consecutive"），所以这不会导致协议错误。但服务器会看到一次"跳版"，需要据此推断中间发生了它不知道的变更。

**对诊断的影响**：幽灵事务递增 version 后，服务器之前基于旧 version 发出的诊断会在 `handle_lsp_diagnostics()` 中因 version 不匹配而被丢弃。这是正确的行为——幽灵事务改变了文档内容，旧诊断可能已不准确。服务器在收到下一次 `didChange` 后会基于新 version 重新计算诊断。

---

## 11. 反向链路：从 LSP 编辑回写到文档

增量同步是双向的。当语言服务器通过 `workspace/applyEdit` 请求修改文档时：

1. `apply_text_edits()`（helix-view/src/handlers/lsp.rs#L82-L131）校验版本号
2. `generate_transaction_from_edits()`（helix-lsp/src/lib.rs#L410-L477）将 LSP TextEdit 转换回 Transaction
3. `doc.apply(&transaction, view.id)` 应用事务
4. 应用后又会触发 `DocumentDidChange`，再次通过 `didChange` 通知服务器

为避免无限循环，服务器不应该在收到自己发起的编辑后再发回相同的编辑——这由 LSP 协议语义保证。

`generate_transaction_from_edits` 还有一个优化：如果只有一个编辑且覆盖整个文档（全文档替换），它会调用 `compare_ropes()` 做 diff，生成更精细的 ChangeSet 而非暴力替换整篇文本。

**注意反向编辑的版本闭环**：当反向编辑被应用后，`doc.apply()` 会递增 version 并触发 `didChange`。这次 `didChange` 携带的 version 是反向编辑之后的版本号，服务器收到后可以确认编辑已成功应用，并将自己的文档模型同步到最新状态。

---

## 12. 关键代码引用索引

| 环节 | 文件 | 行号 | 函数/结构 |
|------|------|------|-----------|
| Operation 枚举 | helix-core/src/transaction.rs | L13-L20 | `enum Operation` |
| ChangeSet 结构 | helix-core/src/transaction.rs | L25-L32 | `struct ChangeSet` |
| ChangeSet::apply | helix-core/src/transaction.rs | L342-L366 | `fn apply()` |
| Document::apply | helix-view/src/document.rs | L1654-L1656 | `fn apply()` |
| Document::apply_impl | helix-view/src/document.rs | L1435-L1626 | `fn apply_impl()` |
| version 递增 | helix-view/src/document.rs | L1464 | `self.version += 1` |
| version 字段 | helix-view/src/document.rs | L196 | `version: i32` |
| versioned_identifier | helix-view/src/document.rs | L2110-L2112 | `fn versioned_identifier()` |
| DocumentDidChange 事件 | helix-view/src/events.rs | L12-L18 | `DocumentDidChange` |
| LSP Handler 钩子 | helix-view/src/handlers/lsp.rs | L407-L421 | `register_hook!` |
| text_document_did_change | helix-lsp/src/client.rs | L1054-L1097 | `fn text_document_did_change()` |
| changeset_to_changes | helix-lsp/src/client.rs | L944-L1052 | `fn changeset_to_changes()` |
| traverse 辅助函数 | helix-lsp/src/client.rs | L966-L995 | `fn traverse()` |
| pos_to_lsp_pos | helix-lsp/src/lib.rs | L223-L251 | `fn pos_to_lsp_pos()` |
| Client::notify | helix-lsp/src/client.rs | L502-L533 | `fn notify()` |
| Channel 创建 | helix-lsp/src/client.rs | L246 | `unbounded_channel()` |
| Transport::send 循环 | helix-lsp/src/transport.rs | L338-L441 | `async fn send()` |
| pending_messages 缓存 | helix-lsp/src/transport.rs | L345 | `Vec<Payload>` |
| 初始化前通知丢弃 | helix-lsp/src/transport.rs | L418-L421 | `continue` |
| 初始化后排空缓存 | helix-lsp/src/transport.rs | L402-L411 | `pending_messages.drain(..)` |
| handle_lsp_diagnostics | helix-view/src/handlers/lsp.rs | L279-L296 | 诊断版本校验 |
| apply_text_edits | helix-view/src/handlers/lsp.rs | L82-L131 | 反向编辑+版本校验 |
| generate_transaction_from_edits | helix-lsp/src/lib.rs | L410-L477 | LSP 编辑 → Transaction |
| apply_temporary | helix-view/src/document.rs | L1658-L1663 | 幽灵事务入口 |
| ghost_transaction 赋值 | helix-view/src/document.rs | L1610 | `!emit_lsp_notification` |
| LanguageServerInitialized 钩子 | helix-view/src/handlers/lsp.rs | L387-L404 | 重发 didOpen |
| TextDocumentSyncKind | helix-lsp-types/src/lib.rs | L1768-L1790 | 同步模式枚举 |
| VersionedTextDocumentIdentifier | helix-lsp-types/src/lib.rs | L970-L980 | 版本号+URI |
| DidChangeTextDocumentParams | helix-lsp-types/src/lib.rs | L2266-L2273 | didChange 参数 |
| TextDocumentContentChangeEvent | helix-lsp-types/src/lib.rs | L2279-L2292 | 增量变更事件 |
| Client::offset_encoding | helix-lsp/src/client.rs | L410-L424 | 偏移编码选择 |
