# Helix 文档变更处理器边界校准

本文校准 DocumentDidChange 处理器清单，精确区分"始终响应"与"仅位置映射/跳过重新请求"的边界，补清自动保存、文档颜色、文档链接的位置映射细节，并说明手动补全入口的默认按键场景。

---

## 一、处理器对 ghost_transaction 的精确行为分类

`ghost_transaction = true` 表示该事务不应通知 LSP（例如补全预览的临时编辑）。各处理器对此有三种行为模式：

### 1.1 模式 A：完全跳过

整个处理器被 `!ghost_transaction` 守卫包裹，ghost 事务时不执行任何操作。

| 处理器 | 代码位置 | 守卫代码 |
|--------|----------|----------|
| LSP didChange 通知 | [handlers/lsp.rs#L407-L421](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/lsp.rs#L407-L421) | `if !event.ghost_transaction { for ls in ... { did_change(...) } }` |
| 诊断拉取 | [handlers/diagnostics.rs#L43-L83](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/diagnostics.rs#L43-L83) | `if !event.ghost_transaction { send_blocking(...) }` |
| 文档高亮更新 | [handlers/document_highlight.rs#L153-L162](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_highlight.rs#L153-L162) | `if auto_document_highlight && !event.ghost_transaction { dispatch_blocking(...) }` |

> 诊断拉取还包含额外的 LSP feature 守卫 `has_language_server_with_feature(PullDiagnostics)`；文档高亮还包含配置开关 `auto_document_highlight`。

### 1.2 模式 B：位置映射始终执行，重新请求被跳过

处理器分两步：先做位置映射（始终执行），再根据 `!ghost_transaction` 决定是否重新请求。

| 处理器 | 代码位置 | 始终执行部分 | 跳过部分 |
|--------|----------|-------------|----------|
| 文档颜色 | [handlers/document_colors.rs#L153-L184](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs#L153-L184) | 颜色色块位置映射（L156-L172） | `send_blocking(DocumentColorsEvent)` 重新请求（L177-L181） |
| 文档链接 | [handlers/document_links.rs#L132-L146](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs#L132-L146) | 链接 start/end 位置映射（L133-L138） | `send_blocking(DocumentLinksEvent)` 重新请求（L140-L143） |
| 签名帮助 | [handlers/signature_help.rs#L353-L358](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353-L358) | — | `send_blocking(SignatureHelpEvent::ReTrigger)` （L354-L356） |

#### 文档颜色详细分析

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    // 第一部分：始终执行 —— 位置映射
    let apply_color_swatch_changes = |annotations: &mut Vec<InlineAnnotation>| {
        event.changes.update_positions(
            annotations.iter_mut()
                .map(|annotation| (&mut annotation.char_idx, Assoc::After)),
        );
    };

    if let Some(DocumentColorSwatches {
        color_swatches,
        colors: _colors,
        color_swatches_padding,
    }) = &mut event.doc.color_swatches
    {
        apply_color_swatch_changes(color_swatches);        // 始终执行
        apply_color_swatch_changes(color_swatches_padding); // 始终执行
    }

    // 第二部分：仅非 ghost 时执行 —— 重新请求
    if !event.ghost_transaction {
        event.doc.color_swatch_controller.cancel();        // 取消进行中的请求
        helix_event::send_blocking(&tx, DocumentColorsEvent(event.doc.id()));
    }

    Ok(())
});
```

**为什么位置映射始终执行？** 即使是 ghost transaction（如补全预览），颜色色块的 `InlineAnnotation` 仍然需要跟随文本变化更新位置，否则显示位置会错乱。但重新向 LSP 请求颜色信息是不合理的，因为 LSP 不知道 ghost 事务的变更，会返回过时的位置。

#### 文档链接详细分析

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    // 第一部分：始终执行 —— 位置映射
    event.changes.update_positions(
        event.doc.document_links.iter_mut().flat_map(|link| {
            std::iter::once((&mut link.start, Assoc::After))
                .chain(std::iter::once((&mut link.end, Assoc::After)))
        })
    );

    // 第二部分：仅非 ghost 时执行 —— 重新请求
    if !event.ghost_transaction {
        event.doc.document_link_controller.cancel();
        helix_event::send_blocking(&tx, DocumentLinksEvent(event.doc.id()));
    }

    Ok(())
});
```

**链接位置映射使用 `Assoc::After`**：链接的起始和结束位置都关联到变更位置之后，确保链接范围跟随文本增删正确移动。

### 1.3 模式 C：始终执行（不受 ghost_transaction 影响）

| 处理器 | 代码位置 | 说明 |
|--------|----------|------|
| Snippet 映射 | [handlers/snippet.rs#L14-L22](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/snippet.rs#L14-L22) | snippet 必须始终跟随变更，否则占位符失效 |

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    if let Some(snippet) = &mut event.doc.active_snippet {
        let invalid = snippet.map(event.changes);
        if invalid {
            event.doc.active_snippet = None;
        }
    }
    Ok(())
});
```

### 1.4 模式 D：配置+ghost 双重守卫

| 处理器 | 代码位置 | 配置守卫 | ghost 守卫 |
|--------|----------|----------|-----------|
| 签名帮助 | [handlers/signature_help.rs#L353-L358](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353-L358) | `auto_signature_help` | `!ghost_transaction` |
| 文档高亮 | [handlers/document_highlight.rs#L153-L162](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_highlight.rs#L153-L162) | `auto_document_highlight` | `!ghost_transaction` |

> 签名帮助没有位置映射部分，所以 ghost 时完全跳过。

### 1.5 特殊：自动保存

自动保存处理器**不检查 `ghost_transaction`**，只要配置启用就会响应。

**代码位置**：[handlers/auto_save.rs#L101-L114](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/auto_save.rs#L101-L114)

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    let config = event.doc.config.load();
    if config.auto_save.after_delay.enable {
        send_blocking(
            &tx,
            AutoSaveEvent::DocumentChanged {
                save_after: config.auto_save.after_delay.timeout,
            },
        );
    }
    Ok(())
});
```

**没有 `ghost_transaction` 检查的原因**：自动保存关注的是文档是否有未保存的修改，而不关心变更来源。即使是 ghost transaction（如补全预览），文档也可能处于"已修改"状态，需要在超时后触发保存。

但这在实践中不会产生问题，因为：
- ghost transaction 由 `apply_temporary()` 触发，通常不会设置 `modified_since_accessed = true`
- `finish_debounce` 中有 `if editor.mode() == Mode::Insert` 守卫，插入模式下不会执行自动保存

### 1.6 特殊：词索引更新

**代码位置**：[handlers/word_index.rs#L388-L401](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/word_index.rs#L388-L401)

```rust
register_hook!(move |event: &mut DocumentDidChange<'_>| {
    if !event.ghost_transaction && event.doc.word_completion_enabled() {
        helix_event::send_blocking(
            &tx,
            Event::Update(event.doc.id(), Change { ... }),
        );
    }
    Ok(())
});
```

**双重守卫**：`!ghost_transaction` + `word_completion_enabled()`。ghost 事务时跳过词索引更新。

---

## 二、完整处理器清单（校准版）

| # | 处理器 | 模式 | ghost 时行为 | 位置映射 | 重新请求 | 去抖 | 额外守卫 |
|---|--------|------|-------------|---------|---------|------|---------|
| 1 | LSP didChange | A | 完全跳过 | — | — | 无 | — |
| 2 | 诊断拉取（单文档） | A | 完全跳过 | — | — | 250ms | `has PullDiagnostics` |
| 3 | 诊断拉取（跨文档） | A | 完全跳过 | — | — | 1s | `inter_file_dependencies` |
| 4 | 文档高亮 | D | 完全跳过 | — | ✅ | 无 | `auto_document_highlight` |
| 5 | 签名帮助 | D | 完全跳过 | — | — | 有 | `auto_signature_help` |
| 6 | 词索引更新 | A | 完全跳过 | — | — | 有 | `word_completion_enabled` |
| 7 | 文档颜色 | B | 仅位置映射 | ✅ 始终 | 跳过 | 250ms | — |
| 8 | 文档链接 | B | 仅位置映射 | ✅ 始终 | 跳过 | 250ms | — |
| 9 | Snippet 映射 | C | 始终执行 | ✅ 始终 | — | 无 | `active_snippet.is_some()` |
| 10 | 自动保存 | D' | 始终发送事件 | — | — | 配置 `timeout` | `after_delay.enable` |

> **模式说明**：
> - A = 完全跳过（仅 `!ghost_transaction` 守卫）
> - B = 位置映射始终执行 + 重新请求被跳过
> - C = 完全不受 ghost 影响
> - D = 配置+ghost 双重守卫
> - D' = 配置守卫但不检查 ghost（受插入模式运行时守卫保护）

---

## 三、自动保存处理器的完整逻辑

### 3.1 触发源

自动保存有两个触发源：

| 触发源 | 事件 | 代码位置 |
|--------|------|----------|
| 文档变更 | `DocumentDidChange` | [auto_save.rs#L103-L114](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/auto_save.rs#L103-L114) |
| 退出插入模式 | `OnModeSwitch` | [auto_save.rs#L116-L123](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/auto_save.rs#L116-L123) |

### 3.2 去抖与保存决策

```rust
fn handle_event(&mut self, event: Self::Event, existing_debounce: Option<Instant>) -> Option<Instant> {
    match event {
        AutoSaveEvent::DocumentChanged { save_after } => {
            // 设置/重置去抖计时器
            Some(Instant::now() + Duration::from_millis(save_after))
        }
        AutoSaveEvent::LeftInsertMode => {
            if existing_debounce.is_some() {
                // 还有未到期的去抖计时器，等它到期再保存
                existing_debounce
            } else {
                // 没有去抖计时器，但可能有挂起的保存
                if self.save_pending.load(Ordering::Relaxed) {
                    self.finish_debounce();  // 立即保存
                }
                None
            }
        }
    }
}
```

### 3.3 插入模式保护

```rust
fn finish_debounce(&mut self) {
    let save_pending = self.save_pending.clone();
    job::dispatch_blocking(move |editor, _| {
        if editor.mode() == Mode::Insert {
            // 插入模式中不保存，标记为挂起
            save_pending.store(true, Ordering::Relaxed);
        } else {
            request_auto_save(editor);
            save_pending.store(false, Ordering::Relaxed);
        }
    })
}
```

**关键保护机制**：
- `finish_debounce` 中检查 `editor.mode() == Mode::Insert`
- 如果仍在插入模式，将 `save_pending` 设为 `true`，延迟到退出插入模式时执行
- 退出插入模式时，`OnModeSwitch` hook 发送 `LeftInsertMode` 事件，触发保存

### 3.4 保存执行

```rust
fn request_auto_save(editor: &mut Editor) {
    let options = WriteAllOptions {
        force: false,
        write_scratch: false,  // 不保存未命名缓冲区
        auto_format: false,    // 不自动格式化（避免循环触发）
    };
    if let Err(e) = commands::typed::write_all_impl(context, options) {
        context.editor.set_error(format!("{}", e));
    }
}
```

**注意**：`auto_format: false` 防止自动保存触发格式化，避免格式化 → 变更 → 自动保存的死循环。

### 3.5 与 ghost_transaction 的关系

自动保存**不检查 ghost_transaction**，但实际不会因 ghost 事务触发保存：
1. ghost transaction 由 `apply_temporary()` 触发
2. `apply_temporary()` 调用 `apply_inner(transaction, view_id, false)`，`emit_lsp_notification = false`
3. 内部仍会调用 `apply_impl()`，派发 `DocumentDidChange { ghost_transaction: true }`
4. 自动保存 hook 收到事件并发送 `DocumentChanged`
5. 但 `apply_temporary()` 的变更通常不设置 `modified_since_accessed`
6. 即使触发了 `finish_debounce`，插入模式守卫会阻止执行
7. 非插入模式下 `write_all_impl` 只保存有修改的文档

---

## 四、文档颜色位置映射边界

### 4.1 数据结构

```rust
pub struct DocumentColorSwatches {
    pub color_swatches: Vec<InlineAnnotation>,        // ■ 色块
    pub colors: Vec<Style>,                            // 色块颜色
    pub color_swatches_padding: Vec<InlineAnnotation>, // 色块前空格
}
```

### 4.2 位置映射逻辑

**代码位置**：[handlers/document_colors.rs#L153-L184](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs#L153-L184)

```rust
let apply_color_swatch_changes = |annotations: &mut Vec<InlineAnnotation>| {
    event.changes.update_positions(
        annotations.iter_mut()
            .map(|annotation| (&mut annotation.char_idx, Assoc::After)),
    );
};

if let Some(DocumentColorSwatches {
    color_swatches,
    colors: _colors,           // 颜色值不需要位置映射
    color_swatches_padding,
}) = &mut event.doc.color_swatches
{
    apply_color_swatch_changes(color_swatches);         // ■ 色块位置
    apply_color_swatch_changes(color_swatches_padding); // 前置空格位置
}
```

**映射策略**：
- 使用 `Assoc::After`：位置关联到变更点之后
- `color_swatches` 和 `color_swatches_padding` 都需要映射
- `colors`（Style 数组）不需要映射，因为它只存储颜色值，没有位置信息

### 4.3 重新请求边界

```rust
if !event.ghost_transaction {
    event.doc.color_swatch_controller.cancel();  // 取消进行中的请求
    helix_event::send_blocking(&tx, DocumentColorsEvent(event.doc.id()));
}
```

**跳过重新请求的原因**（代码注释原文）：

> Avoid re-requesting document colors if the change is a ghost transaction (completion) because the language server will not know about the updates to the document and will give out-of-date locations.

---

## 五、文档链接位置映射边界

### 5.1 数据结构

```rust
pub struct DocumentLink {
    pub start: usize,
    pub end: usize,
    pub link: lsp::DocumentLink,
    pub language_server_id: LanguageServerId,
}
```

### 5.2 位置映射逻辑

**代码位置**：[handlers/document_links.rs#L132-L146](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs#L132-L146)

```rust
event.changes.update_positions(
    event.doc.document_links.iter_mut().flat_map(|link| {
        std::iter::once((&mut link.start, Assoc::After))
            .chain(std::iter::once((&mut link.end, Assoc::After)))
    })
);
```

**映射策略**：
- `start` 和 `end` 都使用 `Assoc::After`
- 使用 `flat_map` 将每个链接的两个位置展平，一次性更新所有链接
- 链接的 `link`（LSP 元数据）和 `language_server_id` 不需要映射

### 5.3 重新请求边界

```rust
if !event.ghost_transaction {
    event.doc.document_link_controller.cancel();
    helix_event::send_blocking(&tx, DocumentLinksEvent(event.doc.id()));
}
```

与文档颜色相同的模式：ghost 时只做位置映射，不重新请求。

### 5.4 与文档颜色的对比

| 维度 | 文档颜色 | 文档链接 |
|------|----------|----------|
| 位置映射数据 | `InlineAnnotation.char_idx` × 2 | `DocumentLink.start` + `DocumentLink.end` |
| 映射关联策略 | `Assoc::After` | `Assoc::After` |
| 映射方式 | 闭包函数，分别映射两组 | `flat_map` 一次性映射 |
| 重新请求去抖 | 250ms | 250ms |
| ghost 行为 | 位置映射 ✓ / 请求 ✗ | 位置映射 ✓ / 请求 ✗ |
| 额外守卫 | 无 | 无 |

---

## 六、手动补全入口的默认按键场景

### 6.1 默认按键绑定

**代码位置**：[default.rs#L385](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/keymap/default.rs#L385)

```rust
let insert = keymap!({ "Insert mode"
    "esc" => normal_mode,
    "C-s" => commit_undo_checkpoint,
    "C-x" => completion,        // ← 手动补全
    "C-r" => insert_register,
    // ...
});
```

**默认按键**：插入模式下按 `C-x`（Ctrl+X）触发 `completion` 命令。

### 6.2 completion 命令实现

**代码位置**：[commands.rs#L5457-L5466](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/commands.rs#L5457-L5466)

```rust
pub fn completion(cx: &mut Context) {
    let (view, doc) = current!(cx.editor);
    let range = doc.selection(view.id).primary();
    let text = doc.text().slice(..);
    let cursor = range.cursor(text);

    cx.editor
        .handlers
        .trigger_completions(cursor, doc.id(), view.id);
}
```

### 6.3 从按键到补全请求的完整链路

```
插入模式下用户按 C-x
    ↓
EditorView::insert_mode() 处理按键 [editor.rs#L984]
    ↓
handle_keymap_event(Mode::Insert, cx, event)
    ↓
KeymapResult::MatchedSequence(MappableCommand::completion)
    ↓
execute_command(completion)
    ↓
commands::completion(cx)  [commands.rs#L5457]
    ↓
handlers.trigger_completions(cursor, doc.id(), view.id)  [handlers.rs#L34]
    ↓
CompletionEvent::ManualTrigger { cursor, doc, view }
    ↓
CompletionHandler::handle_event()  [request.rs#L110-L121]
    ↓
self.finish_debounce()  // 立即执行，不去抖
    ↓
dispatch_blocking → request_completions()
    ↓
向 LSP 发送 textDocument/completion 请求
```

### 6.4 ManualTrigger 与其他触发方式的区别

| 维度 | ManualTrigger (C-x) | AutoTrigger (打字) | TriggerChar (触发字符) |
|------|---------------------|-------------------|----------------------|
| 去抖时间 | 0ms（立即） | 配置值（默认250ms） | 5ms |
| 是否取消当前请求 | 否 | 否 | 是（`task_controller.cancel()`） |
| 检查 auto_completion 配置 | 否 | 是 | 是 |
| 使用场景 | 用户明确想要补全 | 自动提示 | 输入 `.` `:` 等后快速补全 |

**ManualTrigger 不检查 `auto_completion` 配置**：即使 `auto_completion = false`，用户按 `C-x` 仍可触发补全。这是有意设计——自动补全可以关闭，但手动触发始终可用。

### 6.5 C-x 在不同模式的含义

| 模式 | C-x 绑定 | 功能 |
|------|---------|------|
| 普通模式 | `C-x` → `decrement` | 减小数字 |
| 插入模式 | `C-x` → `completion` | 手动补全 |
| 选择模式 | `C-x` → `decrement` | 减小数字 |

`C-x` 在插入模式中专门映射为补全，不会与数字递减冲突。

---

## 七、校准后的处理器行为总图

```
DocumentDidChange 事件
    │
    ├── 模式 A：完全跳过（ghost_transaction 时）
    │   ├── LSP didChange 通知
    │   ├── 诊断拉取（单文档 250ms / 跨文档 1s）
    │   └── 词索引更新
    │
    ├── 模式 B：位置映射始终 + 重新请求跳过
    │   ├── 文档颜色（色块位置映射 ✓ / LSP 请求 ✗）
    │   └── 文档链接（链接位置映射 ✓ / LSP 请求 ✗）
    │
    ├── 模式 C：始终执行
    │   └── Snippet 映射（占位符跟随变更）
    │
    ├── 模式 D：配置 + ghost 双重守卫
    │   ├── 文档高亮（auto_document_highlight && !ghost）
    │   └── 签名帮助（auto_signature_help && !ghost）
    │
    └── 模式 D'：配置守卫 + 不检查 ghost
        └── 自动保存（after_delay.enable，运行时插入模式保护）
```

---

## 八、关键文件索引

| 主题 | 文件 |
|------|------|
| 自动保存 | [handlers/auto_save.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/auto_save.rs) |
| 文档颜色 | [handlers/document_colors.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs) |
| 文档链接 | [handlers/document_links.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs) |
| 文档高亮 | [handlers/document_highlight.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_highlight.rs) |
| 签名帮助 | [handlers/signature_help.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs) |
| Snippet 映射 | [handlers/snippet.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/snippet.rs) |
| LSP didChange | [handlers/lsp.rs#L407-L421](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/lsp.rs#L407-L421) |
| 词索引 | [handlers/word_index.rs#L388-L401](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/word_index.rs#L388-L401) |
| 手动补全命令 | [commands.rs#L5457-L5466](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/commands.rs#L5457-L5466) |
| 默认按键绑定 | [default.rs#L385](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/keymap/default.rs#L385) |
