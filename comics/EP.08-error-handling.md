# EP.08 错误处理的艺术

> 对应正文：[guide/08-errors.md](../docs/guide/08-errors.md)（panic 与 Result 的边界 / Result 与 ? 传播 / Box<dyn Error> 统一错误类型 / Option 组合子）

## 剧情回顾

- **第 1 格**：齿轮君对 `parse()` 直接 `.unwrap()`，脏输入一来程序 panic 崩掉——图省事的代价是全停。
- **第 2 格**：码叔揭秘：`Result<T, E>` 把失败变成普通值；`?` 成功取值、失败提前返回，并经 `From::from` 自动转换错误类型向上传播。
- **第 3 格**：像素酱实证：自定义错误 enum + `Display`/`Error` 三件套，用 `map_err` 包装标准库错误，`Box<dyn Error>` 统一混合错误。
- **第 4 格**：码叔结案——错误是值不是惊吓，处理它或交给调用方。

## 知识点拆解

### 第 1 格 · unwrap 的代价

- `unwrap()` 在 `Err`/`None` 时 panic，`expect("...")` 额外附带说明；
- 二者适合原型、测试和已确认不会失败的位置；
- 库代码与用户输入路径必须返回 `Result`，把选择权交给调用方。

### 第 2 格 · Result 与 ? 传播

```rust
fn build_address(host: &str, raw_port: &str) -> Result<String, ConfigError> {
    if host.trim().is_empty() {
        return Err(ConfigError::MissingField("host"));
    }
    let port = parse_port(raw_port)?;
    Ok(format!("{}:{}", host.trim(), port))
}
```

- `?` 在 `Ok` 时取出值，在 `Err` 时提前返回；
- 返回前通过 `From::from` 把错误转换成当前函数的错误类型，因此函数签名必须能容纳它；
- `panic!` 只留给"继续执行已无意义"的程序缺陷；预期内的外部失败（文件不存在、输入格式错）走 `Result`。

### 第 3 格 · 自定义错误与统一返回

- 错误 enum 区分失败种类；需要 `Debug`（derive）、`Display`（面向用户输出）与 `std::error::Error`（可与其他错误组合）三个实现；
- `map_err` 把标准库错误包装为自定义变体，保留底层 `source`；
- 函数可能返回多种错误、调用方无需区分时，返回 `Box<dyn Error>`——`?` 会自动装箱；`main` 也可以返回 `Result<(), Box<dyn Error>>`。

### 第 4 格 · 处理或上传

- 能就地恢复就 `match` 处理，不能就用 `?` 交给调用方，二选一，不许装没看见；
- `Option` 表达"没有值不算错误"，需要错误语义时用 `ok_or`/`ok_or_else` 转成 `Result`；
- 在返回 `Option` 的函数里，`?` 会在 `None` 时提前返回 `None`。

## 坑点清单

1. 库代码到处 `unwrap`/`expect`，一遇脏数据就 panic；
2. `?` 用在返回 `Option` 的函数里处理 `Result`（或反之）类型不匹配；在返回 `()` 的 `main` 中用 `?` 会报 `error[E0277]`；
3. `let _ = ...` 吞掉错误装没看见，失败静默流失，排查时无迹可寻。

## 自测 3 题（含答案）

1. `?` 运算符做了什么？——**`Ok` 时取出值继续执行；`Err` 时提前返回，并经 `From::from` 把错误转换为当前函数签名的错误类型**。
2. panic 与 Result 的边界在哪里？——**`panic!` 处理代码缺陷与不可恢复场景；`Result` 处理预期内的外部失败，把如何选择交给调用方**。
3. 自定义错误类型需要哪些实现？——**`Debug`（可 derive）、`Display` 与 `std::error::Error` 三件套；需要暴露底层错误时覆盖 `source`**。
