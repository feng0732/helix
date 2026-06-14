# Helix 语言配置体系：定义、语法、LSP 与运行时资源

本文梳理 Helix 编辑器中语言配置的完整代码路径，厘清 **语言定义**、**语法加载**、**LSP 配置** 和 **运行时资源** 四者之间的关系。

---

## 1. 总览：四个概念及其代码归属

| 概念 | 核心文件 | 职责 |
|------|----------|------|
| 语言定义 | [languages.toml](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/languages.toml)、[syntax/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs) | 声明一种语言的身份（name/scope/file-types）、关联的 LSP、缩进规则等 |
| 语法加载 | [grammar.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs)、[syntax.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs) | 编译/加载 tree-sitter 共享库，将 .scm 查询编译为可运行的对象 |
| LSP 配置 | [helix-lsp/src/lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs) | 根据语言定义中的 `language-servers` 字段启动和管理 LSP 进程 |
| 运行时资源 | [runtime/](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/runtime)、[helix-loader/src/lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs) | 存放 .scm 查询文件、编译后的 grammar 动态库、主题等，由 `runtime_dirs()` 统一查找 |

---

## 2. 语言定义：从 languages.toml 到 LanguageConfiguration

### 2.1 配置文件结构

[languages.toml](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/languages.toml) 包含四个顶层段：

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

### 2.2 字段引用关系（关键！）

四类资源通过以下字段精确关联：

```
[language-server] 段
  └── key = "rust-analyzer"          ◄───────────┐
                                                │
[[language]] 段                                  │
  ├── name = "rust"                   ──► LanguageConfiguration.language_id
  ├── grammar = "rust" (默认=name)    ──► LanguageConfiguration.grammar
  │                                    ──► 用于 get_language(parser_name) 加载 grammar 库
  │
  └── language-servers = ["rust-analyzer"]
        │
        └── 每一项反序列化为 LanguageServerFeatures
              ├── name = "rust-analyzer"  ──► 用于在 language_server HashMap 中查找
              ├── only: HashSet<Feature>   ──► 仅启用这些特性
              └── excluded: HashSet<Feature> ──► 排除这些特性

[[grammar]] 段
  └── name = "rust"                    ──► GrammarConfiguration.grammar_id
                                         ──► 对应编译后的动态库文件名 rust.dll/rust.so
```

精确字段映射：

| TOML 字段 | Rust 结构体字段 | 用途 |
|-----------|-----------------|------|
| `[[language]].name` | [LanguageConfiguration.language_id](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L30) | 语言唯一标识 |
| `[[language]].language-id` | [LanguageConfiguration.language_server_language_id](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L33) | 发送给 LSP 的 `textDocument/didOpen` 中的 languageId |
| `[[language]].grammar` | [LanguageConfiguration.grammar](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L70) | 指向 `[[grammar]].name`，默认等于 `language_id` |
| `[[language]].language-servers[i]`（简写字符串） | [LanguageServerFeatures.name](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L372) | 对应 `[language-server]` 的 key |
| `[[language]].language-servers[i].name`（表形式） | [LanguageServerFeatures.name](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L372) | 同上 |
| `[[language]].language-servers[i].only-features` | [LanguageServerFeatures.only](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L373) | 特性白名单 |
| `[[language]].language-servers[i].except-features` | [LanguageServerFeatures.excluded](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L374) | 特性黑名单 |
| `[language-server].<key>` | [Configuration.language_server](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L20) 的 HashMap key | LSP 全局配置 |
| `[[grammar]].name` | [GrammarConfiguration.grammar_id](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L46) | grammar 动态库标识 |
| `use-grammars` | [GrammarSelection](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L37) | grammar 编译白/黑名单 |

### 2.3 配置加载与合并：工作区信任机制

配置加载的完整流程（包含工作区信任检查）：

```
helix-term/src/main.rs
  └── helix_core::config::user_lang_loader(insecure)
        │
        ├── helix_loader::config::user_lang_config(insecure)
        │     │
        │     ├── 1. 取文件路径
        │     │     ├── global_config = config_dir()/languages.toml
        │     │     │                          (~/.config/helix/languages.toml)
        │     │     └── workspace_config = find_workspace().0/.helix/languages.toml
        │     │
        │     ├── 2. 关键：工作区信任检查 [config.rs#L17]
        │     │     │
        │     │     └── quick_query_workspace(insecure) [workspace_trust.rs#L142]
        │     │           │
        │     │           ├── 如果 insecure = true → TrustStatus::Trusted
        │     │           ├── 否则读取 data_dir()/trusted_workspaces
        │     │           │   逐行匹配当前 workspace 路径
        │     │           ├── 匹配成功 → TrustStatus::Trusted
        │     │           └── 未匹配 → TrustStatus::Untrusted
        │     │
        │     ├── 3. 确定参与合并的文件列表
        │     │     ├── Trusted → [global_config, workspace_config]
        │     │     └── Untrusted → [global_config] （不读取工作区配置！）
        │     │
        │     ├── 4. 读取并解析这些文件
        │     │     ├── 不存在的文件会被过滤掉（filter_map + ok()）
        │     │     └── 全部反序列化为 toml::Value
        │     │
        │     └── 5. 从内置默认配置开始，按优先级折叠合并
        │           ├── 左值（低优先级） = default_lang_config()
        │           │     （通过 include_bytes! 编译进二进制）
        │           ├── 然后依次合并用户全局 → 工作区
        │           └── 使用 merge_toml_values(left, right, 3) 递归合并
        │
        ├── Configuration::try_from(toml::Value)
        │     └── 反序列化为强类型结构体
        │
        └── Loader::new(Configuration)
              └── 构建语法加载器索引
```

**合并算法细节** [merge_toml_values](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L207)：
- 深度参数为 3：`[[language]]` 和 `[language-server]` 下两层字段会递归合并，更深层直接覆盖
- 数组合并通过 `name` 字段匹配：同名条目深度减 1 递归合并，不同名条目追加
- 表合并：key 相同则深度减 1 递归合并，key 不同则直接插入
- 其他类型（字符串、数字、布尔）：右值直接覆盖左值

**信任状态如何影响 LSP 自动启动**：
1. 启动时 `user_lang_loader(insecure)` 决定是否合并工作区 `languages.toml`
2. 打开文档后，如果没有 LSP 自动启动，`DocumentDidOpen` hook 会触发 [prompt](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/handlers/workspace_trust.rs#L36)
3. 用户选择 "AllowAlways" 后，路径写入 `data_dir()/trusted_workspaces`
4. 下次启动时 `quick_query_workspace()` 返回 Trusted，工作区配置生效，LSP 自动启动

**信任相关文件路径**：
- 信任列表：[workspace_trust_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L167) → `data_dir()/trusted_workspaces`
- 排除列表：[workspace_exclude_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L171) → `data_dir()/excluded_workspaces`
- 工作区配置：[workspace_lang_config_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L155) → `<workspace>/.helix/languages.toml`
- 全局配置：[lang_config_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L159) → `config_dir()/languages.toml`

### 2.4 Configuration 数据结构

[Configuration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L17) 是顶层反序列化目标：

```rust
pub struct Configuration {
    pub language: Vec<LanguageConfiguration>,                        // 所有 [[language]]
    pub language_server: HashMap<String, LanguageServerConfiguration>, // 所有 [language-server]
}
```

[LanguageConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L25) 关键字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `language_id` | `String` | 语言标识，如 "rust"、"c-sharp" |
| `language_server_language_id` | `Option<String>` | 发给 LSP 的 languageId，如 "csharp" |
| `scope` | `String` | TextMate scope，如 "source.rust" |
| `file_types` | `Vec<FileType>` | 文件扩展名或 glob 匹配 |
| `shebangs` | `Vec<String>` | shebang 匹配 |
| `roots` | `RootMarkers` | 项目根标记（Cargo.toml 等） |
| `grammar` | `Option<String>` | 指向 grammar 名，默认 = language_id |
| `language_servers` | `Vec<LanguageServerFeatures>` | 引用 LSP 名称 + 特性过滤 |
| `indent` | `Option<IndentationConfiguration>` | 缩进配置 |
| `workspace_lsp_roots` | `Option<Vec<PathBuf>>` | 硬编码的 LSP 根目录 |

[LanguageServerConfiguration](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L435) 关键字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `command` | `String` | LSP 可执行文件路径 |
| `args` | `Vec<String>` | 命令行参数 |
| `config` | `Option<serde_json::Value>` | initializationOptions |
| `timeout` | `u64` | 请求超时（默认 60 秒） |
| `environment` | `HashMap<String, String>` | 环境变量 |
| `required_root_patterns` | `Option<GlobSet>` | 要求项目根中存在的文件模式 |

---

## 3. 语法加载：grammar 编译与运行时加载

### 3.1 Grammar 配置解析

[helix-loader/src/grammar.rs](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs) 中定义了独立的 `Configuration`（仅用于 grammar 处理）：

```rust
struct Configuration {
    #[serde(rename = "use-grammars")]
    pub grammar_selection: Option<GrammarSelection>,  // Only/Except
    pub grammar: Vec<GrammarConfiguration>,            // [[grammar]] 列表
}

pub struct GrammarConfiguration {
    pub grammar_id: String,        // [[grammar]].name
    pub source: GrammarSource,     // Git { remote, rev, subpath } 或 Local { path }
}
```

**注意**：这里的 `Configuration` 和 `helix-core` 中的 `Configuration` 是两个不同的结构体，分别从同一个 `languages.toml` 中反序列化出各自需要的字段。

### 3.2 Grammar 编译（构建时）

流程：
```
hx --grammar fetch   → fetch_grammars()
  → get_grammar_configs()                          # 调用 user_lang_config(false) 解析
  → 按 grammar_selection 过滤                      # 应用 use-grammars 规则
  → 对每个 GrammarConfiguration：
      - Git 源 → clone 到 runtime/grammars/sources/<grammar_id>/
      - Local 源 → 直接引用本地路径

hx --grammar build   → build_grammars()
  → 对每个 GrammarConfiguration：
      - 编译 parser C 源码为共享库
      - 输出到 runtime/grammars/<grammar_id>.so/.dll/.dylib
```

关键点：`get_grammar_configs()` 调用 `user_lang_config(false)` 时 `insecure=false`，意味着**编译 grammar 时也遵循工作区信任机制**，不受信任的工作区不会贡献 `[[grammar]]` 条目。

### 3.3 Grammar 运行时加载

[get_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L74) 在运行时动态加载编译好的 grammar 共享库：

```rust
pub fn get_language(name: &str) -> Result<Option<Grammar>> {
    // 构造相对路径: grammars/<name>.dll/.so/.dylib
    let mut rel_library_path = PathBuf::new().join("grammars").join(name);
    rel_library_path.set_extension(DYLIB_EXTENSION);
    
    // 在 runtime_dirs() 中按优先级查找
    let library_path = crate::runtime_file(&rel_library_path);
    
    if !library_path.exists() { return Ok(None); }
    
    // 动态加载共享库，调用 tree_sitter_<name> 函数
    let grammar = unsafe { Grammar::new(name, &library_path) }?;
    Ok(Some(grammar))
}
```

调用路径：`LanguageData::compile_syntax_config()` → `get_language(parser_name)`

### 3.4 查询文件加载

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
    let path = crate::runtime_file(
        PathBuf::new().join("queries").join(language).join(filename)
    );
    std::fs::read_to_string(path)
}
```

**重要**：查询文件路径始终使用 `language_id`（来自 `LanguageConfiguration.language_id`），而非 `grammar` 字段。

查询文件种类与用途：

| 文件 | 编译时机 | 编译函数 | 结果类型 |
|------|----------|----------|----------|
| `highlights.scm` | 首次访问语法时 | `compile_syntax_config()` | `SyntaxConfig` |
| `injections.scm` | 首次访问语法时 | `compile_syntax_config()` | `SyntaxConfig` |
| `locals.scm` | 首次访问语法时 | `compile_syntax_config()` | `SyntaxConfig` |
| `indents.scm` | 首次调用 indent_query 时 | `compile_indent_query()` | `IndentQuery` |
| `textobjects.scm` | 首次调用 textobject_query 时 | `compile_textobject_query()` | `TextObjectQuery` |
| `tags.scm` | 首次调用 tag_query 时 | `compile_tag_query()` | `TagQuery` |
| `rainbows.scm` | 首次调用 rainbow_query 时 | `compile_rainbow_query()` | `RainbowQuery` |

### 3.5 LanguageData：懒加载中枢

[LanguageData](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L40) 是每种语言的运行时数据容器，采用 `OnceCell` 懒加载：

```
LanguageData
  ├── config: Arc<LanguageConfiguration>          # 始终存在
  ├── syntax: OnceCell<Option<SyntaxConfig>>      # 依赖: get_language(grammar) + 3 个 .scm
  ├── indent_query: OnceCell<Option<IndentQuery>> # 依赖: syntax → Grammar + indents.scm
  ├── textobject_query: OnceCell<Option<...>>      # 依赖: syntax → Grammar + textobjects.scm
  ├── tag_query: OnceCell<Option<...>>             # 依赖: syntax → Grammar + tags.scm
  └── rainbow_query: OnceCell<Option<...>>         # 依赖: syntax → Grammar + rainbows.scm
```

`compile_syntax_config()` 的完整流程 [syntax.rs#L67](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L67)：

```
1. let parser_name = config.grammar.as_deref().unwrap_or(&config.language_id)
   ↑ 这里是 language → grammar 关联的核心代码
2. get_language(parser_name)                          # 加载 grammar 动态库
3. read_query(config.language_id, "highlights.scm")   # 用 language_id 查找查询文件
4. read_query(config.language_id, "injections.scm")
5. read_query(config.language_id, "locals.scm")
6. SyntaxConfig::new(grammar, highlights, injections, locals)
7. reconfigure_highlights(&config, &loader.scopes())  # 应用主题 scope 映射
```

---

## 4. LSP 配置：从语言定义到服务器进程

### 4.1 LSP Registry

[Registry](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L581) 持有所有 LSP 客户端实例，并持有 `syn_loader` 引用以查询配置：

```rust
pub struct Registry {
    inner: SlotMap<LanguageServerId, Arc<Client>>,                // id → 客户端
    inner_by_name: HashMap<LanguageServerName, Vec<Arc<Client>>>, // 名称 → 客户端列表
    syn_loader: Arc<ArcSwap<helix_core::syntax::Loader>>,         // 语法加载器引用
    pub incoming: SelectAll<UnboundedReceiverStream<...>>,         // 汇聚所有 LSP 消息流
    pub file_event_handler: file_event::Handler,                   // 文件事件处理
}
```

创建时机：[Editor::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/editor.rs#L1336)
```rust
let language_servers = helix_lsp::Registry::new(syn_loader.clone());
```

### 4.2 LSP 启动流程：精确字段传递

当打开文档时，字段传递路径如下：

```
Document::detect_language(loader)
  → loader.language_for_filename(path)            # 文件扩展名 → Language id (u32)
  → loader.language(lang).config().clone()        # Language id → Arc<LanguageConfiguration>

Editor::refresh_language_servers
  → registry.get(
        language_config,        // &LanguageConfiguration
        doc_path,               // Option<&Path>
        root_dirs,              // &[PathBuf]
        enable_snippets         // bool
    )
    │
    ├── 遍历 language_config.language_servers: Vec<LanguageServerFeatures>
    │     │
    │     ├── 对每个 LanguageServerFeatures { name, only, excluded }:
    │     │     │
    │     │     ├── 检查 self.inner_by_name.get(name) 是否已启动
    │     │     │
    │     │     └── 未启动 → self.start_client(name, language_config, ...)
    │     │           │
    │     │           ├── let syn_loader = self.syn_loader.load()
    │     │           │
    │     │           ├── let config = syn_loader
    │     │           │              .language_server_configs()  # HashMap<String, _>
    │     │           │              .get(&name)                # 用 name 查找
    │     │           │
    │     │           └── start_client(id, name, language_config,
    │     │                                   config,            # LanguageServerConfiguration
    │     │                                   ...)
    │     │                 │
    │     │                 ├── config.command / config.args     # 进程启动参数
    │     │                 ├── config.config                   # initializationOptions
    │     │                 ├── config.timeout                  # 请求超时
    │     │                 └── config.environment              # 环境变量
    │     │
    │     └── client.try_add_doc(
    │           &language_config.roots,       # 来自 [[language]].roots
    │           language_config.workspace_lsp_roots.as_deref()...,
    │           doc_path, ...
    │       )
    │
    └── 返回 (name, Result<Arc<Client>>) 迭代器
```

**[Registry::start_client()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L620)** 中关键的字段桥接：

```rust
fn start_client(
    &mut self,
    name: String,                          // 来自 LanguageServerFeatures.name
    ls_config: &LanguageConfiguration,     // [[language]] 配置
    doc_path: Option<&Path>,
    root_dirs: &[PathBuf],
    enable_snippets: bool,
) -> Result<Arc<Client>, StartupError> {
    let syn_loader = self.syn_loader.load();
    
    // 关键：用 name 在 syn_loader 中查找 LanguageServerConfiguration
    let config = syn_loader
        .language_server_configs()
        .get(&name)
        .ok_or_else(|| anyhow!("Language server '{name}' not defined"))?;
    
    // config 包含 command, args, config, timeout, environment
    // ls_config 包含 roots, workspace_lsp_roots, language_server_language_id
    // 两者配合启动客户端
}
```

### 4.3 特性过滤

[LanguageServerFeatures::has_feature()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L378)：
```rust
pub fn has_feature(&self, feature: LanguageServerFeature) -> bool {
    // only 为空表示不限制，否则必须包含
    (self.only.is_empty() || self.only.contains(&feature)) 
    // excluded 中的特性总是排除
    && !self.excluded.contains(&feature)
}
```

---

## 5. 运行时资源：runtime 目录体系

### 5.1 Runtime 目录优先级

[runtime_dirs()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L85) 返回的搜索路径按优先级从高到低：

1. `CARGO_MANIFEST_DIR` 父目录 + `/runtime`（开发模式，cargo run 时生效）
2. 用户配置目录 + `/runtime`（如 `~/.config/helix/runtime`）
3. `HELIX_RUNTIME` 环境变量指定的路径
4. `HELIX_DEFAULT_RUNTIME` 编译时变量（packager 使用）
5. 可执行文件所在目录 + `/runtime`（发布安装时）

[runtime_file()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/lib.rs#L111) 按优先级查找第一个存在的文件，**高优先级覆盖低优先级**。

### 5.2 Runtime 目录结构与资源对应关系

```
runtime/
├── grammars/
│   ├── <grammar_id>.dll/so/dylib       # 由 [[grammar]].name 决定文件名
│   │                                      # 由 get_language(name) 加载
│   └── sources/
│       └── <grammar_id>/                # 源码检出目录
└── queries/
    └── <language_id>/                   # 由 [[language]].name 决定目录名
        ├── highlights.scm               # 由 read_query(name, "highlights.scm") 读取
        ├── injections.scm               # 由 read_query(name, "injections.scm") 读取
        ├── locals.scm                   # 由 read_query(name, "locals.scm") 读取
        ├── indents.scm                  # 由 read_query(name, "indents.scm") 读取
        ├── textobjects.scm              # 由 read_query(name, "textobjects.scm") 读取
        ├── tags.scm                     # 由 read_query(name, "tags.scm") 读取
        └── rainbows.scm                 # 由 read_query(name, "rainbows.scm") 读取
```

**关键对应**：
- `get_language(name)` 中的 `name` 来自 `LanguageConfiguration.grammar`（默认 = language_id）
- `read_query(lang, filename)` 中的 `lang` 始终是 `LanguageConfiguration.language_id`
- 这意味着多个语言可以共享同一个 grammar 动态库，但各自使用独立的查询文件目录

---

## 6. 四者关系全景图

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                           languages.toml (配置源)                                  │
│                                                                                     │
│  [language-server]              [[language]]                  [[grammar]]            │
│  ┌─────────────────────┐       ┌──────────────────────┐       ┌──────────────────┐ │
│  │ rust-analyzer =     │       │ name = "rust"        │       │ name = "rust"     │ │
│  │   command="ra"      │◄──────│ language-servers =   │       │ source = {git..}  │ │
│  │   config={..}       │       │   ["rust-analyzer"] │       └─────────┬──────────┘ │
│  └──────────┬──────────┘       │ grammar = "rust" ────┼─────────────────────────────┘
│             │                  │ scope/file-types    │                             │
│             │                  └──────────┬───────────┘                             │
└─────────────┼──────────────────────────────┼─────────────────────────────────────────┘
              │                              │
              ▼                              ▼
┌───────────────────────────┐  ┌───────────────────────────┐
│ LanguageServerConfiguration│  │  LanguageConfiguration    │
│ (HashMap value)           │  │  • language_id: "rust"    │
│  • command                │  │  • grammar: "rust"        │◄───┐
│  • args                   │  │  • language_servers: [    │    │
│  • config                 │  │      {name:"rust-analyzer"│    │
│  • timeout                │  │       only: [],           │    │
│  • environment            │  │       excluded: [] } ]    │    │
└─────────────┬─────────────┘  └─────────────┬─────────────┘    │
              │                              │                  │
              │                              ▼                  │
              │                  ┌───────────────────────────┐  │
              │                  │      LanguageData         │  │
              │                  │  (每个语言实例，懒加载)    │  │
              │                  │                           │  │
              │                  │  compile_syntax_config() │  │
              │                  │    1. parser_name =      │  │
              │                  │       config.grammar      │──┘
              │                  │       (unwrap_or "rust")  │
              │                  │    2. get_language(      │
              │                  │         parser_name)      │
              │                  │       ↓ 加载 rust.dll    │
              │                  │    3. read_query(        │
              │                  │         config.language_id│
              │                  │         "highlights.scm") │
              │                  │       ↓ 读 runtime/       │
              │                  │          queries/rust/    │
              │                  │    4. SyntaxConfig::new() │
              │                  └───────────────────────────┘
              │
              ▼
    ┌───────────────────────────┐
    │    LSP Registry::get()    │
    │  遍历 language_servers:   │
    │    [ {name: "ra", ...} ]  │
    │      │                    │
    │      ├── syn_loader       │
    │      │   .language_server_configs()
    │      │   .get("ra")       │
    │      │   → LSP config     │
    │      │                    │
    │      └── start_client(    │
    │            command,args   │
    │          )                │
    └───────────────────────────┘
```

---

## 7. 端到端流程：打开一个 Rust 文件

1. **启动加载配置**：[helix-term/src/main.rs#L133](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/main.rs#L133)
   ```
   user_lang_loader(insecure)
     → quick_query_workspace(insecure) → Trusted/Untrusted
     → 合并 default + global ( + workspace if Trusted )
     → Configuration::try_from() → Loader::new()
   ```

2. **创建 Editor 和 LSP Registry**：[Editor::new()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/editor.rs#L1329)
   ```
   Registry::new(syn_loader.clone())
   ```

3. **打开文件并检测语言**：[Document::detect_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/document.rs#L1195)
   ```
   loader.language_for_filename("main.rs") → Language(0)
   → loader.language(Language(0)).config() → Arc<LanguageConfiguration>
   ```

4. **语法懒加载**：[Document::set_language()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-view/src/document.rs#L1345)
   ```
   Syntax::new(source, Language(0), loader)
     → LanguageData::syntax_config() 首次触发编译
       → get_language("rust") → 加载 rust.dll
       → read_query("rust", "highlights.scm")
       → SyntaxConfig::new(...)
   ```

5. **工作区信任提示（如未信任）**：[DocumentDidOpen hook](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/handlers/workspace_trust.rs#L19)
   ```
   如果 doc.language_servers().next().is_none()
     → quick_query_workspace_with_explicit_untrust() → DenyOnce
     → 弹出信任提示框
   ```

6. **LSP 启动**：[Registry::get()](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L709)
   ```
   遍历 language_config.language_servers:
     → name = "rust-analyzer"
     → syn_loader.language_server_configs().get("rust-analyzer")
       → LanguageServerConfiguration { command: "rust-analyzer", ... }
     → Client::start("rust-analyzer", args, config, ...)
     → 启动进程，发送 initialize 请求
   ```

---

## 8. 字段引用速查表

| 来源字段 | 目标字段 | 代码位置 |
|----------|----------|----------|
| `[[language]].name` | `LanguageConfiguration.language_id` | [syntax/config.rs#L30](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L30) |
| `[[language]].grammar` | `LanguageConfiguration.grammar` | [syntax/config.rs#L70](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L70) |
| `[[language]].grammar` | `get_language(parser_name)` 参数 | [syntax.rs#L72](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax.rs#L72) |
| `[[language]].language-servers[i]` | `LanguageServerFeatures.name` | [syntax/config.rs#L372](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L372) |
| `LanguageServerFeatures.name` | `language_server_configs().get(name)` key | [helix-lsp/src/lib.rs#L631](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-lsp/src/lib.rs#L631) |
| `[language-server].<key>` | `Configuration.language_server` key | [syntax/config.rs#L20](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-core/src/syntax/config.rs#L20) |
| `[[grammar]].name` | `GrammarConfiguration.grammar_id` | [grammar.rs#L46](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L46) |
| `GrammarConfiguration.grammar_id` | 动态库文件名 `grammars/<id>.dll` | [grammar.rs#L75](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L75) |
| `LanguageConfiguration.language_id` | 查询目录 `queries/<id>/` | [grammar.rs#L719](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/grammar.rs#L719) |

| 条件逻辑 | 代码位置 |
|----------|----------|
| 工作区信任检查（决定是否读 workspace languages.toml） | [config.rs#L17](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/config.rs#L17) |
| insecure 模式绕过信任 | [workspace_trust.rs#L143](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-loader/src/workspace_trust.rs#L143) |
| 打开文档后检查信任并提示 | [workspace_trust.rs#L24](file:///d:/fz/0601/solo-dogfeeding/code/282-helix/helix-term/src/handlers/workspace_trust.rs#L24) |
