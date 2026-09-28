---
title: Rust 常用库实战教程
image:
date: 2026-09-28
---

> 面向已有 Rust 基础语法、想快速上手工程开发的读者。
> 每个库都包含：**它解决什么问题、为什么要用它、最常见的用法 + 可运行的小案例**。
> 文中代码基于各库 2025–2026 年的主流版本（serde 1.x / tokio 1.x / reqwest 0.12 / axum 0.8 / clap 4.x / rand 0.9 / sqlx 0.8 等）。

---

## 第 0 章 开始之前：Cargo 与依赖管理

Rust 的库叫 **crate**，统一发布在 [crates.io](https://crates.io)，文档统一托管在 [docs.rs](https://docs.rs)。
看某个库的用法，直接访问 `https://docs.rs/<库名>` 即可，这是 Rust 生态最重要的习惯。

### 0.1 添加依赖

```bash
# 现代方式（推荐）：用 cargo add，自动写入最新兼容版本
cargo add serde --features derive
cargo add tokio --features full
cargo add reqwest --features json
```

也可以手动编辑 `Cargo.toml`：

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
```

**为什么强调 features？** Rust crate 普遍把非核心功能放在 feature 后面（比如 serde 的 `derive`、tokio 的 `full`），默认不开启。这样编译时只编译你真正用到的代码，**编译更快、产物更小**。这是 Rust「零成本抽象」哲学在依赖管理上的体现：不为用不到的东西付出代价。

### 0.2 常用 Cargo 命令

```bash
cargo new my_app        # 新建二进制项目
cargo new my_lib --lib  # 新建库项目
cargo build             # 调试构建（快，无优化）
cargo build --release   # 发布构建（慢，有优化）
cargo run               # 构建并运行
cargo check             # 只检查不生成二进制，改代码时最快
cargo test              # 运行测试
cargo doc --open        # 生成并打开本项目（含依赖）的文档
cargo update            # 在语义化版本允许范围内升级依赖
cargo add / cargo rm    # 增删依赖
cargo tree              # 查看依赖树，排查版本冲突
```

> 经验：写代码时多用 `cargo check`，秒级反馈；需要运行时再 `cargo run`。

---

## 第 1 章 serde / serde_json：序列化与反序列化

### 1.1 它解决什么问题

程序里的结构体和外部世界的 JSON/TOML/YAML 之间需要互相转换。serde 是 Rust 生态事实上的标准，几乎所有库（reqwest、axum、sqlx、config 解析……）都建立在它之上。

**为什么要用 derive 而不是手写？** 手写序列化代码冗长且容易和结构体定义脱节（改了字段忘了改解析逻辑）。`#[derive(Serialize, Deserialize)]` 在编译期自动生成转换代码，**结构体定义即契约**，改一处即可，且零运行时开销。

### 1.2 最小案例

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u8,
    email: Option<String>, // 可空字段用 Option
}

fn main() -> serde_json::Result<()> {
    // 结构体 -> JSON 字符串
    let u = User { name: "小明".into(), age: 20, email: None };
    let s = serde_json::to_string(&u)?;
    println!("{s}"); // {"name":"小明","age":20,"email":null}

    // 美化输出
    println!("{}", serde_json::to_string_pretty(&u)?);

    // JSON 字符串 -> 结构体
    let raw = r#"{"name":"小红","age":18,"email":"a@b.com"}"#;
    let v: User = serde_json::from_str(raw)?;
    println!("{v:?}");
    Ok(())
}
```

### 1.3 常用字段属性

```rust
#[derive(Serialize, Deserialize)]
struct Article {
    #[serde(rename = "id")]            // JSON 里的字段名与 Rust 字段名不同
    article_id: u64,

    #[serde(default)]                   // JSON 缺这个字段时用 Default 值而不是报错
    tags: Vec<String>,

    #[serde(default = "default_status")]// 缺省值用指定函数
    status: String,

    #[serde(skip_serializing)]          // 只反序列化，不序列化（如密码）
    password: String,

    #[serde(skip_serializing_if = "Option::is_none")] // None 时不输出该字段，而不是输出 null
    avatar: Option<String>,

    #[serde(flatten)]                   // 把内嵌结构体的字段拍平到本层
    extra: std::collections::HashMap<String, serde_json::Value>,
}

fn default_status() -> String { "draft".into() }
```

**为什么 `Option` 字段还要加 `skip_serializing_if`？** 默认 `None` 会序列化成 `"avatar": null`，但很多前端/接口约定「没有就不给这个 key」。这个属性让输出更干净，也避免对方把 `null` 和「不存在」混为一谈。

### 1.4 处理不确定结构的 JSON：`Value`

爬第三方接口时，字段经常不稳定，不值得定义完整结构体：

```rust
use serde_json::Value;

let raw = r#"{"code":0,"data":{"users":[{"name":"a"},{"name":"b"}]}}"#;
let v: Value = serde_json::from_str(raw)?;

// 用链式索引取值，类型不符返回 None 而不是 panic
if let Some(name) = v["data"]["users"][0]["name"].as_str() {
    println!("第一个用户：{name}");
}

// json! 宏快速构造 JSON
let payload = serde_json::json!({
    "title": "hello",
    "count": 3,
    "ok": true,
});
```

### 1.5 其他格式

serde 是「中间表示」层，换个后端 crate 就支持别的格式，用法完全一致：

```toml
toml = "0.8"          # TOML
serde_yaml = "0.9"    # YAML
```

```rust
let toml_str = toml::to_string(&u).unwrap();
let back: User = toml::from_str(&toml_str).unwrap();
```

**为什么是这个架构？** serde 把「数据结构 ↔ 抽象数据模型」和「数据模型 ↔ 具体格式」拆开。你写的 derive 代码对 JSON/TOML/YAML 全部通用，社区也只维护一套宏——这就是 serde 能一统江湖的原因。

---

## 第 2 章 anyhow / thiserror：错误处理两件套

### 2.1 它解决什么问题

Rust 没有异常，错误通过 `Result<T, E>` 显式传递。问题是：一个函数里可能产生很多种错误（IO 错、解析错、网络错……），`E` 该写什么类型？

社区共识：

| 场景                         | 用什么      | 原因                                                                 |
| ---------------------------- | ----------- | -------------------------------------------------------------------- |
| 写**应用**（bin、服务端）    | `anyhow`    | 调用者只关心「成功还是失败 + 发生了什么」，不需要细分错误类型        |
| 写**库**（给别人用的 crate） | `thiserror` | 库的调用者可能需要匹配具体错误做不同处理，错误类型必须是结构化的枚举 |

### 2.2 anyhow：应用层的错误处理

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result, bail, ensure};
use std::fs;

// 返回 anyhow::Result<T>，内部是 Box<dyn Error>，任何错误都能用 ? 自动转换
fn load_config(path: &str) -> Result<String> {
    // Context 给底层错误附上业务语义。这是 anyhow 最有价值的功能：
    // 没有它你只能看到 "No such file or directory"，不知道是哪个文件、在干嘛
    let content = fs::read_to_string(path)
        .with_context(|| format!("读取配置文件 {path} 失败"))?;
    Ok(content)
}

fn check_age(age: i32) -> Result<()> {
    ensure!(age >= 0, "年龄不能为负数: {age}");   // 条件不满足则提前返回错误
    if age > 150 {
        bail!("年龄不合理: {age}");              // 直接构造错误并返回，等价于 return Err(...)
    }
    Ok(())
}

fn main() -> Result<()> {
    // main 也可以返回 Result，出错时自动打印错误并退出码非 0
    let cfg = load_config("app.toml")?;
    println!("配置长度: {}", cfg.len());

    // 完整的错误链（含每一层 context）用 {:#} 打印
    if let Err(e) = check_age(-1) {
        eprintln!("错误链: {:#}", e);
        eprintln!("根因: {:?}", e.root_cause()); // 最底层那个错误
    }
    Ok(())
}
```

**为什么要 `Context`？** 底层的 `io::Error` 只说「文件不存在」，但排查线上问题你需要知道「是在加载哪个配置时失败的」。Context 是在错误向上传播的路上不断追加业务上下文，形成错误链——这是用很低成本换来极好排障体验的做法。

### 2.3 thiserror：给库定义错误类型

```toml
[dependencies]
thiserror = "2"
```

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StoreError {
    #[error("记录 {0} 不存在")]                 // 错误信息模板，{0} 引用元组字段
    NotFound(u64),

    #[error("权限不足，需要 {need} 级，当前 {current} 级")] // 命名字段也可以
    PermissionDenied { need: u8, current: u8 },

    #[error("数据库错误")]                       // 展示给上层的文案
    #[from]                                     // 自动生成 From<sqlx::Error>，? 可自动转换
    Db(#[source] sqlx::Error),                  // #[source] 保留底层错误供溯源
}

fn find_user(id: u64) -> Result<String, StoreError> {
    if id == 0 { return Err(StoreError::NotFound(id)); }
    Ok("小明".into())
}

// 调用方可以精确匹配
fn handle() {
    match find_user(0) {
        Err(StoreError::NotFound(id)) => println!("404: {id}"),
        Err(StoreError::Db(e)) => println!("数据库挂了: {e}"),
        _ => {}
        Ok(name) => println!("{name}"),
    }
}
```

**为什么库不用 anyhow？** 如果你的库返回 `anyhow::Error`，调用方就没法 `match` 具体错误种类（anyhow 内部是类型擦除的），只能靠字符串匹配——这对库消费者来说是不可接受的。`thiserror` 只是帮你少写 `impl Display` / `impl Error` / `impl From` 的样板代码，产出的是完全标准、类型安全的错误枚举。

> 补充：应用层也可以用「`thiserror` 定义领域错误 + `anyhow` 做顶层汇总」的混合模式，二者不冲突。

---

## 第 3 章 tokio：异步运行时

### 3.1 它解决什么问题

写网络服务时，一个线程阻塞等 IO 是巨大浪费。Rust 的 `async/await` 只是**语法**，`async fn` 本身不会运行——必须有一个**运行时**来轮询、调度这些 future。tokio 是事实标准的异步运行时（带工作窃取调度器、定时器、异步 IO、异步同步原语）。

**为什么 Rust 异步要显式选运行时？** 因为嵌入式、浏览器 WASM、服务端对调度和 IO 的需求完全不同，标准库不替你决定。代价是入门多一步，收益是各场景都能用最优实现。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] } # 学习中直接 full，生产再裁剪
```

### 3.2 入口与 spawn

```rust
// #[tokio::main] 宏把 main 变成运行时入口，等价于手动 Builder::new_multi_thread()...block_on
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // spawn 创建一个新的异步任务，由运行时并发调度
        // 类似线程，但成本极低：可以轻松起十万级任务
        tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        42
    });

    // JoinHandle 类似线程的 join，await 拿到任务返回值
    let answer = handle.await.unwrap();
    println!("{answer}");
}
```

### 3.3 并发等待：join! / try_join! / select!

```rust
use tokio::time::{sleep, Duration};

async fn fetch_user() -> Result<String, String> { sleep(Duration::from_millis(300)).await; Ok("user".into()) }
async fn fetch_orders() -> Result<Vec<String>, String> { sleep(Duration::from_millis(500)).await; Ok(vec!["o1".into()]) }

#[tokio::main]
async fn main() {
    // join!: 并发执行，全部完成后一起拿结果。总耗时取最长的那个（500ms 而非 800ms）
    let (u, o) = tokio::join!(fetch_user(), fetch_orders());

    // try_join!: 任何一个出错立即返回 Err，其余取消。适合「缺一不可」的聚合请求
    let r: Result<(String, Vec<String>), String> = tokio::try_join!(fetch_user(), fetch_orders());

    // select!: 谁先完成用谁，其余被 drop。适合超时、竞速备份请求
    tokio::select! {
        v = fetch_user() => println!("user 先到: {v:?}"),
        _ = sleep(Duration::from_secs(10)) => println!("超时了"),
    }
    let _ = (u, o, r);
}
```

**为什么要 `join!` 而不是依次 `.await`？** 依次 await 是串行：等 user 回来再发 orders 请求，总延迟翻倍。`join!` 让多个 IO 同时在飞，这是异步最大的收益场景之一——**IO 聚合并发**。

### 3.4 超时与取消

```rust
use tokio::time::{timeout, Duration};

async fn slow() -> &'static str { tokio::time::sleep(Duration::from_secs(5)).await; "done" }

#[tokio::main]
async fn main() {
    // timeout 是 select! 的常见封装：到点未完成就返回 Err
    match timeout(Duration::from_secs(1), slow()).await {
        Ok(v) => println!("{v}"),
        Err(_) => println!("超时取消"), // 超时时 future 被 drop，任务自然取消
    }
}
```

**为什么 drop 就能取消？** Rust 的 future 是惰性状态机，不轮询就不前进。drop 掉 future = 销毁状态机 = 取消。没有线程中断那样的危险操作，这就是 Rust 异步取消模型安全优雅的原因。代价是：被 drop 点必须是不变量的安全点（tokio 的锁等原语都保证了这点）。

### 3.5 异步文件与进程

```rust
use tokio::fs;
use tokio::io::{AsyncWriteExt, AsyncReadExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // 高层 API：一次性读写
    fs::write("a.txt", "你好").await?;
    let s = fs::read_to_string("a.txt").await?;

    // 低层 API：流式处理大文件
    let mut f = fs::File::create("b.txt").await?;
    f.write_all(s.as_bytes()).await?;  // 注意 trait 方法要 use AsyncWriteExt
    f.flush().await?;

    let mut buf = String::new();
    fs::File::open("b.txt").await?.read_to_string(&mut buf).await?;
    Ok(())
}
```

> 注意：tokio 的异步文件 IO 底层其实是线程池（操作系统对普通文件没有真正的异步接口）。**CPU 密集或必须用阻塞 API 的活，用 `tokio::task::spawn_blocking`**：

```rust
let hash = tokio::task::spawn_blocking(|| {
    // 这里跑同步的重计算，不会卡住异步调度器的工作线程
    expensive_hash()
}).await.unwrap();
```

**为什么阻塞代码会卡住整个程序？** tokio 工作线程数量有限（默认 = CPU 核数）。一个任务在工作线程里做阻塞调用（`std::thread::sleep`、同步文件 IO、重计算），这个线程就停摆了，它手上其他几百个异步任务全部挨饿。所以铁律：**async 代码里绝不调用阻塞 API**。

### 3.6 异步同步原语

```rust
use std::sync::Arc;
use tokio::sync::{Mutex, mpsc, oneshot, RwLock, Semaphore, Notify};

#[tokio::main]
async fn main() {
    // 1) tokio::sync::Mutex：跨 .await 持锁时必须用它（std::Mutex 的守卫不是 Send，且会阻塞线程）
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = vec![];
    for _ in 0..10 {
        let c = counter.clone();
        handles.push(tokio::spawn(async move {
            let mut n = c.lock().await; // 异步锁，拿不到时让出线程而不是阻塞
            *n += 1;
        }));
    }
    for h in handles { h.await.unwrap(); }
    println!("{}", *counter.lock().await); // 10

    // 2) mpsc 通道：多生产者单消费者，任务间通信的首选
    let (tx, mut rx) = mpsc::channel::<String>(100); // 100 是缓冲容量，满了 send 会 .await 背压
    tokio::spawn(async move {
        for i in 0..3 {
            tx.send(format!("消息{i}")).await.unwrap();
        }
    });
    while let Some(msg) = rx.recv().await { // 所有 tx 都 drop 后 recv 返回 None，循环结束
        println!("收到: {msg}");
    }

    // 3) oneshot：一次性应答，常用于「请求-响应」式取消通知
    let (tx, rx) = oneshot::channel::<&str>();
    tokio::spawn(async move { let _ = tx.send("pong"); });
    println!("{}", rx.await.unwrap());

    // 4) RwLock / Semaphore / Notify 与 std 对应物语义类似，但都是 async 版
    let sem = Arc::new(Semaphore::new(3)); // 限制并发度：最多 3 个任务同时进入临界区
    let permit = sem.acquire().await.unwrap();
    drop(permit); // 释放名额
    let _ = Notify::new(); // 事件通知：notify_one/notify_waiters + notified().await
    let _ = RwLock::new(0u8);
}
```

**为什么通道容量要有限（100 而不是 unbounded）？** 有限缓冲提供了**背压（backpressure）**：生产快于消费时，`send` 会挂起等位置空出来，内存不会无限膨胀。无界通道在流量高峰时就是内存炸弹。这是 tokio 引导你写出稳态系统的典型设计。

---

## 第 4 章 reqwest：HTTP 客户端

### 4.1 它解决什么问题

发 HTTP 请求调用 REST API、下载文件。reqwest 构建在 tokio + hyper 之上，提供符合直觉的高层 API，是生态绝对主流。

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

### 4.2 GET / POST 与 JSON

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Debug)]
struct Post { id: u64, title: String }

#[derive(Serialize)]
struct NewPost { title: String, body: String }

#[tokio::main]
async fn main() -> reqwest::Result<()> {
    // 最简单的 GET
    let body = reqwest::get("https://jsonplaceholder.typicode.com/posts/1")
        .await?
        .text()          // .text() 拿字符串；.bytes() 拿字节；.json() 直接反序列化
        .await?;
    println!("原始响应: {}...", &body[..50.min(body.len())]);

    // GET + 反序列化为结构体（依赖 serde）
    let post: Post = reqwest::get("https://jsonplaceholder.typicode.com/posts/2")
        .await?
        .json()
        .await?;
    println!("{post:?}");
    Ok(())
}
```

### 4.3 复用 Client：生产环境的正确姿势

```rust
use std::time::Duration;

#[tokio::main]
async fn main() -> reqwest::Result<()> {
    // 为什么要显式建 Client 而不是每次 reqwest::get？
    // Client 内部维护连接池（HTTP keep-alive），复用 TCP/TLS 连接，
    // 高频请求下性能差距巨大；还能统一配置超时、默认 header、代理。
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(10))            // 总超时，防止请求永远挂起拖垮服务
        .connect_timeout(Duration::from_secs(3))     // 连接建立超时单独设置，更快失败
        .user_agent("my-app/1.0")                    // 很多 API 拒绝无 UA 的请求
        .build()?;

    // POST JSON
    let resp = client
        .post("https://jsonplaceholder.typicode.com/posts")
        .json(&NewPost { title: "t".into(), body: "b".into() }) // 自动序列化 + 设 Content-Type
        .send()
        .await?;
    println!("状态: {}", resp.status());

    // GET 带查询参数、自定义 header
    let resp = client
        .get("https://jsonplaceholder.typicode.com/posts")
        .query(&[("userId", "1")])                   // 自动做 URL 编码，别手拼字符串
        .header("Authorization", "Bearer token123")
        .send()
        .await?;

    // error_for_status：把 4xx/5xx 变成 Err，否则 reqwest 认为 HTTP 层成功就是成功
    let posts: Vec<serde_json::Value> = resp.error_for_status()?.json().await?;
    println!("拿到 {} 条", posts.len());
    Ok(())
}
```

**为什么一定要设超时？** 不设超时的 HTTP 调用是对下游故障敞开的：对方服务挂起时你的每个请求都永远占着一个任务和连接，最终耗尽资源级联雪崩。所有出网调用都必须有超时，这是服务端开发的第一戒律。

### 4.4 大文件流式下载

```rust
use futures_util::StreamExt;
use tokio::io::AsyncWriteExt;

async fn download(client: &reqwest::Client, url: &str, path: &str) -> reqwest::Result<()> {
    let mut resp = client.get(url).send().await?.error_for_status()?;
    let mut file = tokio::fs::File::create(path).await.unwrap();

    // bytes_stream() 边下边写，内存占用恒定。如果把整个 body 读进内存，2GB 文件就占 2GB 内存
    while let Some(chunk) = resp.bytes_stream().next().await {
        file.write_all(&chunk?).await.unwrap();
    }
    Ok(())
}
```

### 4.5 表单与 multipart 上传

```rust
async fn forms(client: &reqwest::Client) -> reqwest::Result<()> {
    // application/x-www-form-urlencoded 表单
    client.post("https://httpbin.org/post")
        .form(&[("username", "admin"), ("password", "123456")])
        .send().await?;

    // multipart 文件上传
    let part = reqwest::multipart::Part::bytes(std::fs::read("a.txt").unwrap())
        .file_name("a.txt");
    let form = reqwest::multipart::Form::new()
        .text("desc", "一个文件")
        .part("file", part);
    client.post("https://httpbin.org/post").multipart(form).send().await?;
    Ok(())
}
```

---

## 第 5 章 axum：Web 服务端框架

### 5.1 它解决什么问题

axum 是 tokio 团队官方生态的 Web 框架，基于 tower 抽象，与 tokio/hyper 无缝配合。特点是**没有自己的宏魔法，路由和提取器全是类型系统**——编译器会拦住绝大多数错误。

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

> 注意：axum 0.8 起路径参数语法是 `{id}`（旧版 `:id` 已废弃）。

### 5.2 最小可运行服务

```rust
use axum::{routing::{get, post}, Router, Json};
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct Msg { message: String }

#[derive(Deserialize)]
struct CreateUser { name: String }

async fn hello() -> &'static str { "Hello, world" }       // 返回 &str 自动成为 text/plain
async fn hello_json() -> Json<Msg> {                       // 返回 Json<T> 自动序列化 + 设 Content-Type
    Json(Msg { message: "你好".into() })
}
async fn create_user(Json(payload): Json<CreateUser>) -> Json<Msg> {
    // Json<T> 既是提取器（请求）也是响应器：靠的是函数参数的「提取器模式」
    Json(Msg { message: format!("创建用户 {}", payload.name) })
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello))
        .route("/json", get(hello_json))
        .route("/users", post(create_user));

    // 0.8 写法：TcpListener + serve
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("监听 http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

**为什么 handler 是普通 async fn 而不是 trait？** axum 的提取器模式（handler 参数实现 `FromRequest` 即可）让 handler 签名本身就是 API 文档：参数声明了需要什么，返回类型声明了产出什么，全部编译期检查。请求缺字段、类型不符，根本到不了你的业务代码。

### 5.3 路径参数、查询参数、Header、共享状态

```rust
use axum::{
    extract::{Path, Query, State},
    http::{HeaderMap, StatusCode},
    Json, Router, routing::get,
};
use serde::{Deserialize, Serialize};
use std::{collections::HashMap, sync::{Arc, atomic::{AtomicU64, Ordering}}};

// 共享状态：Arc 包一层。原子计数器之外的可变数据用 Mutex/RwLock
#[derive(Clone)]
struct AppState {
    visit_count: Arc<AtomicU64>,
    app_name: String,
}

#[derive(Deserialize)]
struct PageQuery { page: Option<u32>, size: Option<u32> }

async fn get_user(Path(id): Path<u64>) -> String { format!("用户 {id}") }

async fn list_users(Query(q): Query<PageQuery>) -> String {
    format!("第 {} 页，每页 {} 条", q.page.unwrap_or(1), q.size.unwrap_or(20))
}

async fn visit(State(state): State<AppState>, headers: HeaderMap) -> String {
    let n = state.visit_count.fetch_add(1, Ordering::Relaxed) + 1;
    let ua = headers.get("user-agent")
        .and_then(|v| v.to_str().ok()).unwrap_or("unknown");
    format!("[{}] 第 {n} 次访问，UA: {ua}", state.app_name)
}

// 返回 (StatusCode, Json<T>) 自定义状态码；impl IntoResponse 可以返回任意可转响应的类型
async fn maybe_found(State(s): State<AppState>) -> Result<Json<serde_json::Value>, StatusCode> {
    if s.app_name.is_empty() {
        Err(StatusCode::NOT_FOUND) // 返回 Err(StatusCode) 直接成为对应状态码的空响应
    } else {
        Ok(Json(serde_json::json!({"ok": true})))
    }
}

#[tokio::main]
async fn main() {
    let state = AppState {
        visit_count: Arc::new(AtomicU64::new(0)),
        app_name: "demo".into(),
    };

    let app = Router::new()
        .route("/users/{id}", get(get_user))          // {id} 与 Path<u64> 对应，类型不符自动 400
        .route("/users", get(list_users))
        .route("/visit", get(visit))
        .route("/found", get(maybe_found))
        .with_state(state);                            // 注入状态，handler 用 State 提取

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### 5.4 统一错误响应与中间件

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json, Router, routing::get, middleware,
};
use serde_json::json;

// 应用错误类型：实现 IntoResponse 后，handler 直接返回 Result<T, AppError>
// 为什么？否则每个 handler 都要手写 try/match 转 HTTP 响应，业务代码被噪音淹没
enum AppError { NotFound(String), Internal(anyhow::Error) }

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::NotFound(m) => (StatusCode::NOT_FOUND, m),
            AppError::Internal(e) => {
                eprintln!("内部错误: {e:#}"); // 详细错误只进服务端日志
                (StatusCode::INTERNAL_SERVER_ERROR, "服务器内部错误".into())
                // 为什么不把内部错误详情返回给客户端？——防止泄露路径、SQL 等敏感信息
            }
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}

// From 转换让 ? 直接可用
impl From<anyhow::Error> for AppError {
    fn from(e: anyhow::Error) -> Self { AppError::Internal(e) }
}

async fn handler() -> Result<String, AppError> {
    Err(AppError::NotFound("东西不存在".into()))
}

// 简单中间件：函数式 middleware
async fn log_requests(req: axum::extract::Request, next: middleware::Next) -> Response {
    let start = std::time::Instant::now();
    let method = req.method().clone();
    let path = req.uri().path().to_string();
    let resp = next.run(req).await;          // 调用链路的下一层
    println!("{method} {path} -> {} ({:?})", resp.status(), start.elapsed());
    resp
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(handler))
        .layer(middleware::from_fn(log_requests)) // 挂在整个 Router 上
        .fallback(|| async { (StatusCode::NOT_FOUND, "路由不存在") }); // 自定义 404

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### 5.5 路由嵌套与静态文件

```rust
use axum::{Router, routing::get};

fn api_routes() -> Router {
    Router::new().route("/health", get(|| async { "ok" }))
}

fn app() -> Router {
    Router::new()
        .nest("/api/v1", api_routes())      // 嵌套：/api/v1/health
        // .nest_service("/static", tower_http::services::ServeDir::new("static")) // 静态文件需 tower-http
}
```

---

## 第 6 章 clap：命令行参数解析

### 6.1 它解决什么问题

解析 `mytool --input a.txt -v` 这类参数。clap 的 **derive 模式**让你用结构体定义 CLI，自动生成 `--help`、自动校验、自动类型转换。

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

**为什么用 derive 而不是 builder？** 结构体即文档：参数长什么样、什么类型、什么默认值，一眼看清，且解析结果就是强类型结构体，不存在「字符串忘了 parse」的 bug。

### 6.2 完整案例

```rust
use clap::{Parser, Subcommand, ValueEnum};

#[derive(Parser)]
#[command(name = "mytool", version, about = "一个示例工具", long_about = None)]
struct Cli {
    /// 输入文件（doc comment 自动成为帮助文本！）
    #[arg(short, long)]
    input: String,

    /// 输出目录
    #[arg(short, long, default_value = "./out")]   // 不传时的默认值
    output: String,

    /// 详细日志，-v / -vv / -vvv 可叠加
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,

    /// 并发数，1-32
    #[arg(short, long, default_value_t = 4, value_parser = clap::value_parser!(u8).range(1..=32))]
    jobs: u8,

    /// 输出格式
    #[arg(long, value_enum, default_value_t = Format::Json)]
    format: Format,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Clone, ValueEnum)]
enum Format { Json, Text, Csv }

#[derive(Subcommand)]
enum Commands {
    /// 处理文件
    Process {
        /// 是否强制覆盖
        #[arg(short, long)]
        force: bool,
    },
    /// 清理缓存（位置参数：直接写在命令后面，不需要 --）
    Clean {
        target: String,
    },
}

fn main() {
    let cli = Cli::parse(); // 解析失败（缺参数/类型错）自动打印错误和用法，退出码 2

    match cli.command {
        Commands::Process { force } => {
            println!("处理 {} -> {} (force={force}, jobs={}, format={:?})",
                     cli.input, cli.output, cli.jobs, cli.format);
        }
        Commands::Clean { target } => println!("清理 {target}"),
    }
}
```

运行效果：

```bash
$ mytool --help          # 自动生成的漂亮帮助
$ mytool -i a.txt process --force
$ mytool -i a.txt clean cache_dir
```

### 6.3 常用 arg 属性速查

| 属性                          | 作用                                  | 例                               |
| ----------------------------- | ------------------------------------- | -------------------------------- |
| `short, long`                 | `-x` / `--xxx`                        | `#[arg(short, long)]`            |
| `default_value = "..."`       | 字符串默认值                          | 适合所有 `FromStr` 类型          |
| `default_value_t = 4`         | 类型化默认值                          | 数字、枚举用它可以出现在 help 里 |
| `env = "API_KEY"`             | 从环境变量读（需 feature `env`）      | 密钥不入命令行历史               |
| `value_parser = ...`          | 自定义校验/转换                       | `value_parser!(u16).range(1..)`  |
| `required = true`             | 必填（Option 字段天然可选，无需此项） | 位置参数默认必填                 |
| `conflicts_with` / `requires` | 参数间约束                            | `--json` 与 `--yaml` 互斥        |
| `hide = true`                 | 不在 help 中显示                      | 内部调试参数                     |

---

## 第 7 章 log / env_logger / tracing：日志

### 7.1 它解决什么问题

`println!` 无法控制级别、无法关闭、无法加时间戳和模块名。Rust 日志生态分两层：**`log` 是门面（facade）**——库代码只调 log 的宏，不关心日志去哪；**env_logger / tracing-subscriber 是实现（backend）**——由最终的应用决定日志格式和去向。

**为什么库必须只用 log 门面？** 如果每个库自己初始化输出，日志格式就乱了，而且库不应该替应用做决定。这个「门面 + 实现」分层让任何库的日志都能被应用统一接管。

### 7.2 log + env_logger：简单够用的方案

```toml
[dependencies]
log = "0.4"
env_logger = "0.11"
```

```rust
use log::{trace, debug, info, warn, error};

fn main() {
    // 只在应用的 main 里初始化一次；库代码里绝不要调用 init
    // 默认级别由环境变量 RUST_LOG 控制，无需改代码
    env_logger::init();

    error!("出错了");   // 默认只显示 warn 及以上
    warn!("警告");
    info!("普通信息");  // RUST_LOG=info 才显示
    debug!("调试细节"); // RUST_LOG=debug
    trace!("最啰嗦");   // RUST_LOG=trace
}
```

```bash
RUST_LOG=info cargo run                  # 全局 info 级
RUST_LOG=myapp=debug,reqwest=warn cargo run  # 按模块精细控制：自己代码 debug，第三方库只留 warn
```

**为什么要按模块控制级别？** 开全局 debug 时，tokio/hyper 等底层库的海量日志会淹没你自己的日志。`myapp=debug,reqwest=warn` 这种过滤让你只看想看的。

### 7.3 tracing：结构化日志 + 调用链（服务端推荐）

log 记录的是一条条孤立的字符串；tracing 记录的是**带结构化字段的事件**，外加 `span` 表达「这堆日志属于哪个请求」——异步服务端排查问题的神器。

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use tracing::{info, warn, instrument, info_span};

// #[instrument] 自动给函数建 span，参数自动记录为字段
// 这个函数内的所有日志都会自动带上 request_id 和 user 字段——无需手动传递！
#[instrument(skip(db), fields(request_id = %uuid_v4()))]
async fn handle_request(user: &str, db: &Database) {
    info!("开始处理");
    step1().await;
    warn!(latency_ms = 120, "某步骤偏慢"); // 结构化字段：可以被日志系统检索、聚合
}

#[instrument]
async fn step1() {
    info!("步骤1"); // 自动带上父 span 的 request_id、user
}

struct Database;
fn uuid_v4() -> String { "req-abc123".into() }

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter( // 同样支持 RUST_LOG 环境变量过滤
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "myapp=info".into()))
        // .json()      // 生产环境开 JSON 输出，对接 ELK/Loki 等日志系统
        .init();

    let db = Database;
    handle_request("alice", &db).await;
}
```

**输出（文本模式）：**

```text
INFO handle_request{request_id="req-abc123" user="alice"}: 开始处理
INFO handle_request{request_id="req-abc123" user="alice"}:step1: 步骤1
```

**为什么线上要 JSON 输出？** 文本日志只能靠 grep；JSON 日志里 `latency_ms`、`request_id` 是可索引字段，可以在日志平台做「查 request_id=xxx 的全部日志」「统计 latency_ms 的 P99」这类查询。

> 建议：**写库用 `log`，写服务端应用用 `tracing`**。tracing 兼容 log 的宏（加 `tracing-log` 桥），迁移成本低。

---

## 第 8 章 chrono：日期与时间

### 8.1 它解决什么问题

`std::time::SystemTime` 只给时间戳，不管时区、格式化、日期运算。chrono 是生态标准的日期时间库。

```toml
[dependencies]
chrono = { version = "0.4", features = ["serde"] } # serde feature 让时间字段可直接序列化
```

### 8.2 常用操作

```rust
use chrono::{Local, Utc, NaiveDate, DateTime, Duration, TimeZone, Datelike, Timelike};

fn main() {
    // 1) 获取当前时间
    let now_utc = Utc::now();       // DateTime<Utc>：服务器存时间永远用 UTC
    let now_local = Local::now();   // DateTime<Local>：只用于展示给用户
    println!("UTC:   {now_utc}");
    println!("本地:  {now_local}");

    // 2) 格式化与解析
    println!("{}", now_utc.format("%Y-%m-%d %H:%M:%S"));          // 自定义格式
    println!("{}", now_utc.to_rfc3339());                          // 2026-09-28T12:00:00+00:00，API 交换标准格式
    let dt = DateTime::parse_from_rfc3339("2026-09-28T12:00:00+08:00").unwrap();
    let utc_dt = dt.with_timezone(&Utc);                           // 时区转换是零成本的（内部都是时间戳）

    // 3) 从日期构造时间
    let d = NaiveDate::from_ymd_opt(2026, 2, 29)   // Opt 版本返回 Option，不 panic
        .expect("2026 不是闰年，这里会 panic");
    let dt2 = Utc.from_utc_datetime(&d.and_hms_opt(8, 30, 0).unwrap());

    // 4) 时间运算
    let tomorrow = Utc::now() + Duration::days(1);
    let deadline = Utc::now() + Duration::hours(3) + Duration::minutes(30);
    let diff = deadline - Utc::now();       // 减法得 Duration
    println!("还剩 {} 秒", diff.num_seconds());
    let _ = (d, dt2, tomorrow);

    // 5) 取字段
    println!("{}年{}月{}日 星期{:?}", now_local.year(), now_local.month(), now_local.day(), now_local.weekday());
}
```

**为什么「存 UTC、展示转本地」是铁律？** 夏令时、用户跨时区、服务器迁移时区——只要存了本地时间，这些场景全是坑。UTC 是无歧义的单点时间，本地时间只是「某时区下的一种显示格式」。

### 8.3 与 serde 配合

```rust
use chrono::{DateTime, Utc, NaiveDate};
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct Event {
    name: String,
    at: DateTime<Utc>,                       // 默认序列化为 RFC3339 字符串
    #[serde(with = "date_fmt")]              // 自定义格式时用 with + 模块
    day: NaiveDate,
}

mod date_fmt {
    use chrono::{NaiveDate, serde::ts_microseconds as _}; // 只是示意，下面手写
    use serde::{Deserialize, Deserializer, Serializer};

    const FMT: &str = "%Y/%m/%d";
    pub fn serialize<S: Serializer>(d: &NaiveDate, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&d.format(FMT).to_string())
    }
    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<NaiveDate, D::Error> {
        let s = String::deserialize(d)?;
        NaiveDate::parse_from_str(&s, FMT).map_err(serde::de::Error::custom)
    }
}
```

> 小知识：chrono 自带 `chrono::serde::ts_seconds` / `ts_milliseconds` 模块，`#[serde(with = "chrono::serde::ts_seconds")]` 可以直接把时间序列化为时间戳数字，无需手写。

---

## 第 9 章 rand：随机数

```toml
[dependencies]
rand = "0.9"
```

> rand 0.9（2025 年起的主流版本）API 有变化：`thread_rng()` → `rng()`，`gen()` → `random()`，`gen_range()` → `random_range()`。

```rust
use rand::prelude::*;

fn main() {
    let mut rng = rand::rng();          // 线程局部、密码学安全的默认随机源

    // 1) 基本类型
    let x: u32 = rng.random();          // 任意实现了相应 trait 的类型
    let b: bool = rng.random();
    let f: f64 = rng.random();          // [0, 1) 均匀分布

    // 2) 范围随机
    let dice = rng.random_range(1..=6);         // 1-6 含 6
    let score = rng.random_range(0.0..100.0);   // 浮点也行

    // 3) 概率
    if rng.random_bool(0.3) { println!("30% 概率命中"); }

    // 4) 序列操作
    let mut cards = vec!["A", "K", "Q", "J"];
    cards.shuffle(&mut rng);                       // 洗牌
    let pick = cards.choose(&mut rng).unwrap();    // 随机选一个
    let picks: Vec<_> = cards.choose_multiple(&mut rng, 2).collect(); // 不重复选多个

    // 5) 随机字符串（验证码、临时文件名）
    let code: String = (0..6).map(|_| rng.random_range(0..10).to_string()).collect();
    let token: String = rand::distr::Alphanumeric
        .sample_string(&mut rng, 16);              // 字母数字混合 16 位

    println!("{x} {b} {f} {dice} {score} {pick} {picks:?} {code} {token}");

    // 6) 需要可复现（测试、仿真）时，用固定种子的生成器
    let mut seeded = rand::rngs::StdRng::seed_from_u64(42); // 同种子必然产生同序列
    let _v: u64 = seeded.random();
}
```

**为什么默认用 `rng()` 而不是自己播种？** `rng()` 背后是操作系统熵源，不可预测，适合 token、验证码等安全敏感场景。反过来，测试和仿真实验需要**可复现**，这时必须显式用 `StdRng::seed_from_u64(固定值)`——两种需求对应两种用法，库把选择权交给你，这正是「默认安全、显式可控」的设计。

---

## 第 10 章 regex：正则表达式

```toml
[dependencies]
regex = "1"
```

### 10.1 基本用法

```rust
use regex::Regex;

fn main() {
    // 1) 匹配判断
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap(); // r"..." 原始字符串，不用转义反斜杠
    assert!(re.is_match("2026-09-28"));
    assert!(!re.is_match("2026/09/28"));

    // 2) 查找
    let re = Regex::new(r"\d+").unwrap();
    let m = re.find("房间号是 1024 室").unwrap();
    println!("找到 '{}' 位于 {:?}", m.as_str(), m.range());

    // 3) 捕获组：提取结构化数据
    let re = Regex::new(r"(\w+)@(\w+\.\w+)").unwrap();
    let text = "联系我: alice@example.com";
    if let Some(caps) = re.captures(text) {
        println!("整体: {}", &caps[0]);   // caps[0] 永远是整个匹配
        println!("用户名: {}", &caps[1]);
        println!("域名: {}", &caps[2]);
    }

    // 命名捕获组：长正则里比数字序号更可读
    let re = Regex::new(r"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})").unwrap();
    let caps = re.captures("2026-09-28").unwrap();
    println!("{}月{}日", &caps["month"], &caps["day"]);

    // 4) 迭代所有匹配
    for m in re.find_iter("2026-09-28 和 2025-01-01") {
        println!("{:?}", m.as_str());
    }

    // 5) 替换
    let re = Regex::new(r"\d+").unwrap();
    let masked = re.replace_all("电话 13812345678 备用 01012345", "***");
    println!("{masked}");
    // 用捕获组做模板替换：$name 引用命名组
    let re = Regex::new(r"(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})").unwrap();
    println!("{}", re.replace("2026-09-28", "$d/$m/$y")); // 28/09/2026

    // 6) 分割
    let re = Regex::new(r"[,，;；\s]+").unwrap();
    let parts: Vec<&str> = re.split("苹果, 香蕉，橘子；西瓜 桃子").collect();
    println!("{parts:?}");
}
```

### 10.2 关键性能点：正则只编译一次

```rust
use regex::Regex;
use std::sync::LazyLock;

// Regex::new 要做语法解析+编译自动机，成本不低。
// 如果在循环或 handler 里每次 new，是新手最常见的性能坑。
// 为什么用 LazyLock？标准库 1.80+ 内置，首次访问时才初始化，线程安全，
// 替代过去的 lazy_static / once_cell 第三方宏。
static EMAIL_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^[\w.+-]+@[\w-]+\.[\w.]+$").unwrap()
});

fn validate(email: &str) -> bool {
    EMAIL_RE.is_match(email) // 复用同一份编译结果
}
```

**为什么 regex 是线性时间保证？** Rust 的 regex 用的是有限自动机而不是回溯引擎，**不存在**某些语言正则的「灾难性回溯」（恶意输入让匹配耗时指数爆炸）。代价是不支持反向引用、环视（lookahead）——需要这些时用 `fancy-regex` crate，并接受性能风险。

---

## 第 11 章 itertools：迭代器增强

标准库的迭代器适配器（map/filter/fold…）很强大，但 itertools 补齐了常用缺失操作。

```toml
[dependencies]
itertools = "0.14"
```

```rust
use itertools::Itertools;

fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6];

    // 1) unique：去重（保留顺序）
    let uniq: Vec<_> = v.iter().unique().collect(); // [3, 1, 4, 5, 9, 2, 6]

    // 2) sorted / sorted_by：迭代器直接排序
    let sorted: Vec<_> = v.iter().sorted().collect();
    let desc: Vec<_> = v.iter().sorted_by(|a, b| b.cmp(a)).collect();

    // 3) chunk / chunks：分组
    let chunks: Vec<Vec<_>> = v.iter().chunks(3).into_iter().map(|c| c.collect()).collect();
    // [[3,1,4], [1,5,9], [2,6]]

    // 4) group_by：按 key 分组（类似 SQL GROUP BY）
    let words = vec!["apple", "avocado", "banana", "blueberry", "cherry"];
    let groups = words.iter().into_group_map_by(|w| w.chars().next().unwrap());
    // {'a': [apple, avocado], 'b': [banana, blueberry], 'c': [cherry]}

    // 5) join：迭代器直接拼字符串（比 collect 成 Vec 再 join 少一步）
    let s = words.iter().join(", "); // "apple, avocado, ..."

    // 6) tuple_windows：相邻窗口
    let diffs: Vec<i32> = v.iter().tuple_windows().map(|(a, b)| b - a).collect();

    // 7) cartesian_product：笛卡尔积
    let pairs: Vec<_> = (1..=2).cartesian_product('a'..='b').collect();
    // [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

    // 8) minmax：一次遍历同时拿最小最大值
    let (min, max) = v.iter().minmax().into_option().unwrap();

    // 9) zip_eq：长度必须相等的 zip（不等会 panic，用于防御长度不匹配的静默截断 bug）
    let zipped: Vec<_> = [1, 2, 3].iter().zip_eq(['a', 'b', 'c'].iter()).collect();

    // 10) format! 风格快速打印
    println!("{:?} {:?} {:?} {:?} {:?} {:?} {:?}", uniq, sorted, desc, chunks, groups, diffs, pairs);
    println!("{s} | min={min} max={max} | {}", zipped.iter().map(|(a,_)| a).join("+"));
}
```

**为什么需要 itertools？** 标准库刻意保持精简，这些操作每个项目都要自己写一遍（而且手写 `group_by` 很容易写错）。itertools 的 API 与标准库 Iterator trait 无缝组合——`use itertools::Itertools;` 一行引入后，所有迭代器自动多出这些方法，零成本、零心智负担。

---

## 第 12 章 rayon：一行代码的数据并行

### 12.1 它解决什么问题

CPU 密集型循环想利用多核。传统做法要手动开线程、切分数据、收结果；rayon 只需把 `iter()` 换成 `par_iter()`。

```toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    (2..=((n as f64).sqrt() as u64)).all(|i| n % i != 0)
}

fn main() {
    let nums: Vec<u64> = (1_000_000..2_000_000).collect();

    // 串行版
    let count = nums.iter().filter(|&&n| is_prime(n)).count();

    // 并行版：iter -> par_iter，就这一个改动
    let count_par = nums.par_iter().filter(|&&n| is_prime(n)).count();
    assert_eq!(count, count_par);

    // 其他常见并行操作，API 与迭代器几乎一致
    let sum: u64 = nums.par_iter().sum();                    // 并行求和
    let squares: Vec<u64> = nums.par_iter().map(|n| n * n).collect(); // 并行 map 收集
    let max = nums.par_iter().max();
    nums.par_iter().for_each(|n| { /* 只执行不收集 */ let _ = n; });
    println!("{sum} {} {max:?}", squares.len());
}
```

**为什么 rayon 敢让你无脑换？** 两个设计：一是**工作窃取线程池**，任务自动均衡到所有核；二是**复用 Rust 的所有权系统**——`par_iter` 要求闭包 `Sync`/`Send`，数据竞争在编译期就是错误，所以并行化不会引入并发 bug。这就是 Rust 名言「无畏并发（fearless concurrency）」的最佳例证。

### 12.2 两个独立任务的并行：`join`

```rust
fn heavy_a() -> u64 { 1 }
fn heavy_b() -> u64 { 2 }

let (a, b) = rayon::join(heavy_a, heavy_b); // 两个闭包可能并行执行
println!("{a} {b}");
```

### 12.3 何时不该用 rayon

- **数据量小**（几百个元素）：并行调度开销大于收益，串行更快。
- **IO 密集**（网络/磁盘）：那不是 CPU 瓶颈，该用 tokio 异步而不是线程并行。
- 经验法则：**CPU 密集 + 单机 + 毫秒级以上的循环** → rayon；**IO 密集 + 高并发连接** → tokio。

---

## 第 13 章 sqlx：编译期检查的 SQL

### 13.1 它解决什么问题

Rust 数据库生态两大路线：**ORM（如 diesel、sea-orm）**用代码生成 SQL；**sqlx** 让你直接写 SQL，但用宏在**编译期连接真实数据库校验语句和类型**——SQL 写错了编译就过不去，兼得原生 SQL 的灵活和类型安全。

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "mysql", "chrono"] }
tokio = { version = "1", features = ["full"] }
chrono = "0.4"
```

### 13.2 连接池与基本 CRUD

```rust
use sqlx::MySqlPool;

#[derive(sqlx::FromRow, Debug)]       // FromRow：把查询结果行映射到结构体
struct User {
    id: i64,
    name: String,
    email: Option<String>,
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    // 连接池：为什么必须用池？建一次 TCP+认证握手要几十毫秒，
    // 每个请求新建连接数据库会被打爆。池复用连接，自动处理断线重连。
    let pool = MySqlPool::connect("mysql://user:pass@localhost/mydb").await?;

    // 建表（演示用，生产用 sqlx migrate 管理迁移）
    sqlx::query("CREATE TABLE IF NOT EXISTS user (id BIGINT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50), email VARCHAR(100))")
        .execute(&pool).await?;

    // 1) 插入：? 占位符绑定参数，杜绝 SQL 注入
    // 为什么不拼接字符串？format!("...WHERE id = {id}") 在 id 来自用户输入时就是注入漏洞
    let r = sqlx::query("INSERT INTO user (name, email) VALUES (?, ?)")
        .bind("小明")
        .bind("xm@example.com")
        .execute(&pool).await?;
    println!("插入 id = {}", r.last_insert_id());

    // 2) 查询单条映射到结构体
    let user = sqlx::query_as::<_, User>("SELECT id, name, email FROM user WHERE id = ?")
        .bind(1i64)
        .fetch_one(&pool).await?;      // 没有行时返回 Err(RowNotFound)
    println!("{user:?}");

    // 3) 查询多条
    let users: Vec<User> = sqlx::query_as("SELECT id, name, email FROM user WHERE name LIKE ?")
        .bind("小%")
        .fetch_all(&pool).await?;
    println!("共 {} 人", users.len());

    // 4) fetch_optional：行可能不存在时
    let maybe: Option<User> = sqlx::query_as("SELECT id, name, email FROM user WHERE id = ?")
        .bind(999i64)
        .fetch_optional(&pool).await?;
    println!("{maybe:?}");

    // 5) 更新 / 删除：看受影响行数
    let r = sqlx::query("UPDATE user SET email = ? WHERE id = ?")
        .bind("new@example.com").bind(1i64)
        .execute(&pool).await?;
    println!("更新 {} 行", r.rows_affected());

    // 6) 事务：要么全成，要么全回滚。转账类操作的底线
    let mut tx = pool.begin().await?;
    sqlx::query("UPDATE account SET balance = balance - 100 WHERE id = 1")
        .execute(&mut *tx).await?;
    sqlx::query("UPDATE account SET balance = balance + 100 WHERE id = 2")
        .execute(&mut *tx).await?;
    tx.commit().await?;   // 若中途 panic 或提前返回，tx 被 drop 自动回滚
    Ok(())
}
```

**为什么事务 drop 自动回滚的设计很好？** 你无法「忘记 rollback」——忘记 commit 就什么都不会发生，资金类系统的失败默认方向是安全的（钱不动），而不是危险的（扣了没加）。这是 Rust RAII 哲学在数据库层的体现。

### 13.3 编译期校验宏（进阶）

```rust
// 需要编译时能连到数据库（DATABASE_URL 环境变量或 .sqlx 离线数据）
// let user = sqlx::query_as!(User, "SELECT id, name, email FROM user WHERE id = ?", 1i64)
//     .fetch_one(&pool).await?;
// 表不存在、列名打错、类型不匹配 → 编译报错，不是运行时 500
```

> 与 axum 配合时，把 `MySqlPool` 放进 `State`（见第 5 章），pool 本身是 `Clone` 的廉价句柄。

---

## 第 14 章 高频小工具库速查

按「什么时候会想起它」组织：

### 14.1 环境变量配置：dotenvy

```toml
dotenvy = "0.15"
```

```rust
// 开发时把配置写在项目根目录 .env 文件里（DATABASE_URL=mysql://...）
// main 开头一行加载，之后 std::env::var 就能读到
dotenvy::dotenv().ok();
// 为什么用 .ok() 忽略错误？生产环境没有 .env 文件（配置走真实环境变量），不应 panic
```

### 14.2 UUID：uuid

```toml
uuid = { version = "1", features = ["v4", "serde"] }
```

```rust
let id = uuid::Uuid::new_v4();            // 随机 UUID，做请求 ID、资源 ID
println!("{id}");                          // 550e8400-e29b-41d4-a716-446655440000
```

### 14.3 全局一次性初始化：标准库 LazyLock / OnceLock

```rust
use std::sync::LazyLock;
// 1.80+ 标准库内置，别再引 lazy_static / once_cell 了
static CONFIG: LazyLock<String> = LazyLock::new(|| {
    std::env::var("APP_CONFIG").unwrap_or_else(|_| "default".into())
});
```

### 14.4 编码解码：base64 / hex

```toml
base64 = "0.22"
hex = "0.4"
```

```rust
use base64::{Engine as _, engine::general_purpose::STANDARD};
let b64 = STANDARD.encode("你好");         // 字节 -> base64 字符串
let raw = STANDARD.decode(&b64).unwrap(); // 还原
let h = hex::encode(b"abc");              // "616263"
```

### 14.5 遍历目录：walkdir

```toml
walkdir = "2"
```

```rust
use walkdir::WalkDir;
// 递归遍历，自动处理符号链接和排序
for entry in WalkDir::new("./src").into_iter().filter_map(|e| e.ok()) {
    if entry.path().extension().is_some_and(|e| e == "rs") {
        println!("{}", entry.path().display());
    }
}
```

### 14.6 临时文件：tempfile（测试常用）

```toml
tempfile = "3"
```

```rust
let mut f = tempfile::NamedTempFile::new().unwrap();
use std::io::Write;
writeln!(f, "test").unwrap();
// 为什么用库而不是自己拼 /tmp/xxx？自动防重名、权限正确、离开作用域自动删除，测试不残留垃圾
```

### 14.7 CSV 读写：csv

```toml
csv = "1"
```

```rust
// 读取
let mut rdr = csv::Reader::from_path("data.csv").unwrap();
for row in rdr.records() {
    let row = row.unwrap();
    println!("{:?}", row);
}
// 配合 serde 直接反序列化为结构体
// for row in rdr.deserialize::<MyRow>() { let r: MyRow = row.unwrap(); }

// 写入
let mut wtr = csv::Writer::from_path("out.csv").unwrap();
wtr.write_record(["name", "age"]).unwrap();
wtr.write_record(["小明", "20"]).unwrap();
wtr.flush().unwrap();
```

### 14.8 进度条：indicatif（CLI 工具体验拉满）

```toml
indicatif = "0.17"
```

```rust
use indicatif::{ProgressBar, ProgressStyle};
let pb = ProgressBar::new(1000);
pb.set_style(ProgressStyle::default_bar()
    .template("{bar:40.cyan/blue} {pos}/{len} ETA:{eta}").unwrap());
for _ in 0..1000 {
    std::thread::sleep(std::time::Duration::from_millis(2));
    pb.inc(1);
}
pb.finish_with_message("完成");
```

### 14.9 URL 解析：url

```toml
url = "2"
```

```rust
let u = url::Url::parse("https://example.com:8080/path?a=1#frag").unwrap();
println!("{} {} {}", u.host_str().unwrap(), u.port().unwrap_or(443), u.path());
// 为什么不用字符串切分？URL 有百分号编码、默认端口、相对路径解析等一堆边角，
// 手写必错，用标准库式的解析器一次到位
```

### 14.10 优雅退出：ctrlc / tokio::signal

```rust
// tokio 程序里监听 Ctrl+C，先收尾再退出（关闭连接池、刷日志、保存状态）
tokio::select! {
    _ = run_server() => {},
    _ = tokio::signal::ctrl_c() => { println!("收到退出信号，开始收尾"); }
}
// 为什么不能直接被杀？直接 SIGKILL 可能丢未落库数据、连接不释放。
// 优雅停机（graceful shutdown）是服务端工程素养的基本功。
```

---

## 第 15 章 测试相关的常用工具

### 15.1 内置测试框架回顾

```rust
fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]            // 只在 cargo test 时编译，不进正式产物
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(1, 2), 3);            // 相等断言，失败时打印两边值
        assert_ne!(add(1, 2), 4);
        assert!(add(2, 2) > 3, "自定义失败信息"); // 第二参数是失败时的提示
    }

    #[test]
    #[should_panic(expected = "除以零")]      // 断言「必须 panic 且信息包含指定文本」
    fn test_div_zero() { panic!("除以零"); }

    #[test]
    fn test_result() -> Result<(), String> { // 返回 Result 的测试：Err 即失败，能用 ?
        if add(2, 2) == 4 { Ok(()) } else { Err("算错了".into()) }
    }

    #[test]
    #[ignore = "需要真实数据库，CI 里单独跑"] // cargo test -- --ignored 才运行
    fn test_integration() {}
}
```

### 15.2 浮点数比较：approx

```toml
[dev-dependencies]       # 只在测试时编译的依赖放这里
approx = "0.5"
```

```rust
// 为什么不直接 assert_eq!(0.1 + 0.2, 0.3)？浮点精度问题，0.1+0.2 ≠ 0.3 是成立的！
assert_relative_eq!(0.1 + 0.2, 0.3, epsilon = 1e-10); // 需要 use approx::assert_relative_eq;
```

### 15.3 异步测试：直接用 tokio 宏

```rust
#[tokio::test]           // 自动给你建运行时
async fn test_async() {
    let resp = async { 42 }.await;
    assert_eq!(resp, 42);
}
```

### 15.4 Mock：mockall（简例）

```toml
[dev-dependencies]
mockall = "0.13"
```

```rust
use mockall::{automock, predicate::*};

#[automock]                        // 自动生成 MockDb，可预设每个方法的返回值与调用次数
trait Db { fn get(&self, id: u64) -> Option<String>; }

fn find_name(db: &impl Db, id: u64) -> String {
    db.get(id).unwrap_or("匿名".into())
}

#[test]
fn test_find() {
    let mut mock = MockDb::new();
    mock.expect_get()              // 预期 get 会被调用
        .with(eq(1u64))            // 参数必须是 1
        .times(1)
        .returning(|_| Some("小明".into()));
    assert_eq!(find_name(&mock, 1), "小明");
    // 为什么 mock 而不是连真库？单元测试要快、稳定、可并行，
    // 数据库交互的正确性交给少量集成测试去验证
}
```

---

## 附录：如何为项目选库

**凭这三条原则做绝大多数决策：**

1. **看生态位**：Rust 生态每个领域通常有一个「事实标准」（serde、tokio、reqwest、clap、sqlx……）。没有特殊理由就用它——文档最多、Stack Overflow 答案最全、其他库的集成最好。
2. **看维护状态**：在 crates.io 看最近更新时间和下载量；在 GitHub 看 issue 响应。两年没更新且 issue 堆积的库要警惕。
3. **看编译期 vs 运行时的取舍**：Rust 社区的偏好是**能在编译期抓到的错误绝不拖到运行时**（sqlx 的编译期 SQL 校验、axum 的类型化提取器、serde 的 derive）。选库时倾向这种风格的，长期少踩坑。

**本项目速查表：**

| 需求           | 用这个                            |
| ------------ | ------------------------------ |
| JSON / 配置序列化 | serde + serde_json / toml      |
| 应用错误处理       | anyhow（库用 thiserror）           |
| 异步 / 网络服务    | tokio                          |
| 调 HTTP API   | reqwest                        |
| 写 HTTP 服务    | axum                           |
| 命令行工具        | clap                           |
| 日志           | tracing（简单场景 log + env_logger） |
| 日期时间         | chrono                         |
| 随机数          | rand                           |
| 正则           | regex                          |
| 迭代器不够用       | itertools                      |
| CPU 密集并行     | rayon                          |
| 数据库          | sqlx（要 ORM 再看 sea-orm）         |
| 环境变量         | dotenvy                        |
| UUID         | uuid                           |

> 最后一条建议：**不要一次全用**。新项目从「std + serde + tokio + 领域必需的一两个库」开始，用到什么加什么。Rust 的编译器会接住你，依赖越少，构建越快，心智越轻。
