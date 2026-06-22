# 颜色空间与图像解码、布局渲染的协作脉络

本文梳理 Typst 中 **颜色空间（Color Space）**、**图像解码与缓存（Image Decode & Cache）** 和 **布局渲染（Layout & Render）** 三大模块的协作路径。

---

## 1. 总体架构概览

整个流程可以分为 5 个阶段：

```
源码解析 (typst-eval)
       │
       ▼
① 颜色 / 图像元素构造 (typst-library::visualize)
       │
       ▼
② 图像解码 (ImageElem::decode)  ──►  图像缓存 (comemo::memoize)
       │
       ▼
③ 布局阶段 (typst-layout)         ──►  Frame::push(FrameItem::Image)
       │
       ▼
④ 渲染 / 导出 (typst-render / typst-pdf / typst-svg / typst-html)
       │
       ▼
   最终产物 (PNG / PDF / SVG / HTML)
```

关键的跨模块协作点：
- **颜色**在构造、布局、渲染三阶段都存在「空间转换」
- **图像**在 `decode()` 时首次解码，并通过 `#[comemo::memoize]` 跨渲染复用
- **渐变**在渲染时才逐像素采样，并在采样过程中进行颜色空间混合

---

## 2. 颜色空间系统

核心文件：[color.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs)

### 2.1 颜色类型层级

```rust
enum Color {
    Process(ProcessColor),   // 可互相混合的过程色
    Spot(SpotColor),         // 专色（印刷色）
}

enum ProcessColor {
    Luma(Luma),              // 灰度 (D65 Gray)
    Oklab(Oklab),            // 感知均匀 Lab
    Oklch(Oklch),            // 感知均匀 LCh
    Rgb(Rgb),                // sRGB (Gamma 校正)
    LinearRgb(LinearRgb),    // 线性 RGB (无 Gamma)
    Cmyk(Cmyk),              // CMYK
    Hsl(Hsl),                // HSL
    Hsv(Hsv),                // HSV
}
```

底层类型来自 `palette` crate，并通过类型别名暴露（[color.rs#L22-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L22-L28)）：

```rust
pub type Oklab = palette::oklab::Oklaba<f32>;
pub type Rgb   = palette::rgb::Rgba<encoding::Srgb, f32>;
pub type Luma  = palette::luma::Lumaa<encoding::Srgb, f32>;
// ...
```

### 2.2 跨颜色空间转换路径

所有 `ProcessColor` 都实现了 `to_space(ProcessColorSpace) -> Self`，其核心实现见 [color.rs#L1758-L1873](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L1758-L1873)。

**转换枢纽（Hub & Spoke 模型）**：

```
       ┌──────────────┐
       │  Oklab / sRGB  │  ← 通用中转空间
       └──────┬───────┘
    ┌─────────┼──────────┐
    ▼         ▼          ▼
  Luma      HSL/Hsv   LinearRgb
    │                    ▲
    │    ┌───────────────┘
    ▼    ▼
  Oklch ─► Rgb(sRGB) ─► Cmyk (ICC 转换)
```

关键实现：
- **Oklab / sRGB / Luma / HSL / HSV** 之间：使用 `palette::FromColor` trait 自动完成
- **CMYK → sRGB**：使用 **moxcms** 库进行 ICC profile 转换（[color.rs#L36-L56](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L36-L56)）
  ```rust
  static CMYK_TO_XYZ: LazyLock<ColorProfile> = ...;  // CGATS TR 001-1995
  static SRGB_PROFILE: LazyLock<ColorProfile> = ...;
  static TO_SRGB: LazyLock<Arc<moxcms::Transform8BitExecutor>> = ...;
  ```
- **sRGB → CMYK**：目前仍使用朴素公式（[color.rs#L2194-L2211](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L2194-L2211)）
- **Spot Color → 过程色**：使用 `SpotColor::fallback()`，即 tint × fallback

### 2.3 颜色混合（mix_iter）

在 [color.rs#L1136-L1236](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L1136-L1236) 中实现，流程：

1. **resolve_color_space** 自动选择混合空间（默认 Oklab，spot 色使用自己的色板）
2. **将所有颜色 to_space(target)** → 统一 vec4
3. 加权平均（对色相空间，取色环上**短路径**插值）
4. **to_space(origin)** → 转回源空间（用于 negate、hue rotate 等操作）

---

## 3. 图像解码与缓存

核心文件：
- [image/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs) — 图像元素定义与 decode() 入口
- [image/raster.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs) — 光栅图像解码

### 3.1 图像元素与解码入口

`ImageElem` 元素（[mod.rs#L52-L204](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs#L52-L204)）字段：

```rust
struct ImageElem {
    source: Derived<DataSource, Loaded>,  // 数据源 + 已加载字节
    format: Smart<ImageFormat>,           // 自动探测或手动指定
    width, height, fit, scaling,
    icc: Smart<Derived<DataSource, Bytes>>, // 可选 ICC profile
    page: NonZeroUsize,                   // PDF 图像的页码
}
```

**解码入口**在 `Packed<ImageElem>::decode()`（[mod.rs#L213-L327](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs#L213-L327)）：

```rust
pub fn decode(&self, engine, styles) -> SourceResult<Image> {
    // 1. 确定格式（扩展名 → 内容嗅探 → 显式指定）
    let format = self.determine_format(styles)?;

    // 2. 分支构造 ImageKind
    let kind = match format {
        ImageFormat::Raster(fmt) => ImageKind::Raster(
            RasterImage::new(data, fmt, icc_profile)?   // ← memoized
        ),
        ImageFormat::Vector(VectorFormat::Svg) => ImageKind::Svg(
            SvgImage::with_fonts_images(data, world, fonts, svg_file)?
        ),
        ImageFormat::Vector(VectorFormat::Pdf) => ImageKind::Pdf(
            PdfImage::new(document, page_idx)?
        ),
    };

    Ok(Image::new(kind, alt, scaling))
}
```

### 3.2 光栅图像解码（RasterImage）

实现于 [raster.rs#L31-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs#L31-L152)，核心函数 `new_impl` **被 `#[comemo::memoize]` 装饰**（同参数第二次调用直接返回缓存）。

#### 两种输入格式：

| 类型 | 解码路径 | 颜色通道提取 |
|------|---------|------------|
| `RasterFormat::Exchange(Png/Jpg/Gif/Webp)` | `image::codecs::*Decoder` | `decoder.icc_profile()` 提取 ICC |
| `RasterFormat::Pixel(Rgb8/Rgba8/Luma8/Lumaa8)` | 原始像素 → `ImageBuffer` | 使用用户传入的 `icc` 参数 |

#### 解码附加处理：
1. **EXIF 旋转**（[raster.rs#L85-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs#L85-L93)）：`exif::Reader` 读取 Orientation tag，然后 `apply_rotation()` 翻转动态图像
2. **DPI 提取**（[raster.rs#L362-L445](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs#L362-L445)）：三级回退 EXIF → JFIF APP0 → PNG pHYs

### 3.3 解码后的 Image 结构

```rust
struct Image(Arc<LazyHash<ImageInner>>);  // cheap clone + 懒哈希

struct ImageInner {
    kind: ImageKind,           // Raster / Svg / Pdf
    alt: Option<EcoString>,
    scaling: Smart<ImageScaling>,
}
```

注意：`Image::new_impl` 也被 `#[comemo::memoize]`（[mod.rs#L427-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs#L427-L434)），因此同一张图即使被多个 `ImageElem` 引用，最终也只构造一次 `ImageInner`。

---

## 4. 布局阶段（Layout）

核心文件：
- [typst-layout/src/image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-layout/src/image.rs) — 图像布局
- [frame.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/layout/frame.rs) — 帧与 FrameItem

### 4.1 `layout_image()` 流程

见 [image.rs#L11-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-layout/src/image.rs#L11-L81)：

```
1. decode() → 取得 Image（首次触发解码，后续走 comemo 缓存）
2. 计算像素宽高比 px_ratio
3. 根据 region.expand.{x,y} 确定目标矩形 target
   ├─ 两者都强制 → 使用 region
   ├─ 只指定宽度 → 等比计算高度
   ├─ 只指定高度 → 等比计算宽度
   └─ 都不指定   → 使用 DPI 计算自然尺寸（default 72 dpi）
4. 按 fit (Cover / Contain / Stretch) 计算 fitted 尺寸
5. Frame::soft(fitted) 推入 FrameItem::Image(image, fitted, span)
6. frame.resize(target, Center) 居中对齐，Cover 模式下 clip()
```

### 4.2 Frame 与 FrameItem

见 [frame.rs#L17-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/layout/frame.rs#L17-L30) 与 [frame.rs#L486-L499](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/layout/frame.rs#L486-L499)：

```rust
struct Frame {
    size: Size,
    items: Arc<LazyHash<Vec<(Point, FrameItem)>>>,  // 位置 → 条目
    kind: FrameKind,  // Soft/Hard（Hard 是渐变/平铺的容器边界）
}

enum FrameItem {
    Group(GroupItem),     // 子帧 + 变换 + 裁剪
    Text(TextItem),       // 成形文字
    Shape(Shape, Span),   // 几何形状（含 fill/stroke Paint）
    Image(Image, Size, Span),  // ← 图像引用 + 目标尺寸
    Link(Destination, Size),
    Tag(Tag),
}
```

关键点：**`Image`（含解码后的数据）被原样搬进 FrameItem，此时不做像素级处理**。真正的缩放、颜色转换都发生在渲染/导出阶段。

---

## 5. 渲染 / 导出阶段

共有 4 种导出路径，颜色与图像处理策略各有不同：

| 后端 | crate | 颜色策略 | 图像策略 |
|------|-------|---------|---------|
| 光栅 PNG | `typst-render` | 全部转 sRGB → `tiny_skia::Color` | 按像素放大到视口尺寸（memoized） |
| PDF | `typst-pdf` | 保留原生空间：Luma/CMYK/RGB/Spot | JPEG 直接嵌入，其他转 `CustomImage` |
| SVG | `typst-svg` | 转 CSS 颜色函数（oklab/hsl 等） | 转 base64 data: URL，PDF 转 SVG |
| HTML | `typst-html` | 同上 | 同上 |

### 5.1 光栅渲染（typst-render）

核心文件：
- [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/lib.rs) — 渲染主循环
- [paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs) — 颜色与 Paint
- [image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/image.rs) — 图像纹理

#### 5.1.1 颜色路径

所有颜色最终都到 `to_sk_color()`（[paint.rs#L286-L295](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs#L286-L295)）：

```rust
fn to_sk_color(color: ProcessColor) -> sk::Color {
    let (r, g, b, a) = color.to_rgb().into_components();  // 强制转 sRGB
    sk::Color::from_rgba(r, g, b, a)
}
```

Spot 颜色先 `Color::to_process()` → 用 fallback 颜色渲染。

#### 5.1.2 渐变采样（Gradient → 光栅）

渐变的实际光栅化发生在 `GradientSampler::sample`（[paint.rs#L61-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs#L61-L78)）或 `cached()`（[paint.rs#L146-L172](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs#L146-L172)）预渲染成 pixmap：

```
gradient.sample_at(pixel_xy, container_size)
    │
    ▼  gradient.rs:908
1. 归一化坐标 (x/width, y/height)
2. 根据渐变类型计算参数 t：
   ├─ Linear: t = x*cos + y*sin
   ├─ Radial: t = sqrt(center_dist) / radius
   └─ Conic : t = atan2(dy, dx)/2π
3. sample_stops(stops, space, t)
    │
    ▼  gradient.rs:1399
    3.1 二分查找左右 stop
    3.2 Color::mix_iter(left, right, space)  ← 按指定颜色空间混合
    │
    ▼  回到 paint.rs
4. color.to_process() → to_sk_color() → premultiply
```

#### 5.1.3 图像光栅化

见 [image.rs#L68-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/image.rs#L68-L113)，`build_texture(image, w, h)` 也是 `#[comemo::memoize]`：

```
按变换矩阵计算最终 w,h（考虑旋转后的采样分辨率）
     │
     ▼ build_texture:
     ├─ Raster: image::DynamicImage::resize_exact
     │   ├─ 放大: CatmullRom
     │   ├─ 缩小: Lanczos3
     │   └─ Pixelated 模式: Nearest
     │
     ├─ SVG: resvg::render(tree, scale) → tiny_skia Pixmap
     │
     └─ PDF: hayro::render(page, ...) → bytemuck → Pixmap
     │
     ▼
Rgba 像素预乘 alpha → 作为 sk::Pattern 的 shader
canvas.fill_rect(rect, &paint, transform, mask)
```

### 5.2 PDF 导出（typst-pdf）

核心文件：
- [paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/paint.rs) — PDF 颜色转换
- [image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/image.rs) — PDF 图像嵌入

#### 5.2.1 颜色策略：**尽量保留原生空间**

见 `convert_process_solid()`（[paint.rs#L124-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/paint.rs#L124-L137)）：

```
ProcessColor  ──►  space 判定
   │
   ├─ D65Gray    → krilla::luma::Color (单通道)
   ├─ CMYK       → krilla::cmyk::Color (四通道, 8bit)
   ├─ 其他所有   → krilla::rgb::Color (sRGB 三通道)
   │
SpotColor     ──► krilla::separation::Color (专色 + fallback)
```

**渐变**在复杂空间（Oklab / Oklch / HSL / HSV / LinearRgb）下会被「降级」为大量 RGB 插值 stops：见 `convert_gradient_stops()`（[paint.rs#L303-L402](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/paint.rs#L303-L402)）中 `generate_intermediate_stops_for_rgb_interpolation`。

#### 5.2.2 图像嵌入策略

见 `convert_raster()`（[image.rs#L190-L211](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/image.rs#L190-L211)）：

```
JPEG ──► Image::from_jpeg_with_icc(原始字节, ICC, interpolate)
           （不解码像素，直接把 JPEG 流塞进 PDF 流）

其他 ──► Image::from_custom(PdfRasterImage)
           ├─ color_channel()：动态保证产出 luma8 或 rgb8
           │   （不支持的格式 → to_rgb8()/to_luma8() 转换）
           ├─ alpha_channel()：单独提取 alpha
           ├─ icc_profile()：仅当格式未被转换时才附加
           └─ color_space()：Rgb / Luma
```

**EXIF 旋转**仅对 JPEG 作为 PDF transform 附加（不修改像素），对其他格式已在解码阶段 baked 进像素（[image.rs#L218-L266](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/image.rs#L218-L266)）。

### 5.3 SVG 导出（typst-svg）

核心文件：
- [paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/paint.rs) — SVG 颜色字符串化
- [image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/image.rs) — SVG 图像转 Base64

#### 5.3.1 颜色输出 CSS 函数（尽量表达丰富）

`SvgDisplay for Color`（[paint.rs#L444-L505](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/paint.rs#L444-L505)）：

| 源颜色空间 | SVG 输出 |
|----------|---------|
| Luma / sRGB / CMYK / HSV | `#rrggbb` / `#rrggbbaa`（hex） |
| LinearRgb | `color(srgb-linear r g b / a)` |
| Oklab | `oklab(L% a b / a)` |
| Oklch | `oklch(L% c h / a)` |
| HSL | `hsl(hdeg s% l% / a)` |

**渐变**：与 PDF 类似，Oklab/Oklch 等非原生空间也生成中间 stops 做 rgb 插值（[paint.rs#L244-L275](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/paint.rs#L244-L275)）。

#### 5.3.2 图像策略：全部转 data: URL

`WebImage::new()`（[image.rs#L100-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/image.rs#L100-L131)）也是 `#[comemo::memoize]`：

```
Raster(Exchange)  ──► 原样字节 (png/jpg/gif/webp)
Raster(Pixel)     ──► PngEncoder 重新编码
Svg               ──► 原样字节
Pdf               ──► hayro_svg::convert(page) → SVG 字符串
```

最终 `to_base64_url()` → `data:image/png;base64,xxx...` 放入 `<image xlink:href>`。

---

## 6. 完整调用链示例

以 "在 Typst 源码里写 `#image("photo.jpg", width: 5cm)` 并导出为 PNG" 为例，跟踪完整的处理路径：

```
阶段1：求值 (typst-eval)
  构造 ImageElem {
      source: Path("photo.jpg"),
      format: Smart::Auto,
      width: 5cm
  }
  source.load(world) → 读入字节（Loaded）

阶段2：布局 (typst-layout)
  layout_image(elem, engine, styles, region)
    │
    ├─ elem.decode()
    │    │
    │    ├─ determine_format: 扩展名 .jpg → ExchangeFormat::Jpg
    │    │
    │    └─ RasterImage::new(data, Jpg, Smart::Auto)
    │         ├─ JpegDecoder::new(cursor)
    │         ├─ decoder.icc_profile() → 提取 JPEG 内嵌 ICC
    │         ├─ exif::Reader → Orientation 2 → ops::flip_horizontal
    │         ├─ exif_dpi() → 220 dpi
    │         └─ RasterImageInner { data, dynamic, icc, dpi, exif_rot }
    │              ← 第一次计算，存入 comemo 缓存
    │
    ├─ px_ratio = 宽/高 = 1.5
    ├─ target = Size(5cm, 5cm/1.5)
    ├─ fitted = Contain 模式 → 等比缩放
    └─ FrameItem::Image(Image(...), Size(5cm, 3.33cm), span)
         推入 Frame

阶段3：光栅化 (typst-render::render(page, opts))
  render_frame(canvas, state, &page.frame)
    │
    ├─ 遍历 items，遇到 FrameItem::Image
    │
    └─ image::render_image(canvas, state, &image, size)
         │
         ├─ 计算旋转后的采样分辨率 w,h
         ├─ build_texture(&image, w, h)  ← memoized
         │    │
         │    ├─ 比较 (w,h) vs 原图尺寸 → 缩小
         │    ├─ filter = Lanczos3
         │    ├─ dynamic.resize_exact(w, h, Lanczos3)
         │    └─ 逐像素：Rgba([r,g,b,a]) → ColorU8::premultiply()
         │
         ├─ sk::Pattern::new(pixmap, ..., scale)
         │
         └─ canvas.fill_rect(view_rect, paint, ts, mask)
              │
              └─ 画布上得到最终像素（sRGB + premultiplied alpha）
```

---

## 7. 关键设计决策总结

| 设计点 | 选择 | 位置 |
|-------|-----|------|
| 跨颜色空间转换的中枢 | sRGB / Oklab 双枢纽，CMYK 用 ICC | [color.rs#L1758-L1873](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs#L1758-L1873) |
| 图像解码的缓存时机 | **第一次 layout 时** decode，用 comemo 全局复用 | [mod.rs#L427-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs#L427-L434), [raster.rs#L47-L53](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs#L47-L53) |
| 图像缩放的缓存时机 | **第一次渲染到特定分辨率时** build_texture，用 comemo 复用 | [typst-render/image.rs#L68-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/image.rs#L68-L69) |
| 渐变的颜色混合 | 在用户指定的空间混合（默认 Oklab，感知均匀） | [gradient.rs#L1398-L1422](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/gradient.rs#L1398-L1422) |
| PDF 颜色保真 | 保留 Luma/CMYK/RGB 空间，Spot 颜色用 separation | [typst-pdf/paint.rs#L114-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/paint.rs#L114-L163) |
| SVG 颜色保真 | 利用 CSS Color Level 4 的函数语法（oklab/oklch/color(srgb-linear)） | [typst-svg/paint.rs#L444-L505](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/paint.rs#L444-L505) |
| PDF JPEG 嵌入 | 不解码像素，直接透传 JPEG 字节流 + ICC | [typst-pdf/image.rs#L195-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/image.rs#L195-L207) |
| 光栅化后端统一 | 所有 Shape/Text/Image 都先转 sRGB（因为 tiny_skia 只有 RGBA） | [typst-render/paint.rs#L286-L290](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs#L286-L290) |

---

## 8. 文件索引

| 模块 | 路径 | 职责 |
|-----|------|------|
| 颜色类型 | [color.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/color.rs) | 8 种过程色 + 专色，跨空间转换、混合 |
| 渐变类型 | [gradient.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/gradient.rs) | 线性/径向/圆锥渐变采样与颜色空间插值 |
| 图像元素 | [image/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/mod.rs) | ImageElem 与 decode 入口 |
| 光栅图像 | [image/raster.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/visualize/image/raster.rs) | 4 种格式解码 + EXIF/DPI/ICC 提取 |
| 帧与条目 | [frame.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-library/src/layout/frame.rs) | Frame, FrameItem::Image 定义 |
| 图像布局 | [typst-layout/image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-layout/src/image.rs) | layout_image() 尺寸计算与 fit 逻辑 |
| PNG 渲染主循环 | [typst-render/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/lib.rs) | render() → render_frame() 递归 |
| PNG 颜色/渐变 | [typst-render/paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/paint.rs) | 颜色转 skia，渐变 pixmap 预渲染 |
| PNG 图像光栅化 | [typst-render/image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-render/src/image.rs) | build_texture() 按目标分辨率 resize |
| PDF 颜色 | [typst-pdf/paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/paint.rs) | Luma/CMYK/RGB/Separation 分通道输出 |
| PDF 图像 | [typst-pdf/image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-pdf/src/image.rs) | JPEG 直通 + CustomImage 适配层 |
| SVG 颜色 | [typst-svg/paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/paint.rs) | CSS 颜色函数序列化 |
| SVG 图像 | [typst-svg/image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/135-typst/crates/typst-svg/src/image.rs) | WebImage → base64 DataURL |
