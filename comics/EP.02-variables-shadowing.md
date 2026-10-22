# EP.02 变量、遮蔽与类型

> 对应正文：[guide/02-variables-types.md](../docs/guide/02-variables-types.md)（let、mut 与遮蔽 / 整数溢出 / 控制流是表达式）

## 剧情回顾

- **第 1 格**：齿轮君写下 `let x = 5; x = 6;`，收获 `error[E0384]: cannot assign twice to immutable variable`——变量默认不可变。
- **第 2 格**：码叔区分两招：`let mut x` 允许修改同一个绑定；`let x = x + 1` 是遮蔽，创建全新绑定，还能换类型。
- **第 3 格**：像素酱实证控制流是表达式——`if` 直接算出值，`loop` 用 `break value` 把结果带出循环。
- **第 4 格**：码叔结案三条坑点，特别提醒 debug 与 release 下整数溢出的行为差异。

## 知识点拆解

### 第 1 格 · 默认不可变与 E0384

- `let x = 5;` 创建的是不可变绑定，二次赋值触发 `error[E0384]: cannot assign twice to immutable variable`；
- 编译器会直接给出建议：`help: consider making this binding mutable: mut x`；
- 默认不可变不是风格建议，而是语言规则：编译器据此确认值在作用域内何时被修改，从而支撑借用检查与并发安全。

### 第 2 格 · mut 与遮蔽的区别

```rust
let mut x = 5;
x = 6;            // 修改同一个绑定，类型不变
let x = x + 1;    // 遮蔽：创建全新绑定
let x = "文本";   // 新绑定甚至可以换类型
```

- `mut` 允许重新赋值，但类型不能变；同一作用域内仍只有一个可变绑定；
- 遮蔽复用名字生成新绑定，旧绑定并未被修改，只是被遮住；旧值通常在最后一次使用后结束生命周期；
- 常见误区：忘记 `mut` 后用 `let` 遮蔽，得到的仍是不可变绑定，再赋值照样报 E0384。

### 第 3 格 · 控制流是表达式

```rust
let n = if cond { 1 } else { 2 };
let square = loop {
    count += 1;
    if count * count > 50 {
        break count * count; // break 带值，作为 loop 的结果
    }
};
```

- `if` 是表达式，两个分支必须产生相同类型，否则 `error[E0308]: if and else have incompatible types`；
- 条件必须是 `bool`，不存在"非零即真"的隐式判断；
- `loop` 可以用 `break value` 把值带出循环；`while`、`for` 同样都是表达式。

### 第 4 格 · 整数溢出

- debug 构建默认在溢出时 panic（`attempt to add with overflow`）；release 构建默认按二进制补码回绕，但不应依赖这一行为；
- 需要明确语义时用标准库方法：`checked_add` 返回 `Option`（溢出为 `None`）、`wrapping_add` 回绕、`saturating_add` 停在边界值、`overflowing_add` 返回值加溢出标志；
- 对外部输入或可能越界的计算，优先 `checked_*` 并显式处理 `None`。

## 坑点清单

> 🔗 正文对照：[《变量、类型与控制流 · 常见错误》](../docs/guide/02-variables-types.md)

1. 想改值忘加 `mut` → `error[E0384]`；但也别滥用 `mut`，能用遮蔽表达意图就不必可变；
2. 遮蔽 ≠ 可变：`let x = x + 1` 之后再 `x += 1` 仍报 E0384，因为新绑定默认不可变；
3. 依赖 release 的溢出回绕 → debug 与 release 行为不一致，用 `checked_add` 等显式处理溢出。

## 自测 3 题（含答案）

1. `let x = 5; x = 6;` 会报什么错？——**`error[E0384]`，变量默认不可变；想修改要用 `let mut x` 显式声明**。
2. 遮蔽和 `mut` 有什么区别？——**`mut` 是修改同一个绑定（类型不变）；遮蔽是 `let` 同名创建全新绑定，可以换类型，旧绑定只是被遮住**。
3. `i32::MAX + 1` 在 debug 与 release 下分别会怎样？——**debug 下 panic（attempt to add with overflow），release 下回绕；应使用 `checked_add` 等方法显式处理**。
