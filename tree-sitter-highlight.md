# Tree-sitter 语法高亮代码实现梳理

本文档从代码实现角度，系统梳理 Helix 编辑器中 Tree-sitter 语法高亮的三大核心模块：**语言查询**、**增量解析**、**样式映射**，并阐明它们之间的数据流与调用关系。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        用户编辑 / 文本变更                                │
│                                  │                                        │
│                                  ▼                                        │
│                    Transaction / ChangeSet 生成                          │
│                                  │                                        │
│                    ┌─────────────┴─────────────┐                          │
│                    ▼                           ▼                          │
│         ┌──────────────────┐       ┌─────────────────────┐               │
│         │   增量解析模块    │       │   其它模块(诊断等)   │               │
│         │  (Syntax.update) │       └─────────────────────┘               │
│         └────────┬─────────┘                                              │
│                  │                                                        │
│                  ▼                                                        │
│         输入编辑 (InputEdit)  → tree-sitter 内部增量解析                  │
│                  │                                                        │
│                  ▼                                                        │
│         更新后的语法树 (Tree)                                             │
│                  │                                                        │
│       ┌──────────┴────────────┐                                           │
│       ▼                       ▼                                           │
│ ┌────────────────┐   ┌──────────────────┐                                 │
│ │  语言查询模块  │   │  UI 渲染时调用    │                                 │
│ │(Query 编译匹配)│   │ Syntax.highlighter│                                 │
│ └────────┬───────┘   └────────┬─────────┘                                 │
│          │                    │                                           │
│          ▼                    ▼                                           │
│   capture_name (@type 等)   HighlightEvent 流                            │
│          │                    │                                           │
│          └────────┬───────────┘                                           │
│                   ▼                                                       │
│         ┌─────────────────────┐                                           │
│         │    样式映射模块     │                                           │
│         │  (Theme.highlight)  │                                           │
│         └──────────┬──────────┘                                           │
│                    ▼                                                      │
│              最终 Style (fg/bg/modifier)                                  │
│                    │                                                      │
│                    ▼                                                      │
│         渲染到终端 Buffer (Surface.set_grapheme)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、语言查询（Language Query）

### 2.1 核心职责

语言查询模块负责：
1. **加载语言配置**：从 `languages.toml` 读取每个语言的元信息
2. **编译查询文件**：读取 `.scm` 查询文件并编译为 tree-sitter `Query` 对象
3. **匹配语法树节点**：在渲染时将 Query 应用到语法树，产生 capture

### 2.2 语言配置加载

#### 配置结构

[LanguageConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax/config.rs#L25-L106) 定义了语言的完整配置：

```rust
pub struct LanguageConfiguration {
    pub language_id: String,           // e.g. "rust", "tsx"
    pub scope: String,                 // e.g. "source.rust"
    pub file_types: Vec<FileType>,     // 文件扩展名或 glob
    pub shebangs: Vec<String>,         // shebang 匹配
    pub grammar: Option<String>,       // tree-sitter 语法名
    pub injection_regex: Option<Regex>,// 注入语言的正则
    // ... 其它 LSP/格式化/缩进配置
}
```

#### Loader 初始化

[Loader::new](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L287-L322) 构建语言注册表：

```rust
pub fn new(config: Configuration) -> Result<Self, LoaderError> {
    let mut languages: Vec<LanguageData> = Vec::new();
    // 遍历 languages.toml 中的每种语言
    for mut config in config.language {
        let language = Language(languages.len() as u32);
        // 建立扩展名 → Language 的映射
        for file_type in &config.file_types {
            match file_type {
                FileType::Extension(ext) => {
                    languages_by_extension.insert(ext.clone(), language);
                }
                FileType::Glob(glob) => { /* glob 匹配 */ }
            }
        }
        languages.push(LanguageData::new(config));
    }
}
```

### 2.3 LanguageData：查询的惰性加载

[LanguageData](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L40-L47) 采用 `OnceCell` 实现查询的**懒加载编译**：

```rust
pub struct LanguageData {
    config: Arc<LanguageConfiguration>,
    syntax: OnceCell<Option<SyntaxConfig>>,        // highlights + injections + locals
    indent_query: OnceCell<Option<IndentQuery>>,   // indents.scm
    textobject_query: OnceCell<Option<TextObjectQuery>>,  // textobjects.scm
    tag_query: OnceCell<Option<TagQuery>>,         // tags.scm
    rainbow_query: OnceCell<Option<RainbowQuery>>, // rainbows.scm
}
```

#### 语法配置编译

[compile_syntax_config](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L67-L91) 是核心编译入口：

```rust
pub fn compile_syntax_config(
    config: &LanguageConfiguration,
    loader: &Loader,
) -> Result<Option<SyntaxConfig>> {
    // 1. 加载 tree-sitter 语法动态库
    let grammar = get_language(parser_name)?;

    // 2. 读取三种查询文件
    let highlight_query = read_query(name, "highlights.scm");   // 语法高亮捕获
    let injection_query = read_query(name, "injections.scm");   // 语言注入（如 JS 中的 HTML）
    let local_query     = read_query(name, "locals.scm");       // 本地变量作用域

    // 3. 调用 tree_house 编译查询
    let config = SyntaxConfig::new(grammar, &highlight_query,
                                   &injection_query, &local_query)?;

    // 4. 将 capture 名称映射到主题 scope 索引
    reconfigure_highlights(&config, &loader.scopes());

    Ok(Some(config))
}
```

### 2.4 查询文件读取与继承

[read_query](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L268-L272) 支持查询文件的**链式继承**（如 TypeScript 继承 JavaScript）：

```rust
pub fn read_query(lang: &str, query_filename: &str) -> String {
    tree_house::read_query(lang, |language| {
        helix_loader::grammar::load_runtime_file(language, query_filename)
            .unwrap_or_default()
    })
}
```

`runtime/queries/` 目录下每个语言文件夹包含：
- `highlights.scm` — 定义语法高亮捕获规则
- `injections.scm` — 定义嵌入式语言的注入规则
- `locals.scm` — 定义变量作用域和引用
- `indents.scm` — 定义缩进规则
- `textobjects.scm` — 定义文本对象选择规则
- `tags.scm` — 定义符号跳转标签
- `rainbows.scm` — 定义彩虹括号规则

#### 示例：Rust 语言的 highlights.scm 片段

[runtime/queries/rust/highlights.scm](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/runtime/queries/rust/highlights.scm#L1-L80)：

```scheme
; 基本标识符捕获
"?" @special
(type_identifier) @type
(identifier) @variable
(field_identifier) @variable.other.member

; 操作符捕获
[ "*" "->" "=>" "=" "==" "!" "!=" "+" "-" "/" ">" "<" ... ] @operator

; 命名空间/模块捕获
(use_declaration argument: (identifier) @namespace)
(mod_item name: (identifier) @namespace)
```

Query 语法说明：
- `(node_type)` — 匹配特定类型的语法节点
- `field_name: (pattern)` — 匹配特定字段名的子节点
- `[...]` — 匹配组内任意一个模式
- `@capture_name` — 将匹配结果标记为指定捕获名

### 2.5 Capture 名称到主题 Scope 的映射

[reconfigure_highlights](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L241-L266) 实现了**最长前缀匹配**的 capture → scope 映射：

```rust
fn reconfigure_highlights(config: &SyntaxConfig, recognized_names: &[String]) {
    config.configure(|capture_name| {
        // 对每个 capture 名，在主题 scope 列表中找最长匹配
        let capture_parts: Vec<_> = capture_name.split('.').collect();
        let mut best_index = None;
        let mut best_match_len = 0;

        for (i, recognized_name) in recognized_names.iter().enumerate() {
            let mut len = 0;
            let mut matches = true;
            // 逐段比较，例如 capture "variable.other.member"
            // 匹配 scope "variable" (len=1)
            // 或 scope "variable.other.member" (len=3，最优)
            for (i, part) in recognized_name.split('.').enumerate() {
                match capture_parts.get(i) {
                    Some(cp) if *cp == part => len += 1,
                    _ => { matches = false; break; }
                }
            }
            if matches && len > best_match_len {
                best_index = Some(i);
                best_match_len = len;
            }
        }
        best_index.map(|idx| Highlight::new(idx as u32))
    });
}
```

**关键设计**：`Highlight` 本质上是一个 `u32` 索引，指向主题 `highlights: Vec<Style>` 数组，避免运行时字符串查找。

---

## 三、增量解析（Incremental Parsing）

### 3.1 核心职责

增量解析模块负责：
1. **首次解析**：文档打开时构建完整语法树
2. **增量更新**：编辑时仅重新解析受影响的子树
3. **编辑转换**：将 Helix 内部的 `ChangeSet` 转换为 tree-sitter 的 `InputEdit`

### 3.2 Syntax：语法树的封装

[Syntax](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L514-L702) 是 tree_house::Syntax 的包装：

```rust
pub struct Syntax {
    inner: tree_house::Syntax,  // 内部封装：多语言层(Layer)管理 + 解析
}

const PARSE_TIMEOUT: Duration = Duration::from_millis(500); // 解析超时保护

impl Syntax {
    /// 首次创建语法树
    pub fn new(source: RopeSlice, language: Language, loader: &Loader)
        -> Result<Self, Error>
    {
        let inner = tree_house::Syntax::new(
            source, language, PARSE_TIMEOUT, loader
        )?;
        Ok(Self { inner })
    }

    /// 增量更新语法树
    pub fn update(
        &mut self,
        old_source: RopeSlice,  // 变更前的文本
        source: RopeSlice,      // 变更后的文本
        changeset: &ChangeSet,  // 变更集合
        loader: &Loader,
    ) -> Result<(), Error> {
        let edits = generate_edits(old_source, changeset);
        if edits.is_empty() {
            Ok(())
        } else {
            // 将编辑描述传给 tree-sitter，由其内部做增量解析
            self.inner.update(source, PARSE_TIMEOUT, &edits, loader)
        }
    }
}
```

### 3.3 ChangeSet → InputEdit 的转换

[generate_edits](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L706-L779) 是增量解析的核心桥接函数：

```rust
fn generate_edits(old_text: RopeSlice, changeset: &ChangeSet) -> Vec<InputEdit> {
    let mut edits = Vec::new();
    let mut old_pos = 0; // 字符级游标
    let mut iter = changeset.changes.iter().peekable();

    while let Some(change) = iter.next() {
        let len = match change {
            Delete(i) | Retain(i) => *i,
            Insert(_) => 0,
        };
        let old_end = old_pos + len;

        match change {
            Retain(_) => { /* 未变更，跳过 */ }

            Delete(_) => {
                edits.push(InputEdit {
                    start_byte: old_text.char_to_byte(old_pos) as u32,
                    old_end_byte: old_text.char_to_byte(old_end) as u32,
                    new_end_byte: old_text.char_to_byte(old_pos) as u32, // 删除后结束=开始
                    start_point: Point::ZERO,  // 简化：tree-sitter 内部会重新计算
                    old_end_point: Point::ZERO,
                    new_end_point: Point::ZERO,
                });
            }

            Insert(s) => {
                let start_byte = old_text.char_to_byte(old_pos) as u32;
                // Insert + Delete = Replace（合并优化）
                if let Some(Delete(len)) = iter.peek() {
                    // ... REPLACE 编辑 ...
                } else {
                    edits.push(InputEdit {
                        start_byte,
                        old_end_byte: start_byte,  // 纯插入：无旧内容
                        new_end_byte: start_byte + s.len() as u32,
                        start_point: Point::ZERO,
                        old_end_point: Point::ZERO,
                        new_end_point: Point::ZERO,
                    });
                }
            }
        }
        old_pos = old_end;
    }
    edits
}
```

**三种编辑类型**：

| ChangeSet 操作 | InputEdit 含义 | start_byte | old_end_byte | new_end_byte |
|---|---|---|---|---|
| `Delete(n)` | 删除 n 个字符 | 旧起始位置 | 旧结束位置 | 同 start_byte |
| `Insert(s)` | 插入字符串 s | 插入位置 | 同 start_byte | start_byte + s.len() |
| `Insert(s)` + `Delete(n)` | 替换 n 字符为 s | 旧起始 | 旧结束 | 新结束 |

### 3.4 增量解析的触发链路

解析更新在 [Document::apply_impl](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/document.rs#L1435-L1611) 中被调用：

```rust
fn apply_impl(&mut self, transaction: &Transaction, view_id: ViewId, ...) -> bool {
    let old_doc = self.text().clone();
    let changes = transaction.changes();

    // Step 1: 变更应用到文本 Rope
    changes.apply(&mut self.text);

    // Step 2: 选区位置映射
    for selection in self.selections.values_mut() {
        *selection = selection.clone().map(changes);
    }

    // Step 3: 增量更新 tree-sitter 语法树
    if let Some(syntax) = &mut self.syntax {
        let loader = self.syn_loader.load();
        if let Err(err) = syntax.update(
            old_doc.slice(..),    // 旧文本（用于计算字节偏移）
            self.text.slice(..),  // 新文本
            transaction.changes(),
            &loader,
        ) {
            // 解析失败：降级，禁用当前文档的语法高亮
            log::error!("TS parser failed, disabling TS: {err}");
            self.syntax = None;
        }
    }

    // Step 4: 诊断/嵌入提示等位置更新...
    // Step 5: 派发 DocumentDidChange 事件
    helix_event::dispatch(DocumentDidChange { ... });
}
```

### 3.5 多语言层（Layer）支持

`Syntax` 支持**语言注入**（如 HTML 中的 CSS/JS，Markdown 中的代码块）：

```rust
// 查找包含某个字节范围的最内层语言层
pub fn layer_for_byte_range(&self, start: u32, end: u32) -> Layer;

// 按范围大小返回所有包含该范围的层（从大到小）
pub fn layers_for_byte_range(&self, start: u32, end: u32)
    -> impl Iterator<Item = Layer>;

// 获取每个层的 tree（每种注入语言有自己独立的语法树）
pub fn tree_for_byte_range(&self, start: u32, end: u32) -> &Tree;
```

---

## 四、样式映射（Style Mapping）

### 4.1 核心职责

样式映射模块负责：
1. **加载主题文件**：解析 TOML 主题，构建 `scope → Style` 映射
2. **索引化优化**：将字符串 scope 转换为数组索引（`Highlight`）
3. **运行时查找**：渲染时通过索引快速获取样式
4. **层级合并**：语法样式 + 叠加样式 + 虚拟文本样式合并

### 4.2 Theme：主题数据结构

[Theme](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L272-L284) 采用**双存储**策略优化性能：

```rust
pub struct Theme {
    name: String,

    // UI 样式：HashMap 适合稀疏、按名查找
    styles: HashMap<String, Style>,

    // tree-sitter 高亮：Vec + 索引 适合稠密、按序号查找
    scopes: Vec<String>,                    // 所有 scope 名（与 highlights 一一对应）
    highlights: Vec<Style>,                 // 索引 = Highlight.0
    scope_index: HashMap<String, Highlight>, // scope → Highlight 反向索引

    rainbow_length: usize,                  // 彩虹括号颜色数
}
```

#### 主题解析流程

[from_keys](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L488-L507) → [build_theme_values](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L311-L376)：

```rust
fn build_theme_values(mut values: Map<String, Value>) -> (...) {
    // 1. 解析调色板 palette（颜色别名，如 "my_red" = "#ff0000"）
    let palette: ThemePalette = values.remove("palette")...;

    // 2. 构建彩虹括号颜色数组
    for (i, style) in rainbow_styles.into_iter().enumerate() {
        let name = format!("rainbow.{i}");
        scopes.push(name);
        highlights.push(style);
    }

    // 3. 遍历主题中的所有 scope 定义
    for (name, style_value) in values {
        // name: "keyword", "variable.other.member", "ui.text", ...
        let mut style = Style::default();
        palette.parse_style(&mut style, style_value);

        // 同时存入 HashMap 和 Vec
        styles.insert(name.clone(), style);
        scopes.push(name);
        highlights.push(style);
    }
}
```

#### 示例：base16_theme.toml

[base16_theme.toml](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/base16_theme.toml#L11-L30)：

```toml
"comment" = { fg = "gray" }
"variable" = "red"
"constant.numeric" = "yellow"
"type" = "yellow"
"string" = "green"
"function" = "blue"
"keyword" = "magenta"
"namespace" = "magenta"
"variable.other.member" = "green"
```

### 4.3 Highlight 的类型设计

`Highlight` 是一个包装了 `NonMaxU32` 的新类型，有两种用途：

| 取值范围 | 用途 | 解码方式 |
|---|---|---|
| `0..RGB_START` | 普通 scope 索引 | `highlights[highlight.idx()]` |
| `RGB_START..u32::MAX` | 内联 RGB 颜色 | 小端字节解码 `[B, G, R, 0xFF]` |

[rgb_highlight](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L402-L406) 支持 LSP 文档颜色等动态颜色：

```rust
pub fn rgb_highlight(r: u8, g: u8, b: u8) -> Highlight {
    Highlight::new(u32::from_le_bytes([b, g, r, u8::MAX]) - 1)
}
```

### 4.4 运行时样式查找

[highlight()](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L409-L415) 是高频调用（逐字符），用数组索引 O(1)：

```rust
#[inline]
pub fn highlight(&self, highlight: Highlight) -> Style {
    if let Some((r, g, b)) = Self::decode_rgb_highlight(highlight) {
        // 动态 RGB 颜色
        Style::new().fg(Color::Rgb(r, g, b))
    } else {
        // 普通 scope：数组索引查找
        self.highlights[highlight.idx()]
    }
}
```

[find_highlight()](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L454-L465) 用于反向查找（名称→索引），逐级回退：

```rust
pub fn find_highlight(&self, mut scope: &str) -> Option<Highlight> {
    loop {
        if let Some(h) = self.find_highlight_exact(scope) {
            return Some(h);
        }
        // "variable.other.member" → "variable.other" → "variable" → None
        if let Some(new_end) = scope.rfind('.') {
            scope = &scope[..new_end];
        } else {
            return None;
        }
    }
}
```

### 4.5 渲染时的样式合并：SyntaxHighlighter

[SyntaxHighlighter](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-term/src/ui/document.rs#L468-L530) 是渲染时的核心游标：

```rust
struct SyntaxHighlighter<'h, 'r, 't> {
    inner: Option<Highlighter<'h>>,  // tree_house 的高亮化迭代器
    text: RopeSlice<'r>,
    pos: usize,                      // 下一个高亮事件的字符位置
    theme: &'t Theme,
    text_style: Style,               // 基础文本样式 (ui.text)
    style: Style,                    // 当前合成的样式（供绘制使用）
}

impl SyntaxHighlighter {
    fn advance(&mut self) {
        let Some(highlighter) = self.inner.as_mut() else { return };

        let (event, highlights) = highlighter.advance();
        // HighlightEvent 两种类型：
        // - Push: 在当前样式上叠加
        // - Refresh: 重置为 text_style 再叠加（用于语言层切换）
        let base = match event {
            HighlightEvent::Refresh => self.text_style,
            HighlightEvent::Push    => self.style,
        };

        // 折叠式合并：多个 Highlight 依次 patch
        self.style = highlights.fold(base, |acc, highlight| {
            acc.patch(self.theme.highlight(highlight))
        });
        self.update_pos(); // 计算下一个事件的字节→字符位置
    }
}
```

**HighlightEvent 流的语义**：

```
文本:    fn  main  (  )  {  let  x  =  1  ;  }
事件流:
  Push(@keyword)        → style = text_style + keyword
  Push(@function)       → style = style + function
  (普通字符 "fn" 使用此样式)
  Pop                   → 退出 function 层
  Pop                   → 退出 keyword 层
  Push(@variable)       → style = text_style + variable
  ("main" 使用此样式)
  ...
```

### 4.6 完整渲染管线

[render_text](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-term/src/ui/document.rs#L63-L173) 是最终的渲染循环：

```rust
pub fn render_text(...) {
    // 初始化三个"游标"：
    // 1. 文本格式化器（处理换行、软换行、嵌入注释等）
    let mut formatter = DocumentFormatter::new_at_prev_checkpoint(...);
    // 2. 语法高亮游标
    let mut syntax_highlighter = SyntaxHighlighter::new(...);
    // 3. 叠加高亮游标（LSP 文档高亮、彩虹括号、选中等）
    let mut overlay_highlighter = OverlayHighlighter::new(overlay_highlights, theme);

    loop {
        let Some(grapheme) = formatter.next() else { break };

        // 同步推进高亮游标到当前字素位置
        while grapheme.char_idx >= syntax_highlighter.pos {
            syntax_highlighter.advance();
        }
        while grapheme.char_idx >= overlay_highlighter.pos {
            overlay_highlighter.advance();
        }

        // 三级样式合并：
        //   语法样式 (syntax_style) → 基础
        //   + 叠加样式 (overlay_style) → patch
        //   + 空白字符特殊样式 → 再次 patch
        //   = 最终样式
        let grapheme_style = GraphemeStyle {
            syntax_style: syntax_highlighter.style,
            overlay_style: overlay_highlighter.style,
        };

        renderer.draw_grapheme(&grapheme, grapheme_style, ...);
    }
}
```

---

## 五、关键数据流汇总

### 5.1 文档打开时：从文件到语法树

```
Document::open(path)
  └─ from_reader() → 解码为 Rope
  └─ detect_language(loader)
  │   └─ loader.language_for_filename(path) → Language
  │   └─ Document::set_language(config, loader)
  │       └─ Syntax::new(text_slice, language, loader)
  │           └─ tree_house::Syntax::new()
  │               ├─ 获取 SyntaxConfig（懒编译 Query）
  │               └─ tree-sitter Parser::parse() → 完整语法树 Tree
  └─ detect_indent_and_line_ending()
```

### 5.2 编辑时：从按键到增量更新

```
用户按键 → Command 生成 Transaction
  └─ Document::apply(transaction, view_id)
      └─ apply_impl()
          ├─ ChangeSet::apply(&mut self.text) → 更新 Rope
          ├─ Selection::map(changes) → 更新选区
          ├─ Syntax::update(old, new, changeset, loader)
          │   └─ generate_edits(old, changeset) → Vec<InputEdit>
          │   └─ tree_house::Syntax::update(source, edits)
          │       └─ Parser.parse_with_old_tree(old_tree, edits)
          │           → tree-sitter 内部做增量解析，复用未变子树
          └─ DocumentDidChange 事件派发 → 触发重绘
```

### 5.3 渲染时：从语法树到像素

```
每一帧重绘:
  View::render()
    └─ render_document(surface, viewport, doc, ...)
        └─ render_text()
            ├─ doc.syntax.highlighter(source, loader, byte_range) → Highlighter
            │   └─ 对每个 Layer（含注入语言）：
            │       └─ QueryCursor::matches(highlight_query, tree.root_node)
            │           → 迭代所有匹配的 capture，生成 HighlightEvent 流
            │
            ├─ SyntaxHighlighter::advance()
            │   ├─ highlighter.advance() → (event, highlights)
            │   └─ theme.highlight(h) → Style（O(1) 数组索引）
            │       → 合成当前 style
            │
            ├─ OverlayHighlighter::advance() → 叠加样式
            │
            └─ draw_grapheme(grapheme, {syntax_style, overlay_style}, ...)
                └─ surface.set_grapheme(x, y, grapheme, width, final_style)
                    → 写入终端缓冲区
```

---

## 六、性能优化要点

| 优化点 | 实现位置 | 技术手段 |
|---|---|---|
| 查询编译延迟 | [LanguageData](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L40) | `OnceCell` 按需懒加载 |
| 高亮索引化 | [reconfigure_highlights](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L241) | capture 名 → u32 索引，消除字符串比较 |
| 样式查找 O(1) | [Theme::highlight](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L409) | `Vec<Style>` 按索引直接访问 |
| 增量解析 | [Syntax::update](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L526) | tree-sitter 原生增量 + InputEdit 精确描述 |
| 解析超时保护 | [PARSE_TIMEOUT](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L518) | 500ms 上限，防止大文件卡死 UI |
| 渲染按范围迭代 | [Syntax::highlighter](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L593) | 只对可视字节范围做 query 匹配 |
| 主题继承合并 | [Loader::load_theme](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs#L145) | 递归合并父主题 TOML，减少重复定义 |

---

## 七、相关代码文件索引

| 模块 | 主要文件 | 关键类型/函数 |
|---|---|---|
| 语言配置与查询 | [helix-core/src/syntax.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs) | `Loader`, `LanguageData`, `compile_syntax_config`, `reconfigure_highlights`, `read_query` |
| 语言配置结构 | [helix-core/src/syntax/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax/config.rs) | `LanguageConfiguration`, `Configuration`, `FileType` |
| 增量解析 | [helix-core/src/syntax.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-core/src/syntax.rs#L514-L779) | `Syntax`, `generate_edits` |
| 主题与样式 | [helix-view/src/theme.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/theme.rs) | `Theme`, `Theme::highlight`, `Theme::find_highlight`, `ThemePalette` |
| 文档与语法更新 | [helix-view/src/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-view/src/document.rs#L1435-L1611) | `Document::apply_impl`, `Document::set_language` |
| 渲染层合成 | [helix-term/src/ui/document.rs](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/helix-term/src/ui/document.rs) | `render_text`, `SyntaxHighlighter`, `OverlayHighlighter`, `TextRenderer` |
| 查询文件示例 | [runtime/queries/rust/highlights.scm](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/runtime/queries/rust/highlights.scm) | 示例 query 语法 |
| 主题文件示例 | [base16_theme.toml](file:///d:/fz/0601/solo-dogfeeding/code/265-helix/base16_theme.toml) | 示例 scope → color 映射 |
