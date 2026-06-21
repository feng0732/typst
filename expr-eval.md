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

    // 5. 绑定参数
    // 6. 求值函数体
    let output = body.eval(&mut vm)?;

    // 7. 处理 return 控制流
}
```

**关键行为**：
- **不继承调用方的任何作用域**：`Scopes::new(None)` 且 `scopes.top = closure.captured.clone()`
- 闭包只能访问**捕获的变量 + 参数 + 递归自引用**
- 这就是词法闭包的本质：函数体中的自由变量在**定义时**就已经确定了绑定

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
1. eval "f" → Func (从 scopes 查找)
2. eval Args "(5)" → Args { items: [Arg { name: None, value: Int(5) }] }
3. call_func → Func::call → eval_closure
   a. scopes = Scopes::new(None)  // 不继承调用方
   b. scopes.top = closure.captured.clone()  // { "outer": Int(10) }
   c. vm.define("x", Int(5))  // 绑定参数
   d. eval "x + outer"
      - 查找 x → 当前 top → Int(5)
      - 查找 outer → 当前 top（captured）→ Int(10)
      - 5 + 10 = 15
4. 返回 Value::Int(15)
```

### 8.5 闭包中的变量遮蔽与不可修改

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

### 8.6 闭包修改可变引用（数组/字典的 in-place 方法）

```
源码:
#let arr = (1, 2, 3)
#let f = () => { arr.push(4) }
#f()
```

这里 `arr` 被捕获的是**值的 clone**，所以 `arr.push(4)` 修改的是捕获的副本，不影响外部的 `arr`。Typst 的闭包是**值捕获**（capture by value），不是引用捕获。

### 8.7 for 循环中的作用域

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

### 8.8 二元运算求值路径

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

### 8.9 赋值写入完整路径

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

2. **值捕获，非引用捕获**：闭包通过 `CapturesVisitor` 在**定义时**静态分析需要捕获的变量，然后 clone 值存入 `captured` Scope。调用时完全不继承调用方的作用域。

3. **捕获变量只读**：被捕获的变量标记为 `BindingKind::Captured`，`write()` 会失败。这是"闭包捕获变量不可修改"的实现根源。

4. **代码块/内容块自动创建作用域**：`CodeBlock.eval()` 和 `ContentBlock.eval()` 会 `enter`/`exit` 作用域，`ForLoop` 也为循环体创建新作用域。

5. **let 绑定在 init 之后生效**：`CapturesVisitor` 处理 `LetBinding` 时先访问 `init` 再 `bind` 名称，因此 `#let x = x` 中的右侧 `x` 引用的是外层的 `x`。

6. **闭包默认值在定义时求值**：命名参数的默认值在 `Closure.eval()` 中立即求值，存入 `closure.defaults`，调用时通过 `args.named::<Value>(&name)?.unwrap_or_else(|| default.clone())` 使用。

7. **递归自引用**：`eval_closure()` 中，如果闭包有名字，会把自身绑定到新 Vm 的作用域中，支持递归调用。

8. **模块求值产生独立作用域**：`eval()` 函数求值一个文件后，`vm.scopes.top` 被提取为模块的导出作用域（`Module::new(name, vm.scopes.top)`），外部只能通过 import 访问。

9. **双 trait 求值模型**：`Eval`  trait 用于只读求值（返回 `Value`），`Access` trait 用于可变左值访问（返回 `&mut Value`）。只有标识符、括号、字段访问和访问器方法调用能作为左值。

10. **优先级在解析时确定**：Pratt 算法在解析阶段就根据运算符优先级和结合性构建出正确的 AST 结构，求值阶段只需后序遍历，不需要再处理优先级。

11. **Set/Show 规则的"作用于剩余"语义**：set/show 不是普通表达式，它们通过递归调用 `eval_code`/`eval_markup` 处理所有后续表达式，然后把样式应用到结果上。

12. **值拼接（ops::join）**：顺序求值中每步的结果通过 `ops::join` 累积，内容值会合并成 sequence，`none` 会被吸收，其他不兼容类型会报错。

13. **字典字段赋值的特殊路径**：对字典字段的纯赋值（`=`）走 `dict.insert()` 路径，支持创建新字段；复合赋值（`+=` 等）走通用 Access 路径，要求字段已存在。
