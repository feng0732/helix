# Helix 文件 IO 与编码实现分析

## 概述

Helix 编辑器的文件 IO 与编码处理主要分布在以下几个核心模块中：

- [document.rs](helix-view/src/document.rs)：文件打开、保存、编码识别与转换的核心实现
- [line_ending.rs](helix-core/src/line_ending.rs)：行尾（Line Ending）检测与处理
- [editor_config.rs](helix-core/src/editor_config.rs)：EditorConfig 配置解析（含编码设置）
- [editor.rs](helix-view/src/editor.rs)：编辑器层面的文件操作封装
- [typed.rs](helix-term/src/commands/typed.rs)：命令层的文件操作入口
- [application.rs](helix-term/src/application.rs)：应用层的保存事件处理

编码功能基于 `encoding_rs` crate 实现，自动检测使用 `chardetng` crate。

---

## 一、文件打开链路

### 1.1 调用层级

```
命令层 open_impl() (typed.rs)
    ↓
Editor::open() (editor.rs)
    ↓
Document::open() (document.rs)
    ↓
from_reader() + read_and_detect_encoding() (document.rs)
```

### 1.2 命令层入口

[open_impl()](helix-term/src/commands/typed.rs) 是 `:open` 命令的实现：

- 解析文件路径参数，支持 `~` 展开
- 如果是目录，打开文件选择器
- 如果是文件，调用 `cx.editor.open(&path, action)`
- 支持跳转到指定行列位置

### 1.3 Editor 层

[Editor::open()](helix-view/src/editor.rs)：

- 检查文档是否已打开（通过路径去重）
- 未打开则调用 `Document::open()` 创建新文档
- 初始化诊断、diff 基准、版本控制信息
- 启动语言服务器
- 派发 `DocumentDidOpen` 事件

### 1.4 Document 层 — 核心打开逻辑

[Document::open()](helix-view/src/document.rs) 是文件打开的核心，完整步骤如下：

**步骤 1：文件类型校验**

```rust
if path.metadata().is_ok_and(|metadata| !metadata.is_file()) {
    return Err(DocumentOpenError::IrregularFile);
}
```

非常规文件（如 `/dev/random`）直接返回 `IrregularFile` 错误。

**步骤 2：EditorConfig 加载**

```rust
let editor_config = if config.load().editor_config {
    EditorConfig::find(path)
} else {
    EditorConfig::default()
};
```

如果配置中启用了 `editor_config`，会沿路径向上搜索 `.editorconfig` 文件，提取 `charset`、`end_of_line`、`indent_style` 等配置。

**步骤 3：编码优先级**

```rust
encoding = encoding.or(editor_config.encoding);
```

编码确定优先级：用户手动指定 > EditorConfig `charset` > 自动检测。注意此时若用户已手动指定，EditorConfig 的 `charset` 会被跳过。

**步骤 4：文件读取与行尾初始值**

```rust
let (rope, encoding, has_bom) = if path.exists() {
    let mut file = std::fs::File::open(path)?;
    from_reader(&mut file, encoding)?
} else {
    let line_ending = editor_config
        .line_ending
        .unwrap_or_else(|| config.load().default_line_ending.into());
    let encoding = encoding.unwrap_or(encoding::UTF_8);
    (Rope::from(line_ending.as_str()), encoding, false)
};
```

这是关键分支：

- **路径存在（已有文件）**：调用 `from_reader()` 从磁盘解码。此时 Rope 中保留原始行尾（CRLF 不转 LF，反之亦然），`has_bom` 取决于文件是否有 BOM。
- **路径不存在（新文件）**：创建只包含一个行尾字符的 Rope。行尾来源优先级为：
  1. `editor_config.line_ending`（EditorConfig 中的 `end_of_line`）
  2. `config.load().default_line_ending.into()`（用户配置的默认行尾）

**步骤 5：构造 Document 并做后处理**

```rust
let mut doc = Self::from(rope, Some((encoding, has_bom)), config, syn_loader);
doc.set_path(Some(path));
if detect_language {
    doc.detect_language(&loader);
}
doc.editor_config = editor_config;
doc.detect_indent_and_line_ending();
```

`Document::from()` 内部会将 `line_ending` 设为 `config.load().default_line_ending.into()`（即默认行尾配置），但紧接着 `detect_indent_and_line_ending()` 会覆盖这个值——见下文。

---

## 二、默认行尾处理详解

### 2.1 LineEndingConfig 与 NATIVE_LINE_ENDING

[LineEndingConfig](helix-view/src/editor.rs) 是用户可配置的行尾选项：

```rust
pub enum LineEndingConfig {
    #[default]
    Native,  // 平台原生：Windows 为 CRLF，其他为 LF
    LF,
    Crlf,
    // unicode-lines feature: FF, CR, Nel
}
```

`LineEndingConfig::Native` 通过 `From<LineEndingConfig> for LineEnding` 转换：

```rust
impl From<LineEndingConfig> for LineEnding {
    fn from(line_ending: LineEndingConfig) -> Self {
        match line_ending {
            LineEndingConfig::Native => NATIVE_LINE_ENDING,
            LineEndingConfig::LF => LineEnding::LF,
            LineEndingConfig::Crlf => LineEnding::Crlf,
            // ...
        }
    }
}
```

[NATIVE_LINE_ENDING](helix-core/src/line_ending.rs) 由编译条件决定：

```rust
#[cfg(target_os = "windows")]
pub const NATIVE_LINE_ENDING: LineEnding = LineEnding::Crlf;
#[cfg(not(target_os = "windows"))]
pub const NATIVE_LINE_ENDING: LineEnding = LineEnding::LF;
```

### 2.2 新建文档（无路径）的行尾

[Document::default()](helix-view/src/document.rs) 用于创建空白文档（如 `:new`）：

```rust
pub fn default(config, syn_loader) -> Self {
    let line_ending: LineEnding = config.load().default_line_ending.into();
    let text = Rope::from(line_ending.as_str());  // Rope 内容就是一个行尾字符
    Self::from(text, None, config, syn_loader)
}
```

[Document::from()](helix-view/src/document.rs) 内部也会设置 `line_ending`：

```rust
let line_ending = config.load().default_line_ending.into();
```

所以对于 `Document::default()`，`doc.line_ending` 和 Rope 中的行尾字符是一致的，都来自 `default_line_ending` 配置。

### 2.3 打开已有文件的行尾

对于已有文件，`from_reader()` 解码后的 Rope 保留了原始行尾（CRLF 还是 LF 不做归一化）。然后 `detect_indent_and_line_ending()` 被调用：

```rust
pub fn detect_indent_and_line_ending(&mut self) {
    // ... 缩进检测 ...
    if let Some(line_ending) = self
        .editor_config
        .line_ending
        .or_else(|| auto_detect_line_ending(&self.text))
    {
        self.line_ending = line_ending;
    }
}
```

行尾元信息 `doc.line_ending` 的确定优先级：
1. `editor_config.line_ending`（EditorConfig 的 `end_of_line`）
2. `auto_detect_line_ending(&self.text)`（扫描前 100 行，返回首个匹配的行尾）

**关键理解**：`doc.line_ending` 只是一个元信息标记，**不影响 Rope 中的实际行尾字符**。它的作用是在需要"插入新行尾"时（如 `insert_final_newline`）决定用 `\n` 还是 `\r\n`。

### 2.4 新文件但路径不存在时的行尾

当 `Document::open()` 中 `!path.exists()` 时：

```rust
let line_ending = editor_config
    .line_ending
    .unwrap_or_else(|| config.load().default_line_ending.into());
let encoding = encoding.unwrap_or(encoding::UTF_8);
(Rope::from(line_ending.as_str()), encoding, false)
```

Rope 内容 = 一个行尾字符，行尾来源：EditorConfig > 用户配置默认值 > 平台原生。

### 2.5 stdin 文档的行尾

[Editor::new_file_from_stdin()](helix-view/src/editor.rs) 读取 stdin 内容：

```rust
let (stdin, encoding, has_bom) = crate::document::read_to_string(&mut stdin(), None)?;
let doc = Document::from(
    helix_core::Rope::default(),  // 空 Rope
    Some((encoding, has_bom)),
    // ...
);
// 然后通过 Transaction::insert 将 stdin 内容插入
```

stdin 内容通过 `read_to_string()` 解码后原样插入 Rope，行尾不归一化。

---

## 三、编码识别机制

### 3.1 三级识别策略

[read_and_detect_encoding()](helix-view/src/document.rs) 实现了三级编码识别：

```
第一级：用户手动指定编码（encoding 参数）
    ↓ 未指定
第二级：BOM 检测（encoding_rs::Encoding::for_bom）
    ↓ 未检测到
第三级：chardetng 统计检测（chardetng::EncodingDetector）
```

具体代码：

```rust
fn read_and_detect_encoding<R: std::io::Read + ?Sized>(
    reader: &mut R,
    encoding: Option<&'static Encoding>,
    buf: &mut [u8],
) -> Result<(&'static Encoding, bool, encoding::Decoder, usize), io::Error> {
    let read = reader.read(buf)?;
    let is_empty = read == 0;
    let (encoding, has_bom) = encoding
        .map(|encoding| (encoding, false))           // 第一级：手动指定
        .or_else(|| encoding::Encoding::for_bom(buf)  // 第二级：BOM
            .map(|(encoding, _bom_size)| (encoding, true)))
        .unwrap_or_else(|| {                          // 第三级：chardetng
            let mut encoding_detector =
                chardetng::EncodingDetector::new(chardetng::Iso2022JpDetection::Allow);
            encoding_detector.feed(buf, is_empty);
            (encoding_detector.guess(None, chardetng::Utf8Detection::Allow), false)
        });
    let decoder = encoding.new_decoder();
    Ok((encoding, has_bom, decoder, read))
}
```

注意第一级手动指定时 `has_bom = false`——即使文件实际有 BOM，手动指定编码会跳过 BOM 检测，保存时不会写回 BOM。

### 3.2 BOM 检测

支持的 BOM 类型：
- UTF-8 BOM：`EF BB BF`
- UTF-16BE BOM：`FE FF`
- UTF-16LE BOM：`FF FE`

通过 `encoding_rs::Encoding::for_bom(&buf)` 检测，返回编码和 BOM 大小。BOM 字节会被 `encoding_rs` 的 decoder 自动跳过，不会出现在解码后的 Rope 中。

### 3.3 chardetng 自动检测

当 BOM 检测失败时，使用 `chardetng` 库进行统计编码检测：

- 允许 ISO-2022-JP 检测
- 允许 UTF-8 检测（`Utf8Detection::Allow`）
- 仅基于第一块读取的数据（最多 8KB）进行猜测
- 如果第一块为空（0 字节），回退到 UTF-8

### 3.4 EditorConfig 中的编码

[EditorConfig](helix-core/src/editor_config.rs) 支持从 `.editorconfig` 的 `charset` 键读取编码：

| charset 值 | 对应编码 |
|-----------|---------|
| `latin1` | WINDOWS_1252 |
| `utf-8` | UTF-8 |
| `utf-16le` | UTF-16LE |
| `utf-16be` | UTF-16BE |

> 注意：`utf-8-bom` 被故意忽略，因为规范不推荐使用。

EditorConfig 的编码在 `Document::open()` 中通过 `encoding = encoding.or(editor_config.encoding)` 应用，仅当用户未手动指定编码时生效。

---

## 四、文件读取（解码）流程

### 4.1 核心函数

[from_reader()](helix-view/src/document.rs) 将字节流解码为 UTF-8 并构建 `Rope`。

### 4.2 缓冲区设计

使用两个 8KB（`BUF_SIZE = 8192`）缓冲区：
- `buf`：输入缓冲区，存放从 reader 读取的原始字节
- `buf_out`：输出缓冲区，存放解码后的 UTF-8 字节

### 4.3 双层循环解码

**外层循环**：从 reader 读取数据块
**内层循环**：将输入缓冲区解码到输出缓冲区

```
read_and_detect_encoding() → 首次读取 + 编码检测
    ↓
外层循环:
  内层循环解码:
    ├─ CoderResult::OutputFull → 追加到 RopeBuilder，清空输出缓冲
    └─ CoderResult::InputEmpty → 退出内层循环
  判断 is_empty:
    ├─ 是 → 刷新输出缓冲，完成
    └─ 否 → reader.read() 读取下一块
```

关键变量：
- `total_read`：当前 chunk 中已处理的字节数
- `total_written`：输出缓冲区已写字节数
- `is_empty`：是否到达流末尾

### 4.4 不安全代码

```rust
let buf_str = unsafe { std::str::from_utf8_unchecked_mut(&mut buf_out[..]) };
```

由于 `buf_out` 是零初始化数组且解码输出总是有效 UTF-8，因此使用 `from_utf8_unchecked` 跳过验证以提高性能。

### 4.5 行尾在解码中的处理

`from_reader()` **不做行尾归一化**。原始字节流中的 CRLF (`\r\n`) 解码为 UTF-8 后仍然是 `\r\n`，直接进入 Rope。这意味着：

- 一个 CRLF 文件被读入后，Rope 中每行末尾是 `\r\n`
- 一个 LF 文件被读入后，Rope 中每行末尾是 `\n`
- 混合行尾的文件也会原样保留

### 4.6 read_to_string()

[read_to_string()](helix-view/src/document.rs) 是 `from_reader()` 的简化版本，直接返回 `String` 而不是 `Rope`，用于 stdin 读取等场景。解码逻辑完全相同。

---

## 五、文件保存（编码）流程

### 5.1 调用层级

```
命令层 write_impl() (typed.rs)         ← 保存前预处理
    ↓
Editor::save() (editor.rs)             ← 获取 Future，发送到保存通道
    ↓
Document::save() → save_impl()         ← 构造异步保存 Future
    ↓  Future 被异步执行
to_writer() (document.rs)              ← 编码 + 写入磁盘
    ↓
handle_document_write() (application.rs) ← 事件回调，更新状态
```

### 5.2 命令层 — 保存前预处理

[write_impl()](helix-term/src/commands/typed.rs) 处理 `:w` / `:write` 命令，在真正保存前执行三项预处理：

**1. 修剪行尾空白**

```rust
if doc.trim_trailing_whitespace() {
    trim_trailing_whitespace(doc, view.id);
}
```

[trim_trailing_whitespace()](helix-term/src/commands/typed.rs) 遍历所有行，删除行尾空白字符（行尾字符本身保留）。这会生成一个 `Transaction::delete` 并 apply 到文档。

是否启用的判断：
```rust
pub fn trim_trailing_whitespace(&self) -> bool {
    self.editor_config
        .trim_trailing_whitespace
        .unwrap_or_else(|| self.config.load().trim_trailing_whitespace)
}
```

优先级：EditorConfig > 全局配置。默认 `false`。

**2. 修剪末尾空行**

```rust
if config.trim_final_newlines {
    trim_final_newlines(doc, view.id);
}
```

[trim_final_newlines()](helix-term/src/commands/typed.rs) 删除文件末尾的多个空行，只保留最后一个。使用 `line_ending::get_line_ending()` 逐个检测末尾行尾。此配置无 EditorConfig 覆盖，默认 `false`。

**3. 插入末尾换行**

```rust
if doc.insert_final_newline() {
    insert_final_newline(doc, view_id);
}
```

[insert_final_newline()](helix-term/src/commands/typed.rs) 是**行尾与保存交互的关键位置**：

```rust
fn insert_final_newline(doc: &mut Document, view_id: ViewId) {
    let text = doc.text();
    if text.len_chars() > 0 && line_ending::get_line_ending(&text.slice(..)).is_none() {
        let eof = Selection::point(text.len_chars());
        let insert = Transaction::insert(text, &eof, doc.line_ending.as_str().into());
        doc.apply(&insert, view_id);
    }
}
```

这里 `doc.line_ending.as_str()` 决定了插入的行尾是 `\n` 还是 `\r\n`。这是 `doc.line_ending` 元信息**直接影响 Rope 内容**的唯一位置。

是否启用的判断：
```rust
pub fn insert_final_newline(&self) -> bool {
    self.editor_config
        .insert_final_newline
        .unwrap_or_else(|| self.config.load().insert_final_newline)
}
```

优先级：EditorConfig > 全局配置。默认 `true`。

### 5.3 Editor 层

[Editor::save()](helix-view/src/editor.rs)：

- 调用 `doc.save(path, force)` 获取保存 future
- 将 future 发送到保存通道（`self.saves`），与其他保存操作串行化
- 增加写入计数

### 5.4 Document 层 — 保存核心

[Document::save_impl()](helix-view/src/document.rs) 是保存的核心实现，返回一个 `Future`。

**保存前检查：**

1. **路径解析**：指定路径或使用文档当前路径；无路径时 `bail!("Can't save with no path set!")`
2. **父目录检查**：不存在时，`force=true` 递归创建，否则报错
3. **外部修改检测**：非 force 模式下，比较文件 mtime 与 `last_saved_time`
4. **符号链接解析**：解析符号链接到实际写入路径
5. **只读检查**：只读路径返回 `PermissionDenied` 错误
6. **硬链接/符号链接检测**：影响原子保存策略

**关键：编码与行尾信息的传递**

```rust
let text = self.text().clone();                              // Rope 原样克隆
let encoding_with_bom_info = (self.encoding, self.has_bom);  // 编码 + BOM 标记
```

**核心事实：保存时不做任何行尾转换。** Rope 内容原样传给 `to_writer()`，Rope 里是 `\r\n` 就写 `\r\n`，是 `\n` 就写 `\n`。`doc.line_ending` 在保存路径中不参与行尾转换。

**原子保存机制：**

当 `atomic_save = true` 且文件存在时：
- 创建备份文件（`.bck` 后缀，使用 `tempfile::Builder`）
- 硬链接/符号链接：使用 `copy` 方式备份（因为 rename 会断开硬链接）
- 普通文件：使用 `rename` 方式备份（原子操作）
- 写入失败时从备份恢复
- 写入成功后：普通文件复制元数据并删除备份；硬链接的备份直接删除

**实际写入：**

```rust
let mut dst = tokio::fs::File::create(&write_path).await?;
to_writer(&mut dst, encoding_with_bom_info, &text).await?;
// sync_all 忽略 Unsupported 错误（如 SMB 文件系统）
match dst.sync_all().await {
    Ok(_) => (),
    Err(err) if err.kind() == io::ErrorKind::Unsupported => (),
    #[cfg(unix)]
    Err(err) if matches!(err.raw_os_error(), Some(libc::ENOTSUP | libc::EOPNOTSUPP)) => {}
    Err(err) => return Err(err.into()),
}
```

**保存后操作：**
- 记录保存时间和修订号
- 通知语言服务器 `textDocument/didSave`
- 返回 `DocumentSavedEvent`

### 5.5 编码写入核心 — to_writer()

[to_writer()](helix-view/src/document.rs) 将 `Rope` 编码为指定编码并写入。这是**编码转换发生的唯一位置**。

**写入流程：**

```
1. 如果 has_bom，先在 buf 头部写入 BOM 字节
2. 创建 Encoder（根据编码类型）
3. 遍历 Rope 的所有非空 chunk + 末尾一个空 chunk
4. 对每个 chunk 做双层循环编码：
   ├─ CoderResult::OutputFull → writer.write_all()，清空 buf
   └─ CoderResult::InputEmpty → 继续下一个 chunk
5. 最后一个空 chunk 触发刷新：writer.write_all() + writer.flush()
```

**BOM 写入（[apply_bom()](helix-view/src/document.rs)）：**

保存时如果 `has_bom = true`，在输出缓冲区开头写入 BOM：

```rust
fn apply_bom(encoding: &'static encoding::Encoding, buf: &mut [u8; BUF_SIZE]) -> usize {
    if encoding == encoding::UTF_8 {
        buf[0] = 0xef; buf[1] = 0xbb; buf[2] = 0xbf;  // 3 字节
        3
    } else if encoding == encoding::UTF_16BE {
        buf[0] = 0xfe; buf[1] = 0xff;                   // 2 字节
        2
    } else if encoding == encoding::UTF_16LE {
        buf[0] = 0xff; buf[1] = 0xfe;                   // 2 字节
        2
    } else {
        0  // 非 UTF 编码不写 BOM
    }
}
```

BOM 写入与首个 chunk 的编码输出共享同一个 `buf`，通过 `total_written` 偏移量来管理。

### 5.6 Encoder 封装

[Encoder](helix-view/src/document.rs) 是 Helix 对编码器的封装：

```rust
enum Encoder {
    Utf16Be,                      // 手动实现 UTF-16BE
    Utf16Le,                      // 手动实现 UTF-16LE
    EncodingRs(encoding::Encoder), // 其他编码使用 encoding_rs
}
```

**为什么 UTF-16 需要手动实现？**

`encoding_rs` 的 `encode_from_utf8()` 在处理 UTF-16 时，可能在多字节字符的中间位置返回 `OutputFull`，导致字符被拆分。Helix 的手动实现按字符逐个编码，确保不会在字符中间断开：

```rust
fn encode_from_utf8(&mut self, src: &str, dst: &mut [u8], is_empty: bool)
    -> (encoding::CoderResult, usize, usize)
{
    // UTF-16 分支：
    let to_write = src.char_indices().map(|(indice, char)| {
        let mut encoded: [u16; 2] = [0, 0];
        (indice, char.encode_utf16(&mut encoded)
            .iter_mut()
            .flat_map(|char| convert(*char))  // u16 → [u8; 2]
            .collect::<Vec<u8>>())
    });
    for (indice, utf16_bytes) in to_write {
        if dst.len() <= (total_written + character_size) {
            return (OutputFull, indice, total_written);  // 在字符边界返回
        }
        // ...
    }
}
```

对于非 UTF-16 编码，直接委托给 `encoding_rs` 的 `Encoder::encode_from_utf8()`。

---

## 六、行尾（Line Ending）处理详解

### 6.1 LineEnding 枚举

[LineEnding](helix-core/src/line_ending.rs) 支持的行尾类型：

- 始终可用：`LF` (`\n`)、`Crlf` (`\r\n`)
- `unicode-lines` feature 启用时额外支持：`VT`、`FF`、`CR`、`Nel`、`LS`、`PS`

### 6.2 自动检测

[auto_detect_line_ending()](helix-core/src/line_ending.rs)：

- 扫描前 100 行
- 返回第一个匹配的行尾（LF 或 Crlf）
- 忽略 VT、FF、PS 等特殊用途行尾（unicode-lines 模式下跳过它们）
- 如果 100 行内没有行尾，返回 `None`

### 6.3 doc.line_ending 与 Rope 内容的关系

这是理解 Helix 行尾处理的核心：

| 场景 | Rope 中的行尾 | doc.line_ending |
|------|-------------|----------------|
| 打开 CRLF 文件 | `\r\n`（原样保留） | `LineEnding::Crlf`（自动检测） |
| 打开 LF 文件 | `\n`（原样保留） | `LineEnding::LF`（自动检测） |
| 新建文档 | `default_line_ending` 对应的字符 | `default_line_ending` 转换结果 |
| 打开混合行尾文件 | 原样保留混合行尾 | 首个检测到的行尾类型 |

**doc.line_ending 的用途**：只在以下场景决定用什么行尾字符：
1. `insert_final_newline()` — 在文件末尾追加行尾时
2. 状态栏显示 — 显示当前文档的行尾类型

**doc.line_ending 不做的事**：不做行尾转换。Helix 没有 CRLF ↔ LF 的自动转换功能。

### 6.4 行尾优先级汇总

**打开文件时 `doc.line_ending` 的确定**：

```
1. EditorConfig end_of_line（如 "lf"、"crlf"）
   ↓ 未设置
2. auto_detect_line_ending() 扫描 Rope 内容
   ↓ 无行尾
3. 保持 Document::from() 中的初始值，即 config.default_line_ending
```

**新建文件时 Rope 初始内容的行尾**：

```
1. EditorConfig end_of_line
   ↓ 未设置
2. config.default_line_ending（默认 Native → 平台原生）
```

---

## 七、编码转换在保存时的具体位置

### 7.1 完整数据流

```
Rope (内部始终为 UTF-8)
    ↓  rope.chunks() 遍历
UTF-8 字符串片段（&str）
    ↓  Encoder::encode_from_utf8()
目标编码字节流（&mut [u8]）
    ↓  writer.write_all()
磁盘文件
```

### 7.2 编码转换发生的精确位置

编码转换的**唯一位置**是 [to_writer()](helix-view/src/document.rs) 中的内层循环：

```rust
let (result, read, written, ..) =
    encoder.encode_from_utf8(&chunk[total_read..], &mut buf[total_written..], is_empty);
```

这一行完成了从 UTF-8 `&str` 到目标编码 `&mut [u8]` 的转换。

- **UTF-8 编码保存**：`encode_from_utf8` 实际上是直通（几乎无开销），因为 Rope 本身就是 UTF-8
- **UTF-16 保存**：经过手动 `char.encode_utf16()` → 字节序转换
- **其他编码保存**：通过 `encoding_rs` 的 `Encoder::encode_from_utf8()` 处理

### 7.3 行尾在编码转换中的处理

**编码转换不感知行尾**。`Encoder::encode_from_utf8()` 接收的是原始 UTF-8 字符串，其中包含的 `\r\n` 或 `\n` 会被原样编码为目标编码的对应字节序列。

例如，UTF-8 中的 `\r\n`（`0x0D 0x0A`）在 UTF-16LE 编码中变为 `0x0D 0x00 0x0A 0x00`。

### 7.4 保存时无行尾转换

Helix 保存时**不做 CRLF ↔ LF 转换**。Rope 中是什么行尾就写什么行尾。这与一些编辑器（如 VS Code 的 `files.eol` 设置）不同。

---

## 八、异常处理链路

### 8.1 错误类型

#### DocumentOpenError

[DocumentOpenError](helix-view/src/document.rs)：

```rust
pub enum DocumentOpenError {
    IrregularFile,           // 非常规文件（如设备文件）
    IoError(#[from] io::Error),  // IO 错误
}
```

使用 `thiserror` 派生，`IoError` 通过 `#[from]` 自动实现 `From<io::Error>`。

#### CloseError

[CloseError](helix-view/src/editor.rs)：

```rust
pub enum CloseError {
    DoesNotExist,              // 文档不存在
    BufferModified(String),    // 缓冲区已修改（提示用户保存）
    SaveError(anyhow::Error),  // 保存失败
}
```

#### FormatterError

[FormatterError](helix-view/src/document.rs)：格式化相关错误。

#### anyhow::Error

保存操作的大部分错误使用 `anyhow::Error` 传播，包括：
- 路径无父目录
- 外部修改冲突
- 只读文件
- IO 写入失败
- sync_all 失败

### 8.2 错误传播路径

**打开文件错误：**

```
Document::open() → Result<Self, DocumentOpenError>
    ↓
Editor::open() → Result<DocumentId, DocumentOpenError>
    ↓
typed::open_impl() → anyhow::Result<()>
    ↓
命令执行 → 错误显示在状态栏
```

**保存文件错误（同步部分）：**

```
Document::save_impl() 构造 Future → 同步返回 Err(anyhow::Error)
    ↓
Editor::save() → anyhow::Result<()>
    ↓
typed::write_impl() → anyhow::Result<()>
    ↓
状态栏显示错误
```

**保存文件错误（异步部分）：**

```
Future 执行中出错 → Err(anyhow::Error)
    ↓
保存通道 (mpsc stream) 传递
    ↓
Application 主循环接收 EditorEvent::DocumentSaved(Err)
    ↓
handle_document_write() → self.editor.set_error(err.to_string())
    ↓
状态栏显示错误信息
```

### 8.3 应用层处理

[handle_document_write()](helix-term/src/application.rs)：

- 成功：更新 `last_saved_revision` 和 `save_time`、设置文档路径、显示保存状态（文件名 + 行数 + 大小）
- 失败：调用 `self.editor.set_error(err.to_string())` 在状态栏显示错误

### 8.4 启动时的特殊处理

应用启动时打开文件会忽略 `IrregularFile` 错误（跳过非常规文件继续处理下一个），其他 IO 错误直接导致启动失败。

---

## 九、关键数据结构

### 9.1 Document 中的编码相关字段

```rust
pub struct Document {
    text: Rope,                                // 文档内容（内部始终 UTF-8）
    encoding: &'static encoding::Encoding,     // 文件编码
    has_bom: bool,                             // 是否有 BOM（影响保存时是否写 BOM）
    line_ending: LineEnding,                   // 行尾风格（元信息，不控制 Rope 内容）
    last_saved_time: SystemTime,               // 上次保存时间（用于外部修改检测）
    last_saved_revision: usize,                // 上次保存的修订号
    editor_config: EditorConfig,               // EditorConfig 设置
    // ...
}
```

### 9.2 DocumentSavedEvent

```rust
pub struct DocumentSavedEvent {
    pub revision: usize,
    pub save_time: SystemTime,
    pub doc_id: DocumentId,
    pub path: PathBuf,
    pub text: Rope,
}
```

---

## 十、重新加载（Reload）

[Document::reload()](helix-view/src/document.rs) 重新从磁盘加载文件：

- 使用文档当前编码（不重新检测），通过 `Some(encoding)` 传入 `from_reader()`
- `from_reader()` 会跳过自动编码检测，使用指定编码解码
- 比较新旧内容差异，以 `Transaction` 方式应用（保留 undo 历史）
- 重置修改状态
- 重新检测缩进和行尾（可能覆盖 `doc.line_ending`）
- 更新 diff 基准和版本控制信息

---

## 十一、编码切换

[Document::set_encoding()](helix-view/src/document.rs) 允许运行时切换编码：

```rust
pub fn set_encoding(&mut self, label: &str) -> Result<(), Error> {
    let encoding = Encoding::for_label(label.as_bytes())
        .ok_or_else(|| anyhow!("unknown encoding"))?;
    self.encoding = encoding;
    Ok(())
}
```

- 通过 `Encoding::for_label()` 查找编码（支持 "utf-8"、"gbk"、"shift_jis" 等标签）
- **只影响后续保存操作**，不重新解码当前 Rope 内容
- 不修改 `has_bom` 标志
- 不影响已有行尾

---

## 十二、自动保存

[AutoSaveHandler](helix-term/src/handlers/auto_save.rs) 实现自动保存：

- 基于事件驱动的防抖（debounce）机制
- 文档变更事件 `DocumentDidChange` 触发定时保存
- 离开插入模式 `OnModeSwitch` 时立即保存（如果有待保存内容）
- 插入模式下不执行实际保存（避免修改状态混乱）
- 内部调用 `write_all_impl` 保存所有修改的文档
- 保存出错时在状态栏显示错误

---

## 总结

Helix 的文件 IO 与编码系统设计要点：

1. **三级编码识别**：手动指定 → BOM 检测 → chardetng 统计检测
2. **流式处理**：8KB 双缓冲 + 双层循环，适配大文件，读取与写入结构对称
3. **行尾原样保留**：Rope 中保持原始行尾，`doc.line_ending` 仅作为元信息，在 `insert_final_newline` 时决定插入哪种行尾
4. **无行尾转换**：保存时不做 CRLF ↔ LF 转换，与部分编辑器行为不同
5. **编码转换仅在 to_writer()**：Rope (UTF-8) → 目标编码，是唯一的编码转换点
6. **异步保存**：保存操作返回 Future，不阻塞编辑；通过通道串行化多个保存请求
7. **原子保存**：支持备份和恢复，区分硬链接/符号链接/普通文件
8. **外部修改检测**：基于 mtime 的冲突检测，防止覆盖外部修改
9. **完整的错误处理**：从底层 IO 到 UI 展示的完整错误链路
10. **EditorConfig 集成**：编码、行尾、缩进、空白修剪等可通过 `.editorconfig` 统一管理
