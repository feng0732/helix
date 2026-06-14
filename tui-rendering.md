# TUI 渲染管线代码协作指南

本文档梳理 Helix 编辑器的 TUI（终端用户界面）渲染管线：从应用状态如何一步步转化为缓冲区绘制、组件布局，最终输出到终端屏幕。

## 目录
- [总体架构概览](#总体架构概览)
- [核心模块划分](#核心模块划分)
- [渲染主循环](#渲染主循环)
- [合成器（Compositor）层](#合成器compositor层)
- [组件渲染层](#组件渲染层)
- [布局系统](#布局系统)
- [缓冲区（Buffer）层](#缓冲区buffer层)
- [终端（Terminal）双缓冲层](#终端terminal双缓冲层)
- [后端（Backend）转义序列层](#后端backend转义序列层)
- [数据流转总结图](#数据流转总结图)

---

## 总体架构概览

Helix 的 TUI 渲染采用 **七层分层架构**，从上到下依次为：

```
┌─────────────────────────────────────────┐
│  1. Application 应用层                 │  驱动渲染循环、管理全局状态
├─────────────────────────────────────────┤
│  2. Compositor 合成器层                │  组件栈管理、事件分发与层叠渲染
├─────────────────────────────────────────┤
│  3. Component 组件渲染层              │  EditorView / Picker / Popup 等 UI 组件
├─────────────────────────────────────────┤
│  4. Layout 布局系统                   │  基于 Cassowary 约束求解的区域划分
├─────────────────────────────────────────┤
│  5. Buffer 缓冲区层                   │  Cell 网格，存储字素+样式
├─────────────────────────────────────────┤
│  6. Terminal 终端双缓冲层             │  前后缓冲 diff，生成增量更新
├─────────────────────────────────────────┤
│  7. Backend 后端转义序列层            │  Crossterm/Termina 输出 ANSI 序列
└─────────────────────────────────────────┘
```

**关键数据类型流动方向**：
```
Editor 状态 → Component::render() → Buffer(Cell 网格) → diff 更新 → Backend.draw() → 终端
```

---

## 核心模块划分

| 模块路径 | 职责 | 关键文件 |
|---------|------|---------|
| `helix-term/` | 终端应用层，包含应用主循环、合成器、UI 组件 | `application.rs`, `compositor.rs`, `ui/*.rs` |
| `helix-tui/` | TUI 基础库：缓冲区、布局、终端、widgets、后端 | `buffer.rs`, `layout.rs`, `terminal.rs`, `backend/*.rs` |
| `helix-view/` | 视图与图形类型定义：Rect、Style、Color、Modifier | `graphics.rs`, `view.rs`, `editor.rs` |

---

## 渲染主循环

一切渲染的入口：[Application::render()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/application.rs#L255-L286)

### 完整步骤（逐行解析）

**步骤 1：检查全量重绘标志**
```rust
if self.compositor.full_redraw {
    self.terminal.clear().expect("Cannot clear the terminal");
    self.compositor.full_redraw = false;
}
```
- `full_redraw` 标志由 `Compositor::need_full_redraw()` 设置（如切换主题、调整窗口大小后）
- `terminal.clear()` 会同时清屏并**重置后缓冲**，确保下次 diff 时对比空白，强制所有单元格更新

**步骤 2：构建渲染上下文 Context**
```rust
let mut cx = crate::compositor::Context {
    editor: &mut self.editor,   // 全局编辑器状态（文档、视图树、主题...）
    jobs: &mut self.jobs,       // 异步任务调度
    scroll: None,               // 滚动信息（子组件可填充）
};
```
- `Context` 是组件访问外部世界的唯一通道
- 贯穿整个 `render()` 调用链，所有组件共享可变引用

**步骤 3：同步终端尺寸（自动 resize）**
```rust
let area = self.terminal.autoresize().expect("Unable to determine terminal size");
```
- 调用链：[Terminal::autoresize()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/terminal.rs#L169-L175) → `Backend::size()` → `terminal.resize()`
- 如果检测到尺寸变化，会 resize 两个缓冲区并触发清屏

**步骤 4：获取当前缓冲区（前缓冲）**
```rust
let surface = self.terminal.current_buffer_mut();
```
- Terminal 内部有双缓冲数组 `buffers: [Buffer; 2]`
- `current` 字段索引当前正在写入的缓冲（下一帧将显示的内容）

**步骤 5：合成器驱动所有组件渲染**
```rust
self.compositor.render(area, surface, &mut cx);
```
- **这是最核心的协作点**：Compositor 遍历所有层，将状态写入 Buffer
- 详见下节

**步骤 6：收集光标位置**
```rust
let (pos, kind) = self.compositor.cursor(area, &self.editor);
self.editor.cursor_cache.reset();
let pos = pos.map(|pos| (pos.col as u16, pos.row as u16));
```
- 反向遍历组件栈，第一个返回 `Some` 的组件决定光标
- 支持多种光标样式：`Block / Bar / Underline / Hidden`

**步骤 7：刷新到终端**
```rust
self.terminal.draw(pos, kind).unwrap();
```
- 内部执行 diff → 后端绘制 → 交换缓冲 → flush
- 详见双缓冲层章节

### 主循环中的渲染触发时机

在 [event_loop_until_idle()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/application.rs#L301-L365) 中，以下情况会调用 `self.render().await`：
1. 键盘/鼠标终端事件处理返回 `should_redraw = true`
2. 异步 Jobs 回调完成
3. Editor 内部事件（Redraw、DocumentSaved、ConfigEvent...）
4. IdleTimeout（LSP 补全、诊断等延时更新）
5. 每次进入循环前的初始渲染

---

## 合成器（Compositor）层

[Compositor](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/compositor.rs#L78-L227) 是组件的容器与调度器，借鉴了 Cursive 的设计。

### 数据结构
```rust
pub struct Compositor {
    layers: Vec<Box<dyn Component>>,  // 组件栈：索引小 = 底层，大 = 顶层
    area: Rect,                       // 整个终端可视区域
    last_picker: Option<Box<dyn Component>>,
    full_redraw: bool,                // 是否需要全量重绘
}
```

**组件栈顺序约定**：
- `layers[0]`：最底层（通常是 `EditorView`）
- `layers.last()`：最顶层（如弹出的 Picker、Prompt、Popup）
- 事件从顶层向下传递（冒泡反向）
- 渲染从底层向上绘制（后画的覆盖先画的）

### Component trait：所有 UI 元素的统一接口

[Component](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/compositor.rs#L40-L76) trait 是渲染管线的核心抽象：

```rust
pub trait Component: Any + AnyComponent {
    // 事件处理：true 表示消费了事件
    fn handle_event(&mut self, _event: &Event, _ctx: &mut Context) -> EventResult;

    // 性能优化：返回 false 则跳过重绘
    fn should_update(&self) -> bool { true }

    // ===== 核心渲染方法 =====
    // area:   分配给该组件的矩形区域（全局坐标）
    // frame:  Terminal Buffer 的可变引用（写入目标）
    // ctx:    全局上下文（可访问 Editor）
    fn render(&mut self, area: Rect, frame: &mut Surface, ctx: &mut Context);

    // 光标查询（子组件覆盖，返回自己内部光标位置）
    fn cursor(&self, _area: Rect, _ctx: &Editor) -> (Option<Position>, CursorKind);

    // 向父组件建议尺寸（用于 Popup 等自适应组件）
    fn required_size(&mut self, _viewport: (u16, u16)) -> Option<(u16, u16)>;
}
```

### Compositor::render：层叠渲染流程

[Compositor::render()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/compositor.rs#L184-L188)

```rust
pub fn render(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    for layer in &mut self.layers {     // 从底向上遍历
        layer.render(area, surface, cx); // 每层都拿到完整 area，自己决定画在哪
    }
}
```

**关键点**：
- 每层接收到的 `area` 是**整个终端区域**，组件自己做内部子布局
- 所有层共享**同一个 Buffer**，后绘制的单元格直接覆盖之前的
- 顶层组件（如 Popup）通过自己的位置计算，将内容写入 buffer 对应坐标，实现"浮层"效果

### Compositor::cursor：光标冒泡

[Compositor::cursor()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/compositor.rs#L190-L197)

```rust
pub fn cursor(&self, area: Rect, editor: &Editor) -> (Option<Position>, CursorKind) {
    for layer in self.layers.iter().rev() {  // 从顶层开始
        if let (Some(pos), kind) = layer.cursor(area, editor) {
            return (Some(pos), kind);         // 第一个返回光标的组件获胜
        }
    }
    (None, CursorKind::Hidden)
}
```

---

## 组件渲染层

以最复杂的 [EditorView](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-term/src/ui/editor.rs) 为例，展示组件内部如何协作。

### EditorView::render 内部工作流

```
EditorView::render()
  ├─ 遍历 view tree（编辑器的窗口分割树）
  │    └─ 对每个 View：
  │         ├─ 计算 inner_area（扣除状态栏、行号栏）
  │         ├─ 构建语法高亮器
  │         ├─ 叠加多种高亮 overlay（诊断/选择/括号/搜索...）
  │         ├─ 调用 render_document() 将文档文本写入 Buffer
  │         └─ 渲染行号栏、标尺、光标装饰
  ├─ 绘制全局状态栏
  └─ 处理浮层组件：补全菜单、弹出框等
```

### 组件的布局策略：自顶向下分配

组件通常遵循以下模式来处理子区域：

```rust
// 1. 先算自己的子布局（使用 Layout 系统或手动 Rect 计算）
let chunks = Layout::default()
    .direction(Direction::Vertical)
    .constraints([
        Constraint::Length(3),   // 标题栏占 3 行
        Constraint::Min(0),      // 内容区自适应
        Constraint::Length(1),   // 状态栏占 1 行
    ])
    .split(area);

// 2. 对每个子区域递归绘制内容
self.render_title(chunks[0], surface, cx);
self.render_content(chunks[1], surface, cx);
self.render_status(chunks[2], surface, cx);
```

### Widget trait：无状态可消耗部件

[Widget](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/widgets/mod.rs#L46-L49) 是更轻量的渲染单元，用于 Block、Table 等"即绘即弃"的元素：

```rust
pub trait Widget {
    fn render(self, area: Rect, buf: &mut Buffer);  // 注意 self 是 move，不持有状态
}
```

**典型用法**（在 Component::render 内部）：
```rust
// 一次性使用，构建 + 渲染
Block::bordered()
    .title("My Popup")
    .border_style(Style::default().fg(Color::Blue))
    .render(popup_area, surface);  // self 被 move，构建器消费
```

**Component vs Widget 对比**：

| 维度 | Component | Widget |
|------|-----------|--------|
| 生命周期 | 长期存活，有内部状态 | 临时构建，渲染即丢弃 |
| 事件处理 | 有 `handle_event()` | 无 |
| 光标查询 | 有 `cursor()` | 无 |
| 自身状态 | `&mut self` 可变 | `self` 消耗性 |
| 示例 | EditorView, Picker, Prompt | Block, Table, Paragraph |
| 所在位置 | helix-term/ui/ | helix-tui/widgets/ |

---

## 布局系统

[Layout](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/layout.rs) 基于 **Cassowary 线性约束求解器**实现区域划分。

### 核心类型

```rust
// 约束：决定每个子区域占多少空间
pub enum Constraint {
    Percentage(u16),     // 按父区域百分比：50 = 50%
    Ratio(u32, u32),     // 按比例：1:3 = 1/3, 2:3 = 2/3
    Length(u16),         // 固定长度（行/列数）
    Max(u16),            // 最大长度
    Min(u16),            // 最小长度
}

// 布局描述
pub struct Layout {
    direction: Direction,        // Horizontal(水平) / Vertical(垂直)
    margin: Margin,              // 四周边距
    constraints: Vec<Constraint>, // 子区域约束列表
}
```

### Layout::split() 执行流程

[Layout::split()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/layout.rs#L179-L187)

```
1. 线程本地缓存查询
   LAYOUT_CACHE (HashMap<(Rect, Layout), Vec<Rect>>)
        │ 命中 → 直接返回缓存结果（性能优化）
        └─ 未命中 → 调用 split(area, layout) 求解

2. Cassowary 约束求解（split 内部）
   ├─ 为每个子元素创建 Element（含 x, y, width, height 四个变量）
   ├─ 构建约束方程组：
   │   ├─ 必需约束(REQUIRED)：不重叠、不越界、首尾对齐
   │   ├─ 约束：相邻元素首尾相接 (elem[i].right == elem[i+1].left)
   │   └─ 弱约束(WEAK)：按 Constraint 类型生成目标尺寸
   ├─ solver.add_constraints() → 单纯形法求解
   └─ 遍历求解结果，填充 Vec<Rect>

3. 浮点精度修正
   最后一个元素的 width/height 强制扩展到边界（防止 1px 缝隙）
```

### 约束强度与求解策略

```rust
// 必需约束（REQUIRED，不可违反）：
elt.width >= 0
elt.height >= 0
elt.left >= dest_area.left
elt.top >= dest_area.top
elt.right <= dest_area.right
elt.bottom <= dest_area.bottom
first.left == area.left    // 首个元素贴左/上边
last.right == area.right   // 末尾元素贴右/下边
elem[i].right == elem[i+1].left  // 相邻元素无间隙

// 弱约束（WEAK，尽量满足）：
// Constraint::Length(10) → elem.width = 10
// Constraint::Percentage(30) → elem.width = 父宽 * 0.3
// Constraint::Min(5) → elem.width >= 5
// Constraint::Max(20) → elem.width <= 20
```

### 常见布局模式

**模式 1：编辑器垂直三栏布局**
```rust
Layout::default()
    .direction(Direction::Vertical)
    .constraints([
        Constraint::Length(1),   // 顶部标签页栏
        Constraint::Min(0),      // 主内容区（弹性）
        Constraint::Length(1),   // 状态栏
        Constraint::Length(1),   // 命令行 / 错误提示
    ])
    .split(area)
```

**模式 2：水平双栏 + 边距**
```rust
Layout::default()
    .direction(Direction::Horizontal)
    .horizontal_margin(2)
    .constraints([
        Constraint::Ratio(1, 3),  // 左栏 1/3
        Constraint::Ratio(2, 3),  // 右栏 2/3
    ])
    .split(area)
```

---

## 缓冲区（Buffer）层

[Buffer](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/buffer.rs) 是渲染的**中间表示**：一个由 `Cell` 组成的二维网格，所有组件都往这里写。

### 核心数据结构

```rust
// 单个终端单元格 = 一个字素 + 完整样式
pub struct Cell {
    pub symbol: Symbol,              // 字素（ArrayString<28>，支持超长 Unicode/emoji）
    width: u8,                       // 缓存显示宽度（1=窄, 2=宽 CJK/Emoji）
    pub fg: Color,                   // 前景色
    pub bg: Color,                   // 背景色
    pub underline_color: Color,      // 下划线颜色
    pub underline_style: UnderlineStyle, // 下划线样式（卷曲/虚线/实线...）
    pub modifier: Modifier,          // 修饰位：粗体/斜体/反色/闪烁...
}

// 缓冲区 = Rect 区域 + 平铺的 Cell 向量
pub struct Buffer {
    pub area: Rect,                  // 缓冲区覆盖的矩形区域（全局坐标）
    pub content: Vec<Cell>,          // 行优先存储：len = width * height
}
```

**Cell 存储的内存布局**：
```
content[i] 的坐标:
  行号 = (i / width) + area.y
  列号 = (i % width) + area.x

坐标 → 索引公式（index_of 方法）：
  index = (y - area.y) * width + (x - area.x)
```

### 核心写入 API（组件层使用的主要方法）

| 方法 | 用途 | 适用场景 |
|------|------|---------|
| `buf[(x, y)].set_symbol("x")` | 单格写入字素 | 边框、特殊符号 |
| `buf.set_string(x, y, &str, style)` | 写字符串（Unicode 分段） | 普通文本 |
| `buf.set_stringn(x, y, &str, n, style)` | 写最多 n 列的字符串 | 截断文本 |
| `buf.set_grapheme(x, y, g, w, style)` | 已知宽度的单字素写入（**热路径**） | 语法高亮逐字渲染 |
| `buf.set_style(area, style)` | 给区域批量设置样式 | 选中区、反色块 |
| `buf.set_span / set_spans` | 写带样式的 Span/Spans | 富文本 |
| `buf.set_string_truncated` | 带省略号的截断写入 | 文件名、状态栏 |

### set_stringn 的内部流程（逐字素渲染）

```
输入: (x, y, string, max_width, style)

1. 边界检查：如果 (x,y) 越界，直接返回
2. 按 Unicode 字素分段（graphemes(true)）
3. 对每个字素 s：
   a. 计算显示宽度 s.width()
   b. 如果剩余空间不够，中止（不会渲染半个宽字符）
   c. buf[index].set_symbol(s) + set_style(style)
   d. 如果是宽字符（width=2），后续 1 个 Cell 重置为空格（防重叠）
   e. index += width, x_offset += width
4. 返回写入后的末尾坐标
```

### Buffer::diff()：增量更新的核心算法

[Buffer::diff()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/buffer.rs#L730-L755)

对比前后两帧的 buffer，只输出变化了的单元格，减少终端 I/O。

```
输入: &self（前缓冲）, &other（当前缓冲）
输出: Vec<(x, y, &Cell)>  // 需要重绘的单元格列表

算法：
  invalidated = 0  // 前一个宽字符导致后续需强制刷新的格子数
  to_skip = 0      // 当前宽字符占据的后续空格子数

  对 i in 0..len：
    current = next[i], previous = self[i]

    // 判断是否需要刷新
    needs_update = (current != previous || invalidated > 0) && to_skip == 0

    if needs_update: push (x, y, &current) 到输出

    // 更新宽字符状态
    current_w = current.width()
    to_skip = current_w - 1                    // 当前字符后续跳过数
    affected_w = max(current_w, previous.w)
    invalidated = max(affected_w, invalidated) - 1
```

**宽字符处理示例**：
```
前帧: `コ`   (index 0 是宽字符，index 1 是隐藏空格)
现帧: `aa`   (index 0 和 1 都变成 'a')

diff 结果：即使 index 1 的 'a' 和前帧的空格不同，也需要一起输出
invalidated 机制确保 index 1 也会被标记为需要刷新
```

---

## 终端（Terminal）双缓冲层

[Terminal](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/terminal.rs) 是 Buffer 与 Backend 之间的桥梁。

### 双缓冲设计

```rust
pub struct Terminal<B: Backend> {
    backend: B,
    buffers: [Buffer; 2],   // [0] 和 [1] 两个缓冲交替使用
    current: usize,         // 当前写入的缓冲索引（0 或 1）
    cursor_kind: CursorKind,
    viewport: Viewport,
}
```

**双缓冲工作原理**：
```
帧 N:
  current = 0
  → 组件写入 buffers[0]
  → diff(buffers[1], buffers[0]) → 对比前帧状态
  → flush 输出差异
  → reset(buffers[1])           ← 清空下一帧要写入的缓冲
  → current = 1

帧 N+1:
  current = 1
  → 组件写入 buffers[1]
  → diff(buffers[0], buffers[1])
  → ... 循环往复
```

### Terminal::draw()：完整提交流程

[Terminal::draw()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/terminal.rs#L179-L214)

```rust
pub fn draw(&mut self, cursor_position: Option<(u16, u16)>, cursor_kind: CursorKind) -> io::Result<()> {
    // 1. Diff + 输出到后端内部缓冲
    self.flush()?;

    // 2. 设置光标位置
    if let Some((x, y)) = cursor_position {
        self.set_cursor(x, y)?;
    }

    // 3. 显示/隐藏光标
    match cursor_kind {
        CursorKind::Hidden => self.hide_cursor()?,
        kind => self.show_cursor(kind)?,
    }

    // 4. 重置后缓冲 + 交换
    self.buffers[1 - self.current].reset();
    self.current = 1 - self.current;

    // 5. 真正 flush 到 stdout/stderr
    self.backend.flush()?;
    Ok(())
}
```

### flush()：diff 到后端

[Terminal::flush()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/terminal.rs#L151-L156)

```rust
pub fn flush(&mut self) -> io::Result<()> {
    let previous_buffer = &self.buffers[1 - self.current];
    let current_buffer = &self.buffers[self.current];
    let updates = previous_buffer.diff(current_buffer);  // 增量计算
    self.backend.draw(updates.into_iter())               // 交给后端编码为转义序列
}
```

---

## 后端（Backend）转义序列层

[Backend trait](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/backend/mod.rs#L26-L52) 定义了终端抽象接口，有三种实现：
- Windows: `CrosstermBackend<Stdout>`（基于 crossterm 库）
- Linux/macOS: `TerminaBackend`（基于 termina 库）
- 测试: `TestBackend`（内存缓冲，用于集成测试）

### Backend::draw()：Cell → ANSI 转义序列

以 [CrosstermBackend::draw()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/backend/crossterm.rs#L228-L291) 为例，核心是**状态机压缩**：只在样式/位置变化时发送转义码。

```
输入: Iterator<Item = (u16, u16, &Cell)>  // diff 产生的增量单元格

状态变量（避免重复发送相同序列）:
  fg, bg, underline_color: Color     // 上次的颜色
  underline_style: UnderlineStyle    // 上次下划线样式
  modifier: Modifier                 // 上次修饰位
  last_pos: Option<(u16, u16)>       // 上次光标位置

对每个 (x, y, cell):
  // 1. 光标移动优化
  如果 last_pos != (x-1, y)：        // 非连续写入，需要显式移动
    queue MoveTo(x, y)
  last_pos = Some((x, y))

  // 2. 修饰符变化（粗体/斜体等）
  如果 cell.modifier != modifier：
    计算 ModifierDiff { from, to }
    分别发送移除/添加的 Attribute 转义码
    modifier = cell.modifier

  // 3. 前景/背景色变化
  如果 cell.fg != fg || cell.bg != bg：
    queue SetColors(new_fg, new_bg)
    fg = cell.fg; bg = cell.bg

  // 4. 下划线颜色（需终端能力检测）
  如果 capabilities.has_extended_underlines && 颜色变化：
    queue SetUnderlineColor(new_color)  // 自定义 ANSI: \x1b[58:2::R:G:Bm

  // 5. 下划线样式
  如果 underline_style 变化：
    queue SetAttribute(对应样式)

  // 6. 输出实际字符
  queue Print(&cell.symbol)

最后（重置样式）:
  queue SetUnderlineColor(Reset)
  queue SetForegroundColor(Reset)
  queue SetBackgroundColor(Reset)
  queue SetAttribute(Reset)
```

### ModifierDiff：增量修饰符切换

[ModifierDiff::queue()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/backend/crossterm.rs#L345-L404)

```
removed = from - to    // 之前有、现在要去掉的修饰
added = to - from      // 之前没有、现在要加上的修饰

对 removed 中每个标志：
  BOLD     → SetAttribute(NormalIntensity)  （注意：NormalIntensity 同时清除 DIM）
  ITALIC   → SetAttribute(NoItalic)
  REVERSED → SetAttribute(NoReverse)
  ...

对 added 中每个标志：
  BOLD     → SetAttribute(Bold)
  ITALIC   → SetAttribute(Italic)
  REVERSED → SetAttribute(Reverse)
  ...
```

### 终端能力检测（Capabilities）

[Capabilities::from_env_or_default()](file:///d:/fz/0601/solo-dogfeeding/code/280-helix/helix-tui/src/backend/crossterm.rs#L76-L94)

```
从 terminfo 数据库 + 环境变量检测：
  has_extended_underlines:
    ✅ force_enable_extended_underlines (config.undercurl)
    ✅ terminfo 有 Smulx 扩展能力
    ✅ terminfo 有 Su 能力（kitty 风格）
    ✅ VTE_VERSION >= 5102（GNOME 终端等）
    ✅ TERM_PROGRAM == "WezTerm"
    ❌ 否则退化为普通下划线

  reset_cursor_command:
    Se 扩展能力 + CursorNormal terminfo + 兜底 \x1B[0 q
```

---

## 数据流转总结图

把所有层串起来，**一帧渲染的完整调用链**：

```
Application::render() [helix-term/application.rs:255]
 │
 ├─ terminal.autoresize()                 ← 查询真实终端尺寸
 │    └─ Backend::size()                  ← ioctl(TIOCGWINSZ) 或同等
 │
 ├─ surface = terminal.current_buffer_mut()  ← 获取 &mut Buffer
 │
 ├─ compositor.render(area, surface, cx)  ← 组件栈渲染
 │    │
 │    ├─ layers[0] = EditorView::render()
 │    │    ├─ Layout::split() 划分状态栏/内容区...
 │    │    ├─ 对每个 View:
 │    │    │    ├─ 语法高亮 → Vec<(Range, Style)>
 │    │    │    ├─ 诊断高亮叠加
 │    │    │    ├─ 选择区高亮叠加
 │    │    │    └─ render_document:
 │    │    │         逐行 → 逐字素 → buf.set_grapheme(x, y, g, w, style)
 │    │    │         （热路径：无 Unicode 重新分段）
 │    │    └─ 状态栏: set_string(...)
 │    │
 │    ├─ layers[1] = Picker::render()
 │    │    └─ Block::bordered().render()  → 画边框
 │    │       + set_string_truncated()    → 画条目
 │    │
 │    └─ layers[N-1] = Prompt::render()
 │         └─ 画命令行 + 光标
 │
 ├─ (pos, kind) = compositor.cursor(...)  ← 冒泡查询光标
 │
 └─ terminal.draw(pos, kind)              ← 提交到终端
      │
      ├─ ① flush()
      │    ├─ prev.diff(curr)             ← Buffer::diff 增量计算
      │    └─ Backend::draw(updates)
      │         ├─ 状态机: MoveTo / SetColors / SetAttribute...
      │         └─ queue!(Print(symbol))  → 写入 Stdout 缓冲
      │
      ├─ ② set_cursor(x, y)
      ├─ ③ show/hide_cursor(kind)
      │
      ├─ ④ buffers[1-current].reset()     ← 清空下一帧缓冲
      │    current = 1 - current          ← 交换
      │
      └─ ⑤ backend.flush()                ← Stdout.flush() → 显示到屏幕
```

**从状态到屏幕的关键协作关系表**：

| 步骤 | 发生位置 | 输入 | 输出 | 协作对象 |
|------|---------|------|------|---------|
| 状态产生 | 事件处理层 | 键盘/LSP/IO | `Editor` 字段变更 | 命令处理器 |
| 组件触发 | `Component::render` | `&mut Editor` + Rect | `&mut Buffer` 修改 | Compositor |
| 区域计算 | `Layout::split` | `Rect` + 约束数组 | `Vec<Rect>` | Cassowary Solver |
| 内容填充 | `Buffer::set_*` | 字符串/字素 + Style | Cell 网格写入 | 组件内部逻辑 |
| 增量提取 | `Buffer::diff` | 前后两个 Buffer | `Vec<(x,y,&Cell)>` | Terminal |
| 编码输出 | `Backend::draw` | diff 迭代器 | ANSI 转义字节流 | Crossterm/Termina |
| 物理显示 | OS 终端程序 | stdout 字节流 | 像素/字符渲染 | 用户屏幕 |

---

## 关键协作点速查表

### 组件要画东西，需要访问什么？
- 写单元格：`surface[(x, y)].set_symbol(...)` 或 `surface.set_string(x, y, text, style)`
- 读状态：`cx.editor` → `doc!(cx.editor)` / `view!(cx.editor)`
- 读主题：`let style = cx.editor.theme.get("scope.name")`
- 子布局：`Layout::default()....split(area)`

### 组件要在特定位置画浮层？
- Popup 组件通常在 `required_size()` 返回想要的尺寸
- 父组件（或自己）根据 `area` 计算出 `popup_rect`
- 直接往 `surface` 对应坐标写入即可，无需注册

### 如何强制所有组件重绘？
- `compositor.need_full_redraw()` 设置标志
- 下一帧 `Application::render()` 会调用 `terminal.clear()` 重置后缓冲

### 如何判断某帧哪些单元格需要实际发送到终端？
- 看 `Buffer::diff()` 的输出：返回的 Vec 长度就是变化量
- 宽字符、样式变化、前帧残留都会触发刷新

### 颜色/样式最终如何变成字节？
- `Color::Rgb(r,g,b)` → `CColor::Rgb {r,g,b}` → crossterm 编码为 `\x1b[38;2;R;G;Bm`
- `Modifier::BOLD` → `SetAttribute(Bold)` → `\x1b[1m`
- `UnderlineStyle::Curl`（需能力）→ `\x1b[4:3m`
