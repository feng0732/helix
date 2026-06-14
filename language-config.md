# Helix 语言配置体系：定义、语法、LSP 与运行时资源

本文梳理 Helix 编辑器中语言配置的完整代码路径，厘清 **语言定义**、**语法加载**、**LSP 配置** 和 **运行时资源** 四者之间的关系。

---

## 1. 总览：四个概念及其代码归属

| 概念 | 核心文件 | 职责 |
|------|----------|------|
| 语言定义 | [languages.toml](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/languages.toml)、[config.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs) | 声明一种语言的身份（name/scope/file-types）、关联的 LSP、缩进规则等 |
| 语法加载 | [grammar.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs)、[syntax.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs) | 编译/加载 tree-sitter 共享库，将 .scm 查询编译为可运行的对象 |
| LSP 配置 | [lib.rs (helix-lsp)](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs) | 根据语言定义中的 `language-servers` 字段启动和管理 LSP 进程 |
| 运行时资源 | [runtime/](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/runtime)、[lib.rs (helix-loader)](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs) | 存放 .scm 查询文件、编译后的 grammar 动态库、主题等，由 `runtime_dirs()` 统一查找 |

---

## 2. 语言定义：从 languages.toml 到 LanguageConfiguration

### 2.1 配置文件结构

[languages.toml](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/languages.toml) 包含三个顶层段：

```toml
use-grammars = { except = ["wren", "gemini"] }   # 控制 grammar 编译范围

[language-server]                                  # LSP 服务器全局定义
rust-analyzer = { command = "rust-analyzer" }

[[language]]                                       # 每种语言一条记录
name = "rust"
scope = "source.rust"
file-types = ["rs"]
roots = ["Cargo.toml", "Cargo.lock"]
language-servers = ["rust-analyzer"]
grammar = "rust"                                   # 可选，默认等于 name
indent = { tab-width = 4, unit = "    " }

[[grammar]]                                        # 每种语法一条记录
name = "rust"
source = { git = "https://github.com/tree-sitter/tree-sitter-rust", rev = "..." }
```

关键对应关系：
- `[[language]]` 的 `name` → [LanguageConfiguration.language_id](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L30)
- `[[language]]` 的 `grammar` 字段 → 决定使用哪个 `[[grammar]]` 的共享库（默认与 `name` 相同）
- `[[language]]` 的 `language-servers` → 引用 `[language-server]` 中定义的服务名
- `[[grammar]]` 的 `name` → grammar 动态库的文件名（如 `rust.so` / `rust.dll`）

### 2.2 配置加载与合并

配置加载的调用链：

```
main.rs
  → helix_core::config::user_lang_loader(insecure)
    → helix_loader::config::user_lang_config(insecure)        # 返回 toml::Value
      → default_lang_config()                                  # 内置 languages.toml（include_bytes!）
      → merge_toml_values(default, user_global, 3)            # 与 ~/.config/helix/languages.toml 合并
      → merge_toml_values(merged, workspace, 3)               # 与 .helix/languages.toml 合并
    → Configuration::try_from(toml::Value)                     # 反序列化为 Configuration
    → Loader::new(Configuration)                               # 构建语法加载器
```

相关代码位置：
- 默认配置读取：[helix-loader/src/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/config.rs#L6) — `include_bytes!("../../languages.toml")`
- 用户配置合并：[helix-loader/src/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/config.rs#L13) — `merge_toml_values(a, b, 3)`
- 合并深度逻辑：[helix-loader/src/lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L207) — 按 `name` 字段匹配同名的 `[[language]]` 条目，深度 3 层递归合并
- 入口调用：[helix-term/src/main.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/main.rs#L133)

### 2.3 Configuration 数据结构

[Configuration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L17) 包含两个核心字段：

```rust
pub struct Configuration {
    pub language: Vec<LanguageConfiguration>,                        // 所有 [[language]]
    pub language_server: HashMap<String, LanguageServerConfiguration>, // 所有 [language-server]
}
```

[LanguageConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L25) 的关键字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `language_id` | `String` | 语言标识，如 "rust"、"c-sharp" |
| `language_server_language_id` | `Option<String>` | 发给 LSP 的 languageId，如 "csharp" |
| `scope` | `String` | TextMate scope，如 "source.rust" |
| `file_types` | `Vec<FileType>` | 文件扩展名或 glob 匹配 |
| `shebangs` | `Vec<String>` | shebang 匹配 |
| `roots` | `RootMarkers` | 项目根标记（Cargo.toml 等） |
| `grammar` | `Option<String>` | 指向 grammar 名，默认 = language_id |
| `language_servers` | `Vec<LanguageServerFeatures>` | 引用 LanguageServerConfiguration 的名称 + 特性过滤 |
| `indent` | `Option<IndentationConfiguration>` | 缩进配置 |

[LanguageServerConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L435) 的关键字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `command` | `String` | LSP 可执行文件路径 |
| `args` | `Vec<String>` | 命令行参数 |
| `config` | `Option<serde_json::Value>` | 初始化选项（initializationOptions） |
| `timeout` | `u64` | 请求超时 |
| `required_root_patterns` | `Option<GlobSet>` | 要求项目根中存在的文件模式 |

---

## 3. 语法加载：grammar 编译与运行时加载

### 3.1 Grammar 编译（构建时）

[helix-loader/src/grammar.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs) 负责将 tree-sitter grammar 源码编译为共享库。

流程：
```
hx --grammar fetch   → fetch_grammars()
  → get_grammar_configs()                # 从 languages.toml 读取 [[grammar]]
  → 对每个 GrammarConfiguration：
      - Git 源 → clone 到 runtime/grammars/sources/<name>/
      - Local 源 → 直接引用本地路径

hx --grammar build   → build_grammars()
  → 对每个 GrammarConfiguration：
      - 编译 parser C 源码
      - 输出到 runtime/grammars/<name>.so (或 .dll / .dylib)
```

[GrammarConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L44)：
```rust
pub struct GrammarConfiguration {
    pub grammar_id: String,            // [[grammar]] name
    pub source: GrammarSource,         // Git { remote, rev, subpath } 或 Local { path }
}
```

`use-grammars` 控制哪些 grammar 参与编译：[GrammarSelection](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L37)

### 3.2 Grammar 运行时加载

[get_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L74) 在运行时动态加载编译好的 grammar 共享库：

```rust
pub fn get_language(name: &str) -> Result<Option<Grammar>> {
    let mut rel_library_path = PathBuf::new().join("grammars").join(name);
    rel_library_path.set_extension(DYLIB_EXTENSION);    // .dll / .so / .dylib
    let library_path = crate::runtime_file(&rel_library_path);  // 在 runtime_dirs 中查找
    if !library_path.exists() { return Ok(None); }
    let grammar = unsafe { Grammar::new(name, &library_path) }?;
    Ok(Some(grammar))
}
```

调用路径：`LanguageData::compile_syntax_config()` → `get_language(parser_name)`

### 3.3 查询文件加载

[read_query()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L268) 读取 .scm 查询文件：

```rust
pub fn read_query(lang: &str, query_filename: &str) -> String {
    tree_house::read_query(lang, |language| {
        helix_loader::grammar::load_runtime_file(language, query_filename).unwrap_or_default()
    })
}
```

[load_runtime_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L718) 查找路径：
```rust
pub fn load_runtime_file(language: &str, filename: &str) -> Result<String, std::io::Error> {
    let path = crate::runtime_file(PathBuf::new().join("queries").join(language).join(filename));
    std::fs::read_to_string(path)
}
```

所以查询文件的实际路径为 `runtime/queries/<language_id>/<filename>.scm`。

查询文件种类与用途：

| 文件 | 用途 | 编译结果 |
|------|------|----------|
| `highlights.scm` | 语法高亮捕获 | `SyntaxConfig` |
| `injections.scm` | 嵌入语言标记 | `SyntaxConfig` |
| `locals.scm` | 局部变量追踪 | `SyntaxConfig` |
| `indents.scm` | 缩进规则 | `IndentQuery` |
| `textobjects.scm` | 文本对象选择 | `TextObjectQuery` |
| `tags.scm` | 符号索引 | `TagQuery` |
| `rainbows.scm` | 彩虹括号 | `RainbowQuery` |

### 3.4 LanguageData：懒加载中枢

[LanguageData](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L40) 是每种语言的运行时数据容器，采用 `OnceCell` 懒加载：

```
LanguageData
  ├── config: Arc<LanguageConfiguration>      # 语言定义（始终存在）
  ├── syntax: OnceCell<Option<SyntaxConfig>>  # 首次访问时编译
  ├── indent_query: OnceCell<Option<...>>     # 依赖 syntax → 再读 indents.scm
  ├── textobject_query: OnceCell<Option<...>> # 依赖 syntax → 再读 textobjects.scm
  ├── tag_query: OnceCell<Option<...>>        # 依赖 syntax → 再读 tags.scm
  └── rainbow_query: OnceCell<Option<...>>    # 依赖 syntax → 再读 rainbows.scm
```

`compile_syntax_config()` 的完整流程（[syntax.rs#L67](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L67)）：

```
1. 取 parser_name = config.grammar.unwrap_or(config.language_id)
2. get_language(parser_name)                          # 加载 grammar 动态库
3. read_query(name, "highlights.scm")                 # 读取高亮查询
4. read_query(name, "injections.scm")                 # 读取注入查询
5. read_query(name, "locals.scm")                     # 读取局部变量查询
6. SyntaxConfig::new(grammar, highlights, injections, locals)  # 编译为可执行查询
7. reconfigure_highlights(&config, &loader.scopes())  # 应用主题 scope 映射
```

---

## 4. LSP 配置：从语言定义到服务器进程

### 4.1 LSP Registry

[Registry](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L581) 持有所有 LSP 客户端实例：

```rust
pub struct Registry {
    inner: SlotMap<LanguageServerId, Arc<Client>>,          // id → 客户端
    inner_by_name: HashMap<LanguageServerName, Vec<Arc<Client>>>,  // 名称 → 客户端列表
    syn_loader: Arc<ArcSwap<helix_core::syntax::Loader>>,  # 引用语法加载器
    pub incoming: SelectAll<...>,                            # 汇聚所有 LSP 消息流
}
```

创建时机：[Editor::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/editor.rs#L1336)

```rust
let language_servers = helix_lsp::Registry::new(syn_loader.clone());
```

### 4.2 LSP 启动流程

当打开文档时，Registry 根据语言配置中的 `language_servers` 列表启动 LSP：

```
Document::detect_language(loader)
  → loader.language_for_filename(path)      # 文件名 → Language id
  → loader.language(lang).config().clone()  # Language → Arc<LanguageConfiguration>

Editor::refresh_language_servers / 打开文档
  → registry.get(language_config, doc_path, root_dirs, enable_snippets)
    → 遍历 language_config.language_servers:
        → 对每个 LanguageServerFeatures { name, only, excluded }:
            → 检查 inner_by_name 中是否已有客户端
            → 若无，调用 start_client(name, ls_config, ...)
              → syn_loader.language_server_configs().get(&name)  # 从 Loader 取 LSP 配置
              → Client::start(command, args, config, env, ...)  # 启动进程
```

[Registry::get()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L709) 关键逻辑：
- 遍历 `LanguageConfiguration.language_servers`（有序列表，优先级从前到后）
- 检查已启动的客户端能否处理该文档（`client.try_add_doc()`）
- 否则启动新客户端
- 支持 `only_features` / `except_features` 过滤各 LSP 提供的能力

### 4.3 语言与 LSP 的配置桥接

`language_servers` 字段在 `LanguageConfiguration` 中是 `Vec<LanguageServerFeatures>`，每条记录包含：
- `name`：对应 `[language-server]` 段的 key
- `only`：该 LSP 仅提供的特性子集
- `excluded`：该 LSP 排除的特性

示例配置中常见的多 LSP 协作：
```toml
[[language]]
name = "typescript"
language-servers = [
  "typescript-language-server",   # 主 LSP：提供所有特性
  { name = "biome-lsp-proxy", except-features = ["format"] },  # 辅助 LSP：仅提供非格式化特性
]
```

---

## 5. 运行时资源：runtime 目录体系

### 5.1 Runtime 目录优先级

[runtime_dirs()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L85) 返回的搜索路径按优先级从高到低：

1. `CARGO_MANIFEST_DIR` 的父目录 + `/runtime`（开发模式）
2. 用户配置目录 + `/runtime`（如 `~/.config/helix/runtime`）
3. `HELIX_RUNTIME` 环境变量
4. `HELIX_DEFAULT_RUNTIME` 编译时变量
5. 可执行文件所在目录 + `/runtime`（发布安装）

所有文件查找都通过 [runtime_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L111) 按此优先级搜索，**高优先级覆盖低优先级**。

### 5.2 Runtime 目录结构

```
runtime/
├── grammars/                         # 编译后的 tree-sitter 共享库
│   ├── rust.dll                      # Grammar 动态库（由 build_grammars 生成）
│   ├── python.dll
│   └── sources/                      # Grammar 源码（由 fetch_grammars 拉取）
│       ├── rust/
│       └── python/
├── queries/                          # .scm 查询文件
│   ├── rust/
│   │   ├── highlights.scm
│   │   ├── injections.scm
│   │   ├── locals.scm
│   │   ├── indents.scm
│   │   ├── textobjects.scm
│   │   └── tags.scm
│   ├── python/
│   └── ...
├── themes/                           # 主题文件
│   ├── onedark.toml
│   └── ...
└── tutor                             # 教程文件
```

### 5.3 用户覆盖机制

用户可在 `~/.config/helix/runtime/` 下放置同名文件来覆盖内置资源：
- `~/.config/helix/runtime/queries/rust/highlights.scm` → 覆盖内置高亮查询
- `~/.config/helix/runtime/grammars/rust.dll` → 覆盖内置 grammar
- `~/.config/helix/languages.toml` → 合并覆盖内置语言配置

---

## 6. 四者关系全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        languages.toml                               │
│                                                                     │
│  [language-server]          [[language]]           [[grammar]]      │
│  ┌──────────────────┐      ┌──────────────────┐   ┌─────────────┐  │
│  │ rust-analyzer =  │      │ name = "rust"     │   │ name="rust" │  │
│  │   command=...    │◄─────│ language-servers= │   │ source=git  │  │
│  │   config=...     │      │   ["rust-analyzer"]│   └──────┬──────┘  │
│  └──────────────────┘      │ grammar = "rust"──┼──────────┘         │
│                             │ scope, file-types │                    │
│                             └────────┬──────────┘                    │
└──────────────────────────────────────┼──────────────────────────────┘
                                       │
                    ┌──────────────────┼───────────────────┐
                    ▼                  ▼                   ▼
        ┌──────────────────┐ ┌─────────────────┐ ┌───────────────────┐
        │ LanguageConfiguration │  Grammar 加载     │  LSP 启动         │
        │ (内存数据结构)    │ │                  │ │                   │
        │ • language_id     │ │ get_language()   │ │ Registry::get()   │
        │ • scope           │ │   ↓              │ │   ↓               │
        │ • file_types      │ │ 加载 .dll/.so    │ │ start_client()    │
        │ • language_servers│ │   ↓              │ │   ↓               │
        │ • grammar         │ │ Grammar 对象     │ │ Client::start()   │
        └───────┬───────────┘ └────────┬─────────┘ └───────────────────┘
                │                      │
                ▼                      ▼
        ┌──────────────────────────────────────────┐
        │        LanguageData (懒加载)              │
        │                                          │
        │  compile_syntax_config()                 │
        │    1. get_language(parser_name) ──────► grammar 动态库
        │    2. read_query(name, "highlights.scm") ► runtime/queries/<name>/
        │    3. read_query(name, "injections.scm")
        │    4. read_query(name, "locals.scm")
        │    5. SyntaxConfig::new(grammar, ...)     ► 编译查询
        │                                          │
        │  后续懒加载：                              │
        │    • indent_query   ← indents.scm        │
        │    • textobject_query ← textobjects.scm  │
        │    • tag_query     ← tags.scm            │
        │    • rainbow_query ← rainbows.scm        │
        └──────────────────────────────────────────┘
```

---

## 7. 端到端流程：打开一个 Rust 文件

1. **启动**：[main.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/main.rs#L133) 调用 `user_lang_loader()`，将 `languages.toml` 反序列化为 `Configuration`，再构建 `Loader`
2. **传入 Editor**：[Application::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/application.rs#L94) 将 `lang_loader` 包装为 `Arc<ArcSwap<Loader>>`，传给 [Editor::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/editor.rs#L1329)
3. **创建 LSP Registry**：`Editor::new()` 用 `syn_loader` 创建 [Registry::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L590)
4. **打开文件**：`editor.open(path)` → `Document::open()`
5. **语言检测**：[Document::detect_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/document.rs#L1195) → `loader.language_for_filename(path)` → 匹配到 `Language(0)` (Rust)
6. **设置语言**：[Document::set_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/document.rs#L1345) → `Syntax::new(source, language, loader)` → 触发 `LanguageData::syntax_config()` 懒加载
7. **语法编译**：`compile_syntax_config()` → 加载 `rust.dll` → 读取 `runtime/queries/rust/*.scm` → 编译 `SyntaxConfig`
8. **启动 LSP**：遍历 `LanguageConfiguration.language_servers` → 找到 `"rust-analyzer"` → `Registry::start_client()` → 从 `syn_loader.language_server_configs()` 取 `LanguageServerConfiguration` → `Client::start("rust-analyzer", ...)` → 启动进程
9. **文档关联**：将 `LanguageServerId` 和 `Document` 关联，开始接收诊断、补全等

---

## 8. grammar 与 language 的映射规则

- **默认映射**：`grammar` 字段未设置时，使用 `language_id` 作为 grammar 名
- **显式映射**：如 protobuf 的配置中 `grammar = "proto"`，说明 `[[language]] name="protobuf"` 使用 `[[grammar]] name="proto"` 的共享库
- **共享 grammar**：多个语言可以共用同一个 grammar（如 JSX/TSX 共享 ecma grammar）
- **查询文件路径**：始终以 `language_id`（而非 grammar 名）为目录名，即 `runtime/queries/<language_id>/`
- **tree_house 回退**：`tree_house::read_query()` 支持通过 injection 等机制回退查找关联语言的查询文件
