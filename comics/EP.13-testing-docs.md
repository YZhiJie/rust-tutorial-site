# EP.13 测试与文档

> 对应正文：[guide/13-testing.md](../docs/guide/13-testing.md)（测试函数的形状 / 断言宏 / 模块内单元测试 / 集成测试 / 文档测试）

## 剧情回顾

- **第 1 格**：齿轮君改完代码全靠手点一遍验证——不可重复，回归全靠记忆力，谁也不敢发布。
- **第 2 格**：码叔给出最小形状：`#[test]` + `cargo test`；`assert!`/`assert_eq!`/`assert_ne!` 表达期望；`#[cfg(test)]` 模块只在测试构建时编译。
- **第 3 格**：像素酱补齐武器库：`#[should_panic]` 验证报错路径、返回 `Result` 的测试用 `?`、`tests/` 目录做集成测试、`///` 文档示例随 `cargo test` 真跑。
- **第 4 格**：码叔结案：单元、集成、文档三层测试，构成可重复的质量基线。

## 知识点拆解

### 第 1 格 · 为什么手点不算验证

- 借用检查器能排除内存错误与数据竞争，却无法判断业务规则是否正确：税率算错、边界漏判，编译器都不会报错；
- 手工运行加 `println!` 验证的问题在于不可重复——改动一行代码后，没人能保证之前验证过的路径仍然正确；
- 测试的本质是把期望写成可执行代码，让机器在每次修改后重新验证；Rust 把测试工具直接放进 Cargo，不需要额外框架。

### 第 2 格 · #[test] 与断言宏

- 一个测试就是一个带 `#[test]` 属性的普通函数：不接受参数，返回 `()` 或 `Result`；正常返回即通过，panic 即失败；
- 测试默认多线程并行执行，先后顺序不保证，因此测试之间不能依赖共享状态；
- `assert!`、`assert_eq!`、`assert_ne!` 失败时 panic 并打印左右两边的值；`assert_eq!`/`assert_ne!` 要求类型实现 `Debug` 和 `PartialEq`，宏末尾可追加格式化参数解释业务含义；浮点数应比较误差范围而不是直接 `assert_eq!`；
- 单元测试与被测代码同文件，放在 `#[cfg(test)] mod tests` 里，`use super::*` 访问私有项，且不会进入发布构建。

```rust
#[test]
fn addition_works() {
    assert_eq!(2 + 2, 4);
}
```

### 第 3 格 · 报错路径、集成与文档

- `#[should_panic(expected = "...")]` 用来验证 panic，`expected` 字符串应稳定，避免依赖完整错误文本；不写 `expected` 则任何 panic 都算通过，容易掩盖真正的错误；
- 测试可以返回 `Result`，用 `?` 传播错误，`Err` 会让测试失败并以 `Debug` 格式输出；
- 集成测试放在 `tests/` 目录，每个 `tests/*.rs` 编译成独立 crate，只能使用公开接口；共享辅助代码放 `tests/common/mod.rs`；
- `///` 文档注释中的代码块会被 rustdoc 抽取为文档测试，`cargo test` 一并运行；代码围栏上可写 `ignore`、`no_run`、`should_panic`，以 `#` 开头的行会被隐藏。

```rust
#[test]
#[should_panic(expected = "输入不能为空")]
fn rejects_blank_input() {
    slugify("   ");
}
```

### 第 4 格 · 质量基线

- 覆盖快乐路径之外，每个返回 `Result` 的函数至少覆盖一个错误分支，边界值（0、空字符串、最大值）优先；
- 时间、随机数、网络、真实数据库会造成偶发失败（flaky test），把依赖作为参数注入，测试中传固定值；
- CI 最小门禁：`cargo fmt --all -- --check`、`cargo clippy --all-targets -- -D warnings`、`cargo test`；慢测试标 `#[ignore]`，需要时 `cargo test -- --ignored`。

## 坑点清单

1. 测试之间依赖执行顺序或共享可变状态 → 单线程跑通过、并行跑随机翻车，每个测试自己构造数据；
2. 只测快乐路径 → 错误分支与边界值漏网，每个 `Result` 函数至少补一条错误断言；
3. 文档示例年久失修腐烂 → 用文档测试兜底，让示例随 `cargo test` 一起编译运行。

## 自测 3 题（含答案）

1. `#[cfg(test)]` 起什么作用？——**让测试模块只在 `cargo test` 构建时编译，不会进入发布产物**。
2. `assert_eq!` 对被比较的类型有什么要求？——**要实现 `Debug` 与 `PartialEq`，失败时才能打印左右两边的值**。
3. 文档注释里的代码示例会被执行吗？——**会，rustdoc 把它们抽成文档测试随 `cargo test` 一起运行，编译失败或断言失败都算测试失败**。
