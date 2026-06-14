# Helix 主题加载机制分析

本文通过代码分析 Helix 编辑器的主题加载完整流程，包括主题文件读取、样式合并、默认值兜底和界面应用四个核心环节。

## 一、核心文件与数据结构

### 1.1 关键文件

| 文件 | 作用 |
|------|------|
| `helix-view/src/theme.rs` | 主题加载核心逻辑：Loader、Theme、ThemePalette |
| `helix-loader/src/lib.rs` | TOML 合并工具 `merge_toml_values` |
| `helix-term/src/application.rs` | 应用启动时主题加载入口 `load_configured_theme` |
| `helix-term/src/config.rs` | 配置文件解析，主题配置读取 |
| `helix-term/src/ui/editor.rs` | 编辑器视图渲染，整体背景和文档渲染调度 |
| `helix-term/src/ui/document.rs` | 文档文本渲染，语法高亮应用 |
| `helix-term/src/ui/statusline.rs` | 状态栏渲染 |
| `helix-term/src/ui/menu.rs` | 通用菜单组件渲染 |
| `helix-term/src/ui/completion.rs` | 补全菜单渲染 |
| `helix-term/src/compositor.rs` | 组件组合器，统一渲染调度 |
| `theme.toml` | 内置默认主题（24位真彩色） |
| `base16_theme.toml` | 内置 16 色默认主题 |
| `runtime/themes/` | 内置主题文件目录 |

### 1.2 核心数据结构

#### Theme 结构体 [helix-view/src/theme.rs#L272-L284](helix-view/src/theme.rs#L272-L284)

```rust
pub struct Theme {
    name: String,
    styles: HashMap<String, Style>,      // UI 样式表，键为作用域字符串
    scopes: Vec<String>,                 // 所有高亮作用域名称列表
    highlights: Vec<Style>,              // 语法高亮样式数组（按索引 O(1) 查找）
    scope_index: HashMap<String, Highlight>, // 作用域到高亮索引的反向映射
    rainbow_length: usize,               // 彩虹括号颜色数量
}
```

#### Loader 结构体 [helix-view/src/theme.rs#L89-L92](helix-view/src/theme.rs#L89-L92)

```rust
pub struct Loader {
    theme_dirs: Vec<PathBuf>,  // 主题搜索目录，按优先级从高到低排列
}
```

## 二、主题文件读取流程

### 2.1 运行时目录优先级 [helix-loader/src/lib.rs#L42-L78](helix-loader/src/lib.rs#L42-L78)

主题目录搜索优先级（从高到低）：

1. **CARGO_MANIFEST_DIR 同级 runtime**（开发环境，最高优先级）
2. **用户配置目录** `~/.config/helix/runtime/themes`
3. **HELIX_RUNTIME 环境变量** 指定目录
4. **HELIX_DEFAULT_RUNTIME 构建时变量** 指定目录
5. **可执行文件同级 runtime** 目录（最低优先级）

### 2.2 Loader 初始化 [helix-view/src/theme.rs#L98-L102](helix-view/src/theme.rs#L98-L102)

```rust
pub fn new(dirs: &[PathBuf]) -> Self {
    Self {
        theme_dirs: dirs.iter().map(|p| p.join("themes")).collect(),
    }
}
```

应用启动时的初始化代码 [helix-term/src/application.rs#L100-L102](helix-term/src/application.rs#L100-L102)：

```rust
let mut theme_parent_dirs = vec![helix_loader::config_dir()];
theme_parent_dirs.extend(helix_loader::runtime_dirs().iter().cloned());
let theme_loader = theme::Loader::new(&theme_parent_dirs);
```

### 2.3 主题文件定位 [helix-view/src/theme.rs#L224-L250](helix-view/src/theme.rs#L224-L250)

`path()` 方法按优先级顺序查找主题文件：

```rust
fn path(&self, name: &str, visited_paths: &mut HashSet<PathBuf>) -> Result<PathBuf> {
    let filename = format!("{}.toml", name);
    
    self.theme_dirs.iter().find_map(|dir| {
        let path = dir.join(&filename);
        if !path.exists() {
            None
        } else if visited_paths.contains(&path) {
            cycle_found = true;
            None  // 检测循环继承，跳过已访问路径
        } else {
            visited_paths.insert(path.clone());
            Some(path)
        }
    })
}
```

**关键特性**：
- 支持循环继承检测（通过 `visited_paths` 记录已访问文件）
- 高优先级目录的同名主题会覆盖低优先级目录
- 同一文件名可在不同优先级目录存在，形成继承链

## 三、样式合并机制

### 3.1 继承链递归加载 [helix-view/src/theme.rs#L145-L170](helix-view/src/theme.rs#L145-L170)

主题通过 `inherits` 字段指定父主题，形成继承链：

```rust
fn load_theme(&self, name: &str, visited_paths: &mut HashSet<PathBuf>) -> Result<Value> {
    let path = self.path(name, visited_paths)?;
    let theme_toml = self.load_toml(path)?;
    
    if let Some(parent_theme_name) = theme_toml.get("inherits") {
        let parent_theme_toml = match parent_theme_name.as_str().unwrap() {
            "default" => DEFAULT_THEME_DATA.clone(),
            "base16_default" => BASE16_DEFAULT_THEME_DATA.clone(),
            _ => self.load_theme(parent_theme_name, visited_paths)?,  // 递归
        };
        self.merge_themes(parent_theme_toml, theme_toml)
    } else {
        theme_toml
    }
}
```

**主题继承示例**（来自实际主题文件）：
```toml
# runtime/themes/yo_berry.toml
inherits = "yo"

# runtime/themes/zed_onelight.toml
inherits = "zed_onedark"

# runtime/themes/vesper-transparent.toml
inherits = "vesper"
```

### 3.2 TOML 合并算法 [helix-loader/src/lib.rs#L207-L256](helix-loader/src/lib.rs#L207-L256)

`merge_toml_values(left, right, merge_depth)` 核心逻辑：

```rust
pub fn merge_toml_values(left: Value, right: Value, merge_depth: usize) -> Value {
    match (left, right) {
        // 数组合并：根据 name 字段匹配，递归合并
        (Value::Array(mut left_items), Value::Array(right_items)) => {
            if merge_depth > 0 {
                for rvalue in right_items {
                    let lvalue = get_name(&rvalue)
                        .and_then(|rname| left_items.iter().position(|v| get_name(v) == Some(rname)))
                        .map(|lpos| left_items.remove(lpos));
                    let mvalue = match lvalue {
                        Some(lvalue) => merge_toml_values(lvalue, rvalue, merge_depth - 1),
                        None => rvalue,
                    };
                    left_items.push(mvalue);
                }
                Value::Array(left_items)
            } else {
                Value::Array(right_items)  // 深度不足，直接覆盖
            }
        }
        // 表格合并：按键匹配，递归合并
        (Value::Table(mut left_map), Value::Table(right_map)) => {
            if merge_depth > 0 {
                for (rname, rvalue) in right_map {
                    match left_map.remove(&rname) {
                        Some(lvalue) => {
                            let merged = merge_toml_values(lvalue, rvalue, merge_depth - 1);
                            left_map.insert(rname, merged);
                        }
                        None => {
                            left_map.insert(rname, rvalue);  // 新增键直接添加
                        }
                    }
                }
                Value::Table(left_map)
            } else {
                Value::Table(right_map)  // 深度不足，直接覆盖
            }
        }
        // 其他类型：右值直接覆盖左值
        (_, value) => value,
    }
}
```

### 3.3 主题专用合并策略 [helix-view/src/theme.rs#L188-L211](helix-view/src/theme.rs#L188-L211)

主题合并对 `palette` 调色板有特殊处理：

```rust
fn merge_themes(&self, parent_theme_toml: Value, theme_toml: Value) -> Value {
    // palette 使用 merge_depth=2 进行深合并
    let palette_values = match (parent_palette, palette) {
        (Some(parent_palette), Some(palette)) => {
            merge_toml_values(parent_palette.clone(), palette.clone(), 2)
        }
        // ... 其他情况
    };
    
    // 其他样式使用 merge_depth=1 进行浅合并
    let theme = merge_toml_values(parent_theme_toml, theme_toml, 1);
    
    // 将合并后的 palette 重新注入主题
    merge_toml_values(theme, palette.into(), 1)
}
```

**合并深度差异**：
- `palette`: `merge_depth=2` — 调色板颜色可以逐个覆盖，子主题只需定义修改的颜色
- 其他样式: `merge_depth=1` — 样式整体覆盖，不深入合并内部属性

## 四、默认值兜底机制

### 4.1 内置默认主题 [helix-view/src/theme.rs#L18-L36](helix-view/src/theme.rs#L18-L36)

```rust
// 编译时嵌入默认主题文件
pub static DEFAULT_THEME_DATA: Lazy<Value> = Lazy::new(|| {
    let bytes = include_bytes!("../../theme.toml");
    toml::from_str(str::from_utf8(bytes).unwrap()).unwrap()
});

pub static BASE16_DEFAULT_THEME_DATA: Lazy<Value> = Lazy::new(|| {
    let bytes = include_bytes!("../../base16_theme.toml");
    toml::from_str(str::from_utf8(bytes).unwrap()).unwrap()
});
```

### 4.2 默认调色板 [helix-view/src/theme.rs#L514-L538](helix-view/src/theme.rs#L514-L538)

`ThemePalette::default()` 提供基础 ANSI 颜色映射（17 种基础颜色）：

```rust
fn default() -> Self {
    Self {
        palette: hashmap! {
            "default" => Color::Reset,
            "black" => Color::Black,
            "red" => Color::Red,
            "green" => Color::Green,
            "yellow" => Color::Yellow,
            "blue" => Color::Blue,
            "magenta" => Color::Magenta,
            "cyan" => Color::Cyan,
            "gray" => Color::Gray,
            "light-red" => Color::LightRed,
            "light-green" => Color::LightGreen,
            "light-yellow" => Color::LightYellow,
            "light-blue" => Color::LightBlue,
            "light-magenta" => Color::LightMagenta,
            "light-cyan" => Color::LightCyan,
            "light-gray" => Color::LightGray,
            "white" => Color::White,
        },
    }
}
```

用户自定义颜色会扩展默认调色板：
```rust
pub fn new(palette: HashMap<String, Color>) -> Self {
    let mut default = ThemePalette::default().palette;
    default.extend(palette);  // 用户颜色覆盖默认颜色
    Self { palette: default }
}
```

### 4.3 默认彩虹括号 [helix-view/src/theme.rs#L378-L387](helix-view/src/theme.rs#L378-L387)

```rust
fn default_rainbow() -> Vec<Style> {
    vec![
        Style::default().fg(Color::Red),
        Style::default().fg(Color::Yellow),
        Style::default().fg(Color::Green),
        Style::default().fg(Color::Blue),
        Style::default().fg(Color::Cyan),
        Style::default().fg(Color::Magenta),
    ]
}
```

### 4.4 作用域逐级回退查找 [helix-view/src/theme.rs#L433-L436](helix-view/src/theme.rs#L433-L436)

`try_get()` 方法实现点分隔作用域的逐级回退：

```rust
pub fn try_get(&self, scope: &str) -> Option<Style> {
    std::iter::successors(Some(scope), |s| Some(s.rsplit_once('.')?.0))
        .find_map(|s| self.styles.get(s).copied())
}
```

**查找示例**：
- 查询 `ui.text.focus` → 依次尝试 `ui.text.focus` → `ui.text` → `ui`
- 查询 `keyword.directive` → 依次尝试 `keyword.directive` → `keyword`

`get()` 方法在 `try_get()` 基础上增加最终兜底：
```rust
pub fn get(&self, scope: &str) -> Style {
    self.try_get(scope).unwrap_or_default()  // 返回 Style::default()
}
```

### 4.5 加载失败兜底 [helix-term/src/application.rs#L455-L491](helix-term/src/application.rs#L455-L491)

`load_configured_theme()` 函数的完整兜底逻辑：

```rust
fn load_configured_theme(editor: &mut Editor, config: &Config, 
                         terminal: &mut Terminal, mode: Option<theme::Mode>) {
    let true_color = terminal.backend().supports_true_color() 
        || config.editor.true_color || crate::true_color();
    
    let theme = config.theme.as_ref().and_then(|theme_config| {
        let theme = theme_config.choose(mode);  // 自适应主题选择（明/暗模式）
        editor.theme_loader.load(theme)
            .map_err(|e| { log::warn!("failed to load theme `{}` - {}", theme, e); e })
            .ok()
            .filter(|theme| {
                let colors_ok = true_color || theme.is_16_color();
                if !colors_ok {
                    log::warn!("loaded theme but cannot use it because true color not enabled");
                }
                colors_ok
            })
    }).unwrap_or_else(|| editor.theme_loader.default_theme(true_color));  // 最终兜底
    
    let _ = editor.set_theme(theme);
}
```

**兜底层级（从左到右）**：
1. 用户配置的主题 → 2. 终端真彩色检测 → 3. 16 色主题兼容性检查 → 4. 默认主题

## 五、界面应用过程

主题进入编辑器后，通过多层组件渲染体系作用于界面。以下按组件从整体到局部的顺序详细说明。

### 5.1 应用启动加载与渲染循环

#### 启动时加载 [helix-term/src/application.rs#L128](helix-term/src/application.rs#L128)

```rust
Self::load_configured_theme(&mut editor, &config.load(), &mut terminal, theme_mode);
```

#### 渲染循环 [helix-term/src/application.rs#L255-L286](helix-term/src/application.rs#L255-L286)

```rust
async fn render(&mut self) {
    if self.compositor.full_redraw {
        self.terminal.clear().expect("Cannot clear the terminal");
        self.compositor.full_redraw = false;
    }

    let mut cx = crate::compositor::Context { ... };
    cx.editor.needs_redraw = false;

    let area = self.terminal.autoresize().expect("...");
    let surface = self.terminal.current_buffer_mut();

    self.compositor.render(area, surface, &mut cx);  // 渲染所有组件
    let (pos, kind) = self.compositor.cursor(area, &self.editor);
    
    self.terminal.draw(pos, kind).unwrap();
}
```

#### 组件组合器渲染 [helix-term/src/compositor.rs#L184-L188](helix-term/src/compositor.rs#L184-L188)

```rust
pub fn render(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    for layer in &mut self.layers {
        layer.render(area, surface, cx);  // 逐层渲染，上层覆盖下层
    }
}
```

### 5.2 背景刷新：EditorView 根组件

`EditorView` 是最底层组件，负责绘制编辑器背景和调度所有子视图渲染。

#### 背景色设置 [helix-term/src/ui/editor.rs#L1603-L1606](helix-term/src/ui/editor.rs#L1603-L1606)

```rust
fn render(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    // clear with background color
    surface.set_style(area, cx.editor.theme.get("ui.background"));
    // ... 后续渲染
}
```

**关键作用**：
- 在每帧渲染开始时，用 `ui.background` 样式填充整个编辑器区域
- 这是所有 UI 元素的底色基础

#### 全屏重绘触发 [helix-term/src/compositor.rs#L220-L222](helix-term/src/compositor.rs#L220-L222)

```rust
pub fn need_full_redraw(&mut self) {
    self.full_redraw = true;
}
```

`full_redraw` 标志会导致下一帧先调用 `terminal.clear()` 清除整个终端，再重新绘制所有内容。通常在主题切换、窗口大小变化等场景触发。

### 5.3 主题设置到编辑器：`set_theme`

#### 设置入口 [helix-view/src/editor.rs#L1499-L1525](helix-view/src/editor.rs#L1499-L1525)

```rust
pub fn set_theme(&mut self, theme: Theme) -> anyhow::Result<()> {
    self.set_theme_impl(theme, ThemeAction::Set)
}

fn set_theme_impl(&mut self, theme: Theme, preview: ThemeAction) -> anyhow::Result<()> {
    // 最低要求：必须定义 ui.selection 样式
    if theme.find_highlight_exact("ui.selection").is_none() {
        bail!("Invalid theme: `ui.selection` required");
    }
    
    // 更新语法高亮加载器的作用域列表
    let scopes = theme.scopes();
    (*self.syn_loader).load().set_scopes(scopes.to_vec());
    
    // 更新主题
    self.theme = theme;
    
    // 触发界面刷新和事件通知
    self._refresh();
    self.config_events.0.send(ConfigEvent::ThemeChanged)?;
    
    Ok(())
}
```

**关键步骤**：
1. 验证主题必须包含 `ui.selection`（这是编辑器正常工作的最低要求）
2. 更新语法高亮加载器的作用域列表（影响语法高亮的解析）
3. 设置新主题到 `self.theme`
4. 调用 `_refresh()` 标记需要重绘
5. 发送 `ThemeChanged` 事件通知其他组件

### 5.4 状态栏：多作用域分层应用

状态栏是主题样式最集中的 UI 组件之一，使用了多种作用域。

#### 基础样式选择 [helix-term/src/ui/statusline.rs#L53-L60](helix-term/src/ui/statusline.rs#L53-L60)

```rust
pub fn render(context: &mut RenderContext, viewport: Rect, surface: &mut Surface) {
    let base_style = if context.focused {
        context.editor.theme.get("ui.statusline")
    } else {
        context.editor.theme.get("ui.statusline.inactive")
    };
    surface.set_style(viewport.with_height(1), base_style);
    // ... 渲染左中右三部分内容
}
```

#### 模式指示器样式 [helix-term/src/ui/statusline.rs#L181-L190](helix-term/src/ui/statusline.rs#L181-L190)

```rust
let style = if visible && config.color_modes {
    match context.editor.mode() {
        Mode::Insert => context.editor.theme.get("ui.statusline.insert"),
        Mode::Select => context.editor.theme.get("ui.statusline.select"),
        Mode::Normal => context.editor.theme.get("ui.statusline.normal"),
    }
} else {
    Style::default()
};
```

#### 诊断指示器样式 [helix-term/src/ui/statusline.rs#L234-L260](helix-term/src/ui/statusline.rs#L234-L260)

```rust
for sev in &context.editor.config().statusline.diagnostics {
    match sev {
        Severity::Error if errors > 0 => {
            write(context, Span::styled("●", context.editor.theme.get("error")));
            write(context, format!(" {} ", errors).into());
        }
        Severity::Warning if warnings > 0 => {
            write(context, Span::styled("●", context.editor.theme.get("warning")));
            // ...
        }
        // info、hint 同理
        _ => {}
    }
}
```

#### 分隔符样式 [helix-term/src/ui/statusline.rs#L518-L526](helix-term/src/ui/statusline.rs#L518-L526)

```rust
fn render_separator<'a, F>(context: &mut RenderContext<'a>, write: F) {
    let sep = &context.editor.config().statusline.separator;
    let style = context.editor.theme.get("ui.statusline.separator");
    write(context, Span::styled(sep.to_string(), style));
}
```

#### 样式层叠机制 [helix-term/src/ui/statusline.rs#L123-L126](helix-term/src/ui/statusline.rs#L123-L126)

```rust
fn append<'a>(buffer: &mut Spans<'a>, mut span: Span<'a>, base_style: Style) {
    span.style = base_style.patch(span.style);  // 基础样式 + 元素特定样式
    buffer.0.push(span);
}
```

**状态栏使用的主题作用域汇总**：

| 作用域 | 用途 |
|--------|------|
| `ui.statusline` | 状态栏基础样式（聚焦视图） |
| `ui.statusline.inactive` | 非聚焦视图的状态栏 |
| `ui.statusline.insert` | 插入模式指示器（需 color-modes 开启） |
| `ui.statusline.select` | 选择模式指示器 |
| `ui.statusline.normal` | 普通模式指示器 |
| `ui.statusline.separator` | 状态栏分隔符 |
| `error` | 错误诊断计数圆点 |
| `warning` | 警告诊断计数圆点 |
| `info` | 信息诊断计数圆点 |
| `hint` | 提示诊断计数圆点 |

### 5.5 文档渲染：语法高亮 + 虚拟文本 + 叠加层

文档渲染是主题应用最复杂的部分，涉及语法高亮、虚拟文本、诊断叠加等多层样式叠加。

#### 渲染入口 [helix-term/src/ui/editor.rs#L77-L117](helix-term/src/ui/editor.rs#L77-L117)

```rust
pub fn render_view(&self, editor: &Editor, doc: &Document, view: &View, 
                   viewport: Rect, surface: &mut Surface, is_focused: bool) {
    let inner = view.inner_area(doc);
    let theme = &editor.theme;
    // ...
    
    let text_annotations = view.text_annotations(doc, Some(theme));
    let mut decorations = DecorationManager::default();
    
    // 光标行高亮
    if is_focused && config.cursorline {
        decorations.add_decoration(Self::cursorline(doc, view, theme));
    }
    
    // 语法高亮器
    let syntax_highlighter =
        Self::doc_syntax_highlighter(doc, view_offset.anchor, inner.height, &loader);
    let mut overlays = Vec::new();
    
    // 彩虹括号叠加
    if config.rainbow_brackets {
        if let Some(overlay) = Self::doc_rainbow_highlights(...) {
            overlays.push(overlay);
        }
    }
    
    // 诊断叠加
    Self::doc_diagnostics_highlights_into(doc, theme, &mut overlays);
    
    // ... 更多叠加层
    
    // 最终调用 render_document
    render_document(surface, inner, doc, view_offset, doc_annotations,
                    syntax_highlighter, overlays, theme, decorations);
}
```

#### TextRenderer 初始化 [helix-term/src/ui/document.rs#L201-L273](helix-term/src/ui/document.rs#L201-L273)

```rust
pub fn new(surface: &'a mut Surface, doc: &Document, theme: &Theme,
           offset: Position, viewport: Rect) -> TextRenderer<'a> {
    // ... 空白字符配置
    
    let text_style = theme.get("ui.text");  // 文档文本基础样式
    
    TextRenderer {
        surface,
        text_style,
        whitespace_style: theme.get("ui.virtual.whitespace"),
        indent_guide_style: text_style.patch(
            theme
                .try_get("ui.virtual.indent-guide")
                .unwrap_or_else(|| theme.get("ui.virtual.whitespace")),
        ),
        // ...
    }
}
```

#### 逐字符渲染 [helix-term/src/ui/document.rs#L314-L385](helix-term/src/ui/document.rs#L314-L385)

```rust
pub fn draw_grapheme(&mut self, grapheme: &FormattedGrapheme, grapheme_style: GraphemeStyle,
                    is_virtual: bool, ...) -> usize {
    let mut style = grapheme_style.syntax_style;      // 语法高亮样式
    if is_whitespace {
        style = style.patch(self.whitespace_style);   // 空白字符样式叠加
    }
    style = style.patch(grapheme_style.overlay_style); // 叠加层样式叠加（诊断等）
    
    self.surface.set_grapheme(x, y, grapheme, width, style);
}
```

**样式叠加顺序（从底到顶）**：

1. **基础样式** `ui.text` — 文档文本的默认颜色和背景
2. **语法高亮** `syntax_style` — tree-sitter 解析后的语法高亮（如 keyword、string）
3. **空白字符样式** `ui.virtual.whitespace` — 空白字符显示时的样式
4. **叠加层样式** `overlay_style` — 诊断、选择、搜索高亮等叠加效果

#### 语法高亮样式获取

语法高亮通过 `Highlight` 索引在 `highlights` 数组中 O(1) 查找 [helix-view/src/theme.rs#L409-L415](helix-view/src/theme.rs#L409-L415)：

```rust
#[inline]
pub fn highlight(&self, highlight: Highlight) -> Style {
    if let Some((red, green, blue)) = Self::decode_rgb_highlight(highlight) {
        Style::new().fg(Color::Rgb(red, green, blue))  // RGB 颜色直接编码在索引中
    } else {
        self.highlights[highlight.idx()]  // 索引查找，O(1) 复杂度
    }
}
```

**文档渲染使用的主题作用域汇总**：

| 作用域 | 用途 |
|--------|------|
| `ui.text` | 文档文本基础样式 |
| `ui.virtual.whitespace` | 可见空白字符样式 |
| `ui.virtual.indent-guide` | 缩进参考线（回退到 whitespace） |
| `ui.virtual.ruler` | 列标尺样式 |
| `ui.cursor` | 光标样式 |
| `ui.selection` | 选区背景 |
| `ui.selection.primary` | 主选区背景 |
| `ui.highlight` | 搜索匹配高亮 |
| `ui.cursorline.primary` | 光标行高亮 |
| `diagnostic.error` | 错误诊断下划线 |
| `diagnostic.warning` | 警告诊断下划线 |
| `diagnostic.info` | 信息诊断下划线 |
| `diagnostic.hint` | 提示诊断下划线 |
| `keyword`、`string`、`function` 等 | 语法高亮作用域（上百种） |
| `rainbow.0` ~ `rainbow.N` | 彩虹括号颜色 |

### 5.6 补全菜单：多层样式组合

补全菜单结合了通用 Menu 组件和补全项的自定义样式。

#### 菜单基础样式 [helix-term/src/ui/menu.rs#L335-L342](helix-term/src/ui/menu.rs#L335-L342)

```rust
fn render(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    let theme = &cx.editor.theme;
    let style = theme
        .try_get("ui.menu")
        .unwrap_or_else(|| theme.get("ui.text"));  // 回退到 ui.text
    let selected = theme.get("ui.menu.selected");
    
    surface.clear_with(area, style);  // 填充菜单背景
    // ...
}
```

#### 选中项与滚动条 [helix-term/src/ui/menu.rs#L362-L419](helix-term/src/ui/menu.rs#L362-L419)

```rust
let table = Table::new(rows)
    .style(style)           // 普通项样式
    .highlight_style(selected)  // 选中项样式
    .column_spacing(1)
    .widths(&self.widths);

// ... 滚动条样式
let scroll_style = theme.get("ui.menu.scroll");
if !fits {
    // 绘制滚动条，使用 scroll_style 的 fg 和 bg
    cell.set_fg(scroll_style.fg.unwrap_or(Color::Reset));
}
```

#### 补全项自定义样式 [helix-term/src/ui/completion.rs#L28-L116](helix-term/src/ui/completion.rs#L28-L116)

`CompletionItem` 实现了 `menu::Item` trait，在 `format()` 中定义每一行的样式：

```rust
impl menu::Item for CompletionItem {
    type Data = Style;  // 目录样式作为额外数据传入
    
    fn format(&self, dir_style: &Self::Data) -> menu::Row<'_> {
        let deprecated = /* 判断是否弃用 */;
        let label = /* 补全标签 */;
        let kind = /* 补全类型（如 function、variable） */;
        
        let label = Span::styled(
            label,
            if deprecated {
                Style::default().add_modifier(Modifier::CROSSED_OUT)  // 弃用项加删除线
            } else if kind.0[0].content == "folder" {
                *dir_style  // 文件夹使用目录样式
            } else {
                Style::default()
            },
        );
        
        menu::Row::new([menu::Cell::from(label), menu::Cell::from(kind)])
    }
}
```

#### 补全菜单初始化 [helix-term/src/ui/completion.rs#L132-L136](helix-term/src/ui/completion.rs#L132-L136)

```rust
pub fn new(editor: &Editor, items: Vec<CompletionItem>, trigger_offset: usize) -> Self {
    let dir_style = editor.theme.get("ui.text.directory");
    let menu = Menu::new(items, dir_style, move |editor: &mut Editor, item, event| {
        // ... 回调逻辑
    });
    // ...
}
```

#### 补全文档弹窗 [helix-term/src/ui/completion.rs#L570-L579](helix-term/src/ui/completion.rs#L570-L579)

```rust
// 清除文档区域
let background = cx.editor.theme.get("ui.popup");
surface.clear_with(doc_area, background);
```

**补全菜单使用的主题作用域汇总**：

| 作用域 | 用途 | 回退 |
|--------|------|------|
| `ui.menu` | 菜单背景和普通文字 | `ui.text` |
| `ui.menu.selected` | 选中项样式 | 无 |
| `ui.menu.scroll` | 滚动条样式（fg 滑块，bg 轨道） | 无 |
| `ui.text.directory` | 文件夹类补全项 | 无 |
| `ui.popup` | 补全文档弹窗背景 | 无 |

### 5.7 其他 UI 组件

以下是其他常见组件使用的主题作用域：

| 组件 | 文件 | 主要作用域 |
|------|------|-----------|
| 行号栏 | `helix-term/src/ui/editor.rs` | `ui.linenr`、`ui.linenr.selected` |
| 分隔线 | `helix-term/src/ui/editor.rs` | `ui.background.separator` |
| 命令行/提示框 | `helix-term/src/ui/prompt.rs` | `ui.text`、`ui.text.focus` |
| 弹出窗口 | `helix-term/src/ui/popup.rs` | `ui.popup`、`ui.window` |
| 信息提示 | `helix-term/src/ui/info.rs` | `ui.help` |
| 选取器 | `helix-term/src/ui/picker.rs` | `ui.menu` 系列 |
| 悬浮提示 | `helix-term/src/ui/lsp/hover.rs` | `ui.popup` |

### 5.8 主题切换与重绘触发

#### 主题变更事件流

```
用户执行 :theme 命令
    │
    ▼
Editor::set_theme(theme)
    ├─► 验证 ui.selection 必需样式
    ├─► 更新语法高亮 scopes
    ├─► 设置 self.theme
    ├─► self._refresh()         → 标记 needs_redraw = true
    └─► ConfigEvent::ThemeChanged → 通知其他配置消费者
    │
    ▼
事件循环检测到 needs_redraw
    │
    ▼
Application::render()
    ├─► terminal.clear() (如果 full_redraw)
    ├─► compositor.render() → 逐层调用各组件 render()
    │   └─► EditorView::render()
    │       ├─► 设置 ui.background 背景色
    │       ├─► 渲染 bufferline
    │       ├─► 渲染各 view（文档 + 状态栏）
    │       │   ├─► render_document() → 语法高亮 + 叠加层
    │       │   └─► statusline::render() → 状态栏各元素
    │       └─► 渲染 status message
    ├─► 计算光标位置
    └─► terminal.draw() → 输出到终端
```

## 六、完整流程图

```
用户配置 config.toml
        │
        ▼
Config::load_default() 读取全局和工作区配置
        │
        ▼
Application::new() 创建 theme_loader
        │
        ▼
load_configured_theme()
        ├─► 检测终端真彩色支持
        ├─► 自适应主题选择（明/暗模式）
        └─► loader.load(theme_name)
            │
            ▼
Loader::load_with_warnings()
        ├─► 特殊处理 "default" / "base16_default"
        └─► load_theme(name, visited_paths)
            │
            ▼
Loader::load_theme() 递归加载
        ├─► path(name, visited_paths) 按优先级查找文件
        │   ├─► 检测循环继承
        │   └─► 返回第一个找到的文件路径
        ├─► load_toml(path) 解析 TOML
        ├─► 检查 inherits 字段
        │   └─► 递归 load_theme(parent_name)
        └─► merge_themes(parent, child)
            ├─► palette 深合并 (depth=2)
            └─► 其他样式浅合并 (depth=1)
        │
        ▼
Theme::from_toml() 解析样式
        ├─► 解析 palette（扩展默认调色板 17 色）
        ├─► 解析 rainbow（兜底默认 6 色）
        ├─► 解析所有样式项
        ├─► 构建 styles HashMap
        ├─► 构建 scopes Vec 和 highlights Vec
        └─► 构建 scope_index 反向映射
        │
        ▼
Editor::set_theme()
        ├─► 检查必需的 ui.selection 样式
        ├─► 更新语法高亮 scopes
        ├─► 设置 self.theme
        ├─► 触发界面刷新 (_refresh)
        └─► 发送 ThemeChanged 事件
        │
        ▼
渲染循环 (每帧)
        ├─► EditorView::render()
        │   └─► surface.set_style(area, theme.get("ui.background"))
        │       └─► try_get() 逐级回退 → 兜底 Style::default()
        ├─► 文档渲染 render_document()
        │   ├─► text_style = theme.get("ui.text")
        │   ├─► syntax_style = theme.highlight(highlight_idx)  [O(1) 索引]
        │   ├─► whitespace_style 叠加
        │   └─► overlay_style 叠加（诊断、选区等）
        ├─► 状态栏 statusline::render()
        │   ├─► base_style = ui.statusline / ui.statusline.inactive
        │   ├─► 模式: ui.statusline.insert/select/normal
        │   ├─► 诊断: error/warning/info/hint
        │   └─► 分隔符: ui.statusline.separator
        └─► 补全菜单 Menu::render()
            ├─► style = ui.menu (回退 ui.text)
            ├─► selected = ui.menu.selected
            └─► scroll = ui.menu.scroll
```

## 七、关键设计要点

1. **多目录优先级**：支持用户自定义主题覆盖内置主题，方便定制

2. **循环继承检测**：通过 `visited_paths` 防止 `A inherits B, B inherits A` 导致的死循环

3. **调色板与样式分离合并**：调色板深合并（depth=2），样式浅合并（depth=1），平衡灵活性与性能

4. **作用域逐级回退**：`ui.text.focus` → `ui.text` → `ui`，减少重复定义，支持层级化主题设计

5. **多层兜底机制**：配置失败 → 真彩色检测 → 16 色兼容 → 默认主题 → Style::default()，确保编辑器始终可用

6. **性能优化**：
   - 语法高亮使用索引数组（O(1)）而非 HashMap 查找
   - UI 样式使用 HashMap + 逐级回退缓存
   - RGB 颜色直接编码在 Highlight 索引中，无需额外存储

7. **样式叠加机制**：基础样式 → 语法高亮 → 空白字符 → 叠加层，各层通过 `patch()` 方法合并，实现丰富的视觉效果

8. **完整的主题变更事件**：`ThemeChanged` 事件确保主题切换时所有相关组件都能正确响应
