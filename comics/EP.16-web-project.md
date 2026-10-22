# EP.16 实战：axum Web 服务

> 对应正文：[guide/16-web-project.md](../docs/guide/16-web-project.md)（项目需求 / 依赖与目录 / 分步实现 / 测试与部署）

## 剧情回顾

- **第 1 格**：像素酱接到写 todo API 的任务，面对五条路由与一堆状态码犯了难——码叔说其实就三件套。
- **第 2 格**：码叔揭秘 axum 0.8 三件套：`Router` 组路由、提取器解析请求、handler 是普通 `async fn`；并提醒 0.8 的路径参数是 `{id}`，旧写法 `:id` 已失效。
- **第 3 格**：齿轮君落地共享状态与状态码语义：`Arc<Mutex<Store>>` 做进程内存储，创建回 `201`、删除回 `204`、不存在回 `404`、空标题回 `400`。
- **第 4 格**：码叔结案——锁别跨 `.await`、状态显式注入、`/health` 探活与接口测试补齐后再上线。

## 知识点拆解

### 第 1 格 · 需求与依赖

- 五条路由覆盖增删改查：`GET /todos` 列表、`POST /todos` 创建、`GET /todos/{id}` 单条、`PATCH /todos/{id}` 切换完成、`DELETE /todos/{id}` 删除；
- 标题为空返回 `400`，资源不存在返回 `404`，另有 `GET /health` 供部署探活；数据只存进程内存，重启即失；
- 依赖最小集：`axum = "0.8"` 管路由与提取器，`tokio = "1"`（`rt-multi-thread`/`macros`/`net`）提供异步运行时，`serde = "1"`（`derive`）生成 JSON 转换代码。

### 第 2 格 · 路由、提取器与 handler

```rust
let app = Router::new()
    .route("/todos/{id}", get(get_todo))
    .with_state(state);

async fn get_todo(Path(id): Path<u64>) -> Result<Json<Todo>, StatusCode> { /* ... */ }
```

- `Router::new()` 链式注册路由，`with_state` 必须在所有 `route` 之后调用，否则 `State` 取不到共享数据；
- 提取器 `Path<u64>` 解路径参数、`Json` 消费请求体、`State` 取共享状态；只有最后一个参数能消费请求体，因此 `Json` 要放在 `State`、`Path` 之后；
- handler 就是 `async fn`，返回 `Result<T, StatusCode>` 时 `Err` 直接生成对应状态码；axum 0.8 路径参数用花括号 `{id}`，旧 `:id` 不再匹配；
- 未注册的方法由框架返回 `405`；`Json` 解析失败返回 `400` 或 `422`，内容类型不匹配返回 `415`。

### 第 3 格 · 共享状态与状态码语义

```rust
struct AppState {
    store: Arc<Mutex<Store>>,
}
```

- `Arc` 让每份状态副本指向同一块数据，`Mutex` 保证同一时刻只有一个处理器写；这是进程内共享状态的最小方案；
- `std::sync::Mutex` 的守卫不是 `Send`，不能跨 `.await` 持有；本项目临界区不含挂起点所以可用标准库锁，需要跨挂起点持锁时改用 `tokio::sync::Mutex`；
- 状态码是业务语义：`POST` 创建回 `201 Created`，`DELETE` 成功回 `204`，资源不存在回 `404 Not Found`，空标题回 `400 Bad Request`——两次删除同一资源返回不同状态码，因为「资源是否存在」决定响应。

### 第 4 格 · 测试与上线

- handler 是普通异步函数，`#[tokio::test]` 里可以直接调用，无需启动 TCP 服务：`create_todo(State(state), input).await` 断言返回 `201`；
- 部署时监听地址改为 `0.0.0.0` 或从环境变量读取；`GET /health` 供探活；
- 内存状态重启即清空，生产环境应换数据库；跨域请求需配置 CORS；用进程管理器托管 `cargo build --release` 产物。

## 坑点清单

1. `std::sync::Mutex` 守卫跨 `.await` 持锁——守卫不是 `Send`，编译失败；要么缩短临界区，要么换 `tokio::sync::Mutex`；
2. 状态藏进全局 `static`——测试无法隔离状态，handler 也没法直接调用验证；
3. 不写 `/health` 探活与接口测试就上线——部署后无从探活，回归没有保障。

## 自测 3 题（含答案）

1. axum 0.8 的路径参数怎么写？——**花括号语法 `{id}`，旧写法 `:id` 在 0.8 中不再匹配**。
2. 为什么 `Json` 提取器必须放在参数列表最后？——**只有最后一个参数可以消费请求体，`State`、`Path` 要先于 `Json` 提取**。
3. 为什么 `std::sync::Mutex` 的守卫不能跨 `.await` 持有？——**守卫不是 `Send`，无法随任务在线程间迁移；需跨挂起点持锁时改用 `tokio::sync::Mutex`**。
