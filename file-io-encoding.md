# Helix 文件 IO 与编码实现分析

## 概述

Helix 编辑器的文件 IO 与编码处理主要分布在以下几个核心模块中：

- [document.rs](helix-view/src/document.rs)：文件打开、保存、编码识别与转换的核心实现
- [line_ending.rs](helix-core/src/line_ending.rs)：行尾（Line Ending）检测与处理
- [editor_config.rs](helix-core/src/editor_config.rs)：EditorConfig 配置解析（含编码设置）
- [editor.rs](helix-view/src/editor.rs)：编辑器层面的文件操作封装
- [typed.rs](helix-term/src/commands/typed.rs)：命令层的文件操作入口
- [commands.rs](helix-term/src/commands.rs)：普通模式编辑命令（换行、粘贴、o/O等）
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

[Document::default()](helix-view/src/document.rs) 用于创建空白文档（如 `:new` 命令）：

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
3. 保持 `Document::from()` 中的初始值（`default_line_ending`）

**重要理解**：`detect_indent_and_line_ending()` 只设置元信息 `self.line_ending`，**不修改 Rope 中的实际行尾字符**。

### 2.4 新文件但路径不存在时的行尾

当 `Document::open()` 中 `!path.exists()` 时：

```rust
let line_ending = editor_config
    .line_ending
    .unwrap_or_else(|| config.load().default_line_ending.into());
let encoding = encoding.unwrap_or(encoding::UTF_8);
(Rope::from(line_ending.as_str()), encoding, false)
```

Rope 内容 = 一个行尾字符，行尾来源：EditorConfig `end_of_line` > 用户配置默认值 > 平台原生。

### 2.5 stdin 文档的行尾

[Editor::new_file_from_stdin()](helix-view/src/editor.rs) 读取 stdin 内容：

```rust
let (stdin, encoding, has_bom) = crate::document::read_to_string(&mut stdin(), None)?;
let doc = Document::from(
    helix_core::Rope::default(),  // 先创建空 Rope
    Some((encoding, has_bom)),
    // ...
);
// 然后通过 Transaction::insert 将 stdin 内容插入
```

stdin 内容通过 `read_to_string()` 解码后原样插入 Rope，行尾不做归一化。随后 `doc.detect_indent_and_line_ending()` 会根据实际内容检测并设置元信息。

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

[insert_final_newline()](helix-term/src/commands/typed.rs)：

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

这里 `doc.line_ending.as_str()` 决定了插入的行尾是 `\n` 还是 `\r\n`。这是 `doc.line_ending` 元信息影响 Rope 内容的场景之一（详见第十三章）。

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

**BOM 写入（apply_bom()）：**

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

**doc.line_ending 的作用**：在需要"插入新行尾字符"的所有编辑路径中决定用哪种行尾（详见第十三章）。

**doc.line_ending 不做的事**：

- ❌ 不会自动将已有内容的行尾统一为元信息指定的格式
- ❌ 不会在保存时做行尾转换
- ❌ 不会在打开时做行尾归一化

**行尾不归一化原则**：

Helix 的核心设计原则是"所见即所得"——文件在磁盘上是什么行尾，Rope 中就是什么行尾，保存时原样写出。`doc.line_ending` 只是一个"预期的行尾风格"元信息，用于指导后续编辑操作中新行尾的插入，而非强制已有内容的格式。

### 6.4 行尾检测函数

Helix 提供了三个层级的行尾检测函数（详见 13.10 节）：

- `get_line_ending(&RopeSlice)`：检测单行 Rope 切片的行尾，用于遍历文档时
- `get_line_ending_of_str(&str)`：检测普通字符串的行尾，用于寄存器、剪贴板内容
- `auto_detect_line_ending(&Rope)`：扫描前 100 行返回首个匹配的行尾，用于打开文件时的自动检测

### 6.5 行尾优先级汇总

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

**编辑操作中新插入行尾的来源**：

```
始终来自 doc.line_ending.as_str()
（第十三章详细列出了所有调用位置）
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

编码转换的**唯一位置**是 `to_writer()` 中的内层循环：

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
    line_ending: LineEnding,                   // 行尾风格（元信息，控制新行尾插入）
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

## 十三、行尾处理边界详解

本章详细说明：显式切换行尾时是否改写已有内容、以及各编辑路径如何按当前行尾元信息插入新行尾。

### 13.1 doc.line_ending 的读取位置汇总

`doc.line_ending.as_str()` 是行尾元信息实际生效的方式。它在代码中的调用位置：

| 位置 | 用途 |
|------|------|
| [insert_newline()](helix-term/src/commands.rs) | Enter 键换行时插入行尾 |
| [open()](helix-term/src/commands.rs) | `o`/`O` 命令在上方/下方开新行时插入行尾 |
| [add_newline_impl()](helix-term/src/commands.rs) | `<a-j>`/`<a-k>` 在上方/下方添加纯换行时插入 |
| [paste_impl()](helix-term/src/commands.rs) | 粘贴时用作行尾正则替换的目标 |
| [replace_selections_with_register()](helix-term/src/commands.rs) | 用寄存器内容替换选区时用作行尾正则替换的目标 |
| [insert_final_newline()](helix-term/src/commands/typed.rs) | 保存前在末尾追加行尾时插入 |
| [Document::default()](helix-view/src/document.rs) | 新建空白文档时初始化 Rope 内容 |
| [Document::open()](helix-view/src/document.rs) | 路径不存在的新文件初始化 Rope 内容 |
| [expansion::Variable::LineEnding](helix-view/src/expansion.rs) | `$line_ending` 变量扩展返回值 |
| [Document::snippet_ctx()](helix-view/src/document.rs) | LSP 片段渲染时的行尾上下文 |

### 13.2 显式切换行尾（:line-ending 命令）

`:line-ending` 命令是唯一的**显式行尾切换**入口。完整实现在 [typed.rs](helix-term/src/commands/typed.rs)。

**核心代码：**

```rust
// 解析参数：crlf / lf 等
let line_ending = match arg {
    arg if arg.starts_with("crlf") => Crlf,
    arg if arg.starts_with("lf") => LF,
    // ... unicode-lines 额外支持 cr、ff、nel
    _ => bail!("invalid line ending"),
};

// 步骤 1：先更新元信息
let (view, doc) = current!(cx.editor);
doc.line_ending = line_ending;

// 步骤 2：创建 Transaction 替换所有行的实际行尾字符
let mut pos = 0;
let transaction = Transaction::change(
    doc.text(),
    doc.text().lines().filter_map(|line| {
        pos += line.len_chars();
        match helix_core::line_ending::get_line_ending(&line) {
            Some(ending) if ending != line_ending => {
                // 行尾不匹配 → 替换
                let start = pos - ending.len_chars();
                let end = pos;
                Some((start, end, Some(line_ending.as_str().into())))
            }
            _ => None,  // 行尾匹配或无行尾 → 跳过
        }
    }),
);
doc.apply(&transaction, view.id);
doc.append_changes_to_history(view);
```

**关键结论：**

| 问题 | 答案 |
|------|------|
| 是否修改 `doc.line_ending` 元信息？ | **是**，立即更新 `doc.line_ending = line_ending` |
| 是否改写 Rope 中已有内容？ | **是**，创建 Transaction 遍历所有行，凡是行尾与目标不同就替换 |
| 无行尾的行会被修改吗？ | **不会**，`get_line_ending()` 返回 None 的行被跳过（如最后一行没有行尾时） |
| 可以 undo 吗？ | **可以**，`append_changes_to_history()` 将 Transaction 写入历史 |
| 混合行尾如何处理？ | 每行独立判断，只替换与目标不同的行尾 |

**处理流程：**

```
用户输入 :line-ending crlf
    ↓
doc.line_ending = Crlf    ← 元信息先更新
    ↓
遍历 Rope 所有行:
  行尾 == LF? → Transaction 变更 (start, end, Some("\r\n"))
  行尾 == Crlf? → 跳过
  无行尾? → 跳过
    ↓
doc.apply(transaction)    ← 统一执行所有变更
doc.append_changes_to_history(view)
```

### 13.3 Enter 键换行（insert_newline）

在插入模式下按 Enter 键触发 [insert_newline()](helix-term/src/commands.rs)。

**核心代码：**

```rust
pub fn insert_newline(cx: &mut Context) {
    // ...
    let text = doc.text().slice(..);
    let line_ending = doc.line_ending.as_str();  // 关键：读取当前行尾元信息
    // ...
    let transaction = Transaction::change_by_selection(contents, selection, |range| {
        // ...
        // 场景 1：在行尾处有尾随空白时（常规情况）
        if let Some(idx) = text.slice(line_start..pos).last_non_whitespace_char() {
            // ...
            let local_offs = if let Some(token) = continue_comment_token {
                // 注释续行
                new_text.push_str(line_ending);
                new_text.push_str(&indent);
                new_text.push_str(token);
                new_text.push(' ');
                // ...
            } else if on_auto_pair {
                // 在括号对中间换行：插入两行
                new_text.push_str(line_ending);    // ← 使用元信息
                new_text.push_str(&inner_indent);
                // ...
                new_text.push_str(line_ending);    // ← 使用元信息
                new_text.push_str(&indent);
                // ...
            } else {
                // 普通换行
                new_text.push_str(line_ending);    // ← 使用元信息
                new_text.push_str(&indent);
                // ...
            }
            // ...
        } else {
            // 场景 2：整行都是空白
            new_text.push_str(line_ending);        // ← 使用元信息
            // ...
        }
    });
    doc.apply(&transaction, view.id);
    // ...
}
```

**关键结论：**

- `doc.line_ending.as_str()` 在函数开头被**一次性提取**为局部变量 `line_ending`
- 所有 `new_text.push_str(line_ending)` 都使用这个值
- 如果当前文档是 CRLF 模式，Enter 插入的就是 `\r\n`
- 如果当前文档是 LF 模式，Enter 插入的就是 `\n`
- **不会**修改已有内容的行尾，只会在新插入的位置使用元信息指定的行尾
- 另外会处理尾随空白删除、括号对智能换行、注释续行、自动缩进等

### 13.4 o / O 命令（open_below / open_above）

普通模式下按 `o` 在下方开新行，`O` 在上方开新行。实现位于 [open()](helix-term/src/commands.rs)。

**核心代码：**

```rust
fn open(cx: &mut Context, open: Open, comment_continuation: CommentContinuation) {
    enter_insert_mode(cx);
    let config = cx.editor.config();
    let (view, doc) = current!(cx.editor);
    // ...

    let mut transaction = Transaction::change_by_selection(contents, selection, |range| {
        // ...
        let mut text = String::with_capacity(1 + indent_len);

        if open == Open::Above && next_new_line_num == 0 {
            // 在第一行上方开新行：先写缩进 + 可选注释，后写行尾
            text.push_str(&indent);
            if let Some(token) = continue_comment_token {
                text.push_str(token);
                text.push(' ');
            }
            text.push_str(doc.line_ending.as_str());  // ← 使用元信息
        } else {
            // 常规情况：先写行尾，后写缩进 + 可选注释
            text.push_str(doc.line_ending.as_str());  // ← 使用元信息
            text.push_str(&indent);
            if let Some(token) = continue_comment_token {
                text.push_str(token);
                text.push(' ');
            }
        }

        let text = text.repeat(count);  // 重复 count 次（支持 3o 开 3 行）
        // ...
        (above_next_line_end_index, above_next_line_end_index, Some(text.into()))
    });
    // ...
    doc.apply(&transaction, view.id);
}
```

**关键结论：**

- 通过 `doc.line_ending.as_str()` 直接读取元信息（不缓存为局部变量）
- 第一行上方开新行的顺序特殊：内容在行尾**之前**（因为是在第 0 行位置插入）
- 其他情况：行尾 + 缩进 + 可选注释，行尾在最前
- 支持数字计数：`3o` 会重复插入 3 次
- 进入插入模式，光标定位在新行的缩进之后
- **不会**修改已有内容的行尾

### 13.5 粘贴操作（paste）

粘贴操作有两种主要入口：`p`（`paste_after`）和 `P`（`paste_before`）。核心逻辑在 [paste_impl()](helix-term/src/commands.rs) 和 [replace_selections_with_register()](helix-term/src/commands.rs)。

**行尾转换的核心机制：**

```rust
static LINE_ENDING_REGEX: Lazy<Regex> = Lazy::new(|| Regex::new(r"\r\n|\r|\n").unwrap());
```

这个正则匹配**所有常见行尾格式**：`\r\n`（CRLF）、`\r`（单独的 CR）、`\n`（LF）。

**paste_impl() 中的转换：**

```rust
fn paste_impl(values: &[String], doc: &mut Document, view: &mut View,
               pos: Paste, count: usize, mode: Mode) {
    // ...
    let map_value = |value| {
        // 关键：正则替换所有行尾为当前文档行尾
        let value = LINE_ENDING_REGEX.replace_all(value, doc.line_ending.as_str());
        let mut out = Tendril::from(value.as_ref());
        for _ in 1..count {
            out.push_str(&value);
        }
        out
    };
    // ...
}
```

**replace_selections_with_register() 中的转换（粘贴到选区）：**

```rust
fn replace_selections_with_register(editor: &mut Editor, register: char, count: usize) {
    // ...
    let map_value = |value: &Cow<str>| {
        // 同样的正则替换
        let value = LINE_ENDING_REGEX.replace_all(value, doc.line_ending.as_str());
        let mut out = Tendril::from(value.as_ref());
        for _ in 1..count {
            out.push_str(&value);
        }
        out
    };
    // ...
}
```

**关键结论：**

| 问题 | 答案 |
|------|------|
| 来源内容是 Windows CRLF 呢？ | 用 `LINE_ENDING_REGEX` 把所有 `\r\n` 替换为目标行尾 |
| 来源内容是老 Mac CR 呢？ | `\r` 也会被正则匹配并替换 |
| 来源是混合行尾？ | 每种行尾分别被正则捕获，统一替换 |
| 会修改已粘贴内容之外的行吗？ | **不会**，只处理被粘贴的 `value` 字符串 |
| yank 时的行尾是怎么存的？ | 见 13.7 节（寄存器使用 `NATIVE_LINE_ENDING` 拼接） |

**数据流：**

```
剪贴板内容 (可能含任意行尾)
    ↓ LINE_ENDING_REGEX.replace_all(value, doc.line_ending.as_str())
统一行尾后的字符串
    ↓ Transaction::change 或 Transaction::insert
插入到 Rope 中（行尾已与文档一致）
```

### 13.6 add_newline_above / add_newline_below 命令

绑定键：`<a-j>`（在下方添加换行，不进入插入模式）和 `<a-k>`（在上方添加）。

实现位于 [add_newline_impl()](helix-term/src/commands.rs)：

```rust
fn add_newline_impl(cx: &mut Context, open: Open) {
    let count = cx.count();
    let (view, doc) = current!(cx.editor);
    let selection = doc.selection(view.id);
    let text = doc.text();
    let slice = text.slice(..);

    let changes = selection.into_iter().map(|range| {
        let (start, end) = range.line_range(slice);
        let line = match open {
            Open::Above => start,
            Open::Below => end + 1,
        };
        let pos = text.line_to_char(line);
        (
            pos,
            pos,
            // 直接用元信息重复 count 次
            Some(doc.line_ending.as_str().repeat(count).into()),
        )
    });

    let transaction = Transaction::change(text, changes);
    doc.apply(&transaction, view.id);
}
```

**关键结论：**

- 纯插入行尾字符，**不进入插入模式**
- 与 `o`/`O` 不同：不添加任何缩进或注释续行
- 光标位置不变（仍在原行）
- 常用于在代码块之间快速插入空行

### 13.7 Yank / 寄存器与剪贴板的行尾

寄存器写入时使用**平台原生行尾 `NATIVE_LINE_ENDING`**，而非 `doc.line_ending`。这是一个特殊的边界行为。

实现位于 [register.rs](helix-view/src/register.rs)：

```rust
pub fn write(&mut self, name: char, mut values: Vec<String>) -> Result<()> {
    match name {
        '*' | '+' => {  // 系统剪贴板寄存器
            self.clipboard_provider.load().set_contents(
                // 用 NATIVE_LINE_ENDING 拼接多个选区值
                &values.join(NATIVE_LINE_ENDING.as_str()),
                ClipboardType::Clipboard,  // 或 Selection
            )?;
            // ...
        }
        _ => { /* 普通寄存器直接存储，不做处理 */ }
    }
}

pub fn push(&mut self, name: char, mut value: String) -> Result<()> {
    match name {
        '*' | '+' => {  // 追加到剪贴板
            // ...
            saved_values.push(value.clone());
            if !contents.is_empty() {
                // 追加时用 NATIVE_LINE_ENDING 分隔旧内容和新内容
                value.push_str(NATIVE_LINE_ENDING.as_str());
            }
            value.push_str(&contents);
            // ...
        }
        _ => { /* 普通寄存器直接存储 */ }
    }
}
```

**完整的 Yank → 粘贴数据流：**

```
yank（从 Rope 中选区内容原样复制）
    ↓
寄存器/剪贴板 write():
  ├─ 多选区 → 用 NATIVE_LINE_ENDING 拼接后写入剪贴板
  └─ 单选区 → 原样写入剪贴板（Rope 中是什么行尾就是什么）
    ↓
系统剪贴板（Windows 上拼接用 \r\n，Linux/macOS 用 \n）
    ↓
粘贴时 paste_impl():
  LINE_ENDING_REGEX.replace_all(内容, doc.line_ending.as_str())
    ↓ 统一为文档行尾
插入到 Rope
```

**特殊情况说明：**

- **单选区 yank → 同文档粘贴**：Rope 中是 `\r\n`，剪贴板中也是 `\r\n`，粘贴时如果文档是 CRLF 模式则不做实际替换
- **单选区 yank → 跨文档粘贴**：文档 A 是 CRLF，yank 后在文档 B（LF 模式）中粘贴，粘贴时 CRLF 会被转为 LF
- **多选区 yank**：无论源文档行尾是什么，写入剪贴板时多值之间用 `NATIVE_LINE_ENDING` 分隔；粘贴回时所有行尾（包括拼接用的分隔符）会被统一转换
- **yank 内部寄存器**：`"ay` 到 a 寄存器时不做处理，原样存储；`"ap` 时仍然经过 `LINE_ENDING_REGEX` 转换

### 13.8 LSP Snippet 渲染

LSP 代码片段（Snippet）渲染时，行尾由当前文档的 `doc.line_ending` 决定。

实现位于 [Document::snippet_ctx()](helix-view/src/document.rs)：

```rust
pub fn snippet_ctx(&self) -> SnippetRenderCtx {
    SnippetRenderCtx {
        resolve_var: Box::new(|_| None),
        tab_width: self.tab_width(),
        indent_style: self.indent_style,
        line_ending: self.line_ending.as_str(),  // ← 使用元信息
    }
}
```

`SnippetRenderCtx` 会传递给 LSP 服务，用于渲染代码片段模板。片段模板中的换行符会被转换为 `line_ending` 指定的格式。

### 13.9 变量扩展 $line_ending

Helix 支持通过 `$line_ending` 变量在命令行或配置中引用当前文档的行尾。

实现位于 [expansion.rs](helix-view/src/expansion.rs)：

```rust
pub fn expand(&self, doc: &Document, view: &View) -> Result<Cow<str>> {
    match self {
        // ...
        Variable::LineEnding => Ok(Cow::Borrowed(doc.line_ending.as_str())),
        // ...
    }
}
```

例如在状态栏配置中显示 `$line_ending` 会显示为 `\r\n` 或 `\n`（实际显示的是原始字符）。

### 13.10 行尾检测函数详解

Helix 提供了两个行尾检测函数，分别处理 `RopeSlice` 和 `&str`。

**get_line_ending(&line) - 检测单行 Rope 切片的行尾**

位于 [line_ending.rs](helix-core/src/line_ending.rs)：

```rust
pub fn get_line_ending(line: &RopeSlice) -> Option<LineEnding> {
    // 最后 1 个字符作为 str
    let g1 = line.slice(line.len_chars().saturating_sub(1)..)
        .as_str().unwrap();
    // 最后 2 个字符作为 str
    let g2 = line.slice(line.len_chars().saturating_sub(2)..)
        .as_str().unwrap_or("");
    // 先检查 2 字符的 CRLF，再检查 1 字符的行尾
    LineEnding::from_str(g2).or_else(|| LineEnding::from_str(g1))
}
```

- 用于 `RopeSlice`（来自 `rope.lines()` 迭代）
- 必须先检查 `g2`（2 字符）再检查 `g1`（1 字符），否则 `\r\n` 会被误判为 `\n`
- Ropey 保证 CRLF 始终连续，不会跨 chunk

**get_line_ending_of_str(line) - 检测字符串的行尾**

位于 [line_ending.rs](helix-core/src/line_ending.rs)：

```rust
pub fn get_line_ending_of_str(line: &str) -> Option<LineEnding> {
    if line.ends_with("\u{000D}\u{000A}") {
        Some(LineEnding::Crlf)
    } else if line.ends_with('\u{000A}') {
        Some(LineEnding::LF)
    }
    // unicode-lines feature 额外检查 VT, FF, CR, Nel, LS, PS
    else {
        None
    }
}
```

- 用于普通 `&str`（如寄存器内容、剪贴板内容）
- 使用 `\u{XXXX}` 转义序列而非直接写 `\r\n`，确保跨平台一致

**auto_detect_line_ending(doc) - 自动检测文档行尾**

位于 [line_ending.rs](helix-core/src/line_ending.rs)：

```rust
pub fn auto_detect_line_ending(doc: &Rope) -> Option<LineEnding> {
    for line in doc.lines().take(100) {
        match get_line_ending(&line) {
            None => {}
            #[cfg(feature = "unicode-lines")]
            Some(LineEnding::VT) | Some(LineEnding::FF) | Some(LineEnding::PS) => {}
            ending => return ending,
        }
    }
    None
}
```

- 只扫描前 100 行，保证性能
- 跳过 VT、FF、PS 等特殊用途行尾（unicode-lines 模式）
- 返回第一个匹配的行尾

### 13.11 编辑路径总览表

| 操作 | 触发方式 | 是否使用 `doc.line_ending.as_str()` | 是否修改已有内容行尾 |
|------|---------|-----------------------------------|---------------------|
| Enter 换行 | 插入模式 Enter | ✅ 是（开头提取到局部变量） | ❌ 否 |
| 下方开新行 | 普通模式 `o` | ✅ 是（直接读取） | ❌ 否 |
| 上方开新行 | 普通模式 `O` | ✅ 是（直接读取） | ❌ 否 |
| 添加纯换行 | `<a-j>` / `<a-k>` | ✅ 是（repeat） | ❌ 否 |
| 粘贴 | `p` / `P` | ✅ 是（LINE_ENDING_REGEX 替换目标） | ❌ 否（只改粘贴内容） |
| 选区替换粘贴 | `R` / `gp` 等 | ✅ 是（LINE_ENDING_REGEX 替换目标） | ❌ 否（只改选区内容） |
| 保存末尾补换行 | 写入命令 | ✅ 是（insert_final_newline） | ❌ 否（只在末尾无行尾时添加） |
| 显式切换行尾 | `:line-ending crlf/lf` | ✅ 是（Transaction 替换源） | ✅ **是**，遍历所有行替换 |
| 新建空白文档 | `:new` | ✅ 是（初始化 Rope 内容） | N/A（空文档） |
| 打开不存在路径的文件 | `:open new.txt` | ✅ 是（初始化 Rope 内容） | N/A（新文档） |
| LSP Snippet 插入 | 自动补全/代码段 | ✅ 是（SnippetRenderCtx） | ❌ 否 |
| 变量扩展 | `$line_ending` | ✅ 是（原样返回） | ❌ 否 |
| 保存到磁盘 | `:w` | ❌ 否（Rope 原样写出） | ❌ 否 |
| 重新加载 | `:reload` | ❌ 否（按磁盘内容） | 🔄 可能（磁盘内容覆盖时） |

### 13.12 行尾处理边界完整总结

**Rope 内容与 doc.line_ending 的关系：**

| 场景 | Rope 中的行尾 | doc.line_ending 元信息 | 一致性 |
|------|-------------|-----------------------|--------|
| 打开已有 LF 文件 | `\n` | `LF`（自动检测） | ✅ 一致 |
| 打开已有 CRLF 文件 | `\r\n` | `Crlf`（自动检测） | ✅ 一致 |
| 新建文档（Native=LF） | `\n` | `LF` | ✅ 一致 |
| 打开 EditorConfig 设为 crlf 的新文件 | `\r\n` | `Crlf`（来自 EditorConfig） | ✅ 一致 |
| `:line-ending lf` 后（原 CRLF 文件） | 所有 `\r\n` → `\n` | `LF` | ✅ 一致 |
| 粘贴 LF 内容到 CRLF 文档 | 粘贴内容中 `\n` → `\r\n` | `Crlf` | ✅ 粘贴部分一致 |
| 混合行尾文件（既有 LF 也有 CRLF） | 保留混合 | 首个检测到的行尾类型 | ❌ 部分不一致 |
| 其他编辑器修改了文件行尾，尚未 reload | 还是旧行尾 | 还是旧元信息 | ✅ 一致（但与磁盘不一致） |

**关键边界原则：**

1. **元信息只驱动新插入**：`doc.line_ending` 只在插入新行尾时生效，从不自动修改已有内容
2. **唯一例外是 :line-ending**：显式切换行尾命令会主动遍历替换所有行的行尾字符
3. **粘贴做输入转换**：外部内容进入时（剪贴板、寄存器、Snippet）做行尾归一化到文档当前行尾
4. **保存不做转换**：Rope 中的行尾原样写入磁盘，不做任何 CRLF ↔ LF 转换
5. **yank 不做转换**：从 Rope 复制内容时，保持 Rope 中的原始行尾（多选区拼接除外）
6. **自动检测只在打开时**：`detect_indent_and_line_ending()` 仅在打开和 reload 时调用

---

## 总结

Helix 的文件 IO 与编码系统设计要点：

1. **三级编码识别**：手动指定 → BOM 检测 → chardetng 统计检测
2. **流式处理**：8KB 双缓冲 + 双层循环，适配大文件，读取与写入结构对称
3. **行尾原样保留**：Rope 中保持原始行尾，打开和保存时都不做归一化
4. **元信息驱动新行尾插入**：`doc.line_ending` 在 Enter、`o`/`O`、粘贴、添加换行、补末尾换行、Snippet 渲染等所有插入新行尾的编辑路径中起作用
5. **粘贴统一行尾**：通过正则 `\r\n|\r|\n` 匹配所有行尾并替换为当前文档行尾
6. **显式切换行尾会改内容**：`:line-ending` 命令除了设置元信息，还会创建 Transaction 替换所有行的行尾字符，可 undo
7. **寄存器跨平台处理**：yank 到剪贴板时多选区用 `NATIVE_LINE_ENDING` 拼接，粘贴回时通过正则统一为文档行尾
8. **编码转换仅在 to_writer()**：Rope (UTF-8) → 目标编码，是唯一的编码转换点
9. **行尾检测函数区分**：`get_line_ending()` 处理 RopeSlice，`get_line_ending_of_str()` 处理普通字符串，`auto_detect_line_ending()` 扫描前 100 行
10. **异步保存**：保存操作返回 Future，不阻塞编辑；通过通道串行化多个保存请求
11. **原子保存**：支持备份和恢复，区分硬链接/符号链接/普通文件
12. **外部修改检测**：基于 mtime 的冲突检测，防止覆盖外部修改
13. **完整的错误处理**：从底层 IO 到 UI 展示的完整错误链路
14. **EditorConfig 集成**：编码、行尾、缩进、空白修剪等可通过 `.editorconfig` 统一管理
