# 诊断(Diagnostics)展示脉络分析

本文梳理 Helix 编辑器中诊断信息从产生到最终展示在界面上的完整代码路径，涵盖四大维度：诊断来源、文档绑定、界面标记、刷新时机。

---

## 1. 诊断来源

诊断数据统一来自 LSP（Language Server Protocol），分为 **Push** 和 **Pull** 两种模式。

### 1.1 Push 模式：`textDocument/publishDiagnostics`

LSP 服务器主动推送诊断通知。入口在 [application.rs](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/application.rs#L831-L836)：

```
Notification::PublishDiagnostics(params) => {
    self.editor.handle_lsp_diagnostics(
        &provider,
        uri,
        params.version,
        params.diagnostics,
    );
}
```

### 1.2 Pull 模式：`textDocument/diagnostic`

编辑器主动向 LSP 服务器请求诊断。核心逻辑在 [helix-term/src/handlers/diagnostics.rs](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L162-L259)。

- `request_document_diagnostics` / `request_document_diagnostics_for_language_severs`：构造并发请求
- `handle_pull_diagnostics_response`：处理响应，区分 `Full` 和 `Unchanged` 两种报告
- 支持 `result_id` 增量拉取（存储在 [Document.previous_diagnostic_ids](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/document.rs#L210)）
- 支持服务端取消后重试（`retrigger_request`）

### 1.3 诊断数据结构

核心结构定义在 [helix-core/src/diagnostic.rs](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-core/src/diagnostic.rs#L33-L47)：

```rust
pub struct Diagnostic {
    pub range: Range,          // 字符范围
    pub line: usize,           // 行号（用于 gutter 快速查找）
    pub message: String,       // 诊断消息
    pub severity: Option<Severity>,  // Hint/Info/Warning/Error
    pub code: Option<NumberOrString>,
    pub provider: DiagnosticProvider,  // 来源标识
    pub tags: Vec<DiagnosticTag>,      // Unnecessary/Deprecated
    pub source: Option<String>,
    pub data: Option<serde_json::Value>,
}
```

[DiagnosticProvider](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-core/src/diagnostic.rs#L53-L68) 标识诊断来源：

```rust
pub enum DiagnosticProvider {
    Lsp {
        server_id: LanguageServerId,
        identifier: Option<Arc<str>>,  // Pull 诊断的命名空间
    },
}
```

---

## 2. 文档绑定

诊断与文档的绑定分为两层：**Editor 级原始缓存** 和 **Document 级转换后数据**。

### 2.1 Editor 级缓存

[Editor.diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/editor.rs#L1188) 类型为：

```rust
type Diagnostics = BTreeMap<Uri, Vec<(lsp::Diagnostic, DiagnosticProvider)>>;
```

- 以 URI 为键，存储**原始 LSP 诊断**（未经 offset 编码转换）
- 即使文档未打开也保留，用于 diagnostics picker 等功能
- `handle_lsp_diagnostics` 方法同时维护此缓存

### 2.2 Document 级数据

[Document.diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/document.rs#L199) 类型为 `Vec<Diagnostic>`（已转换的 helix_core::Diagnostic）。

绑定流程通过 [handle_lsp_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/handlers/lsp.rs#L279-L365)：

1. **版本校验**：若 `version` 不匹配 `doc.version()`，丢弃过时诊断
2. **persistent_diagnostic_sources 优化**：语言配置中标记为持久源的 source，如果新旧诊断内容一致则跳过替换
3. **Editor.diagnostics 更新**：移除同一 provider 的旧诊断，追加新诊断，按 `(severity, range.start, provider)` 排序
4. **Document.diagnostics 更新**：调用 [doc.replace_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/document.rs#L2221-L2255)
   - 先清除同 provider 的旧诊断（保留 `unchanged_sources` 中的）
   - 追加新诊断
   - 按 `(range, severity, provider)` 排序（保证 InlineDiagnosticAccumulator 可顺序扫描）
5. **派发事件**：`DiagnosticsDidChange`

### 2.3 URI → Document 查找

`handle_lsp_diagnostics` 通过 URI 在 `editor.documents` 中查找匹配文档：

```rust
let doc = self.documents.values_mut()
    .find(|doc| doc.uri().is_some_and(|u| u == uri));
```

### 2.4 诊断转换

[doc_diagnostics_with_filter](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/editor.rs#L2291-L2326) 负责 `lsp::Diagnostic` → `helix_core::Diagnostic` 的转换：

- 检查 LS 是否支持 `Diagnostics` 特性
- 使用 LS 的 `offset_encoding` 将 LSP Position 转为字符偏移
- 应用 filter 过滤

### 2.5 诊断清除

- [clear_diagnostics_for_language_server](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/document.rs#L2258-L2261)：移除指定 LS 的诊断
- `replace_diagnostics` 中传入 `provider` 参数时只清除同 provider 的诊断

---

## 3. 界面标记

诊断在界面上的展示有四个层面：**行内高亮**、**Gutter 标记**、**行内文本**、**底部面板**。

### 3.1 行内高亮（Overlay Highlights）

在 [editor.rs#doc_diagnostics_highlights_into](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/ui/editor.rs#L350-L449) 中实现：

- 遍历 `doc.diagnostics()`，按 severity 分组收集 range
- 对每个诊断的 `range` 应用对应的主题 scope：
  - `diagnostic`（fallback）
  - `diagnostic.hint` / `diagnostic.info` / `diagnostic.warning` / `diagnostic.error`
  - `diagnostic.unnecessary`（dim 修饰）
  - `diagnostic.deprecated`（crossed_out 修饰）
- 若 severity 为 Hint/Info 且有 tags，则只渲染 tag 高亮而不渲染 severity 高亮
- Warning/Error 级别同时渲染 severity 和 tag 高亮
- 重叠 range 会被合并

### 3.2 Gutter 标记

在 [gutter.rs#diagnostic](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/gutter.rs#L48-L83) 中实现：

- 使用 `partition_point` 按行号二分查找 `doc.diagnostics`
- 取该行最高 severity 的诊断，渲染 `●` 符号
- 颜色对应 `warning`/`error`/`info`/`hint` 主题色
- 还会过滤掉当前 LS 不支持 Diagnostics 特性的诊断
- 与断点标记、执行暂停标记组合（`diagnostics_or_breakpoints`），优先级：执行暂停 > 断点 > 诊断

### 3.3 行内文本展示（Inline Diagnostics）

这是最复杂的展示方式，分两层实现：

#### 3.3.1 注解层（LineAnnotation）

[InlineDiagnosticAccumulator](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/annotations/diagnostics.rs#L122-L255) 在文档格式化阶段工作：

- 顺序扫描 `doc.diagnostics`（依赖已排序特性）
- `process_anchor`：在诊断 range.start 对应的 grapheme 位置收集诊断到 stack
- 跟踪 `cursor_line`：当 grapheme.char_idx == cursor 时标记
- `compute_line_diagnostics`：根据 cursor_line/other_lines 的 filter 过滤 stack，截断到 max_diagnostics
- `insert_virtual_lines`：计算诊断消息文本所需的额外虚拟行数

[InlineDiagnosticsConfig](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/annotations/diagnostics.rs#L53-L60) 控制行为：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| cursor_line | Enable(Warning) | 光标行显示 >= Warning 的诊断 |
| other_lines | Disable | 非光标行不显示行内诊断 |
| min_diagnostic_width | 40 | 最小诊断文本宽度 |
| prefix_len | 1 | 连接线前缀长度 |
| max_wrap | 20 | 最大软换行数 |
| max_diagnostics | 10 | 每行最大诊断数 |

#### 3.3.2 渲染层（Decoration）

[InlineDiagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/ui/text_decorations/diagnostics.rs#L46-L314) 负责实际绘制：

**EOL 诊断**：在行尾直接绘制诊断消息（由 `end_of_line_diagnostics` 配置控制），优先显示比 inline filter 更高 severity 的诊断

**行下诊断**：使用 Box-drawing 字符绘制连接线和消息文本：
- `└` (BR_CORNER)：诊断起始列的底部角
- `┌` (TR_CORNER)：多诊断合并后的起始角
- `─` (HOR_BAR)：水平连接线
- `│` (VER_BAR)：堆叠诊断的垂直连接线
- `├` (STACK)：堆叠连接点
- `┴` (MULTI)：合并连接点
- `┘` (BL_CORNER)：末端角

当多个诊断的 anchor 列超过 `max_diagnostic_start` 时，使用 `draw_multi_diagnostics` 将它们合并绘制到屏幕左侧。

主题 scope：`hint.diagnostic.inline` / `info.diagnostic.inline` / `warning.diagnostic.inline` / `error.diagnostic.inline`

### 3.4 底部面板（Fallback）

[render_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/ui/editor.rs#L780-L841) 仅在 inline 和 eol 诊断都禁用时启用：

- 筛选光标所在位置（`range.start <= cursor && range.end >= cursor`）的诊断
- 在编辑区右上角显示右对齐的诊断消息面板（最大 100×15）
- 显示消息和 code

### 3.5 状态栏

[statusline.rs#render_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/ui/statusline.rs#L214-L261)：

- 统计当前文档各 severity 的诊断数量
- 按 `statusline.diagnostics` 配置的顺序显示 `● N` 格式
- [render_workspace_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/ui/statusline.rs#L263)：统计 `Editor.diagnostics`（所有文档）的数量

---

## 4. 刷新时机

### 4.1 诊断数据刷新触发

| 触发场景 | 机制 | 代码位置 |
|----------|------|----------|
| 文档打开 | `DocumentDidOpen` hook → `request_document_diagnostics` | [helix-term/src/handlers/diagnostics.rs#L85-L89](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L85-L89) |
| 文档内容变更 | `DocumentDidChange` hook → Pull 模式自动请求 | [helix-term/src/handlers/diagnostics.rs#L43-L81](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L43-L81) |
| LS 初始化 | `LanguageServerInitialized` hook → 对所有文档请求诊断 | [helix-term/src/handlers/diagnostics.rs#L91-L99](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L91-L99) |
| Push 通知 | LSP 发送 `textDocument/publishDiagnostics` | [application.rs#L831](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/application.rs#L831) |

Pull 模式的 debounce：
- `PullDiagnosticsHandler`：250ms 防抖，合并同一文档的多次变更
- `PullAllDocumentsDiagnosticHandler`：1s 防抖，用于有 `inter_file_dependencies` 的 LS

### 4.2 界面刷新控制（DiagnosticsHandler）

每个 View 持有独立的 [DiagnosticsHandler](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/handlers/diagnostics.rs#L57-L64)，控制光标行诊断的延迟显示。

**核心机制**：generation 计数器 + 异步超时

```
┌──────────────────────────────────────────────────────────┐
│  show_cursorline_diagnostics(doc, view)                  │
│  ├─ 光标行/文档未变 → 比较 generation == active_generation│
│  ├─ 光标行/文档变化  → generation++                      │
│  │   └─ 发送 CursorLineChanged{generation} 事件         │
│  │   └─ 返回 false（本次不显示行内诊断）                 │
│  └─ 返回 true/false 控制行内诊断是否立即显示             │
└──────────────────────────────────────────────────────────┘
         │ (异步 350ms 超时)
         ▼
┌──────────────────────────────────────────────────────────┐
│  DiagnosticTimeout.finish_debounce()                     │
│  └─ active_generation.store(generation)                  │
│  └─ request_redraw()                                    │
│  └─ 下次渲染时 show_cursorline_diagnostics 返回 true     │
└──────────────────────────────────────────────────────────┘
```

**设计意图**：光标快速移动时不频繁计算行内诊断，移动停止 350ms 后才显示。

### 4.3 DiagnosticsDidChange 事件

当诊断数据更新后，[handle_lsp_diagnostics](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/handlers/lsp.rs#L363) 派发 `DiagnosticsDidChange` 事件。

[hook 处理](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L26-L33)：

```rust
register_hook!(move |event: &mut DiagnosticsDidChange<'_>| {
    if event.editor.mode != Mode::Insert {
        for (view, _) in event.editor.tree.views_mut() {
            send_blocking(&view.diagnostics_handler.events, DiagnosticEvent::Refresh)
        }
    }
    Ok(())
});
```

- 非 Insert 模式下，向所有 view 的 diagnostics_handler 发送 `Refresh` 事件
- Insert 模式下跳过（避免输入时频繁刷新）

### 4.4 模式切换

[OnModeSwitch hook](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-term/src/handlers/diagnostics.rs#L34-L39)：

```rust
register_hook!(move |event: &mut OnModeSwitch<'_, '_>| {
    for (view, _) in event.cx.editor.tree.views_mut() {
        view.diagnostics_handler.active = event.new_mode != Mode::Insert;
    }
    Ok(())
});
```

- Insert 模式：`diagnostics_handler.active = false`，光标行诊断不显示
- 退出 Insert：`active = true`，恢复显示

### 4.5 立即显示

[immediately_show_diagnostic](file:///d:/fz/0601/solo-dogfeeding/code/275-helix/helix-view/src/handlers/diagnostics.rs#L97-L106)：

- 直接设置 `active_generation = generation`，跳过 350ms 延迟
- 用于特定操作后需要立即显示诊断的场景（如切换到光标所在的诊断行）

---

## 5. 完整数据流总览

```
LSP Server
    │
    ├── Push: textDocument/publishDiagnostics ──→ application.rs ──┐
    │                                                              │
    └── Pull: textDocument/diagnostic ←── handlers/diagnostics.rs ─┤
         ↑ request                   ↓ response                    │
         │                                                          │
    ┌────┴─────────────────────────────────────────────────────────┘
    │
    ▼
Editor.handle_lsp_diagnostics()
    │
    ├─→ Editor.diagnostics (BTreeMap<Uri, Vec<(lsp::Diagnostic, DiagnosticProvider)>>)
    │     └─ 用于 diagnostics picker、workspace diagnostics 统计
    │
    ├─→ doc_diagnostics_with_filter() → lsp::Diagnostic → helix_core::Diagnostic
    │     └─ offset 编码转换、特性过滤
    │
    ├─→ Document.replace_diagnostics()
    │     └─ Document.diagnostics (Vec<Diagnostic>)
    │           ├─ 按 (range, severity, provider) 排序
    │           └─ 保留 persistent_diagnostic_sources 中未变的诊断
    │
    └─→ dispatch DiagnosticsDidChange
          └─→ 所有 view 的 diagnostics_handler 收到 Refresh 事件
                └─→ 触发 request_redraw

渲染帧:
    │
    ├─ show_cursorline_diagnostics() → 确定是否启用光标行诊断
    │
    ├─ doc_diagnostics_highlights_into() → 行内下划线/删除线高亮
    │     └─ overlay highlights (diagnostic.warning 等 scope)
    │
    ├─ gutter::diagnostic() → gutter 列的 ● 标记
    │
    ├─ InlineDiagnosticAccumulator (LineAnnotation) → 计算虚拟行数
    │     └─ 顺序扫描 doc.diagnostics，收集每行的诊断 stack
    │
    ├─ InlineDiagnostics (Decoration) → 绘制行下诊断文本
    │     ├─ EOL 诊断（行尾显示）
    │     └─ 行下诊断（Box-drawing 连接线 + 消息文本）
    │
    ├─ render_diagnostics() → 底部面板（仅 inline+eol 都禁用时）
    │
    └─ statusline::render_diagnostics() → 状态栏诊断计数
```
