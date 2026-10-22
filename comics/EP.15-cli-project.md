# EP.15 实战：CLI 工具

> 对应正文：[guide/15-cli-project.md](../docs/guide/15-cli-project.md)（项目结构 / 核心逻辑：src/lib.rs / 入口：src/main.rs / 集成测试：tests/cli.rs / release 构建与分发）

## 剧情回顾

- **第 1 格**：齿轮君手写 `std::env::args` 解析参数，校验、帮助、子命令全靠 if-else，越缠越乱——码叔让他交给 clap 4。
- **第 2 格**：码叔展示派生宏魔法：`#[derive(Parser)]` 声明结构体，`#[command(subcommand)]` 注册子命令枚举，文档注释自动变成 `--help` 文本。
- **第 3 格**：像素酱落地工程分层：`src/lib.rs` 放可测试的核心逻辑（如 wc 的行/词/字符统计），`src/main.rs` 只做参数解析与调用，`tests/` 目录做集成测试。
- **第 4 格**：码叔结案——bin 管交互、lib 管逻辑，测试与 release 交付缺一不可。

## 知识点拆解

### 第 1 格 · 手写解析的泥潭

- `std::env::args` 只提供原始字符串迭代器，参数校验、用法说明、子命令分发全要手写；
- 命令行工具与普通程序的区别：输入来自参数与标准流，失败必须给出可读信息和非零退出码，行为要能被脚本稳定调用；
- if-else 缠成一团后，每加一个子命令都要改动多处，测试还只能启动进程。

### 第 2 格 · clap 4 声明式定义

```rust
#[derive(Parser)]
#[command(name = "todo", version, about = "一个最小的命令行待办工具")]
struct Cli {
    /// 任务文件路径
    #[arg(short, long, default_value = "todo.txt", global = true)]
    file: PathBuf,
    #[command(subcommand)]
    command: Command,
}
```

- `#[derive(Parser)]` 根据字段生成解析代码，文档注释会成为 `--help` 文本；
- `#[command(subcommand)]` 把枚举变体注册为子命令；`global = true` 让 `--file` 在子命令前后都能用；
- `Cli::parse()` 在参数非法时打印帮助并以非零码退出；`Cargo.toml` 需启用 `clap = { version = "4", features = ["derive"] }`。

### 第 3 格 · bin + lib 分层

```rust
struct Stats { lines: usize, words: usize, chars: usize }

fn count(text: &str) -> Stats {
    Stats {
        lines: text.lines().count(),
        words: text.split_whitespace().count(),
        chars: text.chars().count(),
    }
}
```

- `lib.rs` 承载可测试的核心逻辑，`main.rs` 只负责解析参数并把错误交给运行时；
- `tests/` 下的文件是独立 crate，只能调用 `pub` 接口，因此同时验证了 API 边界；端到端验证可用 `env!("CARGO_BIN_EXE_todo")` 启动真实二进制；
- 所有 `?` 由 `main` 的 `Result<(), Box<dyn std::error::Error>>` 统一处理，返回 `Err` 时进程以状态码 1 退出。

### 第 4 格 · release 构建与分发

- `cargo build --release` 生成 `target/release/todo`；
- `cargo install --path .` 安装到 Cargo 的 bin 目录，供全局调用；
- 发布前确认版本号、`--help` 文本与退出码符合预期；路径统一用 `PathBuf`/`Path`，不要拿 `String` 存路径。

## 坑点清单

1. 逻辑全堆 `main.rs`——测试只能靠启动进程，核心逻辑无法单测；
2. 出错直接 `panic!`/`unwrap`——只打印 panic，脚本拿不到有意义的非零退出码；
3. 忘记 `--release` 构建分发版——把未优化的 debug 二进制交给了用户。

## 自测 3 题（含答案）

1. 为什么核心逻辑要放进 `lib.rs`？——**`main.rs` 里的代码无法被 `tests/` 引用；lib 暴露 `pub` API 后才能编写集成测试**。
2. clap 派生宏里文档注释起什么作用？——**自动成为 `--help` 的帮助文本**。
3. `main` 返回 `Err` 时进程退出码是多少？——**状态码 1（非零），脚本可据此判断失败原因**。
