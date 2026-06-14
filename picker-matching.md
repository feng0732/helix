# Picker 模糊匹配流程详解

## 概述

Picker 是 Helix 编辑器中的核心组件，用于实现文件选择、符号搜索、命令面板等功能。其模糊匹配系统基于 `nucleo` 库实现，包含四个核心流程：**候选收集**、**匹配打分**、**预览更新**和**交互选择**。

---

## 1. 候选收集流程

### 核心数据结构

**[Picker\<T, D\>](helix-term/src/ui/picker.rs#L241-L272)** 是主要组件结构体，包含：
- `matcher: Nucleo<T>` - 基于 nucleo 的匹配引擎
- `columns: Arc<[Column<T, D>]>` - 列定义，支持多列匹配
- `editor_data: Arc<D>` - 编辑器上下文数据
- `version: Arc<AtomicUsize>` - 版本号，用于取消异步任务

**[Column\<T, D\>](helix-term/src/ui/picker.rs#L189-L197)** 定义了每列的属性：
- `name: Arc<str>` - 列名，用于查询语法 `%column_name`
- `filter: bool` - 是否参与模糊匹配过滤
- `format: ColumnFormatFn<T, D>` - 格式化函数

### 收集方式

#### 1.1 静态收集（同步）

通过 [Picker::new()](helix-term/src/ui/picker.rs#L299-L335) 一次性注入所有候选项：

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

通过 [Picker::stream()](helix-term/src/ui/picker.rs#L275-L297) 支持异步增量注入：

```rust
pub fn stream(
    columns: impl IntoIterator<Item = Column<T, D>>,
    editor_data: D,
) -> (Nucleo<T>, Injector<T, D>) {
    // 创建 Nucleo 匹配器和 Injector
    // 返回给调用者，调用者可以异步推送候选项
}
```

**[Injector\<T, D\>](helix-term/src/ui/picker.rs#L145-L157)** 是线程安全的注入器：
- 内部包装 `nucleo::Injector<T>`
- 通过 `version` 机制防止已过期的注入
- `Drop` 时自动请求重绘，清除 "running" 指示器

#### 1.3 动态查询收集

通过 [Picker::with_dynamic_query()](helix-term/src/ui/picker.rs#L437-L451) 支持根据查询动态生成候选项（如全局搜索）：

- 使用 `DynamicQueryHandler` 进行防抖处理（默认 100ms）
- 查询变化时通过 `version.fetch_add(1)` 取消之前的请求
- 粘贴操作立即执行，不防抖

### 注入逻辑

**[inject_nucleo_item()](helix-term/src/ui/picker.rs#L132-L143)** 是核心注入函数：

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
- 闭包在 `push` 调用时**同步执行**（在调用者线程），将格式化好的列文本写入 boxcar::Vec 的对应 slot
- `dst` 是 boxcar::Vec 提供的 `Utf32String` 缓冲区，每个可过滤列对应一个 slot
- Worker 线程只读已写入的 `matcher_columns`，不做格式化

---

## 2. 匹配打分流程

### 查询解析

**[PickerQuery](helix-term/src/ui/picker/query.rs#L4-L18)** 负责解析用户输入的查询字符串，支持以下语法：

| 语法示例 | 说明 |
|---------|------|
| `hello world` | 主列匹配 "hello world" |
| `hello %field1 world` | 主列匹配 "hello"，field1 列匹配 "world" |
| `hello \%field1` | 转义 `%`，主列匹配 "hello %field1" |
| `%f abc` | 前缀匹配，自动选择最短匹配的列名 |

**[parse()](helix-term/src/ui/picker/query.rs#L46-L141)** 方法解析流程：
1. 遍历输入字符串的每个字符
2. 遇到 `%` 时切换到字段解析模式
3. 遇到空格时结束字段名解析，开始字段值输入
4. 支持 `\` 转义特殊字符
5. 相同列的多次使用用空格连接

### 查询变更处理

**[handle_prompt_change()](helix-term/src/ui/picker.rs#L534-L581)** 处理查询变化：

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

2. **匹配过程**（由 `tick` 派发到 rayon 线程池异步执行，详见第 3.2 节）：
   - 对每个候选项的每个可过滤列执行模糊匹配
   - 计算匹配得分：底层 `nucleo_matcher::Matcher` 的单列匹配返回 `Option<u16>`（0-65535）；高层 `nucleo::MultiPattern::score()` 将各列 `u16` 累加为 `u32`，存入 `Match { score: u32, idx: u32 }`
   - 按得分降序排列结果（同分 tie breaker 见第 3.3 节）
   - 多列匹配时，所有 filter=true 的列都必须匹配成功（任一列返回 `None` 即排除）

3. **匹配高亮**：
   在 [render_picker()](helix-term/src/ui/picker.rs#L760-L835) 中获取匹配索引并高亮：

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

## 3. 候选进入匹配器后的刷新、排序与高亮全流程

> 本节深入分析候选从注入 nucleo 到最终在 UI 上高亮显示、选中项同步的完整链路。

### 3.1 候选注入与 Nucleo 内部管线

Nucleo 匹配器内部不是"无锁队列消费"模型，而是**共享 append-only vector + 游标增量读取**模型：

```
调用方（主线程/异步任务）
    │  Injector::push(item, fill_fn)
    │  → 同步执行 fill_fn：格式化列文本写入 slot
    │  → 追加到 boxcar::Vec<T>（共享）
    │  → notify() [路径 A：候选追加触发重绘]
    ▼
Arc<boxcar::Vec<T>>      ← Injector / Worker / Snapshot 三方共享
    │  Worker.last_snapshot 游标追踪已处理位置
    │  process_new_items() 增量读取自 last_snapshot 以来的新项
    ▼
rayon 线程池（异步）
    │  Worker::run(status, cleared)
    │  → 增量扫描 boxcar::Vec，对新项执行模糊匹配
    │  → 对已有项若 pattern 变更则重新打分
    │  → par_quicksort 并行排序
    │  → notify() [路径 B：匹配完成触发重绘，若 should_notify=true]
    ▼
Worker.matches 结果向量
    │  下一帧 tick 获取锁时 → snapshot.update()
    ▼
Snapshot（只读，暴露给 UI）
```

核心数据结构 `boxcar::Vec<T>` 是一个**append-only、并发安全、分块存储**的 vector：
- **不会被消费**：注入的 item 永久保存在 vector 中，只是 Worker 用 `last_snapshot: u32` 游标记录上次读到了哪里
- **支持无锁等待写入**：如果某个索引暂时未初始化（写入还在进行），Worker 会把它放进 `in_flight: Vec<u32>`，下次 `run()` 时再检查
- **三方共享**：`Injector`、`Worker`、`Snapshot` 都持有同一个 `Arc<boxcar::Vec<T>>`，通过索引 `idx` 引用原始数据

#### Injector::push() 的实际行为（`nucleo/src/lib.rs`）

```rust
pub fn push(&self, value: T, fill_columns: impl FnOnce(&T, &mut [Utf32String])) -> u32 {
    let idx = self.items.push(value, fill_columns);  // 追加到 boxcar::Vec
    (self.notify)();                                  // 立即触发一次重绘通知
    idx
}
```

1. `self.items.push()` 把 `T` 和列文本追加到共享 vector：先分配索引 slot，再在**当前调用线程中同步执行** `fill_fn` 闭包（即 [inject_nucleo_item()](helix-term/src/ui/picker.rs#L132-L143) 中的 `|item, dst| { ... }`），将格式化好的列文本写入 `dst` 缓冲区，最后返回分配的索引 `idx`。
2. **push 之后立刻调用 `notify()`**，通知 UI 有新候选项。这意味着批量注入场景（如文件扫描、动态查询）每追加一批就会触发一次重绘请求，即使匹配还没开始。

> **注意**：普通用户在查询框中输入字符（pattern 变化）**不会**走这条路径触发重绘——那条路径见第 3.2 节的 tick 派发机制。

#### 重绘通知 `notify` 的两条触发路径

Picker 创建时传给 `Nucleo::new()` 的 notify 回调是 `Arc::new(helix_event::request_redraw)`（见 [picker.rs#L282-L287](helix-term/src/ui/picker.rs#L282-L287) 和 [picker.rs#L317-L322](helix-term/src/ui/picker.rs#L317-L322)）。它在**两类共两个时机**被调用：

**路径 A：候选追加触发（Injector 侧）**
- `Injector::push()` / `extend()`：每次追加候选后立即触发
- 适用场景：动态查询、文件扫描等候选在变化的场景
- 特点：不等匹配开始，push 完立刻通知

**路径 B：匹配完成触发（Worker 侧）**
- `Worker::run()` 末尾：匹配和排序完成后，若 `should_notify` 原子标志为 `true`，再触发一次
- 适用场景：**普通查询输入**（pattern 变化 → 下一帧 tick 派发 Worker → Worker 跑完通知）、候选追加后的首次匹配完成
- 特点：在 rayon 线程池异步完成后触发

`request_redraw()` 的实现（[helix-event/src/redraw.rs#L28-L30](helix-event/src/redraw.rs#L28-L30)）非常轻量：
```rust
pub fn request_redraw() {
    REDRAW_NOTIFY.notify_one();  // 唤醒 tokio 等待的渲染循环
}
```
它不会立即重绘，只是向 tokio 的 `Notify` 发送一次通知。Helix 主循环会以**30 FPS 去抖**速率合并这些通知，保证实际重绘不会过于频繁。

> **普通查询输入的重绘时序**：用户键入字符 → prompt 内容变化（当前帧已渲染，由 UI 事件循环驱动）→ `handle_prompt_change()` 调用 `matcher.pattern.reparse()` → 下一帧 `tick()` 检测到 pattern 变化、派发 Worker → Worker 在 rayon 线程池完成匹配排序 → `should_notify=true` 时调用 `notify()` → 再下一帧 `tick()` 拿到新结果、更新 snapshot → 界面显示新匹配列表。也就是说，用户输入一个字符后，**至少经过两帧**才能看到新的匹配结果（前提是匹配足够快）。

### 3.2 tick — 协调 snapshot 与异步 worker

每次 [render_picker()](helix-term/src/ui/picker.rs#L683-L880) 被调用时，首先执行 tick：

```rust
fn render_picker(&mut self, area: Rect, surface: &mut Surface, cx: &mut Context) {
    let status = self.matcher.tick(10);   // 协调 worker，最多等 10ms 拿锁
    let snapshot = self.matcher.snapshot(); // 只读快照
    // ...
}
```

> **重要澄清**：**tick 本身不执行匹配**。匹配完全在 rayon 线程池异步进行。tick 的职责只是**协调**——检查 worker 是否跑完、把结果同步到 snapshot、必要时把 worker 再次派出去。

#### tick 内部流程（`nucleo/src/lib.rs` → `tick_inner`）

```rust
fn tick_inner(&mut self, timeout: u64, canceled: bool, status: pattern::Status) -> Status {
    // 1. 尝试获取 Worker 锁：canceled=true 时立即阻塞获取；
    //    否则最多等 timeout 毫秒（Picker 传 10ms）
    let mut inner = if canceled {
        self.canceled.store(true, Ordering::Relaxed);
        self.worker.lock_arc()
    } else {
        let Some(worker) = self.worker.try_lock_arc_for(Duration::from_millis(timeout)) else {
            // 拿不到锁 → worker 还在跑，本帧没有新结果
            self.should_notify.store(true, Ordering::Release);
            return Status { changed: false, running: true };
        };
        worker
    };

    // 2. 若上一轮 worker 已经跑完（inner.running=true），把结果同步到 snapshot
    let changed = inner.running;  // changed=true 意味着 snapshot 会更新
    let running = canceled || self.items.count() > inner.item_count();
    if inner.running {
        inner.running = false;
        if !inner.was_canceled && !self.state.canceled() {
            self.snapshot.update(&inner);  // ← snapshot 在这里才会更新
        }
    }

    // 3. 若还有未处理的新项或 pattern 变更，把 worker 派到线程池
    if running {
        inner.pattern.clone_from(&self.pattern);
        self.canceled.store(false, Ordering::Relaxed);
        if !canceled {
            self.should_notify.store(true, Ordering::Release);
        }
        let cleared = self.state.cleared();
        if cleared {
            inner.items = self.items.clone();   // restart() 后切换到新 boxcar::Vec
        }
        // 异步执行 Worker::run()，立即返回不阻塞
        self.pool.spawn(move || unsafe { inner.run(status, cleared) });
    }
    Status { changed, running }
}
```

#### Status 字段含义

| 字段 | 含义 |
|------|------|
| `status.changed` | 本帧 snapshot 是否被更新（只有在 `inner.running=true` 且没被取消时才为 `true`） |
| `status.running` | 是否还有未完成的工作——如果为 `true`，下一帧还需要 tick（若 worker 还在跑，或有新候选项待处理） |

Picker 侧对 `status.changed` 的处理（[picker.rs#L686-L690](helix-term/src/ui/picker.rs#L686-L690)）：
```rust
if status.changed {
    self.cursor = self.cursor
        .min(snapshot.matched_item_count().saturating_sub(1))
}
```
只有在 snapshot 真的更新后才修正 cursor 越界，避免不必要的抖动。

#### Snapshot 更新机制（`Snapshot::update`）

```rust
fn update(&mut self, worker: &Worker<T>) {
    self.item_count = worker.item_count();
    self.pattern.clone_from(&worker.pattern);
    self.matches.clone_from(&worker.matches);   // 复制排序后的匹配结果
    if !Arc::ptr_eq(&worker.items, &self.items) {
        self.items = worker.items.clone()        // restart() 后切换共享 vector
    }
}
```

**[matcher.snapshot()](helix-term/src/ui/picker.rs#L685)** 返回一个**不可变引用**：
- 只包含 `item_count`、`matches: Vec<Match>`、`pattern`、`items: Arc<boxcar::Vec<T>>` 四个字段
- 包含 `matched_item_count()`、`item_count()`、`matched_items(range)`、`get_matched_item(idx)`、`pattern()` 等查询接口
- 所有后续渲染读取都基于这个快照，保证同一次渲染期间数据一致

#### Worker 扫描新项的游标机制（`Worker::process_new_items`）

Worker 用 `last_snapshot: u32` 记录上次处理到的索引。每次 `run()` 时：

```rust
let new_snapshot = self.items.par_snapshot(self.last_snapshot);
if new_snapshot.end() != self.last_snapshot {
    let end = new_snapshot.end();
    // 对 [last_snapshot, end) 范围内的项并行匹配
    self.matches.par_extend(new_snapshot.map(|(idx, item)| {
        // 若 item 未初始化（写入还在进行）→ 放入 in_flight
        // 否则 pattern.score() 计算得分
    }));
    self.last_snapshot = end;  // 游标前进
}
```

如果某索引暂未初始化（push 正在并发写入），Worker 把 `idx` 放入 `in_flight: Vec<u32>`，下次 `run()` 时在 `process_new_items` 开头优先重试这些索引。

### 3.3 排序机制

Nucleo 内部的排序在 `nucleo/src/worker.rs` 的 worker 匹配流程中完成，核心是对 `self.matches` 调用并行快速排序 `par_quicksort`：

#### 排序优先级（从高到低）

1. **得分降序**：`Match.score: u32` 越大越靠前。得分是各列 `u16` 得分之和。

2. **同分 tie breaker — 列文本总长度升序**：得分相同时，比较各列 `matcher_columns` 的 UTF-32 码点总长度，**更短的排在前面**。这体现了"更精确的匹配更相关"的直觉——较短的文本中出现相同数量的匹配字符，匹配密度更高。

3. **长度也相同 — 注入索引升序**：如果连列文本总长度都相同，则按注入索引 `idx` 升序（即先注入的排前面，`reverse_items=false` 时）。

对应源码（`worker.rs` 中 `sort_matches` 的比较闭包）：

```rust
|match1, match2| {
    if match1.score != match2.score {
        return match1.score > match2.score;    // 得分降序
    }
    if match1.idx == u32::MAX { return false; } // 已删除项排后面
    if match2.idx == u32::MAX { return true; }
    let len1: u32 = item1.matcher_columns.iter()
        .map(|haystack| haystack.len() as u32).sum();
    let len2: u32 = item2.matcher_columns.iter()
        .map(|haystack| haystack.len() as u32).sum();
    if len1 == len2 {
        match1.idx < match2.idx                  // 注入索引升序
    } else {
        len1 < len2                              // 总长度升序
    }
}
```

> **注意**：这不是稳定排序。`par_quicksort` 不保证相同键的元素保持原始相对顺序。但由于第三优先级是注入索引，实际效果等同于：得分和长度均相同的项按注入顺序排列。

#### 两种得分类型的区分

| 来源 | 得分类型 | 说明 |
|------|---------|------|
| `nucleo_matcher::Matcher` 单列匹配 | `Option<u16>` | 底层匹配器，单列单次匹配的原始得分 |
| `nucleo::MultiPattern::score()` | `Option<u32>` | 高层多列累加得分，各列 `u16` 之和 |
| `nucleo::Match.score` | `u32` | 匹配结果中的得分字段 |
| `helix_core::fuzzy::fuzzy_match()` | `Vec<(T, u16)>` | 便捷函数，使用底层 `Atom::match_list()`，返回单列 `u16` 得分 |

`helix_core::fuzzy::fuzzy_match()`（[helix-core/src/fuzzy.rs#L31-L48](helix-core/src/fuzzy.rs#L31-L48)）是 Helix 提供的便捷函数，适用于小规模同步匹配场景（如命令补全），**不适用于** Picker 的大规模异步匹配。它返回 `Vec<(T, u16)>`——元素和得分对，已按得分降序排列。而 Picker 使用的是 `nucleo::Nucleo<T>` 的异步管线，得分类型为 `u32`（多列累加）。

### 3.4 选中项同步 — cursor 如何跟随匹配结果

Picker 中的 `cursor: u32` 是一个 **0-based 索引**，指向 `snapshot.matched_items()` 中当前选中的位置。在以下场景中 cursor 会被调整：

#### 查询变更时重置

```rust
// handle_prompt_change() L542
self.cursor = 0;
```
查询发生变化时，匹配结果全部重新计算，cursor 重置到第一项。

#### tick 发现结果变化时修正

```rust
// render_picker() L686-L690
if status.changed {
    self.cursor = self.cursor
        .min(snapshot.matched_item_count().saturating_sub(1))
}
```
如果 tick 后结果集缩小（如用户输入更严格的查询导致匹配数减少），cursor 可能超出范围。这里用 `min` 将 cursor 钳制到最后一个有效项。如果结果集增大，cursor 保持不变——用户看到的选中项不变，只是下方可能出现新的候选项。

#### 用户导航时循环移动

```rust
// move_by() L459-L475
Direction::Forward => {
    self.cursor = self.cursor.saturating_add(amount) % len;
}
Direction::Backward => {
    self.cursor = self.cursor.saturating_add(len).saturating_sub(amount) % len;
}
```
取模运算保证 cursor 始终在 `[0, len)` 范围内，并实现循环滚动（到末尾后回到开头，反之亦然）。

#### 选中项的获取

```rust
// selection() L501-L506
pub fn selection(&self) -> Option<&T> {
    self.matcher
        .snapshot()
        .get_matched_item(self.cursor)
        .map(|item| item.data)
}
```
通过 `cursor` 索引从 snapshot 中取出对应的原始数据引用。`get_matched_item` 返回的是排序后的第 `cursor` 项。

### 3.5 高亮显示的完整过程

高亮发生在 [render_picker()](helix-term/src/ui/picker.rs#L760-L835) 中，分为三步：

#### 第一步：获取匹配字符索引

```rust
// L774-L778
snapshot.pattern().column_pattern(matcher_index).indices(
    item.matcher_columns[matcher_index].slice(..),
    &mut matcher,
    &mut indices,
);
```

- `column_pattern(matcher_index)` 获取第 `matcher_index` 列的查询模式
- `item.matcher_columns[matcher_index]` 是注入时 `fill_fn` 填入的列文本
- `indices()` 方法重新对列文本执行匹配，返回所有匹配字符的位置索引（`Vec<u32>`）
- 匹配器 `MATCHER` 是全局单例（[fuzzy.rs](helix-core/src/fuzzy.rs#L25)），需要加锁使用

#### 第二步：排序去重

```rust
// L779-L780
indices.sort_unstable();
indices.dedup();
```

虽然理论上索引不应重复，但去重确保安全性。

#### 第三步：将索引映射到视觉高亮 Span

这是最复杂的部分（[L781-L818](helix-term/src/ui/picker.rs#L781-L818)），核心逻辑：

```rust
let mut next_highlight_idx = indices.next().unwrap_or(u32::MAX);
let mut span_list = Vec::new();
let mut current_span = String::new();
let mut current_style = Style::default();
let mut grapheme_idx = 0u32;

for span in spans {
    for grapheme in span.content.graphemes(true) {
        let style = if grapheme_idx == next_highlight_idx {
            next_highlight_idx = indices.next().unwrap_or(u32::MAX);
            span.style.patch(highlight_style)   // 匹配字符用高亮样式
        } else {
            span.style                           // 非匹配字符用原始样式
        };
        if style != current_style {
            if !current_span.is_empty() {
                span_list.push(Span::styled(current_span, current_style))
            }
            current_span = String::new();
            current_style = style;
        }
        current_span.push_str(grapheme);
        grapheme_idx += 1;
    }
}
span_list.push(Span::styled(current_span, current_style));
cell = Cell::from(Spans::from(span_list));
```

**工作原理**：
1. 遍历 Cell 中每个 Span 的每个字素（grapheme）
2. 用 `grapheme_idx` 追踪当前字素位置
3. 当 `grapheme_idx == next_highlight_idx` 时，该字素应用 `highlight_style`（`special` + BOLD）
4. 相邻同风格的字素合并为一个 Span，减少渲染开销
5. 最终生成一个 `Spans`（Span 列表）替换原始 Cell 内容

**为什么可以用字素遍历对标字符索引**（代码注释原文，[L792-L796](helix-term/src/ui/picker.rs#L792-L796)）：
> this looks like a bug on first glance, we are iterating graphemes but treating them as char indices. The reason that this is correct is that nucleo will only ever consider the first char of a grapheme (and discard the rest of the grapheme) so the indices returned by nucleo are essentially grapheme indices

即 nucleo 在匹配时只取每个字素的第一个 char，因此返回的索引等效于字素索引。

### 3.6 动态查询时的刷新机制

对于 [DynamicQueryHandler](helix-term/src/ui/picker/handlers.rs#L120-L128)，当查询变化后执行 [finish_debounce()](helix-term/src/ui/picker/handlers.rs#L162-L189)：

```rust
fn finish_debounce(&mut self) {
    // ...
    job::dispatch_blocking(move |editor, compositor| {
        // 1. 递增版本号，使所有旧 Injector 失效
        picker.version.fetch_add(1, atomic::Ordering::Relaxed);
        // 2. 清空匹配器中已有的候选
        picker.matcher.restart(false);
        // 3. 创建新 Injector（带新版本号）
        let injector = picker.injector();
        // 4. 执行回调，重新生成候选列表
        let get_options = (callback)(&query, editor, picker.editor_data.clone(), &injector);
        // 5. 异步等待完成
        tokio::spawn(async move { ... });
    })
}
```

**关键步骤**：
1. `version.fetch_add(1)` — 旧 Injector 检测到版本不匹配后 `push()` 返回 `Err(InjectorShutdown)`，安全退出
2. `matcher.restart(false)` — 切换到一个全新的 `boxcar::Vec`，旧 vector 随旧 Injector 引用计数归零后释放；`false` 表示不立刻清除 snapshot
3. 新 Injector 绑定新版本号和新 vector，后台任务通过它 `push()` 追加新候选（每次 push 立即调 `notify()` 请求重绘）
4. 后续每一帧 `tick(10ms)` 会：检测到 boxcar::Vec 有未处理的新项 → 把 Worker 派发到 rayon 线程池 → Worker 用 `last_snapshot` 游标增量扫描新项 → 匹配、排序、更新 snapshot

### 3.7 完整时序图

```
用户键入字符
    │
    ▼
prompt_handle_event()
    │  更新 prompt 内容
    ▼
handle_prompt_change(is_paste)
    │  ① parse 查询 → 检测哪些列的 pattern 变了
    │  ② cursor = 0（重置选中项）
    │  ③ 对变化的列调用 matcher.pattern.reparse()
    │  ④ 如果是动态 picker，发送 DynamicQueryChange
    ▼
异步：nucleo rayon 线程池
    │  （下一帧 render_picker → tick 时 pool.spawn 派发到这里）
    │  ① pattern.status=Rescore 时 reset + 重算所有项得分
    │  ② pattern.status=Unchanged 时 last_snapshot 游标增量扫描新项
    │  ③ par_quicksort 并行排序（得分降序 → 长度升序 → 注入索引升序）
    │  ④ 完成后若 should_notify → request_redraw()
    ▼
render_picker()
    │  ① matcher.tick(10ms) — 拿 worker 锁
    │     · 超时拿不到 → running=true，本帧无新结果
    │     · 拿到且上一轮跑完 → snapshot.update() 复制结果 + matches
    │     · 拿到且有未处理项 → pool.spawn() 再次派发
    │  ② 获取 snapshot（只读一致性视图）
    │  ③ status.changed → 修正 cursor 不越界
    │  ④ 遍历可见行的 matched_items
    │     └─ 对每个 filter 列：pattern.indices() → 字素遍历 → 构造高亮 Span
    │  ⑤ 渲染 Table（带 highlight_style 高亮）
    ▼
render_preview()
    │  ① selection() → snapshot.get_matched_item(cursor) → 原始数据
    │  ② file_fn → 路径 + 行范围
    │  ③ get_preview() → 缓存/磁盘/编辑器文档
    │  ④ 异步触发语法高亮（150ms 防抖）
    │  ⑤ 渲染预览文档（语法高亮 + 范围标记）
    ▼
用户看到更新后的界面
```

---

## 4. 预览更新流程

### 预览获取

**[get_preview()](helix-term/src/ui/picker.rs#L585-L681)** 获取当前选中项的预览内容：

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
            // ...
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

**[PreviewHighlightHandler](helix-term/src/ui/picker/handlers.rs#L14-L113)** 处理预览的异步语法高亮：

1. **防抖**：150ms 防抖，避免频繁切换时重复高亮
2. **流程**：
   - 检测到无语法的文档时发送高亮请求
   - 防抖结束后在后台线程执行语法解析
   - 解析完成后更新缓存中的文档
   - 同时更新诊断信息

### 预览渲染

**[render_preview()](helix-term/src/ui/picker.rs#L882-L1023)** 渲染预览内容：

1. **定位**：根据 `range` 参数定位到指定行范围
2. **高亮**：
   - 语法高亮（`EditorView::doc_syntax_highlighter`）
   - 彩虹括号（如果启用）
   - 诊断信息（错误、警告等）
   - 选中范围高亮（`ui.highlight` 样式）
3. **渲染**：调用 `render_document()` 渲染文档内容

---

## 5. 交互选择流程

### 事件处理

**[handle_event()](helix-term/src/ui/picker.rs#L1053-L1170)** 处理用户交互事件：

#### 5.1 导航操作

| 快捷键 | 功能 | 实现 |
|-------|------|------|
| `Up` / `Shift-Tab` / `Ctrl-p` | 向上移动一项 | [move_by(1, Backward)](helix-term/src/ui/picker.rs#L459-L475) |
| `Down` / `Tab` / `Ctrl-n` | 向下移动一项 | [move_by(1, Forward)](helix-term/src/ui/picker.rs#L459-L475) |
| `PageUp` / `Ctrl-u` | 向上翻一页 | [page_up()](helix-term/src/ui/picker.rs#L478-L480) |
| `PageDown` / `Ctrl-d` | 向下翻一页 | [page_down()](helix-term/src/ui/picker.rs#L483-L485) |
| `Home` | 跳转到第一项 | [to_start()](helix-term/src/ui/picker.rs#L488-L490) |
| `End` | 跳转到最后一项 | [to_end()](helix-term/src/ui/picker.rs#L493-L499) |

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

#### 5.2 选择操作

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

#### 5.3 输入处理

其他按键事件传递给 `prompt_handle_event()` 处理输入：
- 普通字符输入到查询框
- 退格、删除等编辑操作
- 粘贴事件（`Event::Paste`）

### 选择获取

**[selection()](helix-term/src/ui/picker.rs#L501-L506)** 获取当前选中项：

```rust
pub fn selection(&self) -> Option<&T> {
    self.matcher
        .snapshot()
        .get_matched_item(self.cursor)
        .map(|item| item.data)
}
```

### 关闭逻辑

**[close_fn](helix-term/src/ui/picker.rs#L1066-L1087)** 处理 picker 关闭：

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
┌─────────────────┐     ┌──────────────────────┐
│  Prompt 输入框  │────▶│ PickerQuery 解析      │
└─────────────────┘     └──────────────────────┘
    │                          │
    │                          ▼ matcher.pattern.reparse()
    │              ┌─────────────────────────────────────┐
    │              │  Nucleo 匹配引擎                      │
    │              │  ┌───────────────────────────────┐  │
    │              │  │ Arc<boxcar::Vec<T>>           │  │
    │              │  │ (三方共享 append-only vector) │  │
    │              │  └───────────────┬───────────────┘  │
    │              │          Injector│push/extend         │
    │              │                  ▼                    │
    │              │          追加 + notify()             │
    │              │                  │                    │
    │              │  ┌───────────────▼───────────────┐  │
    │              │  │ rayon 线程池（异步匹配）       │  │
    │              │  │  Worker::run()                 │  │
    │              │  │  · last_snapshot 游标增量扫描   │  │
    │              │  │  · par_extend 并行匹配         │  │
    │              │  │  · par_quicksort 并行排序      │  │
    │              │  │  · 完成后若 should_notify 则   │  │
    │              │  │    再调一次 notify()           │  │
    │              │  └───────────────┬───────────────┘  │
    │              └──────────────────┼──────────────────┘
    │                                 │
    │              ┌──────────────────▼──────────────────┐
    │              │        tick(10ms) + snapshot         │
    │              │  · 拿 worker 锁（最多等 10ms）        │
    │              │  · 若上一轮跑完 → snapshot.update()  │
    │              │  · 若还有未处理项 → pool.spawn()     │
    │              │  · status.changed → 修正 cursor     │
    │              │  · indices() → 高亮渲染              │
    │              └──────────────────┬──────────────────┘
    ▼                                 ▼
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
3. **异步匹配**：nucleo 将 Worker 派发到独立 rayon 线程池，匹配完全不阻塞 UI 线程
4. **tick 限时取锁**：`tick(10ms)` 最多等待 10ms 获取 Worker 锁，拿不到就直接返回本帧旧结果，保证帧率
5. **游标增量扫描**：Worker 用 `last_snapshot` 游标追踪 boxcar::Vec 新增项，避免全量重复扫描
6. **并行匹配与排序**：`par_extend` 并行匹配新项，`par_quicksort` 并行排序
7. **重绘去抖**：`request_redraw()` 通过 tokio `Notify` 发送通知，Helix 主循环以 30 FPS 合并
8. **预览缓存**：避免重复读取文件和解析语法
9. **防抖处理**：动态查询（100ms）和语法高亮（150ms）都有防抖
10. **快速路径**：查询无实际变化时直接返回
11. **版本号取消**：`AtomicUsize` 版本机制安全地使旧 Injector 失效

---

## 核心文件索引

| 文件 | 作用 |
|------|------|
| [picker.rs](helix-term/src/ui/picker.rs) | Picker 主组件，包含渲染、事件处理、预览逻辑 |
| [picker/query.rs](helix-term/src/ui/picker/query.rs) | 查询语法解析，支持 `%field` 多列查询 |
| [picker/handlers.rs](helix-term/src/ui/picker/handlers.rs) | 异步处理器：预览高亮、动态查询 |
| [helix-core/src/fuzzy.rs](helix-core/src/fuzzy.rs) | 模糊匹配工具函数，MATCHER 全局实例（小规模同步匹配） |
| [helix-event/src/redraw.rs](helix-event/src/redraw.rs) | `request_redraw()` 重绘通知机制，30 FPS 去抖 |
