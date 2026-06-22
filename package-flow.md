# Typst 包管理流程分析

本文从代码实现角度梳理 Typst 的包管理机制，重点说明包下载、版本选择和本地缓存三者之间的关系。

## 一、核心概念与数据结构

### 1.1 包标识体系

包的标识定义在 [`typst-syntax/src/package.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-syntax/src/package.rs) 中。

- **PackageSpec**：完整的包规格，格式为 `@namespace/name:version`
  - `namespace`: 命名空间（如 `preview`）
  - `name`: 包名
  - `version`: 具体版本号

- **VersionlessPackageSpec**：不带版本的包规格，格式为 `@namespace/name`
  - 用于查询最新版本等场景
  - 可通过 `at(version)` 方法补全为 `PackageSpec`

- **PackageVersion**：语义化版本号（major.minor.patch）
  - 支持版本比较（`matches_eq`、`matches_gt`、`matches_lt`、`matches_ge`、`matches_le`）

- **VersionBound**：版本边界，可只指定 major 或 major.minor
  - 用于包的编译器版本兼容性检查

### 1.2 包清单（Package Manifest）

每个包的根目录下有一个 `typst.toml` 文件，描述包的元数据：

- **PackageInfo**：包基本信息
  - 必填：`name`、`version`、`entrypoint`
  - 可选：`authors`、`license`、`description`、`homepage`、`repository`、`keywords`、`categories`、`disciplines`、`compiler`（最低编译器版本）、`exclude`

- **TemplateInfo**：模板信息（可选）
  - `path`: 模板文件目录
  - `entrypoint`: 模板入口文件
  - `thumbnail`: 缩略图

- **ToolInfo**：第三方工具配置区（`[tool.*]` sections）

## 二、包存储的三级结构

包的获取遵循三级优先级机制，定义在 [`typst-kit/src/packages.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs) 的 `SystemPackages` 中。

### 2.1 三级来源及优先级

```
优先级从高到低：
1. Data 目录（系统级用户包目录）
2. Cache 目录（自动下载的缓存）
3. Typst Universe（远程包仓库）
```

### 2.2 Data 目录（用户包目录）

由 [`FsPackages`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L151-L279) 管理，默认路径：

| 操作系统 | 默认路径 |
|---------|---------|
| Linux | `$XDG_DATA_HOME/typst/packages` 或 `~/.local/share/typst/packages` |
| macOS | `~/Library/Application Support/typst/packages` |
| Windows | `%APPDATA%/typst/packages` |

可通过 `--package-path` 命令行参数或 `TYPST_PACKAGE_PATH` 环境变量覆盖。

### 2.3 Cache 目录（下载缓存目录）

同样由 `FsPackages` 管理，默认路径：

| 操作系统 | 默认路径 |
|---------|---------|
| Linux | `$XDG_CACHE_HOME/typst/packages` 或 `~/.cache/typst/packages` |
| macOS | `~/Library/Caches/typst/packages` |
| Windows | `%LOCALAPPDATA%/typst/packages` |

可通过 `--package-cache-path` 命令行参数或 `TYPST_PACKAGE_CACHE_PATH` 环境变量覆盖。

### 2.4 文件系统目录结构

无论是 data 还是 cache 目录，都遵循相同的三级目录结构：

```
packages/
  namespace/          # 一级：命名空间
    package-name/     # 二级：包名
      1.2.3/          # 三级：版本号
        typst.toml    # 包清单
        lib.typ       # 入口文件（由 entrypoint 指定）
        ...           # 其他文件
```

## 三、包获取流程（Obtain Flow）

### 3.1 核心获取逻辑

包的获取入口是 [`SystemPackages::obtain()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L95-L124) 方法，流程如下：

```
obtain(spec)
  │
  ├─→ 检查 Data 目录
  │     ├─ 存在 → 返回 FsRoot
  │     └─ 不存在 → 继续
  │
  ├─→ 检查 Cache 目录
  │     ├─ 存在 → 返回 FsRoot
  │     └─ 不存在 → 继续
  │
  ├─→ 检查是否为 preview 命名空间
  │     ├─ 是 → 从 Universe 下载并缓存
  │     │     ├─ 下载成功 → 存入 cache 目录 → 返回 FsRoot
  │     │     └─ 下载失败 → 报错
  │     └─ 否 → 报错（包未找到）
  │
  └─→ 返回 PackageError::NotFound
```

### 3.2 下载并缓存的原子性保证

下载的包通过 [`FsPackages::store()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L225-L278) 方法存入 cache 目录，采用**临时目录 + 原子重命名**策略保证并发安全：

1. 在目标包目录旁创建临时目录：`.tmp-{version}-{random}`
2. 将包内容解压/写入临时目录
3. 尝试将临时目录重命名为最终的版本号目录
4. 如果目标目录已存在（其他进程已完成下载），则忽略 `DirectoryNotEmpty` 错误

这种设计确保：
- 并发下载不会导致文件损坏
- 不会出现部分下载的包
- 多个 Typst 实例可以安全地并行下载同一个包

## 四、版本选择机制

### 4.1 最新版本查询

[`SystemPackages::latest_version()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L127-L142) 方法用于确定包的最新版本，策略如下：

```
latest_version(spec)
  │
  ├─→ 如果是 preview 命名空间
  │     └─→ 从 Universe 远程查询最新版本
  │
  └─→ 其他命名空间
        └─→ 从本地 data 目录查找最新版本
              （不查 cache 目录，因为 cache 仅用于自动下载）
```

### 4.2 远程版本查询（Universe）

[`UniversePackages::latest_version()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L385-L412) 通过下载包索引来确定最新版本：

1. 下载 `https://packages.typst.org/preview/index.json`
2. 解析 JSON 数组，懒加载每个包的 `name` 和 `version`
3. 过滤出目标包名的所有版本
4. 返回最大版本号

索引文件通过 `OnceCell` 实现**内存缓存**，首次访问时下载，后续复用。

### 4.3 本地版本查询

[`FsPackages::latest_version()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L199-L214) 通过文件系统遍历实现：

1. 读取 `{namespace}/{name}/` 目录下的所有子目录
2. 尝试将每个子目录名解析为 `PackageVersion`
3. 返回解析成功的最大版本号

## 五、下载器架构

下载器定义在 [`typst-kit/src/downloader.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/downloader.rs) 中，采用 trait 抽象 + 装饰器模式。

### 5.1 Downloader trait

核心 trait，定义两个方法：
- `stream(key, url)`：流式下载，返回大小提示和 Reader
- `download(key, url)`：完整下载，返回 Vec<u8>（有默认实现）

`key` 参数是 `&dyn Any` 类型，用于标识下载内容，供进度报告器决定是否显示进度。

### 5.2 SystemDownloader

系统内置的 HTTPS 下载器，特性：
- 使用 `ureq` HTTP 客户端
- 使用 `native-tls` 进行 TLS 加密
- 支持系统代理环境变量
- 支持自定义 CA 证书
- 404 响应映射为 `io::ErrorKind::NotFound`

### 5.3 ProgressDownloader

装饰器模式的进度下载器，包装另一个 Downloader：
- 通过 `ProgressReporter` trait 报告下载进度
- 周期性采样计算下载速度
- 支持 ETA 估算
- 通过 `key` 判断是否需要显示进度

CLI 中的实现见 [`typst-cli/src/download.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-cli/src/download.rs)：
- `PackageSpec` 类型的 key 会显示进度
- `"release"` 字符串 key 会显示进度
- 其他 key（如 `"package index"`）不显示进度

## 六、包导入流程（Import Flow）

包的导入逻辑在 [`typst-eval/src/import.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-eval/src/import.rs) 中。

### 6.1 导入触发

当代码中出现 `import "@preview/package:1.0.0"` 时：

1. [`import()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-eval/src/import.rs#L212-L223) 函数检测到 `@` 开头的路径
2. 解析为 `PackageSpec`
3. 调用 `import_package()`

### 6.2 包解析

[`resolve_package()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-eval/src/import.rs#L258-L284) 函数负责：

1. 构造包根目录下 `typst.toml` 的 `FileId`
2. 通过 `engine.world.file()` 读取清单文件
   - 这会触发 `SystemWorld` → `FileStore` → `SystemFiles` → `SystemPackages::obtain()` 的调用链
   - 如果包不在本地，会自动下载
3. 解析 TOML 为 `PackageManifest`
4. 调用 `manifest.validate(&spec)` 验证：
   - 包名匹配
   - 版本匹配
   - 编译器版本兼容
5. 根据 `entrypoint` 字段解析入口文件路径
6. 返回包名和入口文件的 `FileId`

### 6.3 文件加载

文件加载的调用链（从世界抽象到实际文件）：

```
World::file(id) / World::source(id)
  ↓
SystemWorld (typst-cli)
  ↓
FileStore<SystemFiles> (typst-kit/files.rs)
  ↓
SystemFiles::load() (typst-cli/world.rs 或 typst-kit/files.rs)
  ↓
  ├─ VirtualRoot::Project → 从项目目录加载
  └─ VirtualRoot::Package(spec) → SystemPackages::obtain(spec)
                                    ↓
                                  三级查找（data → cache → download）
```

[`FileStore`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/files.rs#L36-L126) 还提供了：
- 内存缓存（避免重复读取文件）
- 增量编译支持（stale source 复用）
- 依赖追踪（`dependencies()` 方法）

## 七、完整流程示例

以 `#import "@preview/example:1.0.0": *` 为例，完整流程：

```
1. 解析 import 语句，识别出 @ 开头的包路径
   ↓
2. 解析为 PackageSpec { namespace: "preview", name: "example", version: "1.0.0" }
   ↓
3. 构造 typst.toml 的 FileId（VirtualRoot::Package + "typst.toml"）
   ↓
4. 通过 World::file() 请求文件
   ↓
5. SystemPackages::obtain(spec) 开始三级查找：
   a. 检查 data 目录：%APPDATA%/typst/packages/preview/example/1.0.0/
      → 不存在，继续
   b. 检查 cache 目录：%LOCALAPPDATA%/typst/packages/preview/example/1.0.0/
      → 不存在，继续
   c. 命名空间是 preview，从 Universe 下载：
      → 下载 https://packages.typst.org/preview/example-1.0.0.tar.gz
      → 解压到临时目录 .tmp-1.0.0-xxx/
      → 原子重命名为 1.0.0/
      → 返回 FsRoot
   ↓
6. 读取 typst.toml，解析为 PackageManifest
   ↓
7. 验证清单：名称、版本、编译器兼容性
   ↓
8. 根据 entrypoint 找到入口文件（如 lib.typ）
   ↓
9. 加载并评估入口文件
   ↓
10. 返回 Module，供 import 使用
```

## 八、异常路径分析

### 8.1 缓存目录未配置时的下载行为

缓存目录（`cache`）是自动下载的必要前提。在 [`SystemPackages::obtain()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L95-L124) 中，下载逻辑完全包裹在 `if let Some(cache) = &self.cache` 块内：

```
obtain(spec)
  │
  ├─ data 目录检查
  │
  └─ if cache.is_some()   ← 整个下载逻辑在这个条件内
        ├─ cache 目录检查
        └─ 下载并缓存
```

**结论**：当缓存目录未配置（`cache` 为 `None`）时：
- **不会触发下载**，即使包名属于 `preview` 命名空间
- 只会在 data 目录中查找
- 找不到时返回 `PackageError::NotFound`

**什么情况下 cache 会是 None**：
- `dirs::cache_dir()` 返回 `None`（无可用的系统缓存目录）
- 调用方通过 `from_parts()` 显式传入 `None`

注意：`data` 目录为 `None` 不影响下载，只要 `cache` 存在即可。

### 8.2 下载失败的影响

下载过程可能在多个阶段失败，每种失败对版本选择和后续加载的影响不同。

#### 8.2.1 包下载失败

在 [`UniversePackages::package()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L351-L380) 中，失败场景和对应的错误类型：

| 失败场景 | 错误类型 | 对 `obtain()` 的影响 | 对版本选择的影响 |
|---------|---------|---------------------|----------------|
| 命名空间不是 `preview` | `NotFound` | 下载分支直接返回错误，`obtain()` 透传 | — |
| HTTP 404 包不存在 | `NotFound`（在确认无该包时构造） | 沿 `?` 透传 | 已触发 `latest_version()` 查询，尝试确认版本 |
| HTTP 404 版本不存在（但包存在） | `VersionNotFound(spec, latest)` | 沿 `?` 透传，错误消息包含最新版本号 | 同一调用内已完成查询 |
| 网络错误（超时、连接拒绝、DNS 失败等） | `NetworkFailed(inner)` | 沿 `?` **直接透传**，不会变成 `NotFound` | 不影响 `OnceCell` 索引缓存状态 |

**关键细节**：当返回 404 时，代码会额外调用 `self.latest_version()` 来判断是包不存在还是版本不存在：
- 如果能找到该包的其他版本 → `PackageError::VersionNotFound(spec, latest_version)`
- 如果包本身不存在 → `PackageError::NotFound(spec)`

这意味着下载失败（404）时会**额外触发一次索引查询**，但索引查询的结果仅用于生成更友好的错误消息，不会改变错误本身。

网络错误（非 404）不会触发额外查询，直接返回 `NetworkFailed`。

#### 8.2.2 包索引下载失败

[`UniversePackages::latest_version()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L385-L412) 依赖 `index.json` 的下载。索引下载失败时：

- `latest_version()` 返回 `Err`，无法获取最新版本号
- `OnceCell` 不会被填充，下次调用会**重新尝试下载**（不会缓存失败状态）
- 对于模板创建等需要先确定版本的场景，会直接失败
- 对于明确指定版本的包导入（`@preview/pkg:1.0.0`），**不受影响**，因为 `obtain()` 不依赖索引

#### 8.2.3 解压失败（归档损坏）

下载成功后，在 [`cache.store()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L111-L115) 的回调中解压 `tar.gz` 归档可能失败：

- 错误类型：`PackageError::MalformedArchive`
- 此时临时目录中可能有部分解压的文件
- 由于 [`Tempdir`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L282-L307) 的 `Drop` 实现，临时目录会被自动清理
- **不会污染 cache 目录**（因为还没执行重命名）
- 后续重新调用 `obtain()` 会重新下载

### 8.3 缓存损坏的影响

#### 8.3.1 哪些情况算"缓存损坏"

缓存损坏指缓存目录中包的文件不完整或内容错误，包括：
- `typst.toml` 缺失或格式错误
- `entrypoint` 指定的文件不存在
- 部分文件缺失
- 文件内容损坏

#### 8.3.2 损坏检测时机

[`FsPackages::obtain()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L191-L195) 仅通过 `dir.exists()` 判断包是否存在，**不做任何完整性检查**：

```rust
pub fn obtain(&self, spec: &PackageSpec) -> Option<FsRoot> {
    let subdir = eco_format!("{}/{}/{}", spec.namespace, spec.name, spec.version);
    let dir = self.path().join(subdir.as_str());
    dir.exists().then_some(FsRoot::new(dir))
}
```

因此，缓存损坏**不会在 `obtain()` 阶段被发现**，而是在后续文件加载时才暴露：

1. `resolve_package()` 读取 `typst.toml` 时
   - 文件不存在 → `FileError::NotFound`
   - TOML 格式错误 → "package manifest is malformed"
   - 名称/版本不匹配 → 验证失败
2. 加载 `entrypoint` 指定的入口文件时
   - 文件不存在 → `FileError::NotFound`

#### 8.3.3 损坏后的行为

**缓存损坏不会触发自动重新下载**。原因是：
- `obtain()` 看到目录存在就直接返回 `FsRoot`
- 后续文件加载失败发生在 `World::file()` / `World::source()` 层
- 该层没有机制通知 `SystemPackages` "这个包坏了，请重新下载"

用户需要手动删除损坏的缓存目录来触发重新下载。

#### 8.3.4 Data 目录 vs Cache 目录的损坏行为

两者行为一致：都只检查目录存在性，不验证完整性。但 data 目录是用户手动管理的，损坏时预期用户自行修复。

### 8.4 临时目录清理

#### 8.4.1 清理机制

临时目录由 [`Tempdir`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L282-L307) 结构体管理，通过 `Drop` trait 实现自动清理：

```rust
impl Drop for Tempdir {
    fn drop(&mut self) {
        _ = std::fs::remove_dir_all(&self.0);
    }
}
```

**正常流程**：
1. 创建临时目录 `.tmp-{version}-{random}`
2. 解压/写入包内容
3. 原子重命名为目标版本号目录
4. `Tempdir` 被 drop 时，尝试删除原路径（已被重命名，路径不存在，忽略错误）

**异常流程（下载/解压失败）**：
1. 创建临时目录
2. 解压过程中出错
3. 函数返回错误，`Tempdir` 离开作用域
4. `drop` 被调用，删除整个临时目录

#### 8.4.2 清理失败的情况

以下情况临时目录可能**残留**：

- **进程被强制终止**（如 `kill -9`、断电、蓝屏）：`Drop` 不会执行
- **文件系统错误**：`remove_dir_all` 可能因权限等原因失败（错误被 `_ =` 静默忽略）

残留的临时目录命名为 `.tmp-{version}-{u32}`，位于包的版本目录同级。

#### 8.4.3 残留临时目录的影响

- **不影响包查找**：`obtain()` 只查找严格匹配版本号的目录（如 `1.2.3`），`.tmp-1.2.3-xxxx` 不会被误认为有效包
- **不影响后续下载**：每次下载使用新的随机数，不会与残留目录冲突
- **占用磁盘空间**：残留的临时目录会一直占用空间，直到用户手动清理
- **不影响版本选择**：`latest_version()` 解析目录名为版本号，`.tmp-*` 目录解析失败会被过滤掉

### 8.5 错误传播路径

在 [`SystemPackages::obtain()`](file:///d:/fz/0601-2/solo-dogfeeding/code/128-typst/crates/typst-kit/src/packages.rs#L95-L124) 中，`self.universe.package(spec)?` 通过 `?` 运算符**直接传播**来自 `UniversePackages::package()` 的错误，不做转换：

```rust
// Line 109: ? 直接透传错误，不做映射
let mut archive = self.universe.package(spec)?;
```

只有当代码完全跳过下载逻辑（如 cache 为 None、或命名空间不是 preview）并走到函数末尾时，才会构造返回：

```rust
// Line 123: 仅在所有来源都跳过/不存在时触发
Err(PackageError::NotFound(spec.clone()))
```

因此，`NotFound` 只代表"没有可用的来源去找这个包"，而 `NetworkFailed`/`VersionNotFound` 等则来自实际下载尝试。

### 8.6 对版本选择的综合影响

| 异常情况 | 对 `latest_version()` 的影响 | 对 `obtain()` 的影响 |
|---------|----------------------------|---------------------|
| data 目录未配置 | 非 preview 命名空间返回错误 | 正常降级，仅跳过 data 检查 |
| cache 目录未配置 | 不影响（不查 cache） | **无法下载**，走到函数末尾返回 `NotFound` |
| 命名空间不是 preview + 本地未找到 | — | 跳过下载分支，返回 `NotFound` |
| 网络不可用（连接超时等） | preview 命名空间返回字符串错误（包装底层 io 错误） | `NetworkFailed` 沿 `?` 透传返回 |
| HTTP 404 包不存在 | preview 命名空间：包不存在则返回错误 | `NotFound`（由 404 分支构造） |
| HTTP 404 版本不存在 | preview 命名空间：可查到最新版本号 | `VersionNotFound(spec, latest)` |
| 索引下载失败 | 返回错误，**下次重试**（OnceCell 不填） | 不影响（指定明确版本时不走 `latest_version`） |
| 归档解压失败 | 不影响 | `MalformedArchive` 沿 `?` 透传返回 |
| 缓存目录损坏 | 不影响（不查 cache） | 返回 FsRoot，但后续加载文件时报错 |
| 临时目录残留 | 无影响（解析失败被过滤） | 无影响 |

## 九、关键设计要点

### 9.1 三级缓存设计

- **Data 目录**：用户手动放置的包，优先级最高，不会被自动修改
- **Cache 目录**：自动下载的包，可随时清理，不影响用户数据
- **内存缓存**：`FileStore` 的内存缓存和 `UniversePackages` 的索引缓存，提升重复访问性能

### 9.2 并发安全

- 下载使用临时目录 + 原子重命名，避免部分下载的包
- `FileStore` 使用 `Mutex` 保护内部哈希表

### 9.3 版本策略

- 明确指定版本的包（`@preview/pkg:1.0.0`）直接使用指定版本
- 未指定版本时（如模板创建），`preview` 命名空间查远程，其他查本地 data 目录
- 包清单中可指定最低编译器版本，运行时进行兼容性检查

### 9.4 可扩展性

- `Downloader` trait 允许自定义下载实现
- `FsPackages` 可自定义 data/cache 路径
- `SystemPackages::from_parts()` 可灵活组合三个来源

### 9.5 失败处理原则

- **缓存目录是下载的前提**：没有 cache 就不下载，避免"下载了但没地方存"的问题
- **错误分层，沿 `?` 透传**：`SystemPackages::obtain()` 中 `universe.package()` 的错误通过 `?` 直接向上传播，不会被吞掉或改写成 `NotFound`
  - 网络不可用/超时 → `NetworkFailed`
  - 404 版本不存在 → `VersionNotFound`（带最新版本号提示）
  - 404 包不存在 → `NotFound`
- **`NotFound` 仅用于"没有来源可找"**：函数末尾的 `PackageError::NotFound` 仅在本地未找到 + 完全跳过了下载分支（cache 为 None 或 namespace 不是 preview）时触发
- **目录存在 ≠ 包有效**：`obtain()` 只做存在性检查，完整性校验延迟到文件加载阶段
- **错误静默降级**：`dirs::cache_dir()` 等系统调用失败时，静默降级为 None，不导致整体崩溃
- **临时目录自清理**：正常 panic 和错误路径下 Tempdir 都能自动清理，仅极端情况（强杀进程）可能残留
