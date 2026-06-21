# Typst 表达式求值：从解析到求值的完整路径

本文追踪 Typst 脚本表达式从「源代码字符串」经「语法树」到「运行时值」的完整路径，重点剖析作用域（scope）和闭包（closure）的行为。

---

## 1. 总体架构：两阶段管线

```
源代码字符串
  │
  ▼  Parser (typst-syntax)
CST / AST（SyntaxNode 为骨架，ast::* 为类型化视图）
  │
  ▼  Eval trait (typst-eval)
Value（运行时值）
```

核心 crate 划分：

| Crate | 职责 |
|---|---|
| [typst-syntax](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax) | 词法分析、语法解析、CST/AST 定义 |
| [typst-eval](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval) | 表达式求值、VM、作用域管理、闭包捕获 |
| [typst-library](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library) | `Scope`、`Scopes`、`Binding`、`Closure`、`Func`、`Value` 等运行时基础类型 |

---

## 2. 解析阶段：源代码 → CST/AST

### 2.1 入口函数

三个模式对应三个入口（见 [parser.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/parser.rs#L16-L37)）：

```rust
pub fn parse(text: &str) -> SyntaxNode          // Markup 模式
pub fn parse_code(text: &str) -> SyntaxNode     // Code 模式
pub fn parse_math(text: &str) -> SyntaxNode     // Math 模式
```

它们返回一棵以 `SyntaxKind::Markup` / `SyntaxKind::Code` / `SyntaxKind::Math` 为根的 **CST**（Concrete Syntax Tree）。

### 2.2 CST 与 AST 的关系

CST 节点是 `SyntaxNode`，包含 `SyntaxKind` 和子节点列表，**保留全部源文本细节**（空格、注释、分隔符等）。

AST 是 CST 上的**惰性类型化视图**（见 [ast.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/ast.rs#L1-L78) 头部文档）：

- 每个 AST 节点本质上是一个 `&'a SyntaxNode` 指针（如 `struct Ident<'a>(&'a SyntaxNode)`）
- 通过 `AstNode::from_untyped()` 将 CST 节点转换为具体类型，转换时会检查 `SyntaxKind` 是否匹配
- 只有在**遍历时**才做转换，未访问的分支不付出构造开销

`Expr` 枚举是 AST 的核心，覆盖了所有表达式类型（见 [ast.rs#L249-L375](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/ast.rs#L249-L375)）：

```rust
pub enum Expr<'a> {
    Text(Text<'a>), Ident(Ident<'a>), Bool(Bool<'a>),
    CodeBlock(CodeBlock<'a>), ContentBlock(ContentBlock<'a>),
    Closure(Closure<'a>), FuncCall(FuncCall<'a>),
    LetBinding(LetBinding<'a>), ForLoop(ForLoop<'a>),
    Conditional(Conditional<'a>), ...
}
```

---

## 3. 求值阶段：AST → Value

### 3.1 `Eval` trait

定义在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L178-L184)：

```rust
pub trait Eval {
    type Output;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output>;
}
```

为每种 AST 节点实现了 `Eval`，分布在多个文件中：

| 文件 | 覆盖的 `Eval` 实现 |
|---|---|
| [code.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs) | `Code`, `Expr`, `Ident`, 字面量, `Array`, `Dict`, `CodeBlock`, `ContentBlock`, `Parenthesized`, `FieldAccess`, `Contextual` |
| [call.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs) | `FuncCall`, `Closure`, `Args`, `CapturesVisitor` |
| [binding.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/binding.rs) | `LetBinding`, `DestructAssignment` |
| [flow.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs) | `Conditional`, `WhileLoop`, `ForLoop`, `LoopBreak`, `LoopContinue`, `FuncReturn` |
| [markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/markup.rs) | `Markup`, `Text`, `Space`, `Strong`, `Emph`, `Heading` 等 |
| [import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/import.rs) | `ModuleImport`, `ModuleInclude` |
| [math.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/math.rs) | 数学模式求值 |
| [ops.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/ops.rs) | `Unary`, `Binary`（一元/二元运算符求值） |
| [rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/rules.rs) | `SetRule`, `ShowRule`（set/show 规则求值） |

### 3.2 通用表达式分派：`Expr::eval`

所有表达式求值的总入口是 `impl Eval for ast::Expr`（见 [code.rs#L74-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L74-L154)），这是一个巨大的 `match` 分派：

```rust
impl Eval for ast::Expr<'_> {
    type Output = Value;

    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let span = self.span();
        let value = match self {
            // 文本类
            Self::Text(v)        => v.eval(vm).map(Value::Content),
            Self::Space(v)       => v.eval(vm).map(Value::Content),
            Self::Linebreak(v)   => v.eval(vm).map(Value::Content),
            Self::Parbreak(v)    => v.eval(vm).map(Value::Content),
            Self::Strong(v)      => v.eval(vm).map(Value::Content),
            Self::Emph(v)        => v.eval(vm).map(Value::Content),
            // ...
            // 字面量
            Self::None(v)        => v.eval(vm),
            Self::Bool(v)        => v.eval(vm),
            Self::Int(v)         => v.eval(vm),
            Self::Float(v)       => v.eval(vm),
            Self::Str(v)         => v.eval(vm),
            // 运算
            Self::Unary(v)       => v.eval(vm),
            Self::Binary(v)      => v.eval(vm),
            // 访问与调用
            Self::Ident(v)       => v.eval(vm),
            Self::FieldAccess(v) => v.eval(vm),
            Self::FuncCall(v)    => v.eval(vm),
            Self::Closure(v)     => v.eval(vm),
            // 控制流
            Self::LetBinding(v)  => v.eval(vm),
            Self::Conditional(v) => v.eval(vm),
            Self::WhileLoop(v)   => v.eval(vm),
            Self::ForLoop(v)     => v.eval(vm),
            Self::LoopBreak(v)   => v.eval(vm),
            Self::FuncReturn(v)  => v.eval(vm),
            // ... 更多分支
        }?
        .spanned(span);

        // 每个表达式求值后都调用 trace，支持 IDE 悬停
        vm.trace_at(span, &value);

        Ok(value)
    }
}
```

**关键设计**：
- 每种表达式类型有独立的 `Eval` impl，`Expr::eval` 只是做匹配分派
- 每个值都附带 span（`.spanned(span)`），用于错误定位
- 每个表达式求值后都触发 `vm.trace_at(span, &value)`，这是 IDE 工具提示（tooltip）的基础
- `SetRule` 和 `ShowRule` 在 `Expr::eval` 中直接报错（"only allowed directly in code and content blocks"），因为它们只能在 `eval_code` / `eval_markup` 中作为顶级表达式处理

### 3.3 虚拟机 Vm

`Vm` 是求值的核心状态容器（见 [vm.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L16-L28)）：

```rust
pub struct Vm<'a> {
    pub engine: Engine<'a>,        // 世界访问、诊断、路由
    pub flow: Option<FlowEvent>,   // 控制流事件（break/continue/return）
    pub scopes: Scopes<'a>,        // 作用域栈
    pub inspected: Option<Span>,   // IDE 追踪的 span
    pub context: Tracked<'a, Context<'a>>,  // 上下文样式数据
}
```

关键方法：
- `Vm::define(var, value)` — 在当前作用域顶层绑定变量（见 [vm.rs#L50-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L50-L52)）
- `Vm::bind(var, binding)` — 更底层的绑定，同时触发 IDE 追踪（见 [vm.rs#L58-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L58-L73)）

### 3.4 求值入口

**模块级求值** — [eval()](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L40-L97)：

```
1. 检查循环求值（route.contains）
2. 构建 Engine 和 Vm（Scopes::new(Some(library))，即以标准库为 base）
3. 检查语法错误
4. root.cast::<ast::Markup>() → markup.eval(&mut vm)
5. 组装 Module（把 vm.scopes.top 作为模块导出的作用域）
```

**字符串求值** — [eval_string()](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L101-L175)：

用于 `eval()` 函数等运行时求值场景，支持三种 `SyntaxMode`，还会额外 `push` 一个调用方传入的 `scope`。

### 3.5 顺序求值：`eval_code` 与 `eval_markup`

表达式不是孤立求值的，而是按顺序在代码块、内容块或模块顶层中逐个求值。有两个核心的顺序求值函数：

#### 3.5.1 代码模式：`eval_code`

定义在 [code.rs#L24-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L24-L72)：

```rust
fn eval_code<'a>(
    vm: &mut Vm,
    exprs: &mut impl Iterator<Item = ast::Expr<'a>>,
) -> SourceResult<Value> {
    let flow = vm.flow.take();     // 保存外层 flow
    let mut output = Value::None;

    while let Some(expr) = exprs.next() {
        let span = expr.span();
        let value = match expr {
            // set 规则特殊处理：作用于剩余所有表达式
            ast::Expr::SetRule(set) => {
                let styles = set.eval(vm)?;
                if vm.flow.is_some() { break; }
                let tail = eval_code(vm, exprs)?.display();
                Value::Content(tail.styled_with_map(styles))
            }
            // show 规则特殊处理：作用于剩余所有表达式
            ast::Expr::ShowRule(show) => {
                let recipe = show.eval(vm)?;
                if vm.flow.is_some() { break; }
                let tail = eval_code(vm, exprs)?.display();
                Value::Content(tail.styled_with_recipe(
                    &mut vm.engine, vm.context, recipe,
                )?)
            }
            // 普通表达式：直接求值
            _ => expr.eval(vm)?,
        };

        // 值拼接：把当前结果和前一个 output 合并
        output = ops::join(output, value).at(span)?;

        // 控制流提前终止：遇到 break/continue/return 就停止
        if let Some(event) = &vm.flow {
            warn_for_discarded_content(&mut vm.engine, event, &output);
            break;
        }
    }

    if flow.is_some() {
        vm.flow = flow;   // 恢复外层 flow
    }

    Ok(output)
}
```

**关键机制**：

1. **Set/Show 规则的"作用于剩余"语义**：set/show 不是普通表达式，它们会递归调用 `eval_code` 处理剩余的所有表达式，然后把样式应用到结果上。这就是 `set text(red)` 为什么会影响后面所有内容的原因。

2. **值拼接（ops::join）**：每一步的求值结果都与累积的 `output` 做 `join` 操作。对于内容值会合并成 sequence，对于两个 `none` 还是 `none`，其他类型会报错（取决于 join 规则）。

3. **控制流穿透**：`FlowEvent`（break/continue/return）会在 `vm.flow` 中设置，遇到后立即停止后续求值。

4. **Flow 保存与恢复**：进入 `eval_code` 时保存外层的 `flow`，退出时恢复，保证控制流不会意外"泄漏"到外层。

#### 3.5.2 标记模式：`eval_markup`

定义在 [markup.rs#L26-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/markup.rs#L26-L87)，与 `eval_code` 结构类似，但输出是 `Content` 而非 `Value`，且对标签（Label）有特殊处理：

```rust
fn eval_markup(
    vm: &mut Vm,
    exprs: &mut impl Iterator<Item = ast::Expr<'a>>,
) -> SourceResult<Content> {
    let flow = vm.flow.take();
    let mut seq = Vec::new();

    while let Some(expr) = exprs.next() {
        match expr {
            ast::Expr::SetRule(set) => {
                let styles = set.eval(vm)?;
                // ... 递归 eval_markup 处理尾部，应用样式
                seq.push(eval_markup(vm, exprs)?.styled_with_map(styles))
            }
            ast::Expr::ShowRule(show) => {
                // ... 类似 set
            }
            // Label 特殊处理：附着到前面最近的内容元素上
            expr => match expr.eval(vm)? {
                Value::Label(label) => {
                    if let Some(elem) = seq.iter_mut().rev().find(...) {
                        *elem = std::mem::take(elem).labelled(label);
                    } else {
                        vm.engine.sink.warn(...);  // 标签没附着到任何东西
                    }
                }
                value => seq.push(value.display().spanned(expr.span())),
            },
        }
        if vm.flow.is_some() { break; }
    }

    Ok(Content::sequence(seq))
}
```

**Markup 特有的 Label 语义**：`Label` 不是作为独立内容插入，而是**向后查找**最近的内容元素并附着上去。如果找不到（比如在段首），会发出警告。

---

## 4. 一元与二元运算符求值

运算符表达式通过 `Unary` 和 `Binary` AST 节点表示，其求值逻辑在 [ops.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/ops.rs) 中。

### 4.1 解析阶段的优先级与结合性

解析器用 Pratt 算法处理运算符优先级（见 [parser.rs#L605-L680](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/parser.rs#L605-L680)）：

```
code_expr_prec(p, atomic, min_prec):
  1. 解析一元操作符或主表达式（primary）
  2. 循环：
     - 如果后面是 ( 或 [，则解析函数调用
     - 如果后面是 .ident，则解析字段访问
     - 如果后面是二元运算符且优先级 >= min_prec：
       * 吃入运算符
       * 递归解析右侧（左结合则 prec+1，右结合则 prec 不变）
       * 包成 Binary 节点
     - 否则 break
```

`BinOp` 定义在 [ast.rs#L1799-L1825](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/ast.rs#L1799-L1825)，每个运算符有 `precedence()` 和 `assoc()` 方法。

### 4.2 求值阶段

**一元运算**（[ops.rs#L7-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/ops.rs#L7-L19)）：

```rust
impl Eval for ast::Unary<'_> {
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let value = self.expr().eval(vm)?;
        let result = match self.op() {
            ast::UnOp::Pos => ops::pos(value),
            ast::UnOp::Neg => ops::neg(value),
            ast::UnOp::Not => ops::not(value),
        };
        result.at(self.span())
    }
}
```

**二元运算**（[ops.rs#L21-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/ops.rs#L21-L66)）：

```rust
fn apply_binary(binary, vm, op) -> SourceResult<Value> {
    let lhs = binary.lhs().eval(vm)?;

    // 短路求值：and/or 不计算右侧
    if (binary.op() == ast::BinOp::And && lhs == false.into_value())
        || (binary.op() == ast::BinOp::Or && lhs == true.into_value())
    {
        return Ok(lhs);
    }

    let rhs = binary.rhs().eval(vm)?;
    op(lhs, rhs).at(binary.span())
}
```

具体的运算逻辑（`ops::add`、`ops::mul` 等）定义在 `typst-library` 的 `foundations/ops.rs` 中，处理多态类型的运算（如数字+长度、字符串拼接等）。

---

## 5. 赋值与可变访问

### 5.1 `Access` trait：可变左值

除了 `Eval` trait（只读求值），还有 `Access` trait（可变访问），定义在 [access.rs#L8-L12](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/access.rs#L8-L12)：

```rust
pub(crate) trait Access {
    fn access<'a>(self, vm: &'a mut Vm) -> SourceResult<&'a mut Value>;
}
```

`Access` 返回的是**可变引用**（`&'a mut Value`），它表示一个"左值"——可以被赋值的位置。

实现了 `Access` 的 AST 类型（见 [access.rs#L14-L74](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/access.rs#L14-L74)）：

| 类型 | 行为 |
|---|---|
| `Expr::Ident` | 从作用域获取可变绑定（`vm.scopes.get_mut()`） |
| `Expr::Parenthesized` | 递归访问内部表达式 |
| `Expr::FieldAccess` | 访问字典的字段（`dict.at_mut(field)`） |
| `Expr::FuncCall` | 仅支持访问器方法（`first`/`last`/`at`），返回可变引用 |
| 其他 | 报错"cannot mutate a temporary value" |

**标识符的可变访问**：

```rust
impl Access for ast::Ident<'_> {
    fn access<'a>(self, vm: &'a mut Vm) -> SourceResult<&'a mut Value> {
        vm.scopes
            .get_mut(&self)
            .and_then(|b| b.write().map_err(Into::into))
            .at(self.span())
    }
}
```

注意 `get_mut()` 返回 `&mut Binding`，然后调用 `Binding::write()` —— 这一步会检查 `BindingKind`，如果是 `Captured` 就会报错。

### 5.2 赋值运算：`apply_assignment`

二元赋值操作（`=`, `+=`, `-=`, `*=`, `/=`）通过 `apply_assignment` 函数处理（见 [ops.rs#L68-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/ops.rs#L68-L91)）：

```rust
fn apply_assignment(
    binary: ast::Binary,
    vm: &mut Vm,
    op: fn(Value, Value) -> HintedStrResult<Value>,
) -> SourceResult<Value> {
    let rhs = binary.rhs().eval(vm)?;
    let lhs = binary.lhs();

    // 特殊情况：对字典字段的纯赋值可以创建新字段
    if binary.op() == ast::BinOp::Assign
        && let ast::Expr::FieldAccess(access) = lhs
    {
        let dict = access_dict(vm, access)?;
        dict.insert(access.field().get().clone().into(), rhs);
        return Ok(Value::None);
    }

    // 通用情况：读取左值，运算，写回
    let location = binary.lhs().access(vm)?;
    let lhs = std::mem::take(&mut *location);
    *location = op(lhs, rhs).at(binary.span())?;
    Ok(Value::None)
}
```

**赋值的三步流程**：
1. 求值右值 `rhs`
2. 通过 `Access::access()` 获取左值的可变引用
3. 读出左值的旧值 → 与右值运算 → 写回左值

**字典字段赋值的特殊路径**：
- 纯赋值（`=`）且目标是字段访问时，走 `access_dict` + `dict.insert()` 路径
- 这允许为字典**创建新字段**，而不仅是修改已有字段
- 复合赋值（`+=` 等）不享受此特殊待遇，因为它们需要先读出现有值

### 5.3 解构赋值

`DestructAssignment`（如 `(a, b) = (1, 2)`）的求值在 [binding.rs#L30-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/binding.rs#L30-L42)：

```rust
impl Eval for ast::DestructAssignment<'_> {
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let value = self.value().eval(vm)?;
        destructure_impl(vm, self.pattern(), value, &mut |vm, expr, value| {
            let location = expr.access(vm)?;
            *location = value;
            Ok(())
        })?;
        Ok(Value::None)
    }
}
```

它复用了 `destructure_impl` 泛型函数（与 `let` 绑定共用），只是在叶子节点处用 `expr.access(vm)` 做可变写入，而不是 `vm.define()` 做新绑定。

### 5.4 可变方法调用

数组和字典有一些"原地修改"的方法（`push`、`pop`、`insert`、`remove`）。这些方法的调用路径不同于普通函数调用：

1. **识别可变方法**：`is_mutating_method()` 判断方法名（见 [methods.rs#L8-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/methods.rs#L8-L16)）
2. **可变访问目标**：用 `target.access(vm)` 获取 `&mut Value`
3. **调用可变方法**：`call_method_mut(&mut value, method, args, span)`

相关代码在 [call.rs#L186-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L186-L212) 的 `maybe_resolve_mutating` 函数，以及 [methods.rs#L23-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/methods.rs#L23-L63) 的 `call_method_mut` 函数。

### 5.5 访问器方法与左值穿透

`first`、`last`、`at` 这类"访问器方法"不仅能读取，还能作为左值被赋值（如 `arr.at(0) = 5`）。这通过 `call_method_access` 实现，它返回 `&mut Value`（见 [methods.rs#L65-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/methods.rs#L65-L98)）。

`Access` impl for `FuncCall` 会检查是否是访问器方法，如果是就走 `call_method_access` 路径返回可变引用，否则报错"cannot mutate a temporary value"（见 [access.rs#L56-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/access.rs#L56-L73)）。

---

## 6. 作用域系统：Scopes → Scope → Binding

### 6.1 三层结构

```
Scopes (作用域栈)
  ├── top: Scope (当前最内层作用域)
  ├── scopes: Vec<Scope> (外层作用域，从近到远)
  └── base: Option<&'a Library> (标准库全局作用域)
```

`Scopes` 定义在 [scope.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L17-L25)。

### 6.2 作用域进入与退出

```rust
impl Scopes {
    pub fn enter(&mut self) {
        // 把当前 top 保存到栈中，创建新的空 top
        self.scopes.push(std::mem::take(&mut self.top));
    }
    pub fn exit(&mut self) {
        // 弹出之前保存的 top，丢弃当前层
        self.top = self.scopes.pop().expect("no pushed scope");
    }
}
```

**谁触发 `enter`/`exit`？**

| 场景 | 代码位置 |
|---|---|
| `CodeBlock` 求值 | [code.rs#L320-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L320-L326) |
| `ContentBlock` 求值 | [code.rs#L331-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L331-L337) |
| `ForLoop` 循环体 | [flow.rs#L124-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs#L124-L146) |

注意 `WhileLoop` **没有** enter/exit —— 其循环变量不在新作用域中（因为 while 没有绑定变量的语法）。

### 6.3 变量查找：`Scopes::get()`

```rust
pub fn get(&self, var: &str) -> HintedStrResult<&Binding> {
    std::iter::once(&self.top)
        .chain(self.scopes.iter().rev())
        .find_map(|scope| scope.get(var))
        .or_else(|| {
            self.base.and_then(|base| match base.global.scope().get(var) {
                Some(binding) => Some(binding),
                None if var == "std" => Some(&base.std),
                None => None,
            })
        })
        .ok_or_else(|| unknown_variable(var))
}
```

查找顺序：**当前 top → 栈中外层（由近及远）→ 标准库全局作用域**。这是典型的词法作用域链查找。

数学模式有单独的 `get_in_math()`，它会查找 `base.math.scope()` 而非 `base.global.scope()`。

### 6.4 Binding：绑定值的元数据

```rust
pub struct Binding {
    value: Value,                    // 绑定的值
    kind: BindingKind,               // Normal 或 Captured(Capturer)
    span: Span,                      // 定义位置的 span
    category: Option<Category>,      // 分类
    deprecation: Option<Box<Deprecation>>,  // 弃用信息
}
```

关键区分（见 [scope.rs#L262-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L262-L268)）：

- `BindingKind::Normal` — 普通绑定，**可读写**
- `BindingKind::Captured(Capturer)` — 闭包/上下文捕获的副本，**只读**

`Binding::write()` 方法在遇到 `Captured` 时会报错（见 [scope.rs#L313-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L313-L325)）：

```rust
pub fn write(&mut self) -> StrResult<&mut Value> {
    match self.kind {
        BindingKind::Normal => Ok(&mut self.value),
        BindingKind::Captured(capturer) => bail!(
            "variables from outside the {} are read-only and cannot be modified",
            match capturer {
                Capturer::Function => "function",
                Capturer::Context => "context expression",
            },
        ),
    }
}
```

这就是**闭包捕获的变量不可修改**的根源。

---

## 7. 闭包：从定义到调用

### 7.1 闭包定义阶段

`ast::Closure` 的 `Eval` 实现（见 [call.rs#L562-L595](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L562-L595)）：

```rust
impl Eval for ast::Closure<'_> {
    type Output = Value;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        // 1. 求值命名参数的默认值
        let mut defaults = Vec::new();
        for param in self.params().children() {
            if let ast::Param::Named(named) = param {
                defaults.push(named.expr().eval(vm)?);
            }
        }

        // 2. 收集捕获的变量
        let captured = {
            let mut visitor = CapturesVisitor::new(Some(&vm.scopes), Capturer::Function);
            visitor.visit(self.to_untyped());
            visitor.finish()
        };

        // 3. 构建 Closure 对象
        let closure = Closure {
            node: ClosureNode::Closure(self.to_untyped().clone()),
            defaults,
            captured,
            num_pos_params: ...,
        };

        Ok(Value::Func(Func::from(closure).spanned(self.params().span())))
    }
}
```

**关键步骤解析**：

1. **默认值在定义时求值**：命名参数的默认值表达式在闭包定义时就被求值了，不是在调用时。
2. **CapturesVisitor 静态分析捕获**：遍历 AST，确定哪些自由变量需要从外部作用域捕获。
3. **闭包体（AST 节点）被保存但不在定义时求值**。

### 7.2 CapturesVisitor：闭包捕获分析

`CapturesVisitor` 是理解作用域和闭包行为的核心（见 [call.rs#L709-L882](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L709-L882)）。

它维护两组作用域：
- `external: Option<&'a Scopes<'a>>` — 引用当前 VM 的真实作用域
- `internal: Scopes<'a>` — 模拟的内部作用域（只用于跟踪哪些名字被绑定了，值无关紧要）
- `captures: Scope` — 收集到的捕获

**核心逻辑 `visit()` 方法**，对每种 AST 节点做不同处理：

| 节点类型 | 行为 |
|---|---|
| `Ident` / `MathIdent` | 尝试捕获（若不在 internal 中，则从 external 查找并捕获） |
| `CodeBlock` / `ContentBlock` | `internal.enter()`，遍历子节点，`internal.exit()` |
| `Closure` | 先访问命名参数默认值，再 `internal.enter()` 并 bind 参数名和函数名，然后访问函数体 |
| `LetBinding` | 先访问 init 表达式，再 bind 绑定名（绑定在 init 之后） |
| `ForLoop` | 先访问 iterable，再 `internal.enter()` + bind 模式变量 + 访问循环体 |
| `FieldAccess` | 只访问 target，不捕获 field 名 |
| `Named` | 只访问 expr，不捕获 name |

**捕获时做两件事**：
1. 调用 `binding.capture(capturer)` 创建一个 `BindingKind::Captured` 的副本
2. 将副本存入 `captures` Scope

这意味着**捕获发生在定义时，捕获的是值的快照**（clone），而非引用。

### 7.3 闭包调用阶段

`eval_closure()` 是闭包调用的核心（见 [call.rs#L600-L707](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L600-L707)）：

```rust
pub fn eval_closure(...) -> SourceResult<Value> {
    // 1. 从 AST 节点提取 name, params, body
    // 2. 构建全新的 Scopes，不继承调用方的
    let mut scopes = Scopes::new(None);  // 注意：base = None
    scopes.top = closure.captured.clone();  // 用捕获的变量作为顶层作用域

    // 3. 创建新的 Vm
    let mut vm = Vm::new(engine, context, scopes, body.span());

    // 4. 如果有函数名，绑定为递归调用
    if let Some(name) = name {
        vm.define(name, func.clone());
    }

    // 5. 绑定参数（详见下方）
    // 6. 求值函数体
    let output = body.eval(&mut vm)?;

    // 7. 处理 return 控制流
}
```

**关键行为**：
- **不继承调用方的任何作用域**：`Scopes::new(None)` 且 `scopes.top = closure.captured.clone()`
- 闭包只能访问**捕获的变量 + 参数 + 递归自引用**
- 这就是词法闭包的本质：函数体中的自由变量在**定义时**就已经确定了绑定

#### 7.3.1 闭包参数绑定详细流程

`eval_closure` 中参数绑定的核心代码（见 [call.rs#L647-L695](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L647-L695)）：

```rust
let num_pos_args = args.to_pos().len();
let sink_size = num_pos_args.checked_sub(closure.num_pos_params);

let mut sink = None;
let mut sink_pos_values = None;
let mut defaults = closure.defaults.iter();
for p in params.children() {
    match p {
        // 位置参数
        ast::Param::Pos(pattern) => match pattern {
            ast::Pattern::Normal(ast::Expr::Ident(ident)) => {
                vm.define(ident, args.expect::<Value>(&ident)?)
            }
            pattern => {
                crate::destructure(&mut vm, pattern,
                    args.expect::<Value>("pattern parameter")?)?;
            }
        },
        // 展开参数（argument sink）
        ast::Param::Spread(spread) => {
            sink = Some(spread.sink_ident());
            if let Some(sink_size) = sink_size {
                sink_pos_values = Some(args.consume(sink_size)?);
            }
        }
        // 命名参数
        ast::Param::Named(named) => {
            let name = named.name();
            let default = defaults.next().unwrap();
            let value =
                args.named::<Value>(&name)?.unwrap_or_else(|| default.clone());
            vm.define(name, value);
        }
    }
}

// 处理 argument sink 的剩余参数
if let Some(sink) = sink {
    let mut remaining_args = args.take();
    if let Some(sink_name) = sink {
        if let Some(sink_pos_values) = sink_pos_values {
            remaining_args.items.extend(sink_pos_values);
        }
        vm.define(sink_name, remaining_args);
    }
}

args.finish()?;
```

**参数绑定按 `params.children()` 的声明顺序逐一处理**（Typst 语法规定参数声明顺序为：位置参数 → 展开参数 → 命名参数，但代码层面并不假设固定顺序，而是逐一遍历 `params.children()` 并按每个参数类型分派）：

| 参数类型 | 绑定方式 |
|---|---|
| `Pos(Ident)` | `args.expect::<Value>(&ident)` — 从 `args` 中消费第一个位置参数并绑定 |
| `Pos(Pattern)` | `args.expect` 取值 + `destructure` 解构绑定（支持 `(a, b)` 模式） |
| `Spread(sink_ident)` | 记录 sink 标识符，**立即消费多余的位置参数**：`sink_size = 实际位置参数数 - 已声明的位置参数数`，用 `args.consume(sink_size)` 消费多余部分存入 `sink_pos_values` |
| `Named(name)` | `args.named::<Value>(&name)` 查找命名参数；未提供则使用 `defaults` 中的默认值（定义时已求值） |

**sink 参数的两阶段处理**：

1. **循环内（Spread 分支）：
   - `sink_size` 在循环之前就计算好了（`num_pos_args - num_pos_params`）
   - 遇到 `Spread` 参数时，**立即**调用 `args.consume(sink_size)?` 消费多余的位置参数，存入 `sink_pos_values`
   - 同时记录 `sink = Some(spread.sink_ident())`
   - 这一步消费的是**位置参数**的多余部分，不涉及命名参数

2. **循环后（第 683-692 行）**：
   - `args.take()` 拿走所有**剩余**参数（即命名参数、spread 后未被消费的参数）
   - 如果 sink 有名字（`sink_ident` 是 `Some`），将 `sink_pos_values`（位置参数多余部分）追加到 `remaining_args` 中
   - 然后把整合后的 `remaining_args` 绑定到 sink 变量

3. **如果 sink 没名字（裸 `..`）**：
   - 仍执行 `args.take()` 消费掉所有剩余参数
   - 但不绑定到任何变量（确保 `args.finish()` 通过，不会因为有未消费参数报错）

**为什么分两阶段？**
- 位置参数的多余部分必须在 Spread 分支内消费，因为后续的 Named 参数也在循环中会继续消费命名参数，而命名参数是按名字查找的，不受位置参数顺序无关。如果不先消费多余位置参数，那么 `expect` 等操作可能会从错误的位置取参数。

**命名参数默认值的使用**：`closure.defaults` 是一个与 AST 中命名参数一一对应的 `Vec<Value>`。在参数绑定循环中，`defaults.iter()` 按顺序逐一取出。`args.named()` 如果在 `args` 中找到了对应的命名参数就消费并返回它；如果没找到，就用 `default.clone()` 作为值。这确保了：
- 调用方显式传入的命名参数优先于默认值
- 默认值在**定义时**求值一次，调用时只是 clone

**参数消费模型**：`Args` 的消费是**破坏性的**——`expect`、`named`、`consume`、`take` 都会从 `items` 中移除已处理的参数。最终 `args.finish()` 检查是否还有未消费的参数，有则报错"unexpected argument"。

### 7.4 context 表达式的闭包

`context expr` 也创建闭包（见 [code.rs#L386-L410](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L386-L410)）：

```rust
impl Eval for ast::Contextual<'_> {
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let captured = {
            let mut visitor = CapturesVisitor::new(Some(&vm.scopes), Capturer::Context);
            visitor.visit(body.to_untyped());
            visitor.finish()
        };
        let closure = Closure {
            node: ClosureNode::Context(self.body().to_untyped().clone()),
            defaults: vec![],
            captured,
            num_pos_params: 0,
        };
        Ok(ContextElem::new(Func::from(closure)).pack())
    }
}
```

与函数闭包的区别：
- `Capturer::Context` 而非 `Capturer::Function`
- `ClosureNode::Context` 存储的是 Markup 节点而非 Closure 节点
- 无参数

### 7.5 函数调用的完整链路

一次函数调用 `f(a, b)` 涉及的调用链：

```
FuncCall::eval (call.rs#L23-L81)
  │
  ├─ 非字段访问调用：
  │   1. callee.eval(vm) → Value
  │   2. cast::<Func>() → Func
  │   3. args.eval(vm) → Args（参数求值，见 §7.6）
  │   4. call_func(vm, func, args, span)
  │      └─ func.call(engine, context, args)
  │         └─ func.call_impl(engine, context, args)  (func.rs#L324-L364)
  │            ├─ Native → native.function(engine, context, &mut args)
  │            ├─ Element → elem.construct(engine, &mut args)
  │            ├─ Closure → eval_closure(func, closure, ..., args)  (call.rs#L600)
  │            ├─ Plugin → func.call(inputs)
  │            └─ With → 预置 with 参数 + 递归 call
  │
  └─ 字段访问调用（如 arr.push(4)）：
      1. 检测到 callee 是 FieldAccess
      2. 判断是否是可变方法（is_mutating_method）
         ├─ 是可变方法 → maybe_resolve_mutating(vm, target, field, args, span)
         │   2a. 先求值参数 args.eval(vm)
         │   2b. target.access(vm) → &mut Value  （获取可变引用）
         │   2c. call_method_mut(&mut value, method, args, span)
         │       → 直接返回结果，不走 Func::call
         │   2d. 如果 target 不是 Array/Dict，回退到普通调用路径
         └─ 不是可变方法 → target.eval(vm) → Value
      3. eval_field_callee(vm, access, field, target)
         ├─ 方法查找：target.ty().scope().get(field)
         │   → 找到 → FieldCallee::Method(func, target)
         │   → args.insert(0, target_span, target)  （self 作为首参）
         │   → call_func(vm, func, args, span)
         ├─ 关联函数查找（Symbol/Func/Type/Module）：
         │   → target.field(field) → FieldCallee::Func(func)
         │   → call_func(vm, func, args, span)
         └─ 字典字段调用：报错"cannot directly call dictionary keys as functions"
```

**关键设计**：

1. **可变方法先于普通调用处理**：`maybe_resolve_mutating` 在 `eval_field_callee` 之前被调用，因为可变方法需要通过 `Access` trait 获取 `&mut Value`，而普通方法只需 `Eval` 返回的 `Value` 副本。两者不能混用——一旦 `access(vm)` 获取了可变借用，就无法再调用 `args.eval(vm)`，所以参数必须提前求值。

2. **方法调用隐式传入 self**：当 `eval_field_callee` 返回 `FieldCallee::Method` 时，`target` 被插入 `args` 的第一个位置（`args.insert(0, target_span, target)`），这样 Rust 侧的 native 函数签名统一为 `(engine, context, &self, ...)` 形式。

3. **字典字段不能直接调用**：`eval_field_callee` 对字典和命名参数（`Args`）做了专门拦截——即使字典中存着一个函数值，也不能用 `dict.key()` 语法调用它，因为这会和类型方法产生歧义。需要用 `(dict.key)(args)` 显式调用。

4. **Func::With 的预置参数**：`Func::with` 创建一个 `FuncInner::With` 变体，把预置参数存在 `Arc<(Func, Args)>` 中。调用时，`with.1.items` 被前置到实际参数之前（`args.items = with.1.items.iter().cloned().chain(args.items).collect()`），然后委托给底层函数。

5. **调用深度检查**：`FuncCall::eval` 开头先调用 `vm.engine.route.check_call_depth()`，防止无限递归。

### 7.6 参数求值与展开：`Args::eval`

`ast::Args` 的 `Eval` 实现（见 [call.rs#L367-L417](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L367-L417)）将 AST 参数节点转换为运行时 `Args`：

```rust
impl Eval for ast::Args<'_> {
    type Output = Args;

    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let mut items = EcoVec::with_capacity(self.items().count());

        for arg in self.items() {
            let span = arg.span();
            match arg {
                ast::Arg::Pos(expr) => {
                    items.push(Arg {
                        span,
                        name: None,
                        value: Spanned::new(expr.eval(vm)?, expr.span()),
                    });
                }
                ast::Arg::Named(named) => {
                    let expr = named.expr();
                    items.push(Arg {
                        span,
                        name: Some(named.name().get().clone().into()),
                        value: Spanned::new(expr.eval(vm)?, expr.span()),
                    });
                }
                ast::Arg::Spread(spread) => match spread.expr().eval(vm)? {
                    Value::None => {}
                    Value::Array(array) => {
                        items.extend(array.into_iter().map(|value| Arg {
                            span,
                            name: None,
                            value: Spanned::new(value, span),
                        }));
                    }
                    Value::Dict(dict) => {
                        items.extend(dict.into_iter().map(|(key, value)| Arg {
                            span,
                            name: Some(key),
                            value: Spanned::new(value, span),
                        }));
                    }
                    Value::Args(args) => items.extend(args.items),
                    v => bail!(spread.span(), "cannot spread {}", v.ty()),
                },
            }
        }

        Ok(Args { span: Span::detached(), items })
    }
}
```

**三种参数类型的处理**：

| 语法 | AST 节点 | 运行时 `Arg` |
|---|---|---|
| `f(1, 2)` | `Arg::Pos(expr)` | `{ name: None, value: ... }` |
| `f(x: 1)` | `Arg::Named(named)` | `{ name: Some("x"), value: ... }` |
| `f(..arr)` | `Arg::Spread(spread)` | 展开为多个 `Arg` |

**展开（spread）的规则**：
- `..none` — 忽略，不产生任何参数
- `..array` — 展开为多个**位置参数**（`name: None`）
- `..dict` — 展开为多个**命名参数**（`name: Some(key)`）
- `..args` — 直接拼接 `args.items`（保留原始的 name/value 结构）
- 其他类型 — 报错

**参数求值顺序**：参数按源码中从左到右的顺序逐一求值，**没有**惰性求值或重排序。每个参数表达式在被遍历到时立即求值。

**位置参数与命名参数的统一存储**：`Args.items` 是一个扁平的 `EcoVec<Arg>`，位置参数和命名参数混在一起。消费时通过 `slot.name.is_none()` 区分。`Args::expect()` 从中找第一个 `name: None` 的参数消费，`Args::named()` 找对应 `name` 的参数消费。

---

## 8. 典型求值路径详解

### 8.1 简单变量引用

```
源码: #x
  │
  ▼ 解析
Expr::Ident(Ident)
  │
  ▼ Eval for Ident (code.rs#L156-L168)
vm.scopes.get(&self)        // 从作用域链查找
  .at(span)?                // 找不到则报错
  .read_checked(...)        // 读取值（检查弃用）
  .clone()                  // 克隆返回
```

### 8.2 let 绑定

```
源码: #let x = 1 + 2
  │
  ▼ 解析
Expr::LetBinding(LetBinding)
  │
  ▼ Eval for LetBinding (binding.rs#L9-L28)
1. eval init expr → Value::Int(3)
2. match kind:
   - Normal(pattern) → destructure(vm, pattern, value)
   - Closure(ident) → vm.define(ident, value)
3. 返回 Value::None
```

`vm.define()` 调用 `vm.scopes.top.bind(name, Binding::new(value, span))`，将变量绑定到**当前最内层作用域**。

### 8.3 代码块

```
源码: #{ let y = 1; y + 2 }
  │
  ▼ 解析
Expr::CodeBlock(CodeBlock { body: Code })
  │
  ▼ Eval for CodeBlock (code.rs#L317-L326)
1. vm.scopes.enter()   ← 进入新作用域
2. self.body().eval(vm)  → eval Code → eval_code()
   - eval "let y = 1" → 在新 top 中绑定 y
   - eval "y + 2"     → 从作用域查找 y，得到 1，计算 3
3. vm.scopes.exit()    ← 退出作用域，y 不再可见
4. 返回 Value::Int(3)
```

### 8.4 闭包定义与调用

```
源码:
#let outer = 10
#let f = (x) => x + outer
#f(5)  // 结果: 15
```

**定义 `f` 时**：

```
1. eval "10" → Value::Int(10)
2. vm.define("outer", Value::Int(10))
3. eval Closure "(x) => x + outer"
   a. 无命名参数默认值
   b. CapturesVisitor 分析:
      - "x" → internal 中有绑定（参数），不捕获
      - "outer" → internal 中没有 → 从 external 查找 → 捕获
      - captured = Scope { "outer": Binding { value: Int(10), kind: Captured(Function) } }
   c. Closure { node, defaults: [], captured, num_pos_params: 1 }
4. vm.define("f", Func::from(closure))
```

**调用 `f(5)` 时**：

```
1. FuncCall::eval:
   a. callee = "f" (Ident)，不是 FieldAccess，走普通调用路径
   b. callee.eval(vm) → Func (从 scopes 查找)
   c. cast::<Func>() → Func (closure 类型)
   d. args.eval(vm) → Args { items: [Arg { name: None, value: Int(5) }] }

2. call_func → Func::call → call_impl:
   匹配 FuncInner::Closure → eval_closure(func, closure, ..., args)

3. eval_closure:
   a. scopes = Scopes::new(None)  // 不继承调用方
   b. scopes.top = closure.captured.clone()  // { "outer": Int(10) }
   c. 创建新 Vm
   d. 无函数名，跳过递归自引用
   e. 参数绑定循环：
      - Param::Pos(Ident("x")) → args.expect::<Value>("x") → Int(5)
        → vm.define("x", Int(5))
   f. 无 sink 参数
   g. args.finish() → Ok（无多余参数）
   h. eval body "x + outer"
      - 查找 x → 当前 top → Int(5)
      - 查找 outer → 当前 top（captured）→ Int(10)
      - 5 + 10 = 15
4. 返回 Value::Int(15)
```

### 8.5 带完整参数类型的闭包调用

```
源码:
#let f = (a, b: 10, ..sink) => {
  (a, b, sink.pos(), sink.named())
}
#f(1, c: 20, 2, 3)
// 结果: (1, 10, (2, 3), (c: 20))
```

**定义 `f` 时**：

```
1. 求值命名参数默认值：defaults = [Value::Int(10)]  // b 的默认值
2. CapturesVisitor 分析：无外部自由变量
3. Closure { node, defaults: [Int(10)], captured: Scope::new(), num_pos_params: 1 }
```

**调用 `f(1, c: 20, 2, 3)` 时**：

```
1. args.eval(vm) → Args {
     items: [
       Arg { name: None, value: Int(1) },      // 1
       Arg { name: Some("c"), value: Int(20) }, // c: 20
       Arg { name: None, value: Int(2) },       // 2
       Arg { name: None, value: Int(3) },       // 3
     ]
   }

2. eval_closure 参数绑定:
   num_pos_args = 3（位置参数：1, 2, 3）
   num_pos_params = 1（声明的位置参数：a）
   sink_size = 3 - 1 = 2

   参数绑定循环:
   - Param::Pos(Ident("a")) → args.expect("a") → Int(1)
     args 剩余: [Arg(c:20), Arg(2), Arg(3)]
     vm.define("a", Int(1))

   - Param::Named("b") → args.named("b") → None（没找到）
     default = defaults.next() → Int(10)
     vm.define("b", Int(10))
     args 剩余不变: [Arg(c:20), Arg(2), Arg(3)]

   - Param::Spread(sink_ident=Some("sink")) →
     sink = Some(Some("sink"))
     sink_pos_values = args.consume(2) → [Arg(2), Arg(3)]
     args 剩余: [Arg(c:20)]

   循环结束后处理 sink:
   - remaining_args = args.take() → Args { items: [Arg(c:20)] }
   - remaining_args.items.extend(sink_pos_values)
     → remaining_args = Args { items: [Arg(c:20), Arg(2), Arg(3)] }
   - vm.define("sink", remaining_args)

   args.finish() → Ok

3. eval body → (1, 10, (2, 3), (c: 20))
```

### 8.6 闭包中的变量遮蔽与不可修改

```
源码:
#let x = 1
#let f = () => {
  x = 2  // 错误！
}
```

调用 `f` 时：
- `x` 被捕获为 `BindingKind::Captured(Function)`
- `x = 2` 尝试 `Binding::write()` → 失败，因为 Captured 绑定是只读的

### 8.7 闭包中调用可变方法：完全只读

#### 8.7.1 直接修改捕获变量：报错

```
源码:
#let arr = (1, 2, 3)
#let f = () => { arr.push(4) }
#f()
```

这段代码**会报错**，错误信息为：

```
variables from outside the function are read-only and cannot be modified
```

**调用路径分析**：

```
arr.push(4)
  │
  ▼ FuncCall::eval (call.rs#L23)
  检测到 callee 是 FieldAccess，且 push 是可变方法
  │
  ▼ maybe_resolve_mutating(vm, target=arr, field=push, args, span)
  1. args.eval(vm)  →  Args { [Arg(name=None, value=4)] }
  2. target.access(vm)
     │
     ▼ Ident::access (access.rs#L29-L42)
        vm.scopes.get_mut("arr")  →  &mut Binding
        Binding::write()          →  检查 kind
        kind == Captured(Function) →  报错！
```

错误发生在 `Binding::write()` 处（[scope.rs#L313-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L313-L325)），因为 `Captured` 绑定被设计为完全只读。

#### 8.7.2 通过字典字段间接修改：仍然报错

一个常见的误区是："把数组放进字典里，通过 `data.arr.push(4)` 是不是就能绕过只读检查？"

```
源码:
#let data = (arr: (1, 2, 3))
#let f = () => { data.arr.push(4) }
#f()
```

**答案是：也会报同样的错误。**

原因在于 `FieldAccess::access` 的实现是**递归的**（[access.rs#L50-L53](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/access.rs#L50-L53)）：

```rust
impl Access for ast::FieldAccess<'_> {
    fn access<'a>(self, vm: &'a mut Vm) -> SourceResult<&'a mut Value> {
        access_dict(vm, self)?.at_mut(self.field().get()).at(self.span())
    }
}
```

而 `access_dict` 会递归调用 `access.target().access(vm)`（[access.rs#L76-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/access.rs#L76-L107)）：

```rust
pub(crate) fn access_dict<'a>(vm: &'a mut Vm, access: ast::FieldAccess)
    -> SourceResult<&'a mut Dict>
{
    match access.target().access(vm)? {  // ← 递归获取 target 的可变引用
        Value::Dict(dict) => Ok(dict),
        ...
    }
}
```

**完整调用路径**：

```
data.arr.push(4)
  │
  ▼ FuncCall::eval
  callee = FieldAccess(data.arr), 可变方法 = push
  │
  ▼ maybe_resolve_mutating(vm, target=data.arr, field=push, args, span)
  1. args.eval(vm) → Args { [4] }
  2. target.access(vm)   // target 是 data.arr (FieldAccess)
     │
     ▼ FieldAccess::access (data.arr)
        access_dict(vm, access=data.arr)
          │
          ▼ access.target().access(vm)   // access.target() = data (Ident)
             │
             ▼ Ident::access("data")
                vm.scopes.get_mut("data") → &mut Binding { kind: Captured(Function) }
                Binding::write() → 报错！
```

递归一直追溯到最外层的 `Ident("data")`，而 `data` 是 Captured 绑定，所以 `Binding::write()` 同样会报错。

> **结论**：闭包中对捕获变量的**任何**修改尝试——无论是直接赋值、复合赋值、字段赋值、还是可变方法调用——最终都会在 `Binding::write()` 处被拦截。闭包捕获的变量是**完全只读**的，没有任何方式可以绕过。
>
> 这是设计使然：值捕获（capture by value）+ 不可变语义，确保闭包的行为是可预测的，不会产生意外的副作用。

### 8.8 for 循环中的作用域

```
源码:
#for x in (1, 2, 3) {
  let y = x * 2
}
// y 不可见
```

求值流程（见 [flow.rs#L114-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs#L114-L189)）：

```
1. eval iterable → (1, 2, 3)
2. vm.scopes.enter()   ← 进入新作用域
3. 对每个元素:
   a. destructure(vm, pattern, value) → vm.define("x", 1)
   b. eval body → eval CodeBlock
      - vm.scopes.enter()  ← 代码块又进入一层
      - vm.define("y", 2)
      - vm.scopes.exit()
   c. 下一轮: x 被重新绑定
4. vm.scopes.exit()    ← 退出，x 和 y 都不可见
```

### 8.9 二元运算求值路径

```
源码: #2 + 3 * 4
  │
  ▼ 解析（Pratt 算法处理优先级）
Expr::Binary {
  op: Add,
  lhs: Expr::Int(2),
  rhs: Expr::Binary {
    op: Mul,
    lhs: Expr::Int(3),
    rhs: Expr::Int(4)
  }
}
  │
  ▼ Eval for Binary (ops.rs#L21-L47)
  1. eval lhs → 2
  2. 非短路运算，eval rhs → 12
     （递归调用 Binary.eval，先算 3*4）
  3. ops::add(2, 12) → 14
  4. 返回 Value::Int(14)
```

**关键点**：
- 优先级在**解析阶段**就通过 Pratt 算法确定了，树的结构本身就体现了结合性
- 求值时按树的后序遍历，左→右→根
- `and` / `or` 有短路优化：左操作数已能确定结果时不求右操作数

### 8.10 赋值写入完整路径

```
源码: #x += 5
  │
  ▼ 解析
Expr::Binary { op: AddAssign, lhs: Expr::Ident("x"), rhs: Expr::Int(5) }
  │
  ▼ Eval for Binary → apply_assignment (ops.rs#L68-L91)
  1. eval rhs → Value::Int(5)
  2. lhs.access(vm)  ← 调用 Access trait
     a. vm.scopes.get_mut("x") → &mut Binding
     b. binding.write() → &mut Value  （检查 BindingKind，Captured 会报错）
  3. std::mem::take(location)  ← 取出旧值
  4. ops::add(old_value, 5) → 新值
  5. *location = 新值  ← 写回
  6. 返回 Value::None
```

**字段赋值的特殊路径**：

```
源码: #dict.key = "value"
  │
  ▼ 解析
Expr::Binary { op: Assign, lhs: Expr::FieldAccess(...), rhs: ... }
  │
  ▼ apply_assignment
  1. 检测到 op == Assign 且 lhs 是 FieldAccess
  2. access_dict(vm, access) → &mut Dict  （获取可变字典引用）
  3. dict.insert("key", "value")  ← 直接插入，可创建新字段
  4. 返回 Value::None
```

---

## 9. 完整路径总结图

```
┌──────────────────────────────────────────────────────────────────┐
│                        源代码字符串                               │
│                 "#let f = (x) => x + outer"                      │
└──────────────────────┬───────────────────────────────────────────┘
                       │ parse() / parse_code()
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CST (SyntaxNode 树)                            │
│  root: SyntaxKind::Markup                                        │
│    └─ SyntaxKind::Hash                                           │
│    └─ SyntaxKind::LetBinding                                     │
│        ├─ SyntaxKind::Ident "f"                                  │
│        ├─ SyntaxKind::Eq                                         │
│        └─ SyntaxKind::Closure                                    │
│            ├─ SyntaxKind::Params "(x)"                           │
│            └─ SyntaxKind::Body "=> x + outer"                    │
└──────────────────────┬───────────────────────────────────────────┘
                       │ cast::<ast::Markup>()
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│               AST (类型化视图)                                     │
│  Markup.exprs() → [Expr::LetBinding(LetBinding)]                 │
│    .kind() = Closure(ident="f")                                  │
│    .init() = Expr::Closure(Closure)                              │
│      .params() = [Param::Pos(Pattern::Normal(Expr::Ident("x")))]│
│      .body()  = Expr::Binary(x + outer)                          │
└──────────────────────┬───────────────────────────────────────────┘
                       │ .eval(&mut vm)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    运行时求值                                      │
│                                                                  │
│  Vm { scopes: Scopes { top, scopes, base: Library } }           │
│                                                                  │
│  1. eval LetBinding:                                             │
│     eval init → eval Closure                                     │
│       a. defaults = []                                           │
│       b. CapturesVisitor:                                        │
│          internal: [x]  (参数绑定)                                │
│          external → vm.scopes                                    │
│          "x"     → internal 有，不捕获                            │
│          "outer" → internal 无，从 external 获取并捕获             │
│          captured = Scope { "outer": Binding::Captured(...) }    │
│       c. Closure { node, captured, ... }                         │
│     vm.define("f", Func::from(closure))                          │
│                                                                  │
│  2. 调用 f(5) 时:                                                │
│     eval_closure():                                              │
│       scopes = Scopes::new(None)                                 │
│       scopes.top = closure.captured  // { "outer": ... }         │
│       vm.define("x", 5)                                          │
│       eval body: x + outer → 5 + <captured> → 返回值             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. 关键设计要点总结

1. **词法作用域**：变量查找沿作用域链从内到外，最远到标准库全局。不存在动态作用域。

2. **值捕获，非引用捕获，且捕获变量完全只读**：闭包通过 `CapturesVisitor` 在**定义时**静态分析需要捕获的变量，然后 clone 值存入 `captured` Scope。调用时完全不继承调用方的作用域。被捕获的变量标记为 `BindingKind::Captured`，**任何**修改尝试——赋值语句、复合赋值、字段赋值、可变方法调用——都会在 `Binding::write()` 处被拦截。即使是通过字典字段间接访问（如 `data.arr.push(4)`）也不例外，因为 `FieldAccess::access` 会递归调用 `target.access()` 一直追溯到最外层的标识符。

3. **代码块/内容块自动创建作用域**：`CodeBlock.eval()` 和 `ContentBlock.eval()` 会 `enter`/`exit` 作用域，`ForLoop` 也为循环体创建新作用域。

4. **let 绑定在 init 之后生效**：`CapturesVisitor` 处理 `LetBinding` 时先访问 `init` 再 `bind` 名称，因此 `#let x = x` 中的右侧 `x` 引用的是外层的 `x`。

5. **闭包默认值在定义时求值**：命名参数的默认值在 `Closure.eval()` 中立即求值，存入 `closure.defaults`，调用时通过 `args.named::<Value>(&name)?.unwrap_or_else(|| default.clone())` 使用。

6. **递归自引用**：`eval_closure()` 中，如果闭包有名字，会把自身绑定到新 Vm 的作用域中，支持递归调用。

7. **模块求值产生独立作用域**：`eval()` 函数求值一个文件后，`vm.scopes.top` 被提取为模块的导出作用域（`Module::new(name, vm.scopes.top)`），外部只能通过 import 访问。

8. **双 trait 求值模型**：`Eval`  trait 用于只读求值（返回 `Value`），`Access` trait 用于可变左值访问（返回 `&mut Value`）。只有标识符、括号、字段访问和访问器方法调用能作为左值。

9. **优先级在解析时确定**：Pratt 算法在解析阶段就根据运算符优先级和结合性构建出正确的 AST 结构，求值阶段只需后序遍历，不需要再处理优先级。

10. **Set/Show 规则的"作用于剩余"语义**：set/show 不是普通表达式，它们通过递归调用 `eval_code`/`eval_markup` 处理所有后续表达式，然后把样式应用到结果上。

11. **值拼接（ops::join）**：顺序求值中每步的结果通过 `ops::join` 累积，内容值会合并成 sequence，`none` 会被吸收，其他不兼容类型会报错。

12. **字典字段赋值的特殊路径**：对字典字段的纯赋值（`=`）走 `dict.insert()` 路径，支持创建新字段；复合赋值（`+=` 等）走通用 Access 路径，要求字段已存在。

13. **函数调用的方法分派优先级**：字段访问调用（如 `x.f()`）优先在 `x` 的类型作用域中查找方法（`ty().scope().get(f)`），找不到才查找值自身的字段。字典字段不允许用方法语法调用。

14. **可变方法需要 Access 而非 Eval**：`arr.push(4)` 之所以能修改 `arr`，不是因为 `push` 是特殊的语言级操作，而是因为 `FuncCall::eval` 检测到可变方法名后，改走 `target.access(vm)` → `call_method_mut` 路径，获取 `&mut Value` 实现原地修改。

15. **参数消费模型是破坏性的**：`Args` 的 `expect`/`named`/`consume`/`take` 都会从 `items` 中移除已处理的参数。`eval_closure` 按 `params.children()` 的声明顺序遍历处理每个参数，最后 `args.finish()` 确保无多余参数。

16. **sink 参数的两阶段处理**：`Spread` 参数在循环的 Spread 分支中就调用 `args.consume(sink_size)` 消费掉多余的**位置参数**（`consume` 会跳过命名参数只取位置参数）；循环结束后再 `args.take()` 取走所有剩余参数（主要是命名参数），两部分合并后绑定到 sink 变量。
