# Helix 补全弹层代码链路解析

本文档对照源码，梳理补全弹层从"候选产生"到"排序展示"再到"确认插入"的完整代码链路，讲清三者之间的协作关系。

---

## 一、全局架构概览

补全弹层涉及的核心文件与职责：

| 层 | 文件 | 职责 |
|---|---|---|
| 事件触发层 | [completion.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion.rs) | 注册 hook、触发补全请求、编排 show/replace/clear |
| 请求调度层 | [request.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/request.rs) | `CompletionHandler`(AsyncHook) 防抖、分发并行请求 |
| 候选来源层 | [request.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/request.rs) LSP 请求 / [word.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/word.rs) 单词补全 / [path.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/path.rs) 路径补全 | 各来源独立返回 `CompletionResponse` |
| 数据模型层 | [item.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/item.rs) + [completion.rs(core)](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-core/src/completion.rs) | `CompletionItem` 枚举、`CompletionResponse`、`CompletionProvider` |
| UI 组件层 | [completion.rs(ui)](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/ui/completion.rs) | `Completion` 结构体：包装 `Popup<Menu<CompletionItem>>`，负责评分排序、过滤、渲染文档浮窗 |
| 菜单渲染层 | [menu.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/ui/menu.rs) | `Menu<T>` 通用菜单组件：维护 matches 列表、光标、滚动、键盘事件 |
| 弹层容器层 | [popup.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/ui/popup.rs) | `Popup<T>` 通用弹层容器：定位、边框、滚动条、事件代理 |
| 编辑器视图层 | [editor.rs(ui)](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/ui/editor.rs) | `EditorView` 持有 `completion: Option<Completion>`，提供 set/clear |
| 视图模型层 | [completion.rs(view)](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-view/src/handlers/completion.rs) | `CompletionHandler`(view 侧)：事件通道、`active_completions`、`request_controller` |
| 异步解析层 | [resolve.rs](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/resolve.rs) | `ResolveHandler`：延迟解析 LSP completion item 的 documentation |

---

## 二、候选来源（数据如何产生/获取）

### 2.1 触发入口

补全的触发有三条路径，最终都向 `CompletionHandler`（view 层的 `event_tx`）发送 `CompletionEvent`：

1. **自动触发（AutoTrigger）**：用户在 insert mode 下输入普通字符，hook `PostInsertChar` 调用 `trigger_auto_completion` → 检测光标前 `completion_trigger_len` 个字符均为 word char → 发送 `CompletionEvent::AutoTrigger`
2. **触发字符（TriggerChar）**：输入的字符命中 LSP 声明的 `trigger_characters` 或路径分隔符 `/`/`\` → 发送 `CompletionEvent::TriggerChar`
3. **手动触发（ManualTrigger）**：用户按 `C-x` 执行 `completion` 命令 → 发送 `CompletionEvent::ManualTrigger`

见 [completion.rs#L118-L171](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion.rs#L118-L171)

### 2.2 防抖与请求分发

`CompletionHandler`（request.rs 中的 `AsyncHook` 实现）收到事件后：

- `AutoTrigger`：设置超时（`completion_timeout`，默认 80ms），防抖后调用 `finish_debounce`
- `TriggerChar`：设置极短超时（5ms），几乎立即执行
- `ManualTrigger`：直接调用 `finish_debounce`，无等待

`finish_debounce` 调用 `request_completions`，该函数并行发起三类候选请求：

见 [request.rs#L161-L290](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/request.rs#L161-L290)

### 2.3 三类候选来源

#### (A) LSP 语言服务器

```rust
// request.rs#L292-L339
fn request_completions_from_language_server(ls, doc, view, context, priority, savepoint)
```

- 向每个支持 `Completion` 特性的 language server 发送 `textDocument/completion` LSP 请求
- 每个 LS 的 `provider_priority` 为 `-(enumerate 顺序 as i8)`：**配置顺序越靠后的 LS，priority 数值越小，排序越靠前**
- 返回前按 LSP 的 `sort_text`（fallback 到 `label`）对 items 做 LSP 侧初始排序
- 封装为 `CompletionResponse { items: CompletionItems::Lsp(items), provider: CompletionProvider::Lsp(ls_id), context }`

#### (B) 路径补全（Path）

```rust
// path.rs#L17-L133
pub(crate) fn path_completion(selection, doc, handle, savepoint)
```

- 检查 `doc.path_completion_enabled()`
- 解析光标前行的路径后缀（`get_path_suffix`），得到目录路径 + 已输入文件名
- 读取目录 `std::fs::read_dir`，为每个条目生成 `CompletionItem::Other(core::CompletionItem { provider: CompletionProvider::Path, .. })`
- `ResponseContext.priority` 固定为 `1`，但 `CompletionItem::Other` 的 `provider_priority()` 方法直接返回 1（不使用 context 中的 priority）

#### (C) 单词补全（Word）

```rust
// word.rs#L16-L106
pub(super) fn completion(editor, trigger, handle, savepoint)
```

- 检查 `doc.word_completion_enabled()`
- 从 `editor.handlers.word_index` 中查找匹配当前输入词的单词
- 已输入词长需 >= `trigger_length`（默认 2）
- 生成 `CompletionItem::Other(core::CompletionItem { provider: CompletionProvider::Word, .. })`
- `ResponseContext.priority` 为 `0`，但 `CompletionItem::Other` 的 `provider_priority()` 方法直接返回 1（不使用 context 中的 priority）

### 2.4 响应汇聚与首屏展示

三类请求被 `JoinSet` 并行执行。`request_completions` 中的异步逻辑：

1. 等待第一个响应（`handle_response`），将 items 通过 `take_items` 合并到 `Vec<CompletionItem>`
2. 继续等待后续响应直到 100ms 超时或全部完成
3. 调用 `show_completion(editor, compositor, items, context, trigger)` 将首批候选推送到 UI
4. 如果还有未完成的请求，启动 `replace_completions` 异步循环，后续响应到达后动态替换弹层中的候选

见 [request.rs#L260-L290](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/request.rs#L260-L290)

### 2.5 `CompletionItem` 的统一数据模型

所有来源的候选最终统一为 `CompletionItem` 枚举：

```rust
// item.rs#L71-L75
pub enum CompletionItem {
    Lsp(LspCompletionItem),     // LSP 来源
    Other(helix_core::CompletionItem),  // 路径/单词来源
}
```

其中 `LspCompletionItem` 额外携带 `provider`(LanguageServerId)、`resolved`(是否已 resolve)、`provider_priority`(i8)。

`CompletionProvider` 枚举标识来源类型：

```rust
// completion.rs(core)#L16-L20
pub enum CompletionProvider {
    Lsp(LanguageServerId),
    Path,
    Word,
}
```

### 2.6 `take_items`：来源到统一模型的转换

```rust
// item.rs#L28-L44
impl CompletionResponse {
    pub fn take_items(&mut self, dst: &mut Vec<CompletionItem>) {
        match &mut self.items {
            CompletionItems::Lsp(items) => dst.extend(items.drain(..).map(|item| {
                CompletionItem::Lsp(LspCompletionItem { item, provider, resolved: false, provider_priority })
            })),
            CompletionItems::Other(items) if dst.is_empty() => mem::swap(dst, items),
            CompletionItems::Other(items) => dst.append(items),
        }
    }
}
```

LSP items 在转换时被包装为 `LspCompletionItem`，其中 `provider_priority` 取自 `self.context.priority`（即 `-(enumerate index)`）；Non-LSP items 直接追加，其 `provider_priority()` 方法固定返回 1。

---

## 三、排序与展示（数据如何排序并渲染到弹层）

### 3.1 从 `show_completion` 到 `Completion::new`

```rust
// completion.rs(handlers)#L83-L116
fn show_completion(editor, compositor, items, context, trigger) {
    // 校验模式/文档一致性
    word::retain_valid_completions(trigger, doc, view.id, &mut items);
    editor.handlers.completions.active_completions = context;
    let completion_area = ui.set_completion(editor, items, trigger.pos, size);
}
```

```rust
// editor.rs(ui)#L1101-L1122
pub fn set_completion(&mut self, editor, items, trigger_offset, size) -> Option<Rect> {
    let mut completion = Completion::new(editor, items, trigger_offset);
    if completion.is_empty() { return None; }
    editor.last_completion = Some(CompleteAction::Triggered);
    self.completion = Some(completion);
    Some(area)
}
```

### 3.2 `Completion::new` — 初始化弹层

```rust
// completion.rs(ui)#L132-L321
pub fn new(editor: &Editor, items: Vec<CompletionItem>, trigger_offset: usize) -> Self
```

关键步骤：

1. **创建 `Menu<CompletionItem>`**：传入所有 items 和回调函数 `callback_fn`
2. **创建 `Popup`**：`Popup::new(Self::ID, menu).with_scrollbar(false).ignore_escape_key(true)`
3. **计算初始 filter**：从 `start_offset`(当前 word 起点) 到 `cursor` 的文本片段作为初始过滤字符串
4. **立即调用 `score(false)`**：执行全量评分排序

### 3.3 核心排序逻辑：`score` 方法

```rust
// completion.rs(ui)#L323-L380
fn score(&mut self, incremental: bool)
```

这是排序展示的关键方法，流程如下：

#### Step 1: 模糊匹配评分

使用 nucleo 模糊匹配器对每个候选的 `filter_text` 与当前 `filter` 字符串做匹配：

- **增量模式**（`incremental=true`，用于用户继续输入）：只对现有 matches 重新评分，移除不匹配项
- **全量模式**（`incremental=false`，用于初始化/新响应到达）：遍历所有 options，重新计算

```rust
let pattern = Atom::new(pattern, CaseMatching::Ignore, Normalization::Smart, AtomKind::Fuzzy, false);
// 匹配后 score = nucleo_score / 3 (全量) 或 nucleo_score / 2 (增量)
```

#### Step 2: 多级排序

```rust
// completion.rs(ui)#L370-L379
matches.sort_unstable_by_key(|&(i, score)| {
    let option = &options[i as usize];
    (
        score <= min_score,           // 1. 低于阈值排到末尾
        Reverse(option.preselect()),  // 2. LSP preselect 优先
        option.provider_priority(),   // 3. provider 优先级（越小越前）
        Reverse(score),               // 4. 模糊匹配分高的优先
        i,                            // 5. 原始顺序兜底
    )
});
```

排序键优先级从高到低（按元组字段顺序）：

| 优先级 | 键 | 类型 | 排序方向 | 说明 |
|---|---|---|---|---|
| 1 | `score <= min_score` | bool | 升序（false 在前） | 低于最低阈值的项统一沉底。`false=0 < true=1`，所以匹配质量达标的项全部排在不达标之前 |
| 2 | `Reverse(option.preselect())` | bool | 降序（true 在前） | LSP 标记 `preselect=true` 的项优先。`Reverse` 反转 bool 默认顺序，使 true 排在 false 前面 |
| 3 | `option.provider_priority()` | i8 | **升序（越小越前）** | 来源优先级。数值越小排名越靠前 |
| 4 | `Reverse(score)` | u32 | 降序（越高越前） | 模糊匹配得分。得分越高排名越靠前 |
| 5 | `i` | u32 | 升序（越小越前） | 原始索引保序。当以上所有键都相等时，按在 options 中出现的先后顺序排列 |

其中 `min_score = (7 + needle_len * 14) / 3`，是一个启发式阈值，用于过滤掉匹配质量过差的候选项（但不直接剔除，只是沉底）。

#### Step 3: `provider_priority` 的数值与来源

`provider_priority()` 返回值（升序，越小越靠前）：

| 来源 | provider_priority 值 | 排序位置 |
|---|---|---|
| 第 N 个配置的 LSP（N 越大越靠后配置） | `-(N-1)` 即 0, -1, -2, ... | 越靠后配置的 LSP 数值越小，排名越靠前 |
| Path 补全 | 1 | LSP 之后 |
| Word 补全 | 1 | LSP 之后，与 Path 同级 |

**注意**：所有非 LSP 来源（Path、Word）在 `CompletionItem::Other` 分支中**统一返回 1**，与 `ResponseContext.priority` 无关。

```rust
// item.rs#L105-L127
impl CompletionItem {
    pub fn provider_priority(&self) -> i8 {
        match self {
            CompletionItem::Lsp(item) => item.provider_priority,  // 来自 request 时的 -(enumerate index)
            CompletionItem::Other(_) => 1,  // Path/Word 固定为 1
        }
    }

    pub fn preselect(&self) -> bool {
        match self {
            CompletionItem::Lsp(item) => item.item.preselect.unwrap_or(false),
            CompletionItem::Other(_) => false,  // 非 LSP 项永远不会被 preselect
        }
    }
}
```

##### LSP provider_priority 的计算

在 [request.rs#L203-L241](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/handlers/completion/request.rs#L203-L241) 中：

```rust
for (priority, ls) in language_servers.iter().enumerate() {
    requests.spawn(request_completions_from_language_server(
        ...,
        -(priority as i8),   // priority 是 enumerate 索引：0, 1, 2...
        ...,
    ));
}
```

- `language_servers` 按 `language_config()` 配置顺序返回（第一个配置的 LS 最先返回）
- 取负后：第一个配置的 LS → priority=0，第二个 → priority=-1，第三个 → priority=-2，依此类推
- 排序时按升序排列：-2 < -1 < 0 < 1
- **结论：配置中越靠后的 LSP，在补全列表中优先级越高**

##### Path 和 Word 的优先级关系

Path 和 Word 的 `provider_priority` 都是 1，处于同一层级。它们之间的相对顺序由后续排序键决定：
1. 先比较模糊匹配得分 `Reverse(score)`
2. 得分相同则比较原始索引 `i`（先加入 options 的排在前面）

##### 多 LSP 场景下的完整排序示例

假设有 3 个 LSP（rust-analyzer, rls, clippy）按顺序配置，以及 Path、Word 补全：

| 候选项 | 来源 | provider_priority | preselect | score (假设) | 最终排序位置 |
|---|---|---|---|---|---|
| func_a | clippy (第3个) | -2 | false | 150 | 1 |
| func_b | rls (第2个) | -1 | true | 100 | 2 |
| func_c | rust-analyzer (第1个) | 0 | false | 200 | 3 |
| func_d | rust-analyzer (第1个) | 0 | false | 180 | 4 |
| src/ | Path | 1 | false | 160 | 5 |
| some_word | Word | 1 | false | 140 | 6 |
| low_qual | rls (第2个) | -1 | false | 5 | 7（低于 min_score 沉底） |

> 注意：func_b 虽然 score 只有 100，但因为 preselect=true，所以排在 func_c 前面。
> clippy 的 func_a 虽然 score 不是最高，但因为 provider_priority=-2 最小，所以排第 1。

### 3.4 过滤更新：`update_filter`

```rust
// completion.rs(ui)#L410-L425
pub fn update_filter(&mut self, c: Option<char>) {
    match c {
        Some(c) => self.filter.push(c),   // 追加字符
        None => { self.filter.pop(); ... } // 删除字符
    }
    self.score(c.is_some());              // 增量或全量重排
    self.popup.contents_mut().reset_cursor();
}
```

用户每输入/删除一个字符，在 `PostInsertChar` hook 中调用 `update_filter`，触发增量评分和光标重置。

### 3.5 动态替换候选：`replace_provider_completions`

```rust
// completion.rs(ui)#L427-L441
pub fn replace_provider_completions(&mut self, response, is_incomplete) {
    let (_, options) = menu.update_options();
    if is_incomplete {
        options.retain(|item| item.provider() != response.provider)  // 移除旧的同 provider 候选
    }
    response.take_items(options);  // 合入新候选
    self.score(false);             // 全量重排
    menu.ensure_cursor_in_bounds();
}
```

当后续 LSP 响应异步到达时，`replace_completions` 调用此方法将新候选合入并重排。

### 3.6 渲染层级

`Completion` 的渲染委托链：

```
Completion::render()
  ├── self.popup.render()              // Popup 容器渲染（定位、背景、边框）
  │     └── self.contents.render()     // Menu 渲染（表格行、高亮选中行、滚动条）
  └── 文档浮窗渲染                      // Completion 自行处理
        └── Markdown::render()         // 选中项的 detail + documentation
```

#### Menu 渲染细节

```rust
// menu.rs#L335-L420  Menu::render()
```

- 从 `self.matches`（排序后的 `(index, score)` 列表）取当前可视范围的候选项
- 调用 `option.format(&self.editor_data)` 获取 `Row`（含 label 列和 kind 列）
- 用 tui `Table` 渲染，`highlight_style` 标记当前选中行
- 超出可视区时渲染滚动条

#### `CompletionItem::format` — 每行如何展示

```rust
// completion.rs(ui)#L28-L117
impl menu::Item for CompletionItem {
    fn format(&self, dir_style: &Style) -> Row<'_> {
        // label 列：deprecated 用删除线，folder 用目录样式
        // kind 列：LSP kind 映射为文字（"function", "variable" 等）；Color kind 渲染色块
        Row::new([Cell::from(label), Cell::from(kind)])
    }
}
```

#### 文档浮窗

```rust
// completion.rs(ui)#L469-L580  Completion::render()
```

当有选中项时：
1. 若为 LSP 项，调用 `resolve_handler.ensure_item_resolved` 异步解析 documentation
2. 将 `detail` + `documentation` 组合为 Markdown 渲染
3. 优先放在弹层右侧；空间不足则放在弹层上方或下方

---

## 四、确认插入（用户选择后如何插入到编辑器）

### 4.1 三种事件与回调

`Menu` 的 `callback_fn` 接收三种 `MenuEvent`（即 `PromptEvent`）：

| 事件 | 触发方式 | 语义 |
|---|---|---|
| `Abort` | `Esc` / `C-c` | 取消补全，关闭弹层 |
| `Update` | `Tab`/`C-n`/`Down`/`Shift-Tab`/`C-p`/`Up`/`PageDown`/`PageUp` | 移动选中项 |
| `Validate` | `Enter` | 确认选中项，插入文本 |

见 [menu.rs#L264-L301](file:///d:/fz/0601/solo-dogfeeding/code/267-helix/helix-term/src/ui/menu.rs#L264-L301)

### 4.2 Update — 预览插入（Ghost Transaction）

当 `preview_completion_insert` 配置开启时，`Update` 事件会执行"幽灵事务"：

```rust
// completion.rs(ui)#L161-L199
PromptEvent::Update if preview_completion_insert => {
    // 1. 首次 Update 时保存 savepoint
    if matches!(editor.last_completion, Some(CompleteAction::Triggered)) {
        editor.last_completion = Some(CompleteAction::Selected {
            savepoint: doc.savepoint(view),
        })
    }
    // 2. 恢复到该 provider 的 savepoint（撤销之前的预览）
    doc.restore(view, &context.savepoint, false);
    // 3. 应用当前选中项的 transaction 为"临时事务"（不发送给 LSP）
    doc.apply_temporary(&transaction, view.id);
}
```

关键点：
- `CompleteAction::Triggered` → `Selected { savepoint }` 的转换只在首次 Update 时发生
- 每次切换选中项时，先 restore 到 provider 的 savepoint，再 apply_temporary 新选中项
- `apply_temporary` 不会通知 LSP，避免增量同步混乱

### 4.3 Validate — 确认插入

```rust
// completion.rs(ui)#L202-L283
PromptEvent::Validate => {
    // 1. 如果有预览选中（Selected），先恢复 savepoint
    if let Some(CompleteAction::Selected { savepoint }) = editor.last_completion.take() {
        doc.restore(view, &savepoint, false);
    }
    // 2. 恢复到 provider 的 savepoint（撤销所有预览），并发送撤销给 LSP
    doc.restore(view, &context.savepoint, true);
    // 3. 保存 undo checkpoint
    doc.append_changes_to_history(view);

    // 4. 根据 item 类型生成 transaction
    match item.clone() {
        CompletionItem::Lsp(mut item) => {
            // 4a. 如果未 resolved，同步 resolve
            if !item.resolved {
                if let Some(resolved_item) = Self::resolve_completion_item(ls, item.item.clone()) {
                    item.item = resolved_item;
                }
            }
            // 4b. 生成 transaction（可能含 snippet）
            let (transaction, snippet) = lsp_item_to_transaction(doc, view.id, &item.item, ...);
            // 4c. 额外编辑（additional_text_edits）
            let add_edits = item.item.additional_text_edits;
            (transaction, add_edits.map(|edits| (edits, encoding)), snippet)
        }
        CompletionItem::Other(core::CompletionItem { transaction, .. }) => {
            (transaction, None, None)
        }
    };

    // 5. 应用主 transaction
    doc.apply(&transaction, view.id);

    // 6. 处理 snippet
    if let Some(snippet) = snippet {
        doc.active_snippet = ...;
    }

    // 7. 记录 CompleteAction::Applied
    editor.last_completion = Some(CompleteAction::Applied {
        trigger_offset,
        changes: completion_changes(&transaction, trigger_offset),
        placeholder,
    });

    // 8. 应用 additional_text_edits
    if let Some((additional_edits, offset_encoding)) = additional_edits {
        let transaction = util::generate_transaction_from_edits(doc.text(), additional_edits, offset_encoding);
        doc.apply(&transaction, view.id);
    }

    // 9. 可能触发新一轮自动补全（如 rust 的 `crate::` 触发字符）
    trigger_auto_completion(editor, true);
}
```

### 4.4 `lsp_item_to_transaction` — LSP 项到事务的转换

```rust
// completion.rs(ui)#L582-L657
fn lsp_item_to_transaction(doc, view_id, item, offset_encoding, trigger_offset, replace_mode)
```

两种模式：
1. **有 `text_edit`**：从 LSP `TextEdit` 的 range 计算相对于 cursor 的偏移，得到替换范围和新文本
   - `InsertAndReplace` 类型根据 `replace_mode` 配置选择 `insert` 或 `replace` range
2. **无 `text_edit`**：用 `insert_text` 或 `label` 作为新文本，在 trigger_offset 处插入

如果 `kind == SNIPPET` 或 `insert_text_format == SNIPPET`：
- 解析 snippet → `generate_transaction_from_snippet` → 返回 `(transaction, Some(rendered_snippet))`

否则：
- `generate_transaction_from_completion_edit` → 返回 `(transaction, None)`

### 4.5 `CompleteAction` 状态机

`editor.last_completion` 贯穿整个补全生命周期：

```
None
  │ (set_completion)
  ▼
CompleteAction::Triggered          ← 弹层刚出现，尚无选中
  │ (首次 Update + preview_completion_insert)
  ▼
CompleteAction::Selected { savepoint }  ← 用户开始浏览，保存了预览前的 savepoint
  │ (Validate)
  ▼
CompleteAction::Applied { trigger_offset, changes, placeholder }  ← 已插入，记录变更用于重放
  │ (clear_completion)
  ▼
None
```

### 4.6 清理：`clear_completion`

```rust
// editor.rs(ui)#L1124-L1158
pub fn clear_completion(&mut self, editor) {
    self.completion = None;
    editor.handlers.completions.request_controller.restart();
    editor.handlers.completions.active_completions.clear();
    match editor.last_completion.take() {
        CompleteAction::Triggered => (),
        CompleteAction::Applied { trigger_offset, changes, placeholder } => {
            // 记录 InsertEvent::CompletionApply
            // 如果有 snippet placeholder，注册 on_next_key 回调
        }
        CompleteAction::Selected { savepoint } => {
            // 恢复 savepoint（撤销预览但不通知 LSP）
            doc.restore(view, &savepoint, false);
        }
    }
}
```

### 4.7 异步 Resolve：延迟补全文档

```rust
// resolve.rs#L38-L89
impl ResolveHandler {
    pub fn ensure_item_resolved(&mut self, editor, item: &mut LspCompletionItem) {
        if item.resolved { return; }
        // 如果 documentation + detail + additional_text_edits 都有内容，视为已 resolved
        if is_resolved { item.resolved = true; return; }
        // 发起异步 resolve 请求（150ms 防抖）
        send_blocking(&self.resolver, ResolveRequest { item, ls });
    }
}
```

resolve 完成后通过 `dispatch` 回调替换弹层中的旧 item：

```rust
// resolve.rs#L147-L169
completion.replace_item(&*self.item, resolved_item);
```

---

## 五、协作关系总览图

```
用户输入字符
    │
    ▼
PostInsertChar hook ──→ trigger_auto_completion() ──→ CompletionEvent::AutoTrigger/TriggerChar
    │                                                        │
    │                                                        ▼
    │                                              CompletionHandler (AsyncHook, 防抖)
    │                                                        │
    │                                                        ▼ finish_debounce
    │                                              request_completions()
    │                                               ┌────────┼────────┐
    │                                               ▼        ▼        ▼
    │                                          LSP请求   path补全   word补全
    │                                          (async)  (blocking) (blocking)
    │                                               │        │        │
    │                                               ▼        ▼        ▼
    │                                         CompletionResponse  CompletionResponse  CompletionResponse
    │                                               │        │        │
    │                                               └────┬───┘────────┘
    │                                                    ▼
    │                                         take_items() → Vec<CompletionItem>
    │                                                    │
    │                                                    ▼
    │                                         show_completion() → EditorView::set_completion()
    │                                                    │
    │                                                    ▼
    │                                         Completion::new() → score() → 排序后的 matches
    │                                                    │
    │                                                    ▼
    │                                         Popup<Menu<CompletionItem>> 渲染到屏幕
    │
    │  用户继续输入 ──→ update_filter() ──→ score(incremental) ──→ 增量重排
    │
    │  后续 LSP 响应到达 ──→ replace_provider_completions() ──→ score(false) ──→ 全量重排
    │
    ▼
用户选中 + Enter (Validate)
    │
    ├─ 恢复 savepoint（撤销预览）
    ├─ 同步 resolve LSP item（如需要）
    ├─ lsp_item_to_transaction() / 直接使用 transaction
    ├─ doc.apply(transaction)
    ├─ 处理 snippet placeholder
    ├─ 应用 additional_text_edits
    ├─ trigger_auto_completion()（可能立即触发下一轮）
    └─ editor.last_completion = Applied { ... }
```

---

## 六、关键设计决策

1. **三个来源并行 + 首屏超时**：LSP 请求异步、word/path 同步计算，首批结果立即展示，后续结果 100ms 内继续等待，超时后异步替换
2. **统一的 `CompletionItem` 枚举**：LSP 和非 LSP 来源在排序/过滤/展示层面统一处理，仅在插入时分支；非 LSP 项的 `provider_priority` 固定为 1，与 `ResponseContext.priority` 无关
3. **五级排序策略**：阈值沉底 → LSP preselect → provider_priority（配置越靠后的 LSP 优先级越高） → 模糊匹配分 → 原始索引。兼顾 LSP 意图、来源优先级和匹配质量
4. **Ghost Transaction 机制**：预览插入用 `apply_temporary`（不通知 LSP），确认插入用 `apply`（通知 LSP），通过 savepoint 保证状态一致性
5. **延迟 resolve**：只在用户选中某项时才异步请求 `completionItem/resolve`，避免大量无用的 resolve 请求
6. **Incomplete list 处理**：LSP 标记 `is_incomplete` 的响应在用户继续输入时会被重新请求（`request_incomplete_completion_list`），而非仅做本地过滤
