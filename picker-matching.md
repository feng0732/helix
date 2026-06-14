# Picker 模糊匹配流程详解

## 概述

Picker 是 Helix 编辑器中的核心组件，用于实现文件选择、符号搜索、命令面板等功能。其模糊匹配系统基于 `nucleo` 库实现，包含四个核心流程：**候选收集**、**匹配打分**、**预览更新**和**交互选择**。

---

## 1. 候选收集流程

### 核心数据结构

**[Picker<T, D>](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L241-L272)** 是主要组件结构体，包含：
- `matcher: Nucleo<T>` - 基于 nucleo 的匹配引擎
- `columns: Arc<[Column<T, D>]>` - 列定义，支持多列匹配
- `editor_data: Arc<D>` - 编辑器上下文数据
- `version: Arc<AtomicUsize>` - 版本号，用于取消异步任务

**[Column<T, D>](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L189-L197)** 定义了每列的属性：
- `name: Arc<str>` - 列名，用于查询语法 `%column_name`
- `filter: bool` - 是否参与模糊匹配过滤
- `format: ColumnFormatFn<T, D>` - 格式化函数

### 收集方式

#### 1.1 静态收集（同步）

通过 [Picker::new()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L299-L335) 一次性注入所有候选项：

```rust
pub fn new<C, O, F>(
    columns: C,
    primary_column: usize,
    options: O,    // 预先生成的所有候选项
    editor_data: D,
    callback_fn: F,
) -> Self {
    // ...
    for item in options {
        inject_nucleo_item(&injector, &columns, item, &editor_data);
    }
    // ...
}
```

#### 1.2 流式收集（异步）

通过 [Picker::stream()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L275-L297) 支持异步增量注入：

```rust
pub fn stream(
    columns: impl IntoIterator<Item = Column<T, D>>,
    editor_data: D,
) -> (Nucleo<T>, Injector<T, D>) {
    // 创建 Nucleo 匹配器和 Injector
    // 返回给调用者，调用者可以异步推送候选项
}
```

**[Injector<T, D>](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L145-L157)** 是线程安全的注入器：
- 内部包装 `nucleo::Injector<T>`
- 通过 `version` 机制防止已过期的注入
- `Drop` 时自动请求重绘，清除 "running" 指示器

#### 1.3 动态查询收集

通过 [Picker::with_dynamic_query()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L437-L451) 支持根据查询动态生成候选项（如全局搜索）：

- 使用 `DynamicQueryHandler` 进行防抖处理（默认 100ms）
- 查询变化时通过 `version.fetch_add(1)` 取消之前的请求
- 粘贴操作立即执行，不防抖

### 注入逻辑

**[inject_nucleo_item()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L132-L143)** 是核心注入函数：

```rust
fn inject_nucleo_item<T, D>(
    injector: &nucleo::Injector<T>,
    columns: &[Column<T, D>],
    item: T,
    editor_data: &D,
) {
    injector.push(item, |item, dst| {
        for (column, text) in columns.iter().filter(|column| column.filter).zip(dst) {
            *text = column.format_text(item, editor_data).into()
        }
    });
}
```

关键点：
- 只将 `filter = true` 的列文本传递给 nucleo 进行匹配
- 闭包在 nucleo 内部线程执行，格式化文本供匹配使用

---

## 2. 匹配打分流程

### 查询解析

**[PickerQuery](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker/query.rs#L4-L18)** 负责解析用户输入的查询字符串，支持以下语法：

| 语法示例 | 说明 |
|---------|------|
| `hello world` | 主列匹配 "hello world" |
| `hello %field1 world` | 主列匹配 "hello"，field1 列匹配 "world" |
| `hello \%field1` | 转义 `%`，主列匹配 "hello %field1" |
| `%f abc` | 前缀匹配，自动选择最短匹配的列名 |

**[parse()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker/query.rs#L46-L141)** 方法解析流程：
1. 遍历输入字符串的每个字符
2. 遇到 `%` 时切换到字段解析模式
3. 遇到空格时结束字段名解析，开始字段值输入
4. 支持 `\` 转义特殊字符
5. 相同列的多次使用用空格连接

### 查询变更处理

**[handle_prompt_change()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L534-L581)** 处理查询变化：

```rust
fn handle_prompt_change(&mut self, is_paste: bool) {
    let line = self.prompt.line();
    let old_query = self.query.parse(line);
    if self.query == old_query {
        return;  // 查询无实际变化，快速返回
    }
    self.cursor = 0;  // 重置光标到顶部
    
    // 只重新解析发生变化的列
    for (i, column) in self.columns.iter().filter(|column| column.filter).enumerate() {
        let pattern = self.query.get(&column.name)...;
        let old_pattern = old_query.get(&column.name)...;
        if pattern == old_pattern {
            continue;  // 该列未变化，跳过
        }
        let is_append = pattern.starts_with(old_pattern);
        self.matcher.pattern.reparse(
            i, pattern, CaseMatching::Smart, Normalization::Smart, is_append
        );
    }
}
```

**性能优化点**：
- 快速路径：查询无实际变化时直接返回
- 增量解析：只重新解析发生变化的列
- `is_append` 提示：如果只是追加字符，nucleo 可以优化匹配过程

### Nucleo 匹配引擎

Picker 使用 **nucleo** 库（fzf 匹配算法的 Rust 实现）进行核心匹配：

1. **配置**：
   - `CaseMatching::Smart` - 智能大小写匹配
   - `Normalization::Smart` - 智能 Unicode 标准化
   - `Config::DEFAULT.match_paths()` - 路径匹配优化（`/` 作为分隔符）

2. **匹配过程**（在 nucleo 内部线程执行）：
   - 对每个候选项的每个可过滤列执行模糊匹配
   - 计算匹配得分（0-65535），得分越高越相关
   - 按得分降序排列结果
   - 多列匹配时，所有列都必须匹配成功

3. **匹配高亮**：
   在 [render_picker()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L760-L835) 中获取匹配索引并高亮：

```rust
snapshot.pattern().column_pattern(matcher_index).indices(
    item.matcher_columns[matcher_index].slice(..),
    &mut matcher,
    &mut indices,
);
// indices 包含所有匹配字符的位置，用于高亮显示
```

**注意**：nucleo 返回的索引本质上是字素索引，因为它只考虑每个字素的第一个字符。

---

## 3. 预览更新流程

### 预览获取

**[get_preview()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L585-L681)** 获取当前选中项的预览内容：

```rust
fn get_preview<'picker, 'editor>(
    &'picker mut self,
    editor: &'editor Editor,
) -> Option<(Preview<'picker, 'editor>, Option<(usize, usize)>)> {
    let current = self.selection()?;
    let (path_or_id, range) = (self.file_fn.as_ref()?)(editor, current)?;
    
    match path_or_id {
        PathOrId::Path(path) => {
            // 1. 优先使用编辑器中已打开的文档
            if let Some(doc) = editor.document_by_path(path) {
                return Some((Preview::EditorDocument(doc), range));
            }
            // 2. 检查缓存
            if self.preview_cache.contains_key(path) {
                // 返回缓存内容，异步触发语法高亮
                return Some((Preview::Cached(preview), range));
            }
            // 3. 从磁盘读取并缓存
            let preview = std::fs::metadata(&path).and_then(|metadata| {
                if metadata.is_dir() { /* 目录内容 */ }
                else if metadata.is_file() {
                    if metadata.len() > MAX_FILE_SIZE_FOR_PREVIEW {
                        Ok(CachedPreview::LargeFile)
                    } else if is_binary {
                        Ok(CachedPreview::Binary)
                    } else {
                        // 打开文档，检测语言，异步高亮
                        Ok(CachedPreview::Document(Box::new(doc)))
                    }
                }
            });
            self.preview_cache.insert(path.clone(), preview);
        }
        PathOrId::Id(id) => {
            // 直接使用编辑器中的文档
            let doc = editor.documents.get(&id).unwrap();
            Some((Preview::EditorDocument(doc), range))
        }
    }
}
```

### 缓存策略

- `preview_cache: HashMap<Arc<Path>, CachedPreview>` 缓存已读取的预览
- 缓存类型：
  - `CachedPreview::Document(Box<Document>)` - 文档预览
  - `CachedPreview::Directory(Vec<(String, bool)>)` - 目录内容
  - `CachedPreview::Binary` - 二进制文件提示
  - `CachedPreview::LargeFile` - 大文件提示（> 10MB）
  - `CachedPreview::NotFound` - 文件不存在

### 异步语法高亮

**[PreviewHighlightHandler](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker/handlers.rs#L14-L113)** 处理预览的异步语法高亮：

1. **防抖**：150ms 防抖，避免频繁切换时重复高亮
2. **流程**：
   - 检测到无语法的文档时发送高亮请求
   - 防抖结束后在后台线程执行语法解析
   - 解析完成后更新缓存中的文档
   - 同时更新诊断信息

### 预览渲染

**[render_preview()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L882-L1023)** 渲染预览内容：

1. **定位**：根据 `range` 参数定位到指定行范围
2. **高亮**：
   - 语法高亮（`EditorView::doc_syntax_highlighter`）
   - 彩虹括号（如果启用）
   - 诊断信息（错误、警告等）
   - 选中范围高亮（`ui.highlight` 样式）
3. **渲染**：调用 `render_document()` 渲染文档内容

---

## 4. 交互选择流程

### 事件处理

**[handle_event()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L1053-L1170)** 处理用户交互事件：

#### 4.1 导航操作

| 快捷键 | 功能 | 实现 |
|-------|------|------|
| `Up` / `Shift-Tab` / `Ctrl-p` | 向上移动一项 | [move_by(1, Backward)](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L459-L475) |
| `Down` / `Tab` / `Ctrl-n` | 向下移动一项 | [move_by(1, Forward)](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L459-L475) |
| `PageUp` / `Ctrl-u` | 向上翻一页 | [page_up()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L478-L480) |
| `PageDown` / `Ctrl-d` | 向下翻一页 | [page_down()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L483-L485) |
| `Home` | 跳转到第一项 | [to_start()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L488-L490) |
| `End` | 跳转到最后一项 | [to_end()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L493-L499) |

**光标循环逻辑**：
```rust
pub fn move_by(&mut self, amount: u32, direction: Direction) {
    let len = self.matcher.snapshot().matched_item_count();
    match direction {
        Direction::Forward => {
            self.cursor = self.cursor.saturating_add(amount) % len;
        }
        Direction::Backward => {
            self.cursor = self.cursor.saturating_add(len).saturating_sub(amount) % len;
        }
    }
}
```
使用取模运算实现循环滚动。

#### 4.2 选择操作

| 快捷键 | 功能 |
|-------|------|
| `Enter` | 确认选择，关闭 picker，执行回调 |
| `Alt-Enter` | 确认选择，不关闭 picker（可多选场景） |
| `Ctrl-s` | 水平分屏打开 |
| `Ctrl-v` | 垂直分屏打开 |
| `Ctrl-t` | 切换预览面板显示 |
| `Esc` / `Ctrl-c` | 关闭 picker，不执行选择 |

**选择回调**：
```rust
if let Some(option) = self.selection() {
    (self.callback_fn)(ctx, option, action);
}
```
`action` 参数指定打开方式（`Replace` / `HorizontalSplit` / `VerticalSplit`）。

#### 4.3 输入处理

其他按键事件传递给 `prompt_handle_event()` 处理输入：
- 普通字符输入到查询框
- 退格、删除等编辑操作
- 粘贴事件（`Event::Paste`）

### 选择获取

**[selection()](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L501-L506)** 获取当前选中项：

```rust
pub fn selection(&self) -> Option<&T> {
    self.matcher
        .snapshot()
        .get_matched_item(self.cursor)
        .map(|item| item.data)
}
```

### 关闭逻辑

**[close_fn](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs#L1066-L1087)** 处理 picker 关闭：

- 候选项超过 100 万时直接丢弃，避免内存占用
- 否则保存到 `compositor.last_picker`，可快速重新打开
- 通过 `version.fetch_add(1)` 取消所有后台注入任务
- 保存查询历史到寄存器（如果配置了 `history_register`）

---

## 整体架构图

```
用户输入
    │
    ▼
┌─────────────────┐     ┌─────────────────┐
│  Prompt 输入框  │────▶│ PickerQuery 解析 │
└─────────────────┘     └─────────────────┘
    │                          │
    │                          ▼
    │              ┌──────────────────────────┐
    │              │  Nucleo 匹配引擎（多线程）│
    │              └──────────────────────────┘
    │                          │
    │                          ▼
    │              ┌──────────────────────────┐
    │              │   匹配结果（排序、打分） │
    │              └──────────────────────────┘
    │                          │
    ▼                          ▼
┌─────────────────┐     ┌─────────────────┐
│ render_picker   │     │ get_preview     │
│  - 列表渲染     │────▶│  - 缓存检查     │
│  - 匹配高亮     │     │  - 异步高亮     │
│  - 光标管理     │     └─────────────────┘
└─────────────────┘             │
    │                          ▼
    │              ┌─────────────────┐
    │              │ render_preview  │
    │              │  - 文档渲染     │
    │              │  - 范围高亮     │
    │              └─────────────────┘
    │
    ▼
用户交互（导航/选择）
    │
    ▼
callback_fn 回调执行
```

---

## 关键性能优化点

1. **增量匹配**：只重新解析变化的查询列
2. **追加优化**：`is_append` 提示 nucleo 优化匹配过程
3. **后台匹配**：nucleo 使用多线程进行匹配，不阻塞 UI
4. **预览缓存**：避免重复读取文件和解析语法
5. **防抖处理**：动态查询和语法高亮都有防抖，避免频繁操作
6. **快速路径**：查询无实际变化时直接返回

---

## 核心文件索引

| 文件 | 作用 |
|------|------|
| [picker.rs](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker.rs) | Picker 主组件，包含渲染、事件处理、预览逻辑 |
| [picker/query.rs](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker/query.rs) | 查询语法解析，支持 `%field` 多列查询 |
| [picker/handlers.rs](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-term/src/ui/picker/handlers.rs) | 异步处理器：预览高亮、动态查询 |
| [helix-core/src/fuzzy.rs](file:///d:/fz/0601/solo-dogfeeding/code/268-helix/helix-core/src/fuzzy.rs) | 模糊匹配工具函数，MATCHER 全局实例 |
