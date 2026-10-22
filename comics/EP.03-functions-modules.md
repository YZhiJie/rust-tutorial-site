# EP.03 函数、模块与 Cargo

> 对应正文：[guide/03-functions-modules.md](../docs/guide/03-functions-modules.md)（函数签名与尾表达式 / 模块、可见性与路径 / 常用 Cargo 命令）

## 剧情回顾

- **第 1 格**：齿轮君的 `double` 函数多写一个分号，`n * 2;` 变成语句，报 `error[E0308]: mismatched types`，期望 `i32` 得到 `()`。
- **第 2 格**：码叔讲表达式与语句的分水岭：尾表达式不加分号就是返回值，`return` 只用于提前返回。
- **第 3 格**：像素酱把 `add` 收进 `mod utils`，加 `pub` 后用 `use crate::utils::add` 引入；忘写 `pub` 会报 E0603。
- **第 4 格**：码叔结案——分号、pub、路径起点各管一件事，别混。

## 知识点拆解

### 第 1 格 · 尾表达式与 E0308

- 函数体最后一个表达式不加分号，它的值就是返回值；
- `n * 2;` 加上分号就变成语句，函数返回类型随即变成 `()`，与声明的 `i32` 不匹配——`error[E0308]: mismatched types`，`expected i32, found ()`；
- 函数签名中参数必须标注类型；返回类型省略时等价于 `()`。

### 第 2 格 · 表达式有值，语句没有

```rust
fn double(n: i32) -> i32 {
    if n < 0 { return -n * 2; }  // 提前返回用 return
    n * 2                         // 尾表达式即返回值
}
```

- 表达式产生值；语句以分号结尾，不产生值；
- `return` 用于提前结束函数；需要返回多个值时用元组，调用处用模式解构接收；
- 参数按值传递：`Copy` 类型被复制，`String` 这类类型会发生移动；要保留所有权就传引用。

### 第 3 格 · mod、pub 与 use

```rust
mod utils {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }
}
use crate::utils::add;
```

- 模块内一切默认私有，`pub` 才把项暴露给外部；外部调用私有项报 `error[E0603]: function ... is private`；
- 路径起点有三种：`crate::` 表示当前包根，`self::` 表示当前模块，`super::` 表示父模块；
- `use` 只把路径引入当前作用域，不改变可见性；文件模块写 `mod utils;`，对应 `src/utils.rs` 或 `src/utils/mod.rs`，两者只能选一种，都找不到时报 `E0583: file not found for module`。

### 第 4 格 · Cargo 速览

- `cargo new wordcount --bin` 新建项目；`cargo check` 只做类型检查，比 build 快；`cargo run` 编译并运行；`cargo test` 运行测试；`cargo clippy -- -D warnings` 静态检查；`cargo build --release` 优化构建；
- `Cargo.toml` 的 `[dependencies]` 是运行时依赖，`[dev-dependencies]` 只用于测试与示例；版本 `"1"` 等价于 `^1`，精确锁定写 `=1.2.3`；
- `Cargo.lock` 记录实际解析出的版本：可执行程序应提交它，库通常不提交。

## 坑点清单

> 🔗 正文对照：[《函数、模块与 Cargo · 常见错误》](../docs/guide/03-functions-modules.md)

1. 尾表达式多写分号 → `error[E0308]`，返回值悄悄变成 `()`，是类型不匹配的高频原因；
2. 忘写 `pub` → `error[E0603]`，模块内的项默认私有；声明 `mod utils;` 却没有对应文件 → `E0583`；
3. `crate::` / `self::` / `super::` 三个路径起点混用，路径越绕越晕，能从 `crate::` 起步就别绕。

## 自测 3 题（含答案）

1. 为什么 `n * 2;` 会让函数报 `error[E0308]`？——**分号把表达式变成语句，函数返回值从 `i32` 变成 `()`，与签名声明不符**。
2. 模块里的函数外部默认能调用吗？——**不能，默认私有；必须加 `pub`，否则外部调用报 `error[E0603]`**。
3. `use` 会把私有项变成公开的吗？——**不会，`use` 只把路径引入作用域；可见性只能由 `pub` 控制**。
