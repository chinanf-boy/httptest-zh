# brson/httptest [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 desc 」

[中文](./readme.md) | [english](https://github.com/brson/httptest)

---

## 校对 🀄️

<!-- doc-templite START generated -->
<!-- repo = 'brson/httptest' -->
<!-- commit = '1d2b6c9b81bdd6cb3b67be3c9245389462e89426' -->
<!-- time = '2017-07-04' -->

| 翻译的原文 | 与日期        | 最新更新 | 更多                       |
| ---------- | ------------- | -------- | -------------------------- |
| [commit]   | ⏰ 2017-07-04 | ![last]  | [中文翻译][translate-list] |

[last]: https://img.shields.io/github/last-commit/brson/httptest.svg
[commit]: https://github.com/brson/httptest/tree/1d2b6c9b81bdd6cb3b67be3c9245389462e89426

<!-- doc-templite END generated -->

### 贡献

欢迎 👏 勘误/校对/更新贡献 😊 [具体贡献请看](https://github.com/chinanf-boy/chinese-translate-list#贡献)

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

### 目录

<!-- START doctoc -->
<!-- END doctoc -->

# 让我们用 Rust 搞个 web service 和 client

所以我正在研究这个项目[弹坑]做的[锈]回归测试.研究在 Rust 中编写部件的可行性.我需要一个能说 JSON 的 HTTP 服务器以及相应的客户端.我没有看到很多关于如何做到这一点的文档,所以我正在记录我的调查.

我打算用[铁]和[超],我都没有经历过.

此仓库中的每个提交都对应一个章节,因此如果您愿意,请跟随.

**编辑:内部的一些不准确![Thanks /r/rust!](http://www.reddit.com/r/rust/comments/33k1yn/lets_make_a_web_service_and_client/)**

# 1. Preparing to serve some JSON

我首先要求 Cargo 给我一个名为'httptest'的新可执行项目.通过`--bin`说要创建一个源文件`src/main.rs`这将被编译到一个应用程序.

```text
$ cargo new httptest --bin
```

我会用的[铁]对于服务器,请按照其文档中的说明将以下内容添加到我的`Cargo.toml`文件.

```toml
[dependencies]
iron = "*"
```

然后进入`main.rs`我只是复制他们的例子.

```rust
extern crate iron;

use iron::prelude::*;
use iron::status;

fn main() {
    fn hello_world(_: &mut Request) -> IronResult<Response> {
        Ok(Response::with((status::Ok, "Hello World!")))
    }

    Iron::new(hello_world).http("localhost:3000").unwrap();
    println!("On 3000");
}
```

并输入`cargo build`.

```text
$ cargo build
Compiling hyper v0.3.13
Compiling iron v0.1.16
Compiling httptest v0.2.0 (file:///opt/dev/httptest)
```

迄今为止取得了成功.该死的,Rust 很顺利.让我们尝试运行服务器`cargo run`.

```text
$ cargo run
Running `target/debug/httptest`
```

我坐在这里等待一段时间,期待它打印"On 3000",但它永远不会.货物必须捕获输出.让我们看看我们是否正在服务.

```text
$ curl http://localhost:3000
Hello World!
```

哦,那太酷了.我们现在知道如何构建 Web 服务器.良好的起点.

# 2. Serving a struct as JSON

是[rustc 序列化-]仍然是转换为 JSON 和从 JSON 转换的最简单方法?也许我应该用[serde],但那你真的想要[serde_macros],但这只适用于 Rust nightlies.我应该只使用夜莺吗?没有其他人需要使用它.

让我们继续尝试真实的 rustc-serialize.现在我的 Cargo.toml'依赖关系'部分如下所示.

```toml
[dependencies]
iron = "*"
rustc-serialize = "*"
```

基于[rustc-serialize docs](https://doc.rust-lang.org/rustc-serialize/rustc_serialize/json/index.html)我更新`main.rs`看起来像这样:

```rust
extern crate iron;
extern crate rustc_serialize;

use iron::prelude::*;
use iron::status;
use rustc_serialize::json;

#[derive(RustcEncodable)]
struct Greeting {
    msg: String
}

fn main() {
    fn hello_world(_: &mut Request) -> IronResult<Response> {
        let greeting = Greeting { msg: "Hello, World".to_string() };
        let payload = json::encode(&greeting).unwrap();
        Ok(Response::with((status::Ok, payload)))
    }

    Iron::new(hello_world).http("localhost:3000").unwrap();
    println!("On 3000");
}
```

然后跑`cargo build`.

```text
$ cargo build
   Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading unicase v1.1.1
 <... more downloading here ...>
   Compiling lazy_static v0.1.15
 <... more compiling here ...>
   Compiling httptest v0.2.0 (file:///opt/dev/httptest)
src/main.rs:17:12: 17:47 error: this function takes 1 parameter but 2 parameters were supplied [E0061]
src/main.rs:17         Ok(Response::with(status::Ok, payload))
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/main.rs:17:12: 17:47 help: run `rustc --explain E0061` to see a detailed explanation
error: aborting due to previous error
Could not compile `httptest`.

To learn more, run the command again with --verbose.
```

很多新的依赖现在.但是一个错误.[`Response::with`](http://ironframework.io/doc/iron/response/struct.Response.html#method.with)有这个定义:

```rust
fn with<M: Modifier<Response>>(m: M) -> Response
```

我不知道是什么`Modifier`是.[Some
docs](http://ironframework.io/doc/iron/modifiers/index.html)没有帮助.我不知道该怎么做但我注意到原来的例子传递了一个元组`Response::with`而我的更新处理`Response::with`作为两个参数.似乎元组是一个`Modifier`.

将元组添加到`Ok(Response::with((status::Ok, payload)))`, 执行`cargo run`,卷曲一些 JSON.

```
$ curl http://localhost:3000
{"msg":"Hello, World"}
```

布拉姆!我们正在发送 JSON.是时候休息一下了.

# 3. Routes

接下来我想发布一些 JSON,但在我这样做之前,我需要一个合适的 URL 来发布,所以我想我需要学习如何设置路由.

我看着铁[docs](http://ironframework.io/doc/iron/)并且在正文中没有看到任何明显的东西,但是有一个叫做的箱子[router](http://ironframework.io/doc/router/index.html)这可能很有趣.

模块文档是"`Router`为 Iron 提供了快速灵活的路由,但其他并不多.我如何使用?!`Router`在我发现很多侦探之后.[an
example](https://github.com/iron/router/blob/master/examples/simple.rs)好吧,让我们试着让它适应我们不断发展的实验.

我加`router = "*"`对我`Cargo.toml` `[dependencies]`开始写作.以下是我在阅读 POST 数据之前想到的.

```rust
extern crate iron;
extern crate router;
extern crate rustc_serialize;

use iron::prelude::*;
use iron::status;
use router::Router;
use rustc_serialize::json;

#[derive(RustcEncodable, RustcDecodable)]
struct Greeting {
    msg: String
}

fn main() {
    let mut router = Router::new();

    router.get("/", hello_world);
    router.post("/set", set_greeting);

    fn hello_world(_: &mut Request) -> IronResult<Response> {
        let greeting = Greeting { msg: "Hello, World".to_string() };
        let payload = json::encode(&greeting).unwrap();
        Ok(Response::with((status::Ok, payload)))
    }

    // Receive a message by POST and play it back.
    fn set_greeting(request: &mut Request) -> IronResult<Response> {
        let payload = request.body.read_to_string();
        let request: Greeting = json::decode(payload).unwrap();
        let greeting = Greeting { msg: request.msg };
        let payload = json::encode(&greeting).unwrap();
        Ok(Response::with((status::Ok, payload)))
    }

    Iron::new(router).http("localhost:3000").unwrap();
}
```

这种用途`Router`控制处理程序调度.它构建并仍然响应`curl http://localhost:3000`但是`/set`路由尚未实现.

现在将 POST 正文读入字符串.这个[docs for
`Request`](http://ironframework.io/doc/iron/request/struct.Request.html)说说这一领域`body`是一个迭代器,因此我们只需要将该迭代器收集到一个字符串中.

我第一次尝试`let payload = request.body.read_to_string();`因为我知道它曾经有效.

它不起作用.

```text
$ cargo build
Compiling httptest v0.2.0 (file:///opt/dev/httptest)
src/main.rs:29:36: 29:52 error: no method named `read_to_string` found for type `iron::request::Body<'_, '_>` in the current scope
src/main.rs:29         let payload = request.body.read_to_string();
                                                  ^~~~~~~~~~~~~~~~
src/main.rs:29:36: 29:52 help: items from traits can only be used if the trait is in scope; the following trait is implemented but not in scope, perhaps add a `use` for it:
src/main.rs:29:36: 29:52 help: candidate #1: use `std::io::Read`
error: aborting due to previous error
Could not compile `httptest`.

To learn more, run the command again with --verbose.
```

我厌恶地举起双手.为什么这个方法不再存在?拉斯特队总是捉弄我们!然后,我注意到编译器在一定程度上已经解释了方法的存在,并且我应该导入`std::io::Read`.

我添加导入并发现`read_to_string`我的行为与我想象的不同.

```text
101 $ cargo build
Compiling httptest v0.2.0 (file:///opt/dev/httptest)
src/main.rs:30:36: 30:52 error: this function takes 1 parameter but 0 parameters were supplied [E0061]
src/main.rs:30         let payload = request.body.read_to_string();
                                                  ^~~~~~~~~~~~~~~~
```

好的,是的`Read::read_to_string`现在是`fn read_to_string(&mut self, buf: &mut String) -> Result<usize, Error>`从而提供缓冲区并处理错误.重写`set_greeting`方法.

```rust
    // Receive a message by POST and play it back.
    fn set_greeting(request: &mut Request) -> IronResult<Response> {
        let mut payload = String::new();
        request.body.read_to_string(&mut payload).unwrap();
        let request: Greeting = json::decode(&payload).unwrap();
        let greeting = Greeting { msg: request.msg };
        let payload = json::encode(&greeting).unwrap();
        Ok(Response::with((status::Ok, payload)))
    }
```

让我们运行这个并给它一个卷曲.

```text
$ curl -X POST -d '{"msg":"Just trust the Rust"}' http://localhost:3000/set
{"msg":"Just trust the Rust"}
```

哦,锈.你太糟糕了.

# 4. Mutation

嘿,这些我都知道`.unwrap()`S 是错的.我不在乎.我们正在做原型.

在我们继续编写客户端之前,我想修改这个玩具示例,以便在 POST 上存储一些状态`/set`以后再报告.我会做的`greeting`本地的,在一些闭包中捕获它,然后查看编译器如何抱怨.

这是我的新`main`函数,在尝试编译之前:

```rust
fn main() {
    let mut greeting = Greeting { msg: "Hello, World".to_string() };

    let mut router = Router::new();

    router.get("/", |r| hello_world(r, &greeting));
    router.post("/set", |r| set_greeting(r, &mut greeting));

    fn hello_world(_: &mut Request, greeting: &Greeting) -> IronResult<Response> {
        let payload = json::encode(&greeting).unwrap();
        Ok(Response::with((status::Ok, payload)))
    }

    // Receive a message by POST and play it back.
    fn set_greeting(request: &mut Request, greeting: &mut Greeting) -> IronResult<Response> {
        let mut payload = String::new();
        request.body.read_to_string(&mut payload).unwrap();
        *greeting = json::decode(&payload).unwrap();
        Ok(Response::with(status::Ok))
    }

    Iron::new(router).http("localhost:3000").unwrap();
}
```

```text
src/main.rs:21:12: 21:51 error: type mismatch resolving `for<'r, 'r, 'r> <[closure@src/main.rs:21:21: 21:50 greeting:_] as core::ops::FnOnce<(&'r mut iron::request::Request<'r, 'r>,)>>::Output == core::result::Result<iron::response::Response, iron::error::IronError>`:
 expected bound lifetime parameter ,
    found concrete lifetime [E0271]
src/main.rs:21     router.get("/", |r| hello_world(r, &greeting));
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/main.rs:21:12: 21:51 help: run `rustc --explain E0271` to see a detailed explanation
src/main.rs:22:12: 22:60 error: type mismatch resolving `for<'r, 'r, 'r> <[closure@src/main.rs:22:25: 22:59 greeting:_] as core::ops::FnOnce<(&'r mut iron::request::Request<'r, 'r>,)>>::Output == core::result::Result<iron::response::Response, iron::error::IronError>`:
 expected bound lifetime parameter ,
    found concrete lifetime [E0271]
src/main.rs:22     router.post("/set", |r| set_greeting(r, &mut greeting));
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/main.rs:22:12: 22:60 help: run `rustc --explain E0271` to see a detailed explanation
src/main.rs:21:12: 21:51 error: type mismatch: the type `[closure@src/main.rs:21:21: 21:50 greeting:&Greeting]` implements the trait `core::ops::Fn<(&mut iron::request::Request<'_, '_>,)>`, but the trait `for<'r, 'r, 'r> core::ops::Fn<(&'r mut iron::request::Request<'r, 'r>,)>` is required (expected concrete lifetime, found bound lifetime parameter ) [E0281]
src/main.rs:21     router.get("/", |r| hello_world(r, &greeting));
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/main.rs:21:12: 21:51 help: run `rustc --explain E0281` to see a detailed explanation
src/main.rs:22:12: 22:60 error: the trait `for<'r, 'r, 'r> core::ops::Fn<(&'r mut iron::request::Request<'r, 'r>,)>` is not implemented for the type `[closure@src/main.rs:22:25: 22:59 greeting:&mut Greeting]` [E0277]
src/main.rs:22     router.post("/set", |r| set_greeting(r, &mut greeting));
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
src/main.rs:22:12: 22:60 help: run `rustc --explain E0277` to see a detailed explanation
error: aborting due to 4 previous errors
Could not compile `httptest`.
```

拉斯特克不高兴.但我想不到.我抛出类型只是为了得到响应.告诉我怎么做生锈.

这些错误消息令人困惑,但显然闭包是错误的类型.这个`get`和`post`方法`Router`拿一个[`Handler`](http://ironframework.io/doc/iron/middleware/trait.Handler.html)在 doc 页面中,我看到一个 impl 被定义为

```rust
pub trait Handler: Send + Sync + Any {
    fn handle(&self, &mut Request) -> IronResult<Response>;
}
```

一口,但是`Handler`定义为`Fn`不是`FnOnce`或`FnMut`必须如此`Send + Sync`. 因为它需要发送,所以我们不会捕获任何引用,并且由于环境不是可变的,所以我们必须使用内部可变性来改变问候语.所以我要使用可发送的智能指针,`Arc`为了使它可变,将`Mutex`里面.为此,我们需要像这样从标准库中导入两者:`use std::sync::{Mutex, Arc};`. 我还要移动捕获`move |r| ...`避免通过引用捕获.

像这样更新我的代码会产生相同的错误消息.

```rust
    let greeting = Arc::new(Mutex::new(Greeting { msg: "Hello, World".to_string() }));
    let greeting_clone = greeting.clone();

    let mut router = Router::new();

    router.get("/", move |r| hello_world(r, &greeting.lock().unwrap()));
    router.post("/set", move |r| set_greeting(r, &mut greeting_clone.lock().unwrap()));
```

rustc 不喜欢我关门的日子.为什么?我不知道.我问瑞姆他是否知道该怎么做.

几个小时后,雷姆说

```text
16:02 < reem> brson: Partially hint the type of the request art, rustc has trouble inferring HRTBs
```

HRTB 意味着"更高级别的特征界限",这意味着大致"复杂的寿命".

我将这些相同的行更改为提示`r: &mut Request`一切正常……

```rust
    let greeting = Arc::new(Mutex::new(Greeting { msg: "Hello, World".to_string() }));
    let greeting_clone = greeting.clone();

    let mut router = Router::new();

    router.get("/", move |r: &mut Request| hello_world(r, &greeting.lock().unwrap()));
    router.post("/set", move |r: &mut Request| set_greeting(r, &mut greeting_clone.lock().unwrap()));
```

这似乎是 Rust 的推理器中的一个 bug.那是跛脚的.

现在它再次构建,所以我们可以用 curl 进行测试.

```text
$ curl http://localhost:3000
{"msg":"Hello, World"}
$ curl -X POST -d '{"msg":"Just trust the Rust"}' http://localhost:3000/set
$ curl http://localhost:3000
{"msg":"Just trust the Rust"}
```

现在我们正在玩弄权力.

# 5. The client

我们有一个小 JSON 服务器.现在让我们编写客户端.这次我们将使用[超级]直接.

我把它加在我的身上`Cargo.toml`:`hyper = "*"`然后创建`src/bin/client.rs`:

```rust
extern crate hyper;

fn main() { }
```

源文件`src/bin/`由货物自动构建为可执行文件.跑`cargo build`以及`target/debug/client`程序出现.好,宇宙是理智的.现在找出 Hyper.

趴下[Hyper client example](http://hyperium.github.io/hyper/hyper/client/index.html)我想到了这个片段,它仅请求"/"并打印正文:

```rust
extern crate hyper;

use hyper::*;
use std::io::Read;

fn main() {
    let client = Client::new();
    let mut res = client.get("http://localhost:3000/").send().unwrap();
    assert_eq!(res.status, hyper::Ok);
    let mut s = String::new();
    res.read_to_string(&mut s).unwrap();
    println!("{}", s);
}
```

但是现在`cargo run`不再工作.

```text
$ cargo run
`cargo run` requires that a project only have one executable; use the `--bin` option to specify which one to run
```

我必须打字`cargo run --bin httptest`启动服务器.我这样做,然后`cargo run --bin client`看到

```text
$ cargo run --bin client
Running `target/debug/client`
{"msg":"Hello, World"}
```

哦,伙计,我是个锈迹巫师.最后一件事,我想做的是,发出 POST 请求来设置消息.显然要做的是改变`client.get`到[`client.post`](http://hyperium.github.io/hyper/hyper/client/struct.Client.html#method.post). 返回一个[`RequestBuilder`](http://hyperium.github.io/hyper/hyper/client/struct.RequestBuilder.html)因此,我正在寻找设置有效负载的构建器方法.怎么样[`body`](http://hyperium.github.io/hyper/hyper/client/struct.RequestBuilder.html#method.body)?

我的新创造:

```rust
extern crate hyper;

use hyper::*;
use std::io::Read;

fn main() {
    let client = Client::new();
    let res = client.post("http://localhost:3000/set").body("Just trust the Rust").send().unwrap();
    assert_eq!(res.status, hyper::Ok);
    let mut res = client.get("http://localhost:3000/").send().unwrap();
    assert_eq!(res.status, hyper::Ok);
    let mut s = String::new();
    res.read_to_string(&mut s).unwrap();
    println!("{}", s);
}
```

但是运行它是令人失望的.

```text
101 $ cargo run --bin client
   Compiling httptest v0.2.0 (file:///Users/danieleesposti/workspace/httptest)
     Running `target/debug/client`
thread '<main>' panicked at 'assertion failed: `(left == right)` (left: `InternalServerError`, right: `Ok`)', src/bin/client.rs:9
```

同时我看到服务器也出错:

```text
$ cargo run --bin httptest
   Running `target/debug/httptest`
thread '<unnamed>' panicked at 'called `Result::unwrap()` on an `Err` value: ParseError(SyntaxError("invalid syntax", 1, 1))', src/libcore/result.rs:741
```

那是因为我错误处理不好!我没有将有效的 JSON 传递给`/set`路线.固定身体`.body(r#"{ "msg": "Just trust the Rust" }"#)`让客户端成功:

```text
$ cargo run --bin client
   Compiling httptest v0.2.0 (file:///opt/dev/httptest)
      Running `target/debug/client`
{"msg":"Just trust the Rust"}
```

就像这样,我们在 Rust 中创建了一个 Web 服务和客户端.看来前途不远了.去建立一些东西[熨斗]和[超级].

[crater]: https://github.com/brson/taskcluster-crater
[rust]: https://github.com/brson/taskcluster-crater
[iron]: http://ironframework.io/
[hyper]: http://hyperium.github.io/hyper/hyper/index.html
[rustc-serialize]: https://crates.io/crates/rustc-serialize
[serde]: https://crates.io/crates/serde
[serde_macros]: https://crates.io/crates/serde_macros
