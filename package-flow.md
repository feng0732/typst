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

## 八、关键设计要点

### 8.1 三级缓存设计

- **Data 目录**：用户手动放置的包，优先级最高，不会被自动修改
- **Cache 目录**：自动下载的包，可随时清理，不影响用户数据
- **内存缓存**：`FileStore` 的内存缓存和 `UniversePackages` 的索引缓存，提升重复访问性能

### 8.2 并发安全

- 下载使用临时目录 + 原子重命名，避免部分下载的包
- `FileStore` 使用 `Mutex` 保护内部哈希表

### 8.3 版本策略

- 明确指定版本的包（`@preview/pkg:1.0.0`）直接使用指定版本
- 未指定版本时（如模板创建），`preview` 命名空间查远程，其他查本地 data 目录
- 包清单中可指定最低编译器版本，运行时进行兼容性检查

### 8.4 可扩展性

- `Downloader` trait 允许自定义下载实现
- `FsPackages` 可自定义 data/cache 路径
- `SystemPackages::from_parts()` 可灵活组合三个来源
