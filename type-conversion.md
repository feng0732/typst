# Typst 类型转换机制分析

本文档深入分析 Typst 中类型和值之间的转换规则，包括基础类型系统、自动转换机制和失败诊断三个核心部分的协作方式。

## 1. 基础类型系统

### 1.1 Value 枚举：运行时的值表示

所有 Typst 值都由 `Value` 枚举统一表示，定义在 [value.rs](crates/typst-library/src/foundations/value.rs#L26-L88)：

```rust
pub enum Value {
    None, Auto, Bool(bool), Int(i64), Float(f64),
    Length(Length), Angle(Angle), Ratio(Ratio),
    Relative(Rel<Length>), Fraction(Fr),
    Color(Color), Gradient(Gradient), Tiling(Tiling),
    Symbol(Symbol), Version(Version),
    Str(Str), Bytes(Bytes), Label(Label),
    Datetime(Datetime), Decimal(Decimal), Duration(Duration),
    Content(Content), Styles(Styles),
    Array(Array), Dict(Dict), Func(Func), Args(Args),
    Type(Type), Module(Module), Dyn(Dynamic),
}
```

每个 `Value` 变体都可通过 `ty()` 获取对应的 `Type`（[value.rs#L116-L149](crates/typst-library/src/foundations/value.rs#L116-L149)）。

### 1.2 Type 类型：类型的元信息

`Type` 结构体描述了值的种类（[ty.rs#L64-L65](crates/typst-library/src/foundations/ty.rs#L64-L65)），包含 `short_name()`（如 `str`）、`long_name()`（如 `string`）、`title()`（如 `String`）、`constructor()`、`scope()` 等元信息。

### 1.3 NativeType trait：类型的 Rust 侧定义

`NativeType` trait 将 Rust 类型与 Typst 类型关联（[ty.rs#L190-L203](crates/typst-library/src/foundations/ty.rs#L190-L203)）：

```rust
pub trait NativeType {
    const NAME: &'static str;
    fn ty() -> Type { Type::from(Self::data()) }
    fn data() -> &'static NativeTypeData;
}
```

---

## 2. 转换核心 Trait

类型转换系统由三个核心 trait 组成，定义在 [cast.rs](crates/typst-library/src/foundations/cast.rs)。

### 2.1 Reflect：类型元数据与可转换性检查

```rust
pub trait Reflect {
    fn input() -> CastInfo;
    fn output() -> CastInfo;
    fn castable(value: &Value) -> bool;
    fn error(found: &Value) -> HintedString;
}
```

- `input()` / `output()` 返回 `CastInfo`，用于文档和自动补全
- `castable()` 是快速检查路径，避免通过 `CastInfo` 做堆分配 + 动态检查（[cast.rs#L42-L44](crates/typst-library/src/foundations/cast.rs#L42-L44)）
- `error()` 委托给 `Self::input().error(found)` 生成带提示的错误消息

### 2.2 IntoValue：Rust 类型 → Typst Value（不可失败）

```rust
pub trait IntoValue {
    fn into_value(self) -> Value;
}
```

### 2.3 FromValue：Typst Value → Rust 类型（可失败）

```rust
pub trait FromValue<V = Value>: Sized + Reflect {
    fn from_value(value: V) -> HintedStrResult<Self>;
}
```

注意：`FromValue` 的默认泛型参数是 `Value`，但 `Args` 的各方法约束是 `T: FromValue<Spanned<Value>>`。对于 `Spanned<Value>`，有两个关键的 impl（[cast.rs#L280-L291](crates/typst-library/src/foundations/cast.rs#L280-L291)）：

```rust
// impl A: Self = 任意 T（只要 T: FromValue）
impl<T: FromValue> FromValue<Spanned<Value>> for T {
    fn from_value(value: Spanned<Value>) -> HintedStrResult<Self> {
        T::from_value(value.v)
    }
}

// impl B: Self = Spanned<T>（T: FromValue）
impl<T: FromValue> FromValue<Spanned<Value>> for Spanned<T> {
    fn from_value(value: Spanned<Value>) -> HintedStrResult<Self> {
        let span = value.span;
        T::from_value(value.v).map(|t| Spanned::new(t, span))
    }
}
```

**两个 impl 的约束条件互不重叠，它们不是竞争候选，也不存在"选更具体的"优先级选择**。每个 impl 只在自己的约束范围内匹配：

```rust
// impl A: Self = 任意 T，约束 T: FromValue（即 T 能从 Value 转换）
impl<T: FromValue> FromValue<Spanned<Value>> for T {
    fn from_value(value: Spanned<Value>) -> HintedStrResult<Self> {
        T::from_value(value.v)   // 解包为 Value，调用 T: FromValue 的 from_value
    }
}

// impl B: Self = Spanned<T>，约束 T: FromValue（即内部 T 能从 Value 转换）
impl<T: FromValue> FromValue<Spanned<Value>> for Spanned<T> {
    fn from_value(value: Spanned<Value>) -> HintedStrResult<Self> {
        let span = value.span;
        T::from_value(value.v).map(|t| Spanned::new(t, span))
    }
}
```

**关键点：`FromValue` 是 `FromValue<V = Value>` 的简写，约束 `T: FromValue` 实际是 `T: FromValue<Value>`。**

- **普通目标类型（如 `f64`、`Content`）**：Self = `f64`
  - impl A：`T = f64`，约束 `f64: FromValue<Value>` ✓，匹配
  - impl B：Self 要求是 `Spanned<..>`，与 `f64` 不匹配
  - **只有 impl A 适用**，span 被丢弃——span 已在调用方通过 `.at(span)` 附加到 `SourceDiagnostic` 上。

- **带位置目标类型（`Spanned<Inner>`）**：Self = `Spanned<Inner>`
  - impl A：`T = Spanned<Inner>`，约束 `Spanned<Inner>: FromValue<Value>` ✗——`Spanned<Inner>` 从不单独实现 `FromValue<Value>`（`Spanned` 需要 span 信息才能构造，但 `Value` 本身没有 span），所以约束不成立
  - impl B：Self = `Spanned<Inner>`，`T = Inner`，约束 `Inner: FromValue<Value>` ✓
  - **只有 impl B 适用**，span 被保留在返回的 `Spanned<Inner>` 中。

两个 impl 各自的 Self 类型和 trait bounds 决定了适用范围，不存在"两个候选都匹配、选更具体"的情形。

### 2.4 CastInfo：转换信息描述

`CastInfo` 枚举（[cast.rs#L294-L304](crates/typst-library/src/foundations/cast.rs#L294-L304)）：

```rust
pub enum CastInfo {
    Any,                          // 任意值
    Value(Value, &'static str),   // 特定值 + 文档
    Type(Type),                   // 某类型的任意值
    Union(Vec<Self>),             // 多个可选值
}
```

`CastInfo` 实现了 `Add` trait，可通过 `+` 组合成 `Union`。

---

## 3. 自动转换机制

### 3.1 primitive! 宏：基础类型的自动转换

`primitive!` 宏（[value.rs#L578-L617](crates/typst-library/src/foundations/value.rs#L578-L617)）为基础类型批量实现 `Reflect`、`IntoValue` 和 `FromValue`。

宏签名中，逗号后的每个 `$other:ident$(($binding:ident))? => $out:expr` 定义了一条**自动转换规则**——当 `FromValue::from_value()` 收到不匹配主变体的值时，尝试匹配这些备选分支。

以 `f64` 为例（[value.rs#L621](crates/typst-library/src/foundations/value.rs#L621)）：

```rust
primitive! { f64: "float", Float, Int(v) => v as f64 }
```

展开后：
- `castable()` 接受 `Value::Float(_)` 和 `Value::Int(_)`
- `from_value()` 先匹配 `Value::Float(v)` → 直接返回；再匹配 `Value::Int(v)` → `Ok(v as f64)`；其余 → `Err(Self::error(&v))`

更多自动转换规则（[value.rs#L619-L663](crates/typst-library/src/foundations/value.rs#L619-L663)）：

| 目标类型       | 主变体           | 自动转换源                          | 转换逻辑                        |
|---------------|-----------------|------------------------------------|--------------------------------|
| `f64`         | `Float`         | `Int(v)`                           | `v as f64`                     |
| `Rel<Length>` | `Relative`      | `Length(v)`, `Ratio(v)`            | `v.into()`                     |
| `Str`         | `Str`           | `Symbol(symbol)`                   | `symbol.get().into()`          |
| `Content`     | `Content`       | `None`, `Symbol(v)`, `Str(v)`      | 空内容/符号元素/文本元素         |
| `Func`        | `Func`          | `Type(ty)`, `Symbol(symbol)`       | 构造函数/符号函数（可失败）       |

### 3.2 cast! 宏：自定义类型转换

`cast!` 宏（[cast.rs#L467-L521](crates/typst-library/src/foundations/cast.rs#L467-L521)）用于为自定义类型定义双向转换。

示例（[cast.rs#L467-L480](crates/typst-library/src/foundations/cast.rs#L467-L480)）：

```rust
cast! {
    SyntaxMode,
    self => IntoValue::into_value(match self {
        SyntaxMode::Markup => "markup",
        SyntaxMode::Math => "math",
        SyntaxMode::Code => "code",
    }),
    "markup" => SyntaxMode::Markup,
    "math" => SyntaxMode::Math,
    "code" => SyntaxMode::Code,
}
```

格式：`类型, self=>表达式, 字符串=>变体` 或 `绑定:类型=>表达式`。

### 3.3 cast! 宏的展开过程

由 [macros/cast.rs](crates/typst-macros/src/cast.rs) 中的 `cast()` 函数处理，生成三个 impl 块。

**`FromValue` impl 的展开**（[macros/cast.rs#L289-L336](crates/typst-macros/src/cast.rs#L289-L336)）：按如下优先级依次尝试：

1. **动态类型检查**（仅 `type` 标记的 cast）：如果 `value` 是 `Value::Dyn` 且 `dynamic.is::<Self>()`，则 downcast 返回
2. **字符串匹配**：如果 `value` 是 `Value::Str`，match 其内容对应各个 `"string" => Expr` 分支
3. **类型级转换检查**：对每个 `binding: Ty => Expr` 分支，先调用 `<Ty as Reflect>::castable(&value)` 做预筛，通过则调用 `<Ty as FromValue>::from_value(value)?` 做实际转换
4. **全部失败**：`Err(<Self as Reflect>::error(&value))`

注意 cast! 展开中的 **castable 预筛** 与 `Args::find()` 中的 **castable 预筛** 是不同层面的：前者发生在 `FromValue::from_value()` 内部，后者发生在 `Args` 层。两者可以叠加——`Args::find()` 先用 `T::castable()` 过滤掉不可能匹配的参数，再对被选中的参数调用 `T::from_value()`，后者内部再次对每个子类型做 castable 检查。

### 3.4 运算中的自动转换

`ops.rs` 在算术/比较运算中定义了大量跨类型转换规则（[ops.rs#L91-L172](crates/typst-library/src/foundations/ops.rs#L91-L172)）。这些转换不走 `FromValue`，而是直接 match `(Value, Value)` 对。例如 `Int + Float → Float`、`Length + Ratio → Relative`、`Color + Length → Stroke` 等。

---

## 4. 参数消费与类型转换的完整流程

这是本文档的核心部分。`Args` 定义在 [args.rs](crates/typst-library/src/foundations/args.rs)。每个参数是 `Arg` 结构体（[args.rs#L514-L521](crates/typst-library/src/foundations/args.rs#L514-L521)），包含 `span: Span`、`name: Option<Str>`（命名参数有名称，位置参数为 `None`）、`value: Spanned<Value>`。

### 4.1 eat()：消费第一个位置参数，不做预筛

```rust
// args.rs#L112-L124
pub fn eat<T>(&mut self) -> SourceResult<Option<T>>
where
    T: FromValue<Spanned<Value>>,
{
    for (i, slot) in self.items.iter().enumerate() {
        if slot.name.is_none() {                          // 只看位置参数
            let value = self.items.remove(i).value;       // 取出 Spanned<Value>
            let span = value.span;
            return T::from_value(value).at(span).map(Some);
        }
    }
    Ok(None)  // 没有位置参数时返回 None，不是错误
}
```

**步骤**：
1. 遍历 `items`，找到第一个 `name == None` 的位置参数
2. 从 `items` 中 `remove` 它（**消费**，后续调用不会再看到这个参数）
3. 调用 `T::from_value(value)`——此处 `value` 类型为 `Spanned<Value>`，走 `FromValue<Spanned<Value>> for T` impl，内部解包为 `T::from_value(value.v)`
4. `.at(span)` 将 `HintedStrResult<T>` 转为 `SourceResult<T>`，把参数的 span 附到 `SourceDiagnostic` 上
5. **不做 `T::castable()` 预筛**——无论值的类型是什么，直接尝试转换，失败则报错

### 4.2 expect()：eat() + 缺参数报错

```rust
// args.rs#L150-L158
pub fn expect<T>(&mut self, what: &str) -> SourceResult<T>
where
    T: FromValue<Spanned<Value>>,
{
    match self.eat()? {
        Some(v) => Ok(v),
        None => bail!(self.missing_argument(what)),
    }
}
```

`expect()` 是 `eat()` 的包装。当没有位置参数时，`eat()` 返回 `Ok(None)`，`expect()` 会生成 `"missing argument: {what}"` 错误。

`missing_argument()` 还有一个特殊逻辑（[args.rs#L161-L174](crates/typst-library/src/foundations/args.rs#L161-L174)）：如果用户写了 `name: value` 但参数实际上是位置参数，会提示 `"the argument '{what}' is positional"; hint: "try removing '{name}:'"` 。

**注意**：`expect()` 有两种失败——"缺少参数"（`missing_argument`）和 "类型转换失败"（来自 `eat()` 内 `from_value` 的错误），两者的诊断消息不同。

### 4.3 find()：按类型预筛，消费第一个可转换的位置参数

```rust
// args.rs#L177-L189
pub fn find<T>(&mut self) -> SourceResult<Option<T>>
where
    T: FromValue<Spanned<Value>>,
{
    for (i, slot) in self.items.iter().enumerate() {
        if slot.name.is_none() && T::castable(&slot.value.v) {
            let value = self.items.remove(i).value;
            let span = value.span;
            return T::from_value(value).at(span).map(Some);
        }
    }
    Ok(None)
}
```

**与 `eat()` 的关键区别**：多了一步 `T::castable(&slot.value.v)` 预筛。只有通过预筛的参数才会被消费和转换。

**为什么需要预筛**：`find()` 的典型场景是"在多个位置参数中找到那个类型匹配的"。例如函数签名 `(spacing: Length, body: Content)`，`Content` 和 `Length` 都可能是位置参数，`find::<Content>()` 需要跳过不匹配的 `Length` 参数。如果直接用 `eat()`，遇到类型不匹配就会报错而非继续查找。

**预筛与实际转换的关系**：`castable()` 返回 `true` 时，`from_value()` 通常也会成功，但这不是绝对的。`castable()` 检查的是值的顶层 `Value` 变体，而 `from_value()` 内部可能有更细致的校验（如 `Func` 从 `Type` 转换时要调用 `ty.constructor()?`，`castable` 只看 `Value::Type(_)` 不检查是否有构造函数）。这种情况下 `castable` 通过但 `from_value` 失败，错误会正常传播。

### 4.4 all()：消费所有位置参数，不预筛，收集错误

```rust
// args.rs#L192-L214
pub fn all<T>(&mut self) -> SourceResult<Vec<T>>
where
    T: FromValue<Spanned<Value>>,
{
    let mut list = vec![];
    let mut errors = eco_vec![];
    self.items.retain(|item| {
        if item.name.is_some() {
            return true;  // 保留命名参数
        }
        let span = item.value.span;
        let spanned = Spanned::new(std::mem::take(&mut item.value.v), span);
        match T::from_value(spanned).at(span) {
            Ok(val) => list.push(val),
            Err(diags) => errors.extend(diags),
        }
        false  // 移除所有位置参数
    });
    if !errors.is_empty() {
        return Err(errors);
    }
    Ok(list)
}
```

**行为**：
- 遍历所有位置参数，对每个都调用 `from_value()` 尝试转换
- **不做 `castable()` 预筛**——成功转的进 `list`，失败的攒到 `errors`
- 用 `retain()` 一边遍历一边移除：命名参数保留，位置参数全部移除
- 如果有任何转换失败，一次性返回所有错误（`EcoVec<SourceDiagnostic>`）；全部成功则返回 `Vec<T>`
- 不匹配的参数不会被跳过，而是直接报错

### 4.5 named()：按名称消费命名参数，取最后一个

```rust
// args.rs#L218-L236
pub fn named<T>(&mut self, name: &str) -> SourceResult<Option<T>>
where
    T: FromValue<Spanned<Value>>,
{
    // We don't quit once we have a match because when multiple matches
    // exist, we want to remove all of them and use the last one.
    let mut i = 0;
    let mut found = None;
    while i < self.items.len() {
        if self.items[i].name.as_deref() == Some(name) {
            let value = self.items.remove(i).value;
            let span = value.span;
            found = Some(T::from_value(value).at(span)?);
        } else {
            i += 1;
        }
    }
    Ok(found)
}
```

**行为**：
- 按名称查找命名参数
- **不做 `castable()` 预筛**——找到就尝试转换，失败就报错
- 当存在多个同名参数时，**移除所有同名参数，取最后一个**——循环不会提前退出
- 转换失败时通过 `?` 提前返回错误

### 4.6 named_or_find()：先查命名参数，再按类型查找位置参数

```rust
// args.rs#L239-L247
pub fn named_or_find<T>(&mut self, name: &str) -> SourceResult<Option<T>>
where
    T: FromValue<Spanned<Value>>,
{
    match self.named(name)? {
        Some(value) => Ok(Some(value)),
        None => self.find(),
    }
}
```

**行为**：先调用 `named(name)` 查找命名参数。如果找到就返回；如果没找到（返回 `None`，不是错误），再 fallback 到 `find()` 按类型预筛位置参数。

### 4.7 五种方法的对比

| 方法              | 参数筛选条件                       | castable 预筛 | 消费范围                    | 无参数时返回       |
|------------------|-----------------------------------|--------------|----------------------------|-------------------|
| `eat::<T>()`     | 第一个位置参数                     | ❌ 无         | 1 个位置参数               | `Ok(None)`        |
| `expect::<T>()`  | 第一个位置参数                     | ❌ 无         | 1 个位置参数               | 缺参数错误         |
| `find::<T>()`    | 第一个 `castable` 的位置参数       | ✅ 有         | 1 个位置参数               | `Ok(None)`        |
| `all::<T>()`     | 全部位置参数                       | ❌ 无         | 所有位置参数               | `Ok(vec![])`      |
| `named::<T>(n)`  | 名称为 `n` 的命名参数（取最后一个） | ❌ 无         | 所有同名命名参数           | `Ok(None)`        |

---

## 5. 失败诊断机制

### 5.1 错误类型的层级

转换过程涉及三种错误类型，层级由低到高：

```
HintedStrResult<T> = Result<T, HintedString>     // from_value 直接产出
       ↓ .at(span)
SourceResult<T>   = Result<T, EcoVec<SourceDiagnostic>>  // 最终交给引擎
```

`HintedString`（[diag.rs#L519-L564](crates/typst-library/src/diag.rs#L519-L564)）内部用 `EcoVec<EcoString>` 存储：第一个元素是主消息，后续元素是提示。

### 5.2 from_value 失败 → HintedString 的生成

当 `FromValue::from_value()` 收到不匹配的值时，调用 `<Self as Reflect>::error(&v)`。`Reflect::error()` 的默认实现（[cast.rs#L55-L57](crates/typst-library/src/foundations/cast.rs#L55-L57)）委托给 `Self::input().error(found)`。

`CastInfo::error()`（[cast.rs#L309-L365](crates/typst-library/src/foundations/cast.rs#L309-L365)）的完整逻辑：

1. **walk `CastInfo`**：递归展开 `Union`，收集所有叶子节点的描述到 `parts`
2. **构造主消息**：`"expected {separated_list(parts, "or")}"` + 如果实际值类型不在期望中，追加 `", found {found.ty()}"`
3. **智能提示**：根据 `found` 值和期望类型添加上下文相关提示

智能提示场景：

| 实际值          | 期望含        | 提示示例                                                          |
|----------------|--------------|------------------------------------------------------------------|
| `Value::Int(i)` | `"length"`   | `a length needs a unit - did you mean {i}pt?`                    |
| `Value::Str(s)` | `"label"`    | `use <{s}> or label({s}) to create a label`                      |
| `Value::Decimal` | `"float"`   | `if loss of precision is acceptable, explicitly cast the decimal to a float with float(value)` |

### 5.3 HintedString → SourceDiagnostic 的传播

`.at(span)` 有两个 impl（[diag.rs#L498-L575](crates/typst-library/src/diag.rs#L498-L575)）：

**泛型 `Result<T, S: Into<EcoString>>`**（[diag.rs#L498-L505](crates/typst-library/src/diag.rs#L498-L505)）：
```rust
fn at(self, span: Span) -> SourceResult<T> {
    self.map_err(|message| eco_vec![SourceDiagnostic::error(span, message)])
}
```
只保留主消息，不保留提示。

**`HintedStrResult<T>` 专用**（[diag.rs#L566-L575](crates/typst-library/src/diag.rs#L566-L575)）：
```rust
fn at(self, span: Span) -> SourceResult<T> {
    self.map_err(|err| {
        let mut components = err.0.into_iter();
        let message = components.next().unwrap();
        let diag = SourceDiagnostic::error(span, message).with_hints(components);
        eco_vec![diag]
    })
}
```
第一个元素做主消息，其余全部做提示。**因为 `from_value` 返回的是 `HintedStrResult`，所以 `Args` 方法中的 `.at(span)` 走这个 impl，提示信息不会丢失。**

### 5.4 完整的错误传播路径

以 `eat::<f64>()` 为例，当传入 `Value::Bool(true)` 时：

```
eat() 找到第一个位置参数 Value::Bool(true)
  ↓
T::from_value(Spanned<Value>{ v: Value::Bool(true), span })
  ↓ 目标类型 T = f64（普通类型，非 Spanned<..>）
  ↓ 匹配 impl A: FromValue<Spanned<Value>> for T（T = f64）
  ↓ 内部调用 f64::from_value(Value::Bool(true))
  ↓
primitive! 展开的 from_value:
  match Value::Bool(true) {
    Value::Float(v) => ...    // 不匹配
    Value::Int(v) => ...      // 不匹配
    v => Err(<f64 as Reflect>::error(&v))  // 走这里
  }
  ↓
Reflect::error(&Value::Bool(true))
  → Self::input().error(&Value::Bool(true))
  → CastInfo::Type(Type::of::<f64>()).error(&Value::Bool(true))
  ↓
CastInfo::error() 生成:
  parts = ["float"]
  matching_type = false  (bool ≠ float)
  msg = "expected float, found boolean"
  无智能提示匹配（found 是 Bool 不是 Int/Str/Decimal）
  ↓
返回 Err(HintedString { "expected float, found boolean" })
  ↓
.at(span)
  ↓ HintedStrResult<T> 专用的 At impl
SourceDiagnostic::error(span, "expected float, found boolean")
  ↓
返回 Err(eco_vec![SourceDiagnostic { severity: Error, span, message, hints: [] }])
```

---

## 6. 三者协作总结

### 6.1 不同消费方法中四步骤的参与情况

一次参数消费涉及的四个步骤：

| 步骤            | 作用                          | eat | expect | find | all | named |
|----------------|------------------------------|-----|--------|------|-----|-------|
| ① 参数筛选      | 按位置/名称/类型筛选参数        | 位置 | 位置    | 位置+castable | 位置 | 名称   |
| ② 消费移除      | 从 items 中 remove/retain     | 1个 | 1个    | 1个  | 全部 | 同名全部 |
| ③ 实际转换      | `T::from_value(value)`        | ✅  | ✅     | ✅   | ✅  | ✅    |
| ④ 诊断传播      | `.at(span)` → SourceDiagnostic | ✅  | ✅     | ✅   | ✅  | ✅    |

**关键差异在步骤①**：
- `find()` 在筛选阶段就调用 `T::castable()` 做预筛，跳过不可能匹配的参数
- `eat()` / `expect()` / `all()` / `named()` 不做预筛，拿到参数就直接转换

### 6.2 协作关系图

```
┌──────────────────────────────────────────────────────────┐
│                     基础类型系统                           │
│  Value 枚举 ──→ ty() ──→ Type ──→ NativeTypeData         │
│  (运行时值)        (类型标识)    (元信息:名称/文档/作用域)  │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────┐
│               Args 参数消费层                              │
│  eat()   ── 位置参数，直接转换，不预筛                     │
│  expect() ── eat() + 缺参数报错                           │
│  find()  ── 位置参数，castable 预筛后转换                  │
│  all()   ── 全部位置参数，直接转换，攒错误                  │
│  named() ── 按名称查找命名参数，取最后一个                  │
│  named_or_find() ── named + find 的组合                   │
└──────────────────────┬───────────────────────────────────┘
                       │ 调用 T::from_value(value)
┌──────────────────────▼───────────────────────────────────┐
│              FromValue 转换层                              │
│  FromValue<Spanned<Value>> for T ── 解包 span，委托      │
│  FromValue<Value> for T ── 实际转换逻辑                   │
│  primitive! 展开的 from_value ── match + 自动转换分支     │
│  cast! 展开的 from_value ── dynamic→str→type 级联匹配    │
│    ↳ 内部也有 castable 预筛（对每个子类型检查）            │
└──────────────────────┬───────────────────────────────────┘
                       │ 失败时调用 <Self as Reflect>::error()
┌──────────────────────▼───────────────────────────────────┐
│              失败诊断层                                    │
│  Reflect::error() → CastInfo::error()                    │
│  CastInfo::error() ── walk CastInfo → 期望类型列表        │
│    ↳ 智能提示: Int→length / Str→label / Decimal→float    │
│  HintedString ── 主消息 + hints                          │
│  .at(span) ── HintedStrResult 专用 impl → SourceDiagnostic│
│    ↳ 保留所有 hints（不会丢失提示）                        │
└──────────────────────────────────────────────────────────┘
```

### 6.3 设计要点

1. **两层 castable 的分工**：`Args::find()` 的 castable 预筛用于在多个参数中"挑选"合适的那个；`cast!` 展开中的 castable 预筛用于在多个可选转换路径中"选择"匹配的分支。两者解决不同层面的问题。

2. **eat 不预筛是故意的**：`eat()` 假设调用方已经知道下一个位置参数的预期类型，不需要跳过。如果类型不匹配，说明用户传错了参数，应该立即报错。

3. **Spanned<Value> 的两个 impl 约束互不重叠**：impl `for T` 要求 `T: FromValue<Value>`，impl `for Spanned<T>` 要求内部 `T: FromValue<Value>`。当目标是 `Spanned<Inner>` 时，impl `for T` 的约束退化为 `Spanned<Inner>: FromValue<Value>`——这从不成立，因为 `Spanned<..>` 不单独实现 `FromValue<Value>`。两个 impl 不是竞争候选，各自按自己的约束匹配适用范围。

4. **HintedStrResult 的 At impl 保留提示**：普通的 `StrResult` 通过 `.at(span)` 会丢失提示，但 `from_value` 返回 `HintedStrResult`，其 `At` impl 会拆解 `HintedString` 的 vec，第一个做消息、其余做 hints，确保智能提示不丢失。

5. **named 取最后一个同名参数**：这模仿了命令行参数和 CSS 的"后者覆盖前者"语义。循环不提前退出，确保所有同名参数都被移除。
