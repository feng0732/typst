# CLI Watch 监听、防抖与重编译链路分析

按代码执行顺序，完整梳理 `typst watch` 命令从启动到监听文件变化、防抖合并、错误恢复、触发重编译的全过程。

---

## 1. 命令入口与初始化

### 1.1 进程入口

**文件**: `crates/typst-cli/src/main.rs` (第 50-66 行)

```
main()
  └─ sigpipe::reset()          // 处理管道信号
  └─ dispatch()                // 命令分发
```

### 1.2 命令分发

**文件**: `crates/typst-cli/src/main.rs` (第 69-82 行)

```rust
fn dispatch() -> HintedStrResult<()> {
    match &ARGS.command {
        Command::Watch(command) => crate::watch::watch(command)?,
        // ... 其他命令
    }
}
```

命令定义在 `crates/typst-cli/src/args.rs` [args.rs#L122-L133](crates/typst-cli/src/args.rs#L122-L133)，`WatchCommand` 包含编译参数和可选的 HTTP 服务器参数。

---

## 2. 监视模式初始化

**文件**: `crates/typst-cli/src/watch.rs` [watch.rs#L18-L84](crates/typst-cli/src/watch.rs#L18-L84)

### 2.1 步骤 1：创建编译配置

```rust
let mut config = CompileConfig::watching(command)?;
```

调用 `crates/typst-cli/src/compile.rs` [compile.rs#L98-L100](crates/typst-cli/src/compile.rs#L98-L100) 的 `CompileConfig::watching()`，内部通过 `new_impl(args, Some(command))` 构造配置：

- 推断输出格式（PDF/PNG/SVG/HTML/Bundle）
- 校验 watch 模式下不允许 stdout 输出
- 可选地创建 HTTP 服务器（HTML/Bundle 导出时）
- 设置 `config.watching = true` 标记

### 2.2 步骤 2：创建文件系统监听器

```rust
let mut watcher = Watcher::new(Some(output.clone()))?;
```

**文件**: `crates/typst-kit/src/watcher.rs` [watcher.rs#L47-L70](crates/typst-kit/src/watcher.rs#L47-L70)

`Watcher::new()` 内部：
1. 创建 MPSC 通道 `(tx, rx)` 用于 notify-rs 事件传递
2. 配置 `notify::Config`，将轮询间隔从默认 ~30s 缩短为 `POLL_INTERVAL = 300ms`
3. 构造 `RecommendedWatcher`（底层为 notify-rs，根据系统选 inotify/fsevents/kqueue）
4. 初始化四个核心字段：
   - `output: Option<PathBuf>`：输出文件路径（变化时忽略）
   - `watched: FxHashMap<PathBuf, bool>`：已监听路径集合 + 标记位
   - `missing: FxHashSet<PathBuf>`：尚不存在、需手动轮询的路径
   - `rx: Receiver<notify::Result<Event>>`：事件接收端

### 2.3 步骤 3：创建编译世界（错误恢复场景一）

**文件**: `crates/typst-cli/src/watch.rs` [watch.rs#L31-L49](crates/typst-cli/src/watch.rs#L31-L49)

```rust
let mut world = loop {
    match SystemWorld::new(...) {
        Ok(world) => break world,
        Err(
            ref err @ (WorldCreationError::InputNotFound(ref path)
            | WorldCreationError::RootNotFound(ref path)),
        ) => {
            // 输入文件或根目录不存在时，不退出
            watcher.update([path.clone()])?;  // 把缺失路径加入监听
            Status::Error.print(&config).unwrap();
            print_error(&err.to_string()).unwrap();
            watcher.wait()?;                  // 阻塞等待文件被创建
        }
        Err(err) => return Err(err.into()),   // 其他错误直接退出
    }
};
```

**设计要点**：
- 如果输入文件/根目录不存在，watch 模式 **不会退出**，而是把缺失路径加入监听后进入等待
- 用户创建文件后，`watcher.wait()` 收到事件返回，循环重试 `SystemWorld::new()`
- 只有非 "not found" 类错误（如 I/O 错误、时间戳无效）才会直接终止

### 2.4 步骤 4：预扫描字体 + 首次编译

```rust
if config.output_format.is_paged() {
    world.scan_fonts();   // 预先强制加载字体，不计入编译耗时
}

// 首次编译
timer.record(&mut world, |world| compile_once(world, &mut config))??;
```

`Timer::record` 见 `crates/typst-kit/src/timer.rs` [timer.rs#L57-L95](crates/typst-kit/src/timer.rs#L57-L95)。

注意这里的 `??`：外层是 `timer.record` 的 `StrResult`，内层是 `compile_once` 的 `HintedStrResult`。**但编译错误（语法错误等）不会导致 `compile_once` 返回 Err**，详见第 6 章错误恢复分析。

---

## 3. 核心循环总览

**文件**: `crates/typst-cli/src/watch.rs` [watch.rs#L68-L83](crates/typst-cli/src/watch.rs#L68-L83)

```rust
loop {
    // ① 更新监听路径为上次编译的所有依赖
    watcher.update(world.dependencies())?;

    // ② 阻塞等待变化事件（含防抖/批处理）
    watcher.wait()?;

    // ③ 重置编译世界状态
    world.reset();

    // ④ 触发重编译
    timer.record(&mut world, |world| compile_once(world, &mut config))??;

    // ⑤ 清理 comemo 增量缓存
    comemo::evict(10);
}
```

---

## 4. 依赖监听更新机制

### 4.1 获取依赖列表

**文件**: `crates/typst-cli/src/world.rs` [world.rs#L97-L101](crates/typst-cli/src/world.rs#L97-L101)

```rust
pub fn dependencies(&mut self) -> impl Iterator<Item = PathBuf> + '_ {
    let (loader, deps) = self.files.dependencies();
    deps.filter_map(|id| loader.resolve(id).ok())
}
```

从 `FileStore` 中取出上次编译过程中实际访问过的所有 `FileId`，解析为文件系统路径。这意味着 **每次编译后都会动态刷新监听范围** —— 如果代码中 `#import` 了新文件，下一轮监听会自动包含它们。

**依赖跟踪的实现原理**：见 `crates/typst-kit/src/files.rs` [files.rs#L91-L99](crates/typst-kit/src/files.rs#L91-L99)

```rust
pub fn dependencies(&mut self) -> (&L, impl Iterator<Item = FileId> + '_) {
    let iter = self
        .slots
        .get_mut()
        .iter()
        .filter(|(_, slot)| slot.accessed())  // 只要被访问过就算依赖
        .map(|(&id, _)| id);
    (&self.loader, iter)
}
```

`accessed()` 的定义：[files.rs#L163-L165](crates/typst-kit/src/files.rs#L163-L165)

```rust
fn accessed(&self) -> bool {
    !matches!(self, Self::Empty(_))
}
```

即：只要 `FileSlot` 不是 `Empty` 状态，就视为被访问过。`FileSlot` 有三种状态：`Empty`、`Loaded`、`Parsed`。

### 4.2 更新监听器（Mark-and-Sweep 策略）

**文件**: `crates/typst-kit/src/watcher.rs` [watcher.rs#L76-L118](crates/typst-kit/src/watcher.rs#L76-L118)

```rust
pub fn update(&mut self, iter: impl IntoIterator<Item = PathBuf>) -> StrResult<()> {
    // ① Mark：将所有已监听路径标记为 false（待清理）
    for seen in self.watched.values_mut() {
        *seen = false;
    }
    self.missing.clear();

    // ② 遍历新依赖列表
    for path in iter {
        if !path.exists() {
            self.missing.insert(path);       // 不存在的文件放入 missing 集合
            continue;
        }
        if !self.watched.contains_key(&path) {
            // 新路径：调用 notify-rs 注册监听（非递归）
            self.watcher.watch(&path, RecursiveMode::NonRecursive)?;
        }
        self.watched.insert(path, true);     // 标记为保留
    }

    // ③ Sweep：移除标记为 false 的路径（不再被依赖）
    self.watched.retain(|path, &mut seen| {
        if !seen {
            self.watcher.unwatch(path).ok(); // 取消监听
        }
        seen
    });
}
```

**关键点**：
- 使用 **Mark-and-Sweep** 做增量监听管理，避免每次都全部重新注册
- 对不存在的路径加入 `missing` 集合，后续在 `wait()` 中手动轮询
- 所有监听均为 `NonRecursive`（只监听文件本身，不递归目录）

---

## 5. 等待与防抖机制

**文件**: `crates/typst-kit/src/watcher.rs` [watcher.rs#L121-L192](crates/typst-kit/src/watcher.rs#L121-L192)

### 5.1 整体结构

```rust
pub fn wait(&mut self) -> StrResult<()> {
    loop {
        // 阶段一：等待首个事件
        let first = self.rx.recv_timeout(
            if self.missing.is_empty() { Duration::MAX } else { Self::POLL_INTERVAL }
        );

        // 阶段二：批处理/防抖（Batching）
        let mut relevant = false;
        let batch_start = Instant::now();

        for event in first.into_iter()
            .chain(iter::from_fn(|| self.rx.recv_timeout(Self::BATCH_TIMEOUT).ok()))
            .take_while(|_| batch_start.elapsed() <= Self::STARVE_TIMEOUT)
        {
            // ... 处理每个事件 ...
        }

        // 阶段三：检查触发条件
        if relevant || self.missing.iter().any(|p| p.exists()) {
            return Ok(());
        }
        // 否则继续外层 loop
    }
}
```

### 5.2 三个关键时间常量

| 常量 | 值 | 作用 |
|------|-----|------|
| `BATCH_TIMEOUT` | 100ms | 接收到事件后，继续等待后续连续事件的窗口。编辑器保存文件时通常会产生 Remove+Create/Modify 多个事件，100ms 窗口将它们合并为一次触发 |
| `STARVE_TIMEOUT` | 500ms | 批处理的最大持续时间。防止大量文件修改时，因事件不断涌入而永远在 BATCH_TIMEOUT 内收不到超时信号，导致迟迟不触发编译 |
| `POLL_INTERVAL` | 300ms | 两种用途：① 缺失文件（missing 集合）的存在性轮询间隔；② notify-rs PollWatcher 后端的轮询间隔 |

### 5.3 事件处理流程

```
遍历到一个事件后：
  │
  ├─ ① 事件类型过滤 — 跳过 Access/Metadata/Other 等无关事件
  │
  ├─ ② inotify unwatch 修复
  │     若事件为 Remove(File) 或 Modify(Name(RenameFrom))：
  │     主动从 watched 映射中移除路径（notify-rs 的 inotify 后端
  │     会在文件删除/重命名时隐式 unwatch，需手动清理状态）
  │
  ├─ ③ 输出文件过滤 — 所有路径都是输出文件则跳过（防止自触发死循环）
  │
  └─ ④ 标记 relevant = true
```

### 5.4 事件类型过滤

**文件**: `crates/typst-kit/src/watcher.rs` [watcher.rs#L196-L211](crates/typst-kit/src/watcher.rs#L196-L211)

```
相关事件：Any / Create / Modify::Any / Modify::Data
        / Modify::Name / Remove
忽略事件：Access / Modify::Metadata / Modify::Other / Other
```

### 5.5 防抖迭代器链

```
first                        // 第一个事件（Option 转 iter，可能为空）
  .into_iter()
  .chain(                    // 接上后续持续等待的事件
     iter::from_fn(||
        rx.recv_timeout(BATCH_TIMEOUT=100ms).ok()
        // 100ms 内没新事件 → 返回 None → 迭代结束
     )
  )
  .take_while(|_|            // 强制截止：最多 500ms
     batch_start.elapsed() <= STARVE_TIMEOUT=500ms
  )
```

**示例**：编辑器原子保存（典型流程 `unlink → write → rename`）
```
T+0ms:   Remove 事件到达 → 进入批处理，batch_start 记录
T+10ms:  Create 事件到达 → 在 100ms 窗口内，继续
T+30ms:  Modify(Data) 事件到达 → 继续
T+130ms: BATCH_TIMEOUT=100ms 超时 → 迭代链终止
         累计发现相关事件 → relevant=true → 返回触发编译
```

### 5.6 输出自触发防护与边界分析

#### 5.6.1 防护逻辑

**代码位置**：`crates/typst-kit/src/watcher.rs` [watcher.rs#L172-L181](crates/typst-kit/src/watcher.rs#L172-L181)

```rust
// Don't recompile because the output file changed.
// FIXME: This doesn't work properly for multifile image export.
if let Some(output) = &self.output
    && event
        .paths
        .iter()
        .all(|path| is_same_file(path, output).unwrap_or(false))
{
    continue;
}
```

**防护机制**：
- `Watcher::new(output)` 时传入输出文件路径，存储在 `self.output` 中
- 每个事件到达后，用 `is_same_file`（same_file crate）比较事件路径与输出路径
- `is_same_file` 通过 inode/文件句柄判断两个路径是否指向同一个文件（能正确处理符号链接、相对/绝对路径差异）
- 如果事件中的**所有**路径都是输出文件，则 `continue` 跳过，不标记 `relevant`
- 配合防抖批处理机制，如果一个批处理窗口内只有输出文件事件，整轮等待不会触发重编译

**`all()` 的语义**：只有事件中所有路径都是输出文件时才跳过。如果事件同时涉及输出文件和源文件，则不会跳过 —— 这是合理的，因为源文件变化确实需要重编译。

#### 5.6.2 output 参数的来源

**代码位置**：`crates/typst-cli/src/watch.rs` [watch.rs#L22-L27](crates/typst-cli/src/watch.rs#L22-L27)

```rust
let Output::Path(output) = &config.output else {
    bail!("cannot write document to stdout in watch mode");
};
let mut watcher = Watcher::new(Some(output.clone()))?;
```

传入的 `output` 就是 `config.output`，即用户在命令行指定的原始输出路径。**注意：对于多页图片导出，这个路径可能包含模板占位符（如 `{p}`、`{0p}`），不是实际生成的文件路径。**

#### 5.6.3 输出路径模板与实际生成文件

多页 PNG/SVG 导出时，路径模板在 `crates/typst-cli/src/compile.rs` [compile.rs#L536-L563](crates/typst-cli/src/compile.rs#L536-L563) 的 `output_template` 模块中处理。

支持的模板占位符：

| 占位符 | 含义 | 示例（第3页/共10页） |
|--------|------|---------------------|
| `{p}` | 页码（无前导零） | `3` |
| `{0p}` 或 `{n}` | 页码（有前导零，按总页数宽度对齐） | `03` |
| `{t}` | 总页数 | `10` |

**模板检测**：`has_indexable_template(output)` 检查路径中是否包含 `{p}`、`{0p}` 或 `{n}` 任意一个。

**多页导出的条件**：
- 如果文档有多页，且输出路径不含模板 → 报错：`cannot export multiple images without a page number template`
- 如果文档有多页，且输出路径含模板 → 正常导出，生成 `page-1.png`、`page-2.png`、...

#### 5.6.4 各导出格式的防护有效性

| 导出格式 | 输出类型 | output 参数 | 实际生成 | 自触发防护 |
|---------|---------|------------|---------|-----------|
| PDF | 单文件 | 实际文件路径 | 1 个文件 | ✅ 有效 |
| HTML | 单文件 | 实际文件路径 | 1 个文件 | ✅ 有效 |
| PNG（单页文档） | 单文件 | 实际文件路径 | 1 个文件 | ✅ 有效 |
| PNG（多页 + 模板） | 多文件 | 模板路径（含 `{p}`） | N 个文件 | ❌ 无效 |
| SVG（单页文档） | 单文件 | 实际文件路径 | 1 个文件 | ✅ 有效 |
| SVG（多页 + 模板） | 多文件 | 模板路径（含 `{p}`） | N 个文件 | ❌ 无效 |
| Bundle | 目录 | 目录路径 | 目录下多文件 | ❌ 无效 |

**为什么多文件导出防护失效**：

以 `--format png -o page-{p}.png` 为例：

```
Watcher 中的 output: "page-{p}.png"   （模板路径，文件系统中不存在）
实际生成的文件:
  page-1.png  ← 写入时触发事件
  page-2.png  ← 写入时触发事件
  page-3.png  ← 写入时触发事件

防护判断:
  is_same_file("page-1.png", "page-{p}.png") → false（两个不同文件）
  is_same_file("page-2.png", "page-{p}.png") → false（两个不同文件）
  ...
  all(...) → false
  → 不跳过 → 标记为 relevant → 触发重编译 → 自触发！
```

`page-{p}.png` 作为模板路径，在文件系统中要么不存在（`is_same_file` 返回 `Err`，`.unwrap_or(false)` 得 `false`），要么是另一个不同的文件，所以所有生成文件的事件都不会被过滤。

#### 5.6.5 Bundle 导出的情况

Bundle 导出的输出是一个目录（见 `export_bundle` [compile.rs#L390-L415](crates/typst-cli/src/compile.rs#L390-L415)），目录下生成多个文件（HTML、CSS、JS、图片资源等）。

```
output: "output/"  （目录路径）
生成的文件:
  output/index.html
  output/assets/style.css
  output/assets/script.js
  ...

防护判断:
  is_same_file("output/index.html", "output/") → false（文件 vs 目录）
  → 不跳过 → 自触发！
```

`is_same_file` 比较文件和目录，返回 `false`，所以 bundle 导出的所有生成文件事件都不会被过滤。

#### 5.6.6 自触发会导致什么问题

如果输出文件/生成文件恰好在依赖监听范围内（例如：输出文件与某个被引用的图片资源同名，或输出目录在源文件目录内且被递归监听），则会形成**编译 → 写输出 → 触发事件 → 重编译 → 再写输出 → ...** 的死循环。

在当前架构下，由于：
1. watcher 只监听依赖文件（源文件、资源文件），不监听输出文件
2. 输出文件通常不在依赖列表中

所以自触发问题在大多数情况下不会实际发生。但以下边界场景可能触发：

- **输出文件与资源文件同名**：例如项目里有 `diagram.png` 被 `#image` 引用，同时输出路径设为 `diagram.png`，每次编译覆盖资源文件，触发死循环
- **模板展开后与资源文件同名**：多页导出时，某一页的输出文件名恰好和某个资源文件相同
- **Bundle 输出目录在源目录内**：bundle 生成的文件如果刚好和源目录内的文件路径重合

代码中的 `FIXME` 注释明确标记了这个已知问题，但尚未修复。

#### 5.6.7 单文件场景为何正常工作

单文件导出（PDF、HTML、单页图片）时，`output` 就是实际生成文件的路径。如果这个文件恰好在依赖监听范围内（虽然不常见），`is_same_file` 能正确识别为同一文件，过滤掉写入事件，防止死循环。

**注意**：即使是单文件导出，如果输出文件不在依赖监听范围内，自触发防护也"用不上"——因为 watcher 根本不会收到输出文件的事件。防护是针对"输出文件恰好也在监听范围内"这种边界场景的保险。

---

## 6. 错误恢复机制（核心）

### 6.1 编译错误 ≠ 进程退出

**关键发现**：`compile_once` 即使编译失败（有语法/语义错误），返回值仍然是 `Ok(())`。

**文件**: `crates/typst-cli/src/compile.rs` [compile.rs#L258-L314](crates/typst-cli/src/compile.rs#L258-L314)

```rust
pub fn compile_once(
    world: &mut SystemWorld,
    config: &mut CompileConfig,
) -> HintedStrResult<()> {
    let start = std::time::Instant::now();
    if config.watching {
        Status::Compiling.print(config).unwrap();
    }

    let Warned { output, mut warnings } = compile_and_export(world, config);
    // output: SourceResult<Vec<Output>> — 可能是 Err（编译错误）

    match &output {
        Ok(_) => {
            // 成功：打印 Success / PartialSuccess
            let duration = start.elapsed();
            if config.watching {
                if warnings.is_empty() {
                    Status::Success(duration).print(config).unwrap();
                } else {
                    Status::PartialSuccess(duration).print(config).unwrap();
                }
            }
        }

        Err(errors) => {
            // 编译失败：设置失败标志 + 打印错误
            set_failed();           // 仅设置退出码，不返回错误
            if config.watching {
                Status::Error.print(config).unwrap();
            }
            print_diagnostics(world, errors, &warnings, config.diagnostic_format)
                .map_err(|err| eco_format!("failed to print diagnostics ({err})"))?;
        }
    }

    // 无论成功失败，都走到这里返回 Ok(())
    Ok(())
}
```

**设计意图**：
- 编译错误（语法错误、未定义引用等）是 **用户需要修复的正常错误**，watch 模式不应退出
- 用户修改文件保存后，自动重新编译验证
- 只有真正的系统级错误（如无法打印诊断信息、终端不可写）才会向上传播，导致 watch 退出

### 6.2 watch 循环中的错误传播

回到 `crates/typst-cli/src/watch.rs` [watch.rs#L79](crates/typst-cli/src/watch.rs#L79)：

```rust
timer.record(&mut world, |world| compile_once(world, &mut config))??;
```

两个 `?` 的含义：
- 第一个 `?`：`timer.record()` 的 `StrResult<T>` —— 计时文件写入失败
- 第二个 `?`：`compile_once()` 的 `HintedStrResult<()>` —— 只有系统级错误才会返回 Err

**会导致 watch 退出的错误（系统级）**：
1. `timer.record` 写入计时文件失败
2. `print_diagnostics` 打印诊断失败（如终端不可写）
3. `open_output` 打开输出文件失败
4. `write_deps` 写入依赖文件失败
5. `watcher.update` 监听注册失败
6. `watcher.wait` 事件接收失败（notify-rs 内部错误）

**不会导致 watch 退出的错误（用户/环境级）**：
1. 语法错误、解析错误
2. 类型错误、未定义引用等语义错误
3. 图片/字体等资源加载失败（编译时错误）
4. **输出文件写入失败**（磁盘满、权限不足等）
5. 任何 `SourceDiagnostic` 级别的诊断

### 6.3 输出文件写入失败的错误传播分析

**核心问题**：输出文件写入失败（如磁盘满、权限不足）是否会导致 watch 退出？

**答案**：**不会退出**。输出文件写入失败会被包装为 `SourceDiagnostic` 错误，走与编译语法错误相同的处理路径 —— 打印错误但不退出，继续等待下一次文件变化。

#### 6.3.1 错误类型定义

`SourceResult<T>` 的定义见 `crates/typst-library/src/diag.rs` [diag.rs#L210](crates/typst-library/src/diag.rs#L210-L210)：

```rust
pub type SourceResult<T> = Result<T, EcoVec<SourceDiagnostic>>;
```

即：编译和导出阶段的所有错误（包括语法错误、语义错误、写入错误）都被收集为 `SourceDiagnostic` 列表。

#### 6.3.2 写入错误的完整传播链

以 PDF 导出为例，代码见 `crates/typst-cli/src/compile.rs` [compile.rs#L379-L388](crates/typst-cli/src/compile.rs#L379-L388)：

```rust
fn export_pdf(...) -> SourceResult<()> {
    let buffer = typst_pdf::pdf(document, &options)?;
    config.output.write(&buffer)
        .map_err(|err| eco_format!("failed to write PDF file ({err})"))
        .at(Span::detached())?;  // 包装为 SourceDiagnostic
    Ok(())
}
```

完整的错误转换链路：

```
std::fs::write() 失败
  → io::Error
  → .map_err() 包装为 EcoString 错误消息
  → .at(Span::detached()) 包装为 SourceDiagnostic::error
  → 收集到 EcoVec<SourceDiagnostic>
  → compile_and_export 返回 Warned { output: Err(EcoVec<SourceDiagnostic>), ... }
```

所有导出格式的写入错误处理方式一致：
- **HTML**: `export_html` [compile.rs#L343-L357](crates/typst-cli/src/compile.rs#L343-L357) — `failed to write HTML file`
- **PNG**: `export_image_page` [compile.rs#L566-L592](crates/typst-cli/src/compile.rs#L566-L592) — `failed to write PNG file`
- **SVG**: `export_image_page` [compile.rs#L566-L592](crates/typst-cli/src/compile.rs#L566-L592) — `failed to write SVG file`
- **Bundle**: `write_virtual_fs` [compile.rs#L418-L438](crates/typst-cli/src/compile.rs#L418-L438) — `failed to write file`

#### 6.3.3 在 compile_once 中的处理

代码见 `crates/typst-cli/src/compile.rs` [compile.rs#L295-L305](crates/typst-cli/src/compile.rs#L295-L305)：

```rust
match &output {
    Ok(_) => { /* ... */ }
    Err(errors) => {
        set_failed();                   // 设置进程退出码
        if config.watching {
            Status::Error.print(config).unwrap();  // 显示错误状态
        }
        print_diagnostics(world, errors, &warnings, config.diagnostic_format)
            .map_err(|err| eco_format!("failed to print diagnostics ({err})"))?;
    }
}
```

**关键判断点**：
- 如果 `print_diagnostics` **成功**（终端可写）：错误被内部消化，函数走到最后返回 `Ok(())`，watch 循环继续
- 如果 `print_diagnostics` **失败**（如终端被关闭、磁盘不可读导致诊断打印失败）：通过 `?` 向上传播错误，导致 watch 退出

#### 6.3.4 写入失败后的行为

输出文件写入失败后，watch 的行为与语法错误完全一致：

```
用户保存文件 → 编译成功 → 写入 PDF 失败（磁盘满）
  │
  ├─ compile_and_export 返回 Err([SourceDiagnostic])
  ├─ Status::Error.print() → 显示 "compiled with errors"
  ├─ print_diagnostics() → 打印 "failed to write PDF file (Permission denied)"
  ├─ compile_once() 返回 Ok(()) → 不退出
  ├─ comemo::evict(10) → 清理缓存
  │
  ├─ 回到循环顶
  │
  ├─ watcher.update(world.dependencies())
  │    └─ 注意：编译成功但写入失败，所有依赖文件都已被访问 → 监听范围完整
  │
  ├─ watcher.wait() → 继续等待
  │
  └─ 用户清理磁盘空间后再次保存 → 重新编译 → 写入成功
```

**重要细节**：写入失败发生在编译成功之后，所以 `FileStore` 中所有依赖文件的 `FileSlot` 都已被标记为非 `Empty`，`dependencies()` 返回完整的依赖列表，监听范围不受影响。

#### 6.3.5 设计意图

输出文件写入失败被归类为"可恢复错误"的原因：
1. **写入失败通常是暂时性的**：磁盘满、权限问题等，用户可以在不退出 watch 的情况下修复
2. **保持监视循环的连续性**：如果因一次写入失败就退出，用户需要重新启动 watch，体验不佳
3. **与编译错误保持一致的处理模型**：所有 `SourceDiagnostic` 级别的错误都采用相同的处理策略 —— 打印错误 + 继续等待

**例外**：`--open` 打开输出文件失败会导致退出（见 `compile_once` 中的 `open_output(config)?`），因为这是在 `match` 块之外的独立调用，直接通过 `?` 传播。但 `--open` 只在首次编译生效，后续编译不会重复打开。

### 6.4 编译失败后的依赖跟踪

**问题**：编译失败时，`world.dependencies()` 还能返回正确的依赖列表吗？

**答案**：可以。原因：

1. **`FileStore` 的依赖跟踪基于"是否被访问"**，而非"是否编译成功"
2. `accessed()` 的判断条件只是 `!matches!(self, Self::Empty(_))` [files.rs#L163-L165](crates/typst-kit/src/files.rs#L163-L165)
3. 即使编译在某个文件处失败，该文件以及之前访问过的所有文件，它们的 `FileSlot` 都已处于 `Loaded` 或 `Parsed` 状态，都会被计入依赖

**示例**：`main.typ` 引用了 `a.typ`，`a.typ` 引用了 `b.typ`，但 `b.typ` 有语法错误。
```
编译过程：
  访问 main.typ → slot 变为 Parsed → 计入依赖 ✓
  → import a.typ → slot 变为 Parsed → 计入依赖 ✓
  → import b.typ → slot 变为 Parsed(Err(...)) → 非 Empty → 计入依赖 ✓
  → 编译 b.typ 时发现错误 → 整个编译失败

dependencies() 返回：[main.typ, a.typ, b.typ] —— 三个都在 ✓
```

这确保了：**即使编译失败，watch 仍然能正确监听所有相关文件**，用户修复任何一个文件后都能触发重编译。

### 6.5 Reset 与下一轮的衔接

`world.reset()` 的实现见 `crates/typst-cli/src/world.rs` [world.rs#L104-L107](crates/typst-cli/src/world.rs#L104-L107)：

```rust
pub fn reset(&mut self) {
    self.files.reset();   // 清除 FileStore 的缓存标记
    self.now.reset();     // 重置时间戳
}
```

`FileStore::reset` 的实现：[files.rs#L111-L116](crates/typst-kit/src/files.rs#L111-L116)

```rust
pub fn reset(&mut self) {
    for slot in self.slots.get_mut().values_mut() {
        slot.reset();
    }
}
```

`FileSlot::reset`：[files.rs#L168-L174](crates/typst-kit/src/files.rs#L168-L174)

```rust
fn reset(&mut self) {
    let stale = match mem::take(self) {
        Self::Parsed(Ok(source), _) => Some(source),
        _ => None,
    };
    *self = Self::Empty(stale);  // 变回 Empty 状态
}
```

**Reset 的双重作用**：
1. **依赖跟踪清零**：slot 变为 `Empty`，下一轮 `dependencies()` 只包含新编译中实际访问的文件
2. **增量编译保留**：对于成功解析过的源文件，保留 `stale source`（过期源码），下次可以基于它做增量重解析（reparser），提升编译速度

注意：编译失败时（`Parsed(Err(...))`），`stale` 为 `None`，即不保留旧源码 —— 失败文件下次需要从头解析。

### 6.6 错误恢复的完整场景

**场景 1：初始文件不存在**
```
用户执行 typst watch missing.typ
  │
  ├─ SystemWorld::new() 返回 InputNotFound 错误
  ├─ watcher.update([missing.typ]) — 加入 missing 集合
  ├─ 打印错误信息
  ├─ watcher.wait()
  │    └─ 每 300ms 轮询 missing 集合，检查文件是否存在
  │
  └─ 用户创建 missing.typ
       └─ wait() 检测到文件存在 → 返回
       └─ 循环回到 SystemWorld::new()
       └─ 成功 → 继续首次编译
```

**场景 2：运行时编译语法错误**
```
用户保存文件，内容有语法错误
  │
  ├─ watcher.wait() 收到文件变化事件 → 返回
  ├─ world.reset() — 清空访问标记
  ├─ compile_once() 开始编译
  │    ├─ 编译到错误处 → output = Err(errors)
  │    ├─ set_failed() — 设置进程退出码
  │    ├─ Status::Error.print() — 显示错误状态
  │    └─ print_diagnostics() — 打印具体错误
  │
  ├─ compile_once() 返回 Ok(()) — 不向上传播
  │
  ├─ comemo::evict(10) — 清理缓存
  │
  ├─ 回到循环顶部
  │
  ├─ watcher.update(world.dependencies())
  │    └─ 虽然编译失败，但访问过的文件都在依赖列表里 → 监听仍然正确
  │
  ├─ watcher.wait() — 继续等待下一次修改
  │
  └─ 用户修复错误并保存 → 重新编译 → 成功
```

**场景 3：输出文件写入失败（不退出）**
```
用户保存文件 → 编译成功 → 写入 PDF 时磁盘已满
  │
  ├─ compile_and_export 中 export_pdf() 写入失败
  │    ├─ config.output.write() 返回 io::Error
  │    ├─ .map_err() 包装为 "failed to write PDF file"
  │    └─ .at(Span::detached()) 包装为 SourceDiagnostic
  │
  ├─ compile_and_export 返回 Warned { output: Err([SourceDiagnostic]), ... }
  ├─ match output 进入 Err 分支
  │    ├─ set_failed() — 设置进程退出码
  │    ├─ Status::Error.print() — 显示 "compiled with errors"
  │    └─ print_diagnostics() — 打印写入错误信息
  │
  ├─ compile_once() 返回 Ok(()) — 不向上传播
  │
  ├─ comemo::evict(10) — 清理缓存
  │
  ├─ 回到循环顶部
  │
  ├─ watcher.update(world.dependencies())
  │    └─ 编译成功过，所有依赖文件都已被访问 → 监听范围完整
  │
  ├─ watcher.wait() — 继续等待
  │
  └─ 用户清理磁盘空间 → 再次保存 → 重新编译 → 写入成功
```

**场景 4：系统级错误（watch 退出）**
```
编译过程中磁盘突然不可用
  │
  ├─ print_diagnostics() 写入终端失败 → 返回 Err
  ├─ compile_once() 通过 ? 向上传播 Err
  ├─ timer.record() 内层 ? 捕获 Err → 向上传播
  ├─ watch() 函数通过 ? 返回 Err
  └─ main() 打印错误 → 进程退出
```

---

## 7. 重编译链路细节

### 7.1 compile_once 完整流程

**文件**: `crates/typst-cli/src/compile.rs` [compile.rs#L258-L314](crates/typst-cli/src/compile.rs#L258-L314)

```
compile_once(world, config)
  │
  ├─ [watching] Status::Compiling.print()  // 清屏 + 输出 watching 标题
  │
  ├─ compile_and_export(world, config)
  │    ├─ 根据 output_format 选择编译目标
  │    │    ├─ Paged (PDF/PNG/SVG): typst::compile::<PagedDocument>(world)
  │    │    ├─ Html:                   typst::compile::<HtmlDocument>(world)
  │    │    └─ Bundle:                 typst::compile::<Bundle>(world)
  │    │
  │    └─ 导出到文件
  │         ├─ PDF:  typst_pdf::pdf() → write
  │         ├─ PNG:  typst_render::render() → encode_png() → write
  │         │       └─ ExportCache: 页面内容未变则跳过写入
  │         ├─ SVG:  typst_svg::svg() → write
  │         ├─ HTML: typst_html::html() → write (+ server.set_html)
  │         └─ Bundle: 写虚拟文件系统 (+ server.set_bundle)
  │
  ├─ 合并 config.warnings（静态警告）
  │
  ├─ 根据结果打印状态
  │    ├─ 成功 + 无警告: Status::Success(duration)
  │    ├─ 成功 + 有警告: Status::PartialSuccess(duration)
  │    └─ 失败:          Status::Error + set_failed()
  │
  ├─ 打印诊断信息 (print_diagnostics)
  │
  ├─ [首次编译且 --open] 打开输出文件
  │
  └─ [--deps] 写入依赖列表
```

### 7.2 图片导出缓存

**文件**: `crates/typst-cli/src/compile.rs` [compile.rs#L640-L671](crates/typst-cli/src/compile.rs#L640-L671)

```rust
if config.watching
    && config.export_cache.is_cached(i, page)  // page 帧哈希与上次相同
    && path.exists()                            // 文件也存在
{
    return Ok(Output::Path(path.to_path_buf())); // 跳过重新导出
}
```

`is_cached()` 内部比较页面帧的 128 位哈希，不变则直接跳过写入，减少磁盘 IO。

### 7.3 comemo 缓存清理

```rust
comemo::evict(10);
```

每轮编译后调用 `comemo::evict(10)`，将 comemo 增量编译缓存中引用计数 ≤10 的条目淘汰。comemo 是 Typst 的增量编译框架。

---

## 8. 完整链路时序图

```
用户执行 typst watch input.typ
  │
  ├─ CompileConfig::watching()           构造配置
  ├─ Watcher::new(output)                创建文件监听器（notify-rs + mpsc）
  ├─ loop 等待 SystemWorld::new 成功      若文件缺失则先监听再 wait()
  ├─ world.scan_fonts()                  预加载字体
  └─ 首次 compile_once()                 初始编译
        │
        ▼
┌─ 监听主循环 ◄─────────────────────────────────────────────────────────┐
│     │                                                                 │
│     ├─ watcher.update(world.dependencies())                           │
│     │    └─ Mark-and-Sweep: 增删监听路径 / 维护 missing 集合            │
│     │                                                                 │
│     ├─ watcher.wait()                                                  │
│     │    ├─ recv_timeout(MAX or 300ms)    等待首个事件或轮询超时        │
│     │    ├─ 迭代器链批处理 [100ms 窗口, 上限 500ms]                     │
│     │    │    ├─ 过滤无关事件类型                                       │
│     │    │    ├─ 处理 Remove/RenameFrom（修复 inotify unwatch）         │
│     │    │    └─ 过滤输出文件变化（防止自触发）                          │
│     │    └─ relevant 或 missing 有文件存在 → 退出等待                   │
│     │                                                                 │
│     ├─ world.reset()                    清除文件缓存 + 刷新时间戳       │
│     │                                                                 │
│     ├─ compile_once()                   重编译 + 导出 (+ 图片缓存)      │
│     │    ├─ 编译成功 → 写入成功 → Status::Success/PartialSuccess        │
│     │    ├─ 编译成功 → 写入失败 → Status::Error + set_failed（不退出！）│
│     │    └─ 编译失败 → Status::Error + set_failed（不退出！）          │
│     │                                                                 │
│     └─ comemo::evict(10)               清理增量编译缓存                │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

**错误恢复路径**（编译失败场景）：
```
  compile_once 编译失败
        │
        ├─ 打印错误诊断（不返回 Err）
        │
        ├─ 返回 Ok(()) → 继续 comemo::evict
        │
        ├─ 回到循环顶
        │
        ├─ watcher.update(dependencies)
        │    └─ 即使编译失败，依赖列表仍然完整
        │
        ├─ watcher.wait() — 正常等待
        │
        └─ 用户修复后保存 → 触发下一轮重编译
```

**错误恢复路径**（输出文件写入失败场景）：
```
  compile_once 编译成功 → export_pdf 写入失败（磁盘满）
        │
        ├─ io::Error → SourceDiagnostic（不直接向上传播）
        │
        ├─ 打印写入错误诊断（不返回 Err）
        │
        ├─ 返回 Ok(()) → 继续 comemo::evict
        │
        ├─ 回到循环顶
        │
        ├─ watcher.update(dependencies)
        │    └─ 编译成功过，依赖列表完整
        │
        ├─ watcher.wait() — 正常等待
        │
        └─ 用户清理磁盘 → 再次保存 → 写入成功
```
