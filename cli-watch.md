# CLI Watch 监听、防抖与重编译链路分析

按代码执行顺序，完整梳理 `typst watch` 命令从启动到监听文件变化、防抖合并、触发重编译的全过程。

---

## 1. 命令入口：CLI 参数解析与分发

### 1.1 进程入口

**文件**: [main.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/main.rs#L50-L66)

```
main()
  └─ sigpipe::reset()          // 处理管道信号
  └─ dispatch()                // 命令分发
```

### 1.2 命令分发

**文件**: [main.rs#L69-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/main.rs#L69-L82)

```rust
fn dispatch() -> HintedStrResult<()> {
    match &ARGS.command {
        Command::Watch(command) => crate::watch::watch(command)?,
        // ... 其他命令
    }
}
```

命令定义在 [args.rs#L122-L133](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/args.rs#L122-L133)，`WatchCommand` 包含编译参数和可选的 HTTP 服务器参数。

---

## 2. 监视模式初始化阶段

**文件**: [watch.rs#L18-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/watch.rs#L18-L84)

### 2.1 步骤 1：创建编译配置

```rust
let mut config = CompileConfig::watching(command)?;
```

调用 [compile.rs#L98-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/compile.rs#L98-L100) 的 `CompileConfig::watching()`，内部通过 `new_impl(args, Some(command))` 构造配置：

- 推断输出格式（PDF/PNG/SVG/HTML/Bundle）
- 校验 watch 模式下不允许 stdout 输出
- 可选地创建 HTTP 服务器（HTML/Bundle 导出时）
- 设置 `config.watching = true` 标记

### 2.2 步骤 2：创建文件系统监听器

```rust
let mut watcher = Watcher::new(Some(output.clone()))?;
```

**文件**: [watcher.rs#L47-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-kit/src/watcher.rs#L47-L70)

`Watcher::new()` 内部：
1. 创建 MPSC 通道 `(tx, rx)` 用于 notify-rs 事件传递
2. 配置 `notify::Config`，将轮询间隔从默认 ~30s 缩短为 `POLL_INTERVAL = 300ms`
3. 构造 `RecommendedWatcher`（底层为 notify-rs，根据系统选 inotify/fsevents/kqueue）
4. 初始化四个核心字段：
   - `output: Option<PathBuf>`：输出文件路径（变化时忽略）
   - `watched: FxHashMap<PathBuf, bool>`：已监听路径集合 + 标记位
   - `missing: FxHashSet<PathBuf>`：尚不存在、需手动轮询的路径
   - `rx: Receiver<notify::Result<Event>>`：事件接收端

### 2.3 步骤 3：创建编译世界（SystemWorld）

**文件**: [watch.rs#L31-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/watch.rs#L31-L49)

```rust
let mut world = loop {
    match SystemWorld::new(...) {
        Ok(world) => break world,
        Err(WorldCreationError::InputNotFound(path) | WorldCreationError::RootNotFound(path)) => {
            // 关键：如果输入文件/根目录不存在，先监听它们，然后等待
            watcher.update([path.clone()])?;  // 先把缺失文件加入监听
            Status::Error.print(...);
            print_error(...);
            watcher.wait()?;                  // 阻塞等待文件被创建
        }
        Err(err) => return Err(err.into()),
    }
};
```

这是一个重要的设计：**即使文件还不存在，watch 模式也不会退出**，而是把缺失路径加入监听后进入等待，直到文件被创建。

### 2.4 步骤 4：预扫描字体 + 首次编译

```rust
if config.output_format.is_paged() {
    world.scan_fonts();   // 预先强制加载字体，不计入编译耗时
}

// 首次编译
timer.record(&mut world, |world| compile_once(world, &mut config))??;
```

`Timer::record` 的实现见 [timer.rs#L57-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-kit/src/timer.rs#L57-L95)，若 path 为 None 则直接执行闭包（placeholder 模式）。

---

## 3. 核心循环：监听 → 防抖 → 重编译

**文件**: [watch.rs#L68-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/watch.rs#L68-L83)

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

**文件**: [world.rs#L97-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/world.rs#L97-L101)

```rust
pub fn dependencies(&mut self) -> impl Iterator<Item = PathBuf> + '_ {
    let (loader, deps) = self.files.dependencies();
    deps.filter_map(|id| loader.resolve(id).ok())
}
```

从 `FileStore` 中取出上次编译过程中实际访问过的所有 `FileId`，解析为文件系统路径。这意味着 **每次编译后都会动态刷新监听范围** —— 如果代码中 `#import` 了新文件，下一轮监听会自动包含它们。

### 4.2 更新监听器（Mark-and-Sweep 策略）

**文件**: [watcher.rs#L76-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-kit/src/watcher.rs#L76-L118)

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

## 5. 等待与防抖机制（核心）

**文件**: [watcher.rs#L121-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-kit/src/watcher.rs#L121-L192)

```rust
pub fn wait(&mut self) -> StrResult<()> {
    loop {
        // ========== 阶段一：等待首个事件 ==========
        let first = self.rx.recv_timeout(
            if self.missing.is_empty() { Duration::MAX } else { Self::POLL_INTERVAL }
        );

        // ========== 阶段二：批处理/防抖（Batching） ==========
        let mut relevant = false;
        let batch_start = Instant::now();

        for event in first.into_iter()
            .chain(iter::from_fn(|| self.rx.recv_timeout(Self::BATCH_TIMEOUT).ok()))
            .take_while(|_| batch_start.elapsed() <= Self::STARVE_TIMEOUT)
        {
            let event = event?;

            // 5.1 过滤无关事件类型
            if !is_relevant_event_kind(&event.kind) { continue; }

            // 5.2 inotify Remove/RenameFrom 处理
            if matches!(event.kind,
                Remove(File) | Modify(Name(RenameMode::From)))
            {
                for path in &event.paths {
                    // notify-rs 的 inotify 后端在文件被删除/重命名时会隐式 unwatch，
                    // 这里主动从 watched 移除，下次 update() 时可重新注册
                    self.watcher.unwatch(path).ok();
                    self.watched.remove(path);
                }
            }

            // 5.3 过滤输出文件自身的变化（防止编译输出触发死循环）
            if let Some(output) = &self.output
                && event.paths.iter().all(|p| is_same_file(p, output).unwrap_or(false))
            {
                continue;
            }

            relevant = true;
        }

        // ========== 阶段三：检查触发条件 ==========
        // 条件 A：批处理中发现了相关事件
        // 条件 B：missing 集合中有任何路径现在存在了（用户创建了缺失文件）
        if relevant || self.missing.iter().any(|p| p.exists()) {
            return Ok(());
        }

        // 否则继续外层 loop，重新等待
    }
}
```

### 5.1 三个关键时间常量

| 常量 | 值 | 作用 |
|------|-----|------|
| `BATCH_TIMEOUT` | 100ms | 接收到事件后，继续等待后续连续事件的窗口。编辑器保存文件时通常会产生 Remove+Create/Modify 多个事件，100ms 窗口将它们合并为一次触发 |
| `STARVE_TIMEOUT` | 500ms | 批处理的最大持续时间。防止大量文件修改时，因事件不断涌入而永远在 BATCH_TIMEOUT 内收不到超时信号，导致迟迟不触发编译 |
| `POLL_INTERVAL` | 300ms | 两种用途：① 缺失文件（missing 集合）的存在性轮询间隔；② notify-rs PollWatcher 后端的轮询间隔 |

### 5.2 事件类型过滤

**文件**: [watcher.rs#L196-L211](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-kit/src/watcher.rs#L196-L211)

```
被认为相关的事件：
  ├─ Any
  ├─ Create(_)
  ├─ Modify::Any
  ├─ Modify::Data(_)    // 内容修改（最常见）
  ├─ Modify::Name(_)    // 重命名（某些编辑器的原子保存）
  └─ Remove(_)

被忽略的事件：
  ├─ Access(_)          // 访问/读取
  ├─ Modify::Metadata(_)// 元数据（权限等）变化
  ├─ Modify::Other
  └─ Other
```

### 5.3 防抖算法的迭代器构造

最核心的批处理用迭代器链实现，值得拆解：

```
first                        // 第一个事件（Option 转 iter，可能为空）
  .into_iter()
  .chain(                    // 接上后续持续等待的事件
     iter::from_fn(||        // 反复调用闭包产生新事件
        rx.recv_timeout(BATCH_TIMEOUT=100ms).ok()
        // 100ms 内没新事件 → 返回 None → 迭代结束
     )
  )
  .take_while(|_|            // 强制截止：最多 500ms
     batch_start.elapsed() <= STARVE_TIMEOUT=500ms
  )
```

**示例场景**：编辑器保存（典型流程 `unlink → write → rename`）
```
T+0ms:   Remove 事件到达 → 进入批处理，batch_start 记录
T+10ms:  Create 事件到达 → 在 100ms 窗口内，继续
T+30ms:  Modify(Data) 事件到达 → 继续
T+130ms: BATCH_TIMEOUT=100ms 超时 → 迭代链终止
         累计发现相关事件 → relevant=true → 返回触发编译
```

---

## 6. 世界重置

**文件**: [world.rs#L104-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/world.rs#L104-L107)

```rust
pub fn reset(&mut self) {
    self.files.reset();   // 清除 FileStore 的缓存（源码/二进制缓存）
    self.now.reset();     // 重置时间戳（Time::System 模式下刷新为当前时间）
}
```

---

## 7. 重编译链路

**文件**: [compile.rs#L258-L314](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/compile.rs#L258-L314)

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

### 7.1 图片导出缓存机制

**文件**: [compile.rs#L640-L671](file:///d:/fz/0601-2/solo-dogfeeding/code/132-typst/crates/typst-cli/src/compile.rs#L640-L671)

```rust
// export_image 中：
if config.watching
    && config.export_cache.is_cached(i, page)  // page 帧哈希与上次相同
    && path.exists()                            // 文件也存在
{
    return Ok(Output::Path(path.to_path_buf())); // 跳过重新导出
}
```

`is_cached()` 内部比较页面帧的 128 位哈希，不变则直接跳过写入，减少磁盘 IO。

---

## 8. 缓存清理

```rust
comemo::evict(10);
```

每轮编译后调用 `comemo::evict(10)`，将 comemo 增量编译缓存中引用计数 ≤10 的条目淘汰。comemo 是 Typst 的增量编译框架，缓存过多会占用内存。

---

## 完整链路时序图

```
用户执行 typst watch input.typ
  │
  ├─ CompileConfig::watching()        构造配置
  ├─ Watcher::new(output)             创建文件监听器（notify-rs + mpsc）
  ├─ loop 等待 SystemWorld::new 成功   若文件缺失则先监听再 wait()
  ├─ world.scan_fonts()               预加载字体
  └─ 首次 compile_once()              初始编译
        │
        ▼
┌─ 监听主循环 ◄───────────────────────────────────────────────────┐
│     │                                                           │
│     ├─ watcher.update(world.dependencies())                     │
│     │    └─ Mark-and-Sweep: 增删监听路径 / 维护 missing 集合      │
│     │                                                           │
│     ├─ watcher.wait()                                            │
│     │    ├─ recv_timeout(MAX or 300ms)  等待首个事件或轮询超时    │
│     │    ├─ 迭代器链批处理 [100ms 窗口, 上限 500ms]               │
│     │    │    ├─ 过滤无关事件类型                                 │
│     │    │    ├─ 处理 Remove/RenameFrom（修复 inotify unwatch）   │
│     │    │    └─ 过滤输出文件变化（防止自触发）                    │
│     │    └─ relevant 或 missing 有文件存在 → 退出等待             │
│     │                                                           │
│     ├─ world.reset()              清除文件缓存 + 刷新时间戳       │
│     │                                                           │
│     ├─ compile_once()             重编译 + 导出 (+ 图片缓存)      │
│     │                                                           │
│     └─ comemo::evict(10)          清理增量编译缓存                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
