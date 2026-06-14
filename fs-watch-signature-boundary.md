# Helix 签名帮助与文档颜色/链接边界校准

本文校准签名帮助处理器的分类，明确它与文档颜色、文档链接的根本区别：签名帮助没有位置映射，只有配置+ghost 双重守卫的完全跳过。全文分类表保持一致。

---

## 一、问题说明

前文将签名帮助处理器分类不一致：在某些段落归类为"模式 B（位置映射始终 + 重新请求跳过）"，在另一些段落归类为"模式 D（配置+ghost 双重守卫）"。实际代码证明：**签名帮助没有位置映射，属于纯模式 D**。

---

## 二、三个处理器的代码精确对比

### 2.1 文档颜色 — 模式 B（有位置映射）

**代码位置**：[handlers/document_colors.rs#L153-L184](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs#L153-L184)

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    // ── 第一部分：位置映射，始终执行 ──
    let apply_color_swatch_changes = |annotations: &mut Vec<InlineAnnotation>| {
        event.changes.update_positions(
            annotations.iter_mut()
                .map(|annotation| (&mut annotation.char_idx, helix_core::Assoc::After)),
        );
    };

    if let Some(DocumentColorSwatches {
        color_swatches,
        colors: _colors,
        color_swatches_padding,
    }) = &mut event.doc.color_swatches
    {
        apply_color_swatch_changes(color_swatches);          // ← 始终执行
        apply_color_swatch_changes(color_swatches_padding);  // ← 始终执行
    }

    // ── 第二部分：重新请求，仅非 ghost ──
    if !event.ghost_transaction {                            // ← 仅 ghost 守卫
        event.doc.color_swatch_controller.cancel();
        helix_event::send_blocking(&tx, DocumentColorsEvent(event.doc.id()));
    }

    Ok(())
});
```

**结构特征**：
- ✅ 有位置映射（`InlineAnnotation.char_idx`，使用 `Assoc::After`）
- ✅ 有重新请求（`send_blocking(DocumentColorsEvent)`）
- ❌ 没有配置守卫（无 `auto_*` 配置检查）
- 守卫层次：仅 `!ghost_transaction`

### 2.2 文档链接 — 模式 B（有位置映射）

**代码位置**：[handlers/document_links.rs#L132-L146](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs#L132-L146)

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    // ── 第一部分：位置映射，始终执行 ──
    event.changes.update_positions(
        event.doc.document_links.iter_mut().flat_map(|link| {
            std::iter::once((&mut link.start, Assoc::After))      // ← 始终执行
                .chain(std::iter::once((&mut link.end, Assoc::After)))  // ← 始终执行
        })
    );

    // ── 第二部分：重新请求，仅非 ghost ──
    if !event.ghost_transaction {                                // ← 仅 ghost 守卫
        event.doc.document_link_controller.cancel();
        helix_event::send_blocking(&tx, DocumentLinksEvent(event.doc.id()));
    }

    Ok(())
});
```

**结构特征**：
- ✅ 有位置映射（`DocumentLink.start/end`，使用 `Assoc::After`）
- ✅ 有重新请求（`send_blocking(DocumentLinksEvent)`）
- ❌ 没有配置守卫（无 `auto_*` 配置检查）
- 守卫层次：仅 `!ghost_transaction`

### 2.3 签名帮助 — 模式 D（无位置映射，配置+ghost 双重守卫）

**代码位置**：[handlers/signature_help.rs#L353-L358](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353-L358)

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    if event.doc.config.load().lsp.auto_signature_help && !event.ghost_transaction {
        send_blocking(&tx, SignatureHelpEvent::ReTrigger);
    }
    Ok(())
});
```

**结构特征**：
- ❌ **没有位置映射**（签名帮助不存储位置数据到文档上）
- ✅ 有重新触发（`send_blocking(ReTrigger)`）
- ✅ 有配置守卫（`auto_signature_help`）
- 守卫层次：`auto_signature_help && !ghost_transaction`（双重守卫）

---

## 三、根本区别：为什么签名帮助没有位置映射

### 3.1 数据存储方式不同

| 处理器 | 存储的数据 | 存储位置 | 需要位置映射？ |
|--------|-----------|----------|--------------|
| 文档颜色 | `Vec<InlineAnnotation>` 色块列表 | `doc.color_swatches` | ✅ `char_idx` 需跟随文本移动 |
| 文档链接 | `Vec<DocumentLink>` 链接列表 | `doc.document_links` | ✅ `start/end` 需跟随文本移动 |
| 签名帮助 | Popup 弹窗（非文档数据） | Compositor 管理的 UI 组件 | ❌ 弹窗跟随光标，不需要位置映射 |

### 3.2 渲染方式不同

- **文档颜色**：色块作为 `InlineAnnotation` 插入到文档文本中，位置是相对于文本的字符偏移量。文本变化时偏移量必须更新，否则色块显示在错误位置。

- **文档链接**：链接范围（`start`, `end`）是文档中的字符偏移量。文本变化时范围必须更新，否则链接高亮错位。

- **签名帮助**：以 Popup 弹窗形式显示在编辑器上方，位置由 Compositor 根据**光标位置**自动计算。光标位置由 `Selection` 管理，`apply_impl()` 中已经通过 `selection.map(transaction.changes())` 更新了选区，所以弹窗自动跟随，不需要单独的位置映射。

### 3.3 生命周期不同

| 处理器 | 数据创建 | 数据失效 |
|--------|---------|---------|
| 文档颜色 | LSP 返回 `textDocument/documentColor` 结果 | 文档变更时位置偏移，需映射或重新请求 |
| 文档链接 | LSP 返回 `textDocument/documentLink` 结果 | 文档变更时位置偏移，需映射或重新请求 |
| 签名帮助 | LSP 返回 `textDocument/signatureHelp` 结果 | 文档变更时直接重新请求，旧弹窗被替换 |

签名帮助不需要"保持旧数据的位置正确"，而是直接用新请求的完整结果替换旧弹窗。

---

## 四、校准后的完整分类表

### 4.1 五模式分类

| 模式 | 定义 | ghost 时行为 | 适用处理器 |
|------|------|-------------|-----------|
| **A** | 仅 ghost 守卫，完全跳过 | 不执行任何操作 | LSP didChange、诊断拉取（单文档）、诊断拉取（跨文档） |
| **B** | 位置映射始终 + 重新请求 ghost-only | 位置映射 ✓ / 重新请求 ✗ | **文档颜色**、**文档链接** |
| **C** | 不检查 ghost，始终执行 | 完全不受影响 | Snippet 映射 |
| **D** | 配置+ghost 双重守卫，完全跳过 | 不执行任何操作 | **签名帮助**、**文档高亮** |
| **D'** | 仅配置守卫，不检查 ghost | 正常发送事件 | 自动保存 |

### 4.2 逐处理器精确参数

| # | 处理器 | 模式 | 位置映射 | 重新请求/触发 | 配置守卫 | Ghost 守卫 | 去抖 |
|---|--------|------|---------|-------------|---------|-----------|------|
| 1 | LSP didChange | A | — | — | 无 | ✅ | 无 |
| 2 | 诊断拉取（单文档） | A | — | ✅ | 无（feature 检查） | ✅ | 250ms |
| 3 | 诊断拉取（跨文档） | A | — | ✅ | 无（feature 检查） | ✅ | 1s |
| 4 | 词索引更新 | A | — | ✅ | ✅ `word_completion_enabled` | ✅ | 有 |
| 5 | **文档颜色** | **B** | ✅ 始终 | ✅ ghost-only | 无 | 仅重新请求部分 | 250ms |
| 6 | **文档链接** | **B** | ✅ 始终 | ✅ ghost-only | 无 | 仅重新请求部分 | 250ms |
| 7 | Snippet 映射 | C | ✅ 始终 | — | 无 | 无 | 无 |
| 8 | **签名帮助** | **D** | ❌ 无 | ✅ config+ghost | ✅ `auto_signature_help` | ✅ | 120ms |
| 9 | **文档高亮** | **D** | ❌ 无 | ✅ config+ghost | ✅ `auto_document_highlight` | ✅ | 无 |
| 10 | 自动保存 | D' | — | ✅ config-only | ✅ `after_delay.enable` | ❌ | 配置 timeout |

### 4.3 模式 B vs 模式 D 的关键区别

```
模式 B（文档颜色/链接）              模式 D（签名帮助/文档高亮）
─────────────────────────           ──────────────────────────
有持久化的位置数据                    无持久化的位置数据
  ↓                                   ↓
必须始终映射位置                      不需要位置映射
  ↓                                   ↓
位置映射 + 重新请求 分离              只有重新请求/触发
  ↓                                   ↓
ghost 时：位置映射 ✓ / 请求 ✗        ghost 时：完全跳过
  ↓                                   ↓
无配置守卫（功能始终启用）             有配置守卫（可关闭）
```

---

## 五、签名帮助的完整触发源

签名帮助不仅响应 `DocumentDidChange`，还有其他触发源：

| 触发源 | 事件 | 代码位置 | 条件 | 发送的事件 |
|--------|------|----------|------|-----------|
| 文档变更 | `DocumentDidChange` | [signature_help.rs#L353-L358](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353-L358) | `auto_signature_help && !ghost` | `ReTrigger` |
| 选区变更 | `SelectionDidChange` | [signature_help.rs#L361-L366](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L361-L366) | `auto_signature_help` | `ReTrigger` |
| 输入字符 | `PostInsertChar` | [signature_help.rs#L291-L327](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L291-L327) | `auto_signature_help` + 触发字符匹配 | `Trigger` |
| 模式切换 | `OnModeSwitch` | [signature_help.rs#L331-L345](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L331-L345) | 进入插入模式 | `Trigger` / `Cancel` |
| 手动命令 | 用户调用 | — | — | `Invoked` |

**注意**：只有 `DocumentDidChange` 触发源检查了 `ghost_transaction`。其他触发源（`SelectionDidChange`、`PostInsertChar`、`OnModeSwitch`）不检查 ghost，因为它们只在用户直接操作时触发，不会由 ghost transaction 产生。

---

## 六、签名帮助去抖状态机

**代码位置**：[signature_help.rs#L51-L99](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L51-L99)

去抖时间：**120ms**（来自 VSCode 的经验值）

```
                    ┌──────────┐
          ┌────────►│  Closed  │◄─────────────┐
          │         └─────┬────┘              │
          │               │ Trigger/ReTrigger │ Cancel
          │               ▼                   │
     RequestComplete  ┌──────────┐            │
      (open=false)    │  Pending │────────────┘
          │           └─────┬────┘
          │                 │ finish_debounce
          │                 ▼
          │           发送 LSP 请求
          │                 │
          │                 ▼ 响应到达
          │           RequestComplete(open=true)
          │                 │
          ├─────────────────┘
          │
     ┌────▼────┐
     │  Open   │◄─── ReTrigger ──► 保持 Pending，重设去抖
     └────┬────┘
          │ Cancel / 退出插入模式
          ▼
       Closed
```

**事件处理细节**：
- `Invoked`（手动）：立即执行 `finish_debounce()`，不走去抖
- `Trigger`：设置 trigger 为 Automatic，启动 120ms 去抖
- `ReTrigger`：仅在 `Open` 或 `Pending` 状态下才处理（`Closed` 时忽略），重设去抖
- `Cancel`：关闭状态，取消请求
- `RequestComplete`：更新状态为 Open/Closed，取消进行中的旧请求

---

## 七、关键文件索引

| 主题 | 文件 |
|------|------|
| 签名帮助完整实现 | [handlers/signature_help.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs) |
| 签名帮助 DocumentDidChange hook | [signature_help.rs#L353-L358](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353-L358) |
| 签名帮助去抖状态机 | [signature_help.rs#L51-L99](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L51-L99) |
| 文档颜色 hook | [handlers/document_colors.rs#L153-L184](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs#L153-L184) |
| 文档链接 hook | [handlers/document_links.rs#L132-L146](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs#L132-L146) |
| 文档高亮 hook | [handlers/document_highlight.rs#L153-L162](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_highlight.rs#L153-L162) |
