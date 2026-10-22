# EP.17 实战：WASM 前端

> 对应正文：[guide/17-wasm-project.md](../docs/guide/17-wasm-project.md)（项目需求 / 依赖与目录 / 分步实现 / 测试与部署）

## 剧情回顾

- **第 1 格**：齿轮君听说 Rust 能跑进浏览器，大为震惊——码叔解释：浏览器不认 Rust，但认 WebAssembly，编译成 wasm 就行。
- **第 2 格**：码叔揭秘 `#[wasm_bindgen]` 属性宏：加在 `pub fn` 上导出函数，`wasm-pack build --target web` 一键打出 JS 胶水与 `.wasm` 产物。
- **第 3 格**：像素酱完成 JS 与 Rust 的分工落地：`await init()` 先实例化，再调 `analyze` 出统计，JS 负责渲染 DOM，用完 `free()` 释放内存。
- **第 4 格**：码叔结案——重活给 Rust、界面给 JS，边界越薄越快；发布前记得看产物体积。

## 知识点拆解

### 第 1 格 · 为什么浏览器能跑 Rust

- 浏览器只认 JavaScript 与 WebAssembly（WASM）；Rust 编译到 `wasm32-unknown-unknown` 目标即可在浏览器执行；
- 项目目标：浏览器端文本统计器——输入文本，算出字符数、词数、去重词数、预计阅读时长与高频词 Top 5；
- 分词采用「按空白切分 + 去除首尾标点」规则，适合空格分隔的语言；中文没有空格，整句会被当成一个词，这里只作演示，不追求语言学准确性；
- 分工原则：统计逻辑用 Rust 编写，JavaScript 只读输入、调导出函数并渲染结果，不引入前端框架。

### 第 2 格 · wasm-bindgen 与 wasm-pack

```rust
#[wasm_bindgen]
pub fn analyze(text: &str, limit: u32) -> Stats { /* ... */ }
```

```toml
[lib]
crate-type = ["cdylib", "rlib"]
```

- `#[wasm_bindgen]` 是属性宏：加在 `struct` 上生成 JS 可见的类，加在 `impl` 上导出方法，加在 `pub fn` 上导出函数；导出结构体字段默认私有，需要 `#[wasm_bindgen(getter)]` 才能从 JS 读取；
- `crate-type = ["cdylib", "rlib"]`：`cdylib` 生成供外部加载的动态库，`rlib` 保留普通 Rust 库形态，让 `cargo test` 能在本机运行；
- 工具链：`rustup target add wasm32-unknown-unknown` 加编译目标，`cargo install wasm-pack` 装打包器；
- `wasm-pack build --target web --out-dir pkg` 的产物：`text_stats.js` 是加载器与绑定胶水，`text_stats_bg.wasm` 是编译后的模块，`text_stats.d.ts` 是类型声明；`--target web` 可被 ES module 直接引入。

### 第 3 格 · JS 侧接入

```js
import init, { analyze } from "../pkg/text_stats.js";
await init();
const stats = analyze(input.value, 5);
render(stats);
stats.free();
```

- `init()` 负责实例化 WASM 并连接导入导出表，必须在调用其他导出函数前完成；
- 读取字段走 getter（`stats.chars`、`stats.top` 等）；导出对象由 JS 侧持有，不再使用时必须调用 `stats.free()` 释放 WASM 内存；
- 页面与 `pkg/` 不同目录时，Vite 需配置 `server: { fs: { allow: [".."] } }` 才允许读取父目录；
- 脚手架：`npm create vite@latest web -- --template vanilla`，然后 `npm install`、`npm run dev`。

### 第 4 格 · 测试与部署

- 统计与排序是纯函数，`cargo test` 在本机直接跑，无需浏览器——这正是 `rlib` 形态的价值；
- 构建流水线：`cargo test` → `wasm-pack build --target web --out-dir pkg` → `cd web && npm run build`；
- 部署输出目录是 `web/dist`，其中 `.wasm` 文件必须以 `application/wasm` 类型返回；
- 不能直接双击 `index.html` 用 `file://` 打开，浏览器会拦截模块与 WASM 加载；部署到子路径时用 `vite build --base=/子路径/`。

## 坑点清单

1. 高频跨 JS↔WASM 边界传大数据——每次跨界都要在线性内存与 JS 堆之间复制，边界越薄越快；
2. 导出类型不是 wasm 友好类型——字段私有又没写 getter，JS 侧读不到数据；
3. 不看产物体积，未优化就发布——`.wasm` 体积直接影响首屏加载，发布前应检查并优化。

## 自测 3 题（含答案）

1. `init()` 的作用是什么？——**实例化 WASM 模块并连接导入导出表，必须在调用任何导出函数前完成**。
2. 为什么 `crate-type` 要写 `["cdylib", "rlib"]` 两种形态？——**`cdylib` 生成供浏览器加载的动态库，`rlib` 保留普通 Rust 库形态，使 `cargo test` 可以在本机运行**。
3. 为什么用完要调 `stats.free()`？——**导出对象由 JS 侧持有并占用 WASM 线性内存，不再使用时需显式释放，否则内存无法回收**。
