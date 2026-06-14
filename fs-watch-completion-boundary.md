# Helix 外部文件改动提示与补全触发边界分析

本文从代码实现角度深入分析：外部文件改动提示链路、文档变更与 reload 触发的处理器、补全触发的来源与边界。

---

## 一、外部文件改动的提示链路

### 1.1 核心结论

Helix **没有自动监听文件系统变化**的机制。外部文件改动只能在以下时机被发现并提示用户：

1. **保存时检测**：写入磁盘前对比 mtime，发现冲突则报错
2. **手动 reload**：用户主动执行 `:reload` / `:reload-all` 重新读取磁盘

> 没有后台线程轮询文件，没有 inotify/fsevent/ReadDirectoryChangesW 监听。

### 1.2 保存冲突提示完整链路

```
用户输入 :w
    ↓
write() 命令入口 [typed.rs#L490-L503]
    ↓
write_impl() [typed.rs#L378-L423]
    │
    ├─ 可选：自动格式化
    └─ 无格式化时直接调用 cx.editor.save(id, path, force)
    ↓
Editor::save_in_background() [editor.rs#L2155-L2173]
    ↓
Document::save() [document.rs#L970-L982]
    ↓
save_impl() → 生成异步 future
    ↓
(后台 tokio 任务执行)
    │
    ├─ 读取磁盘 fs::metadata(&path).await
    ├─ 获取 metadata.modified() → mtime
    ├─ 对比 last_saved_time < mtime ?
    │   │
    │   ├─ 是（外部改动）→ bail!("file modified by an external process...")
    │   └─ 否（无冲突）→ 继续写入
    ↓
结果通过 save_queue 传回主线程
    ↓
Editor::wait_event() 接收结果 [editor.rs#L2420-L2427]
    ↓
包装为 EditorEvent::DocumentSaved(Result)
    ↓
Application::handle_editor_event() [application.rs#L648-L652]
    ↓
Application::handle_document_write() [application.rs#L576-L642]
    │
    ├─ Err → self.editor.set_error(err.to_string())
    │      → status_msg = (error_text, Severity::Error)
    │      → 状态栏红色错误提示
    │
    └─ Ok → 更新 last_saved_time、触发 LSP 文件事件、设置成功状态
```

### 1.3 冲突检测核心代码

**代码位置**：[document.rs#L1038-L1047](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1038-L1047)

```rust
// Protect against overwriting changes made externally
if !force {
    if let Ok(metadata) = fs::metadata(&path).await {
        if let Ok(mtime) = metadata.modified() {
            if last_saved_time < mtime {
                bail!("file modified by an external process, use :w! to overwrite");
            }
        }
    }
}
```

**关键变量 `last_saved_time`**：
- 类型：`SystemTime`
- 更新时机：
  - 文档打开时：`pickup_last_saved_time()` [document.rs#L1340](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1340)
  - 保存成功后：[document.rs#L1130-L1133](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1130-L1133)
  - reload 后：[document.rs#L1297](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1297)

### 1.4 用户提示方式

**三级状态消息系统**，定义在 [editor.rs#L1448-L1485](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/editor.rs#L1448-L1485)：

| 级别 | 方法 | Severity | 场景 |
|------|------|----------|------|
| Info | `set_status()` | Info | 保存成功、普通通知 |
| Warning | `set_warning()` | Warning | 警告信息 |
| Error | `set_error()` | Error | 保存冲突、操作失败 |

保存冲突使用 `set_error()`，用户在状态栏看到红色文字：
```
file modified by an external process, use :w! to overwrite
```

---

## 二、文档变更与 reload 触发的处理器

### 2.1 DocumentDidChange 事件总览

`DocumentDidChange` 是文档内容变更的核心事件，在 `Document::apply_impl()` 中派发：

**代码位置**：[document.rs#L1605-L1611](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1605-L1611)

```rust
helix_event::dispatch(DocumentDidChange {
    doc: self,
    view: view_id,
    old_text: &old_doc,
    changes,
    ghost_transaction: !emit_lsp_notification,
});
```

**事件字段**：
- `doc: &mut Document` - 文档可变引用
- `view: ViewId` - 触发变更的视图
- `old_text: &Rope` - 变更前文本
- `changes: &ChangeSet` - 变更集合
- `ghost_transaction: bool` - 是否为幽灵事务（不通知 LSP）

### 2.2 注册的处理器清单

通过 `register_hook!` 注册的 `DocumentDidChange` 处理器共 **10+ 个**，分布如下：

#### helix-view 层（基础层）

| 处理器 | 文件位置 | 触发条件 | 作用 |
|--------|----------|----------|------|
| LSP didChange 通知 | [handlers/lsp.rs#L407](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/lsp.rs#L407) | `!ghost_transaction` | 向所有 LSP 发送 `textDocument/didChange` |
| 词索引更新 | [handlers/word_index.rs#L388](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/word_index.rs#L388) | `!ghost_transaction && word_completion_enabled` | 更新单词索引（用于单词补全） |

#### helix-term 层（终端 UI 层）

| 处理器 | 文件位置 | 触发条件 | 作用 |
|--------|----------|----------|------|
| 诊断拉取 | [handlers/diagnostics.rs#L43](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/diagnostics.rs#L43) | `!ghost_transaction && has PullDiagnostics` | 触发单文档诊断拉取（250ms 去抖） |
| 跨文档诊断拉取 | [handlers/diagnostics.rs#L54-L80](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/diagnostics.rs#L54-L80) | `!ghost_transaction && inter_file_dependencies` | 触发全文档诊断拉取（1s 去抖） |
| 签名帮助重触发 | [handlers/signature_help.rs#L353](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/signature_help.rs#L353) | `auto_signature_help && !ghost_transaction` | 重新请求签名帮助 |
| 文档高亮更新 | [handlers/document_highlight.rs#L153](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_highlight.rs#L153) | `auto_document_highlight && !ghost_transaction` | 重新请求文档高亮 |
| 文档颜色更新 | [handlers/document_colors.rs#L153](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_colors.rs#L153) | `!ghost_transaction` | 重新请求文档颜色 |
| 文档链接更新 | [handlers/document_links.rs#L132](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/document_links.rs#L132) | 总是触发（位置映射+重新请求） | 更新链接位置并重新请求 |
| Snippet 映射 | [handlers/snippet.rs#L14](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/snippet.rs#L14) | 总是触发 | 映射 snippet 占位符位置 |
| 自动保存 | [handlers/auto_save.rs](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/auto_save.rs) | `!ghost_transaction` | 重置自动保存计时器 |

> **重要**：代码补全（completion）**不在** DocumentDidChange 处理器列表中。

### 2.3 reload 触发的处理器

`document.reload()` 内部通过 `self.apply(&transaction, view.id)` 走标准变更路径，因此**上述所有 DocumentDidChange 处理器都会被触发**。

**reload 额外执行的操作**（除了 DocumentDidChange 之外）：

| 操作 | 代码位置 | 说明 |
|------|----------|------|
| `append_changes_to_history` | [document.rs#L1295](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1295) | 将变更加入撤销栈，可撤销 reload |
| `reset_modified()` | [document.rs#L1296](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1296) | 清除"未保存"标记 |
| `pickup_last_saved_time()` | [document.rs#L1297](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1297) | 重新读取磁盘 mtime |
| `detect_indent_and_line_ending()` | [document.rs#L1298](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1298) | 重新检测缩进和换行符 |
| 更新 diff base | [document.rs#L1300-L1303](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1300-L1303) | 更新 git diff 基线 |
| 更新 vcs head | [document.rs#L1305](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1305) | 更新版本控制 HEAD |
| LSP file_changed 通知 | [typed.rs#L1514-L1519](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/commands/typed.rs#L1514-L1519) | 额外触发 `workspace/didChangeWatchedFiles` |

### 2.4 ghost_transaction 的作用

`ghost_transaction = true` 表示"幽灵事务"，不应该通知 LSP 和相关处理器：

```rust
pub fn apply_temporary(&mut self, transaction: &Transaction, view_id: ViewId) -> bool {
    self.apply_inner(transaction, view_id, false)  // emit_lsp_notification = false
}
```

**受 `ghost_transaction` 过滤的处理器**：
- LSP didChange 通知
- 诊断拉取
- 签名帮助
- 文档高亮
- 文档颜色
- 文档链接重新请求（仅跳过重新请求，不跳过位置映射）
- 词索引更新
- 自动保存

**不受影响（始终执行）的处理器**：
- Snippet 位置映射
- 文档链接位置映射（仅位置更新部分）

---

## 三、补全触发为什么只来自插入模式按键事件

### 3.1 核心结论

**代码补全不监听 DocumentDidChange 事件**。补全触发完全来自**插入模式下的按键事件**，通过 `PostInsertChar` 和 `PostCommand` 等事件驱动。

### 3.2 补全触发的全部来源

补全事件触发点共 **6 处**，全部与插入模式相关：

#### 来源 1：PostInsertChar 事件（最主要）

**代码位置**：[handlers/completion.rs#L263-L270](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L263-L270)

```rust
register_hook!(move |event: &mut PostInsertChar<'_, '_>| {
    if event.cx.editor.last_completion.is_some() {
        update_completion_filter(event.cx, Some(event.c))
    } else {
        trigger_auto_completion(event.cx.editor, false);
    }
    Ok(())
});
```

- 每次在插入模式输入字符后触发
- 如果已有补全列表，则更新过滤器
- 否则尝试触发自动补全

#### 来源 2：PostCommand 事件（删除操作）

**代码位置**：[handlers/completion.rs#L200-L243](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L200-L243)

```rust
fn completion_post_command_hook(PostCommand { command, cx }: &mut PostCommand<'_, '_>) -> Result<()> {
    if cx.editor.mode == Mode::Insert {
        if cx.editor.last_completion.is_some() {
            // 已有补全时：处理删除命令
            match command {
                MappableCommand::Static { name: "delete_char_backward", .. } => update_completion_filter(cx, None),
                _ => clear_completions(cx),
            }
        } else {
            // 无补全时：发送 DeleteText 或 Cancel
            let event = match command {
                MappableCommand::Static { name: "delete_char_backward" | "delete_word_forward" | "delete_char_forward", .. } => {
                    CompletionEvent::DeleteText { cursor: primary_cursor }
                }
                MappableCommand::Static { name: "completion" | "insert_mode" | "append_mode", .. } => return Ok(()),
                _ => CompletionEvent::Cancel,
            };
            cx.editor.handlers.completions.event(event);
        }
    }
    Ok(())
}
```

- 处理插入模式下的删除操作
- 删除触发位置之前的内容会取消补全

#### 来源 3：OnModeSwitch 事件（进入/退出插入模式）

**代码位置**：[handlers/completion.rs#L248-L261](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L248-L261)

```rust
register_hook!(move |event: &mut OnModeSwitch<'_, '_>| {
    if event.old_mode == Mode::Insert {
        event.cx.editor.handlers.completions.event(CompletionEvent::Cancel);
        clear_completions(event.cx);
    } else if event.new_mode == Mode::Insert {
        trigger_auto_completion(event.cx.editor, false)
    }
    Ok(())
});
```

- 退出插入模式：取消补全
- 进入插入模式：尝试触发自动补全

#### 来源 4：手动触发命令

**代码位置**：[handlers.rs#L34-L40](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers.rs#L34-L40)

```rust
pub fn trigger_completions(&self, trigger_pos: usize, doc: DocumentId, view: ViewId) {
    self.completions.event(CompletionEvent::ManualTrigger {
        cursor: trigger_pos,
        doc,
        view,
    });
}
```

- 用户按 `C-x` 手动触发补全
- 手动触发**不经过去抖**，立即执行

#### 来源 5：补全接受后重新触发

**代码位置**：[ui/completion.rs#L281](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/ui/completion.rs#L281)

```rust
// we could have just inserted a trigger char (like a `crate::` completion for rust
// so we want to retrigger immediately when accepting a completion.
trigger_auto_completion(editor, true);
```

- 接受补全后，如果插入了触发字符，立即重新触发

#### 来源 6：补全为空后重新触发

**代码位置**：[handlers/completion.rs#L67-L70](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L67-L70)

```rust
if completion.is_empty() {
    editor_view.clear_completion(editor);
    trigger_auto_completion(editor, false);
}
```

- 补全结果为空时，可能是输入了触发字符，重新尝试

### 3.3 trigger_auto_completion 的判定逻辑

**代码位置**：[handlers/completion.rs#L118-L171](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L118-L171)

```
trigger_auto_completion()
    ↓
检查 auto_completion 配置是否开启
    ↓
获取光标位置
    ↓
┌─ 是触发字符吗？（LSP 声明的 trigger_characters）
│   是 → 发送 CompletionEvent::TriggerChar（5ms 去抖）
│   否 → 继续
└─
    ↓
┌─ 是路径补全触发吗？（/ 或 \）
│   是 → 发送 CompletionEvent::TriggerChar（5ms 去抖）
│   否 → 继续
└─
    ↓
┌─ 是自动触发吗？（末尾 N 个字符都是单词字符）
│   是 → 发送 CompletionEvent::AutoTrigger（配置时间去抖，默认250ms）
│   否 → 不触发
└─
```

### 3.4 为什么补全不监听 DocumentDidChange

**设计原因分析**：

1. **触发条件需要上下文**：补全触发需要判断当前是否在单词上、是否在触发字符后，这些需要知道光标位置和编辑方向，而 `DocumentDidChange` 只携带变更集合，不携带"为什么变"的信息。

2. **模式敏感性**：补全只在插入模式有意义，`DocumentDidChange` 不区分模式。

3. **精确控制**：通过 `PostInsertChar` 和 `PostCommand` 可以精确区分是输入字符、删除操作还是其他命令，从而做不同的处理（更新过滤器、取消补全等）。

4. **性能考虑**：`DocumentDidChange` 可能由非用户操作触发（如 reload、格式美化），这些情况下不需要触发补全。

5. **去抖边界不同**：
   - 诊断去抖：基于文档变更频率，关注"多久变一次"
   - 补全去抖：基于用户输入节奏，关注"打字速度"

---

## 四、外部文件改动提示 vs 补全触发：边界对比

### 4.1 触发源对比

| 维度 | 外部文件改动提示 | 代码补全触发 |
|------|------------------|-------------|
| **触发源** | 保存命令（`:w`） | 插入模式按键事件 |
| **事件类型** | 同步函数返回 Result | PostInsertChar / PostCommand |
| **检测机制** | mtime 对比（保存前检查） | 光标处字符特征判断 |
| **是否主动** | 被动（用户保存时才发现） | 主动（每次按键都检查） |

### 4.2 去抖机制对比

| 维度 | 外部文件改动 | 代码补全 |
|------|-------------|----------|
| **是否有去抖** | 无去抖（保存即检查） | 有去抖 |
| **去抖时间** | N/A | 自动：250ms / 触发字符：5ms / 手动：立即 |
| **去抖位置** | N/A | CompletionHandler（AsyncHook） |
| **合并策略** | N/A | 只保留最后一个 trigger 位置 |

### 4.3 处理器关联对比

| 处理器 | DocumentDidChange 触发 | 插入模式按键触发 |
|--------|----------------------|-----------------|
| LSP didChange 通知 | ✅ 是 | ❌ 否（已包含在 didChange 中） |
| 诊断拉取 | ✅ 是 | ❌ 否 |
| 签名帮助 | ✅ 是（重触发） | ✅ 是（PostInsertChar） |
| 文档高亮 | ✅ 是 | ❌ 否（SelectionDidChange 也触发） |
| 文档颜色 | ✅ 是 | ❌ 否 |
| 文档链接 | ✅ 是 | ❌ 否 |
| 自动保存 | ✅ 是 | ❌ 否 |
| 词索引 | ✅ 是 | ❌ 否 |
| Snippet 映射 | ✅ 是 | ❌ 否 |
| **代码补全** | **❌ 否** | **✅ 是** |

### 4.4 边界图示

```
┌──────────────────────────────────────────────────────────────────────┐
│                        文档内容变更                                    │
│                                                                      │
│  来源：                                                              │
│  - 用户打字（插入/普通模式）                                          │
│  - 撤销/重做                                                         │
│  - 自动格式化                                                         │
│  - reload（外部文件改动）                                             │
│  - LSP 编辑（代码动作、补全应用）                                     │
│  - 等等...                                                           │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                DocumentDidChange 事件
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
  LSP didChange         诊断拉取             签名帮助(重触发)
  文档高亮               自动保存             文档颜色/链接
  词索引                 Snippet 映射         ...
        │
        │ （补全不在此层）
        │
        │  另一条独立链路
        ▼
  插入模式按键事件
  (PostInsertChar / PostCommand)
        │
        ▼
  trigger_auto_completion()
        │
        ├─→ TriggerChar（5ms 去抖）
        ├─→ AutoTrigger（250ms 去抖）
        └─→ ManualTrigger（立即）
        │
        ▼
  CompletionHandler → LSP 补全请求
```

---

## 五、总结

### 5.1 外部文件改动提示

1. **没有自动监听**：Helix 不监听文件系统，外部改动只能在保存时或手动 reload 时发现
2. **保存冲突检测**：保存前对比磁盘 mtime 与 `last_saved_time`，冲突则 `set_error()` 显示红色状态栏提示
3. **手动 reload**：用户执行 `:reload` 从磁盘重新读取，通过 diff 事务应用变更

### 5.2 文档变更触发的处理器

`DocumentDidChange` 事件触发 **10+ 个处理器**，包括：
- LSP 内容同步、诊断拉取、签名帮助、文档高亮
- 文档颜色、文档链接、自动保存、词索引、Snippet 映射
- **但不包括代码补全**

### 5.3 补全触发的边界

1. **补全不监听 DocumentDidChange**：完全由插入模式按键事件驱动
2. **6 个触发来源**：PostInsertChar、PostCommand、OnModeSwitch、手动命令、补全接受后、补全清空后
3. **设计原因**：需要光标上下文、模式敏感、精确区分输入类型、避免非用户操作触发
4. **去抖分层**：自动触发 250ms、触发字符 5ms、手动触发 0ms

### 5.4 关键文件索引

| 主题 | 文件 |
|------|------|
| 保存冲突检测 | [document.rs#L1038-L1047](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1038-L1047) |
| 状态消息系统 | [editor.rs#L1448-L1485](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/editor.rs#L1448-L1485) |
| DocumentDidChange 派发 | [document.rs#L1605-L1611](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1605-L1611) |
| reload 实现 | [document.rs#L1270-L1308](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/document.rs#L1270-L1308) |
| 补全触发函数 | [handlers/completion.rs#L118-L171](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L118-L171) |
| 补全 hook 注册 | [handlers/completion.rs#L245-L271](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion.rs#L245-L271) |
| 补全去抖 handler | [handlers/completion/request.rs#L68-L159](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-term/src/handlers/completion/request.rs#L68-L159) |
| LSP didChange hook | [handlers/lsp.rs#L407-L420](file:///d:/fz/0601/solo-dogfeeding/code/281-helix/helix-view/src/handlers/lsp.rs#L407-L420) |
