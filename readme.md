# brson/httptest [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 让我们用 Rust 搞个 web service 和 client ，用[Iron]和[Hyper]  」

[中文](./readme.md) | [english](https://github.com/brson/httptest)

---

## 校对 ✅

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

所以我正在研究这个项目[Crater]做的[Rust]回归测试。研究在 Rust 中编写部件的可行性。我需要一个能说 JSON 的 HTTP 服务器，以及相应的客户端。我没有看到很多关于如何做到这一点的文档，所以我在这里记录我的调查。

我打算用[Iron]和[Hyper]，都没有用过.

此仓库中的每个提交都对应一个章节，因此如果您愿意,请跟随.

**请编辑: 当内部不准确! [Thanks /r/rust!](http://www.reddit.com/r/rust/comments/33k1yn/lets_make_a_web_service_and_client/)**

# 1. 准备服务一些 JSON

我首先要求 Cargo 给我一个名为'httptest'的新可执行项目。传递`--bin`，创建一个源文件`src/main.rs`，终会被编译到一个应用程序(二进制)。

```text
$ cargo new httptest --bin
```

我会用[Iron]作为服务器(框架),请按照对应文档中的说明，将以下内容添加到我的`Cargo.toml`文件.

```toml
[dependencies]
iron = "*"
```

然后进入`main.rs`，只是复制他们的例子.

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

然后，输入`cargo build`.

```text
$ cargo build
Compiling hyper v0.3.13
Compiling iron v0.1.16
Compiling httptest v0.2.0 (file:///opt/dev/httptest)
```

迄今为止，要取得了成功哦。喔呼，Rust 超爽快的。让我们尝试运行服务器`cargo run`.

```text
$ cargo run
Running `target/debug/httptest`
```

我坐在这里等待一段时间，期待它打印"On 3000"，但它永远不会打印。Cargo 要捕获输出，以此让我们看看我们是否正在服务。

```text
$ curl http://localhost:3000
Hello World!
```

哦,酷。我们现在知道如何构建 Web 服务器。良好的起点。

# 2. 让我们把 结构 变为 JSON

[rustc-serialize]仍然是转换为 JSON 和，从 JSON 转换的最简单方法? 也许我应该用[serde]，若你真的想要[serde_macros]的话，但这只适用于 Rust 每晚版本。我应该只使用每晚版本吗? 没有其他人需要使用它。

让我们继续尝试真实的 rustc-serialize。现在我的 Cargo.toml 的'依赖关系(dependencies)'部分如下所示.

```toml
[dependencies]
iron = "*"
rustc-serialize = "*"
```

基于[rustc-serialize 文档](https://doc.rust-lang.org/rustc-serialize/rustc_serialize/json/index.html)，我更新了`main.rs`，看起来像这样:

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

然后，`cargo build`.(以下错误，源自我的失误，上面例子是对的)

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

现在又多了很多新依赖，但，有一个[`Response::with`](http://ironframework.io/doc/iron/response/struct.Response.html#method.with)错误出来:

```rust
fn with<M: Modifier<Response>>(m: M) -> Response
```

我不知道`Modifier`是什么。[查查文档](http://ironframework.io/doc/iron/modifiers/index.html)也没有帮助。我不知道该怎么做，但我注意到原来的例子传递了一个元组给`Response::with`，而我新的`Response::with`处理，则有两个参数。似乎元组是一个`Modifier`.

将元组添加到`Ok(Response::with((status::Ok, payload)))`, 再执行`cargo run`, curl 一些 JSON.

```
$ curl http://localhost:3000
{"msg":"Hello, World"}
```

Boom! 我们正在发送 JSON。是时候加上另一些'佐料'了.

# 3. 路由

接下来我想 POST 一些 JSON，但在我这样做之前，我需要一个合适的 URL 来 POST，所以我想我需要学习如何设置路由。

我看着 Iron 的[文档](http://ironframework.io/doc/iron/)，并且在正文中没有看到任何明显的东西，但是有一个叫做[router](http://ironframework.io/doc/router/index.html)的箱子，可能很有趣。

它的模块文档，说明了"`Router`为 Iron 提供了快速灵活的路由"，但其他内容并不多。我如何使用`Router`?! 在我做了很多侦探工作之后，我发现了[一个示例](https://github.com/iron/router/blob/master/examples/simple.rs)好吧,让我们试着让它适应我们不断发展的实验。

我加`router = "*"`到`Cargo.toml`的 `[dependencies]`部分，然后开始写作。以下是我在阅读 POST 数据之前，想到的。

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

这使用了`Router`来控制，处理程序调度。它构建后，仍会响应`curl http://localhost:3000`，但是`/set`路由并没有真正实现。

现在将 POST body 读入字符串。这个[`Request`的文档](http://ironframework.io/doc/iron/request/struct.Request.html)说这`body`字段是一个迭代器，因此我们只需要将该迭代器收集到一个字符串中。

我第一次尝试`let payload = request.body.read_to_string();`因为我知道它曾经有效过。

但现在没有了。

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

我厌恶地举起双手(屏蔽)。为什么这个方法不再存在?! Rust 团队总是捉弄我们! 但,我注意到编译器在一定程度上已经解释了方法的存在，并告诉我应该导入`std::io::Read`。

我添加导入`read_to_string`并发现其行为与我想象的不同.

```text
101 $ cargo build
Compiling httptest v0.2.0 (file:///opt/dev/httptest)
src/main.rs:30:36: 30:52 error: this function takes 1 parameter but 0 parameters were supplied [E0061]
src/main.rs:30         let payload = request.body.read_to_string();
                                                  ^~~~~~~~~~~~~~~~
```

好吧，现在`Read::read_to_string`的函数签名是`fn read_to_string(&mut self, buf: &mut String) -> Result<usize, Error>`，以此来提供缓冲区并处理错误。重写`set_greeting`方法.

```rust
    // 接收一个来自POST的信息，并使其返回
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

哦，Rust，终于好了。你太烦了.

# 4. 变一变

是的，我知道所有`.unwrap()`们很糟糕。但我不在乎。我们正在做原型.

在我们继续编写客户端之前，我想修改这个玩具示例，可在`/set` POST 上存储一些状态，以便以后再使用。我搞出一个本地`greeting`，在一些闭包中捕获它，然后看看编译器有什么抱怨的。

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

    // 接收一个来自POST的信息，并使其返回
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

rustc 很不高兴。我也想到了。但我只是抛出类型，得到个响应。天啊，快告诉我搞定 rustc 吧。

这些错误消息令人困惑，但显然闭包是个错误类型。这个`Router`的`get`和`post`方法拿一个[`Handler`](http://ironframework.io/doc/iron/middleware/trait.Handler.html)，且从 doc 页面中,我看到一个 impl 被定义为

```rust
pub trait Handler: Send + Sync + Any {
    fn handle(&self, &mut Request) -> IronResult<Response>;
}
```

真是一口`**`，但是`Handler`定义为`Fn`，而不是`FnOnce`或`FnMut`，且必须为`Send + Sync`。 因为它需要发送(send)，所以我们不会捕获任何引用，并且由于环境不是可变的，所以我们必须使用内部可变性来改变`greeting`。所以我要使用可发送的智能指针，`Arc`，为了使它可变，加个`Mutex`在里面。为此，我们需要像这样从标准库中导入两者:`use std::sync::{Mutex, Arc};`。 我还要移动(move)捕获`move |r| ...`避免通过引用捕获。

像这样更新我的代码，`MD`，还是会产生相同的错误消息。

```rust
    let greeting = Arc::new(Mutex::new(Greeting { msg: "Hello, World".to_string() }));
    let greeting_clone = greeting.clone();

    let mut router = Router::new();

    router.get("/", move |r| hello_world(r, &greeting.lock().unwrap()));
    router.post("/set", move |r| set_greeting(r, &mut greeting_clone.lock().unwrap()));
```

rustc 不喜欢我闭包的生命周期。为什么? 我不知道。我问 `#rust 的reem`，他是否知道该怎么做。

几个小时后,reem 说

```text
16:02 < reem> brson: Partially hint the type of the request art, rustc has trouble inferring HRTBs

16:02 < reem> brson: 请求(request) art 相关的部分类型, rustc 无法推断出 HRTBs
```

HRTB 意味着"更高级别的特征界限(higher-ranked trait bounds)",这也大概意味着"复杂的生命周期"。

我将这些相同的行，更改为碰触到`r: &mut Request`，最终一切归于平静……

```rust
    let greeting = Arc::new(Mutex::new(Greeting { msg: "Hello, World".to_string() }));
    let greeting_clone = greeting.clone();

    let mut router = Router::new();

    router.get("/", move |r: &mut Request| hello_world(r, &greeting.lock().unwrap()));
    router.post("/set", move |r: &mut Request| set_greeting(r, &mut greeting_clone.lock().unwrap()));
```

这似乎是 Rust 的推断器中的一个 bug。真蹩脚.

现在它再次构建，所以我们可以用 curl 进行测试了.

```text
$ curl http://localhost:3000
{"msg":"Hello, World"}
$ curl -X POST -d '{"msg":"Just trust the Rust"}' http://localhost:3000/set
$ curl http://localhost:3000
{"msg":"Just trust the Rust"}
```

现在，我们正在掌握'雷电'.

# 5. 客户端

我们有了一个小 JSON 服务器。现在让我们编写客户端。这次我们将直接使用[Hyper].

我把`hyper = "*"`加到`Cargo.toml`身上，然后创建`src/bin/client.rs`:

```rust
extern crate hyper;

fn main() { }
```

`src/bin/`下的源文件，由 Cargo 自动构建为可执行文件。运行`cargo build`后，执行文件在`target/debug/client`出现。好了，‘老天’赐福。现在找出 Hyper 吧。

拔下[Hyper 客户端的示例](http://hyperium.github.io/hyper/hyper/client/index.html)，我改了改这个片段，变成仅请求"/"，并打印 body:

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

但是，现在`cargo run`又又又不工作啦.

```text
$ cargo run
`cargo run` requires that a project only have one executable; use the `--bin` option to specify which one to run
```

我必须输入`cargo run --bin httptest`，启动服务器。这样做之后,运行`cargo run --bin client`,才看到:

```text
$ cargo run --bin client
Running `target/debug/client`
{"msg":"Hello, World"}
```

哦伙计,看，我是个 Rust 巫师`:P`。最后一件事,我想做的是，发出 POST 请求来设置消息。显然要做的是改`client.get`为[`client.post`](http://hyperium.github.io/hyper/hyper/client/struct.Client.html#method.post)。 这会返回一个[`RequestBuilder`](http://hyperium.github.io/hyper/hyper/client/struct.RequestBuilder.html)，因此,我正在寻找设置有效负载的构建器方法。[`body`](http://hyperium.github.io/hyper/hyper/client/struct.RequestBuilder.html#method.body)怎么样?

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

但是运行它是令人失望的

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

看来，是因为我不好的错误处理! 我没有将有效的 JSON 传递给`/set`路由。修正(fix) **body** 为 `.body(r#"{ "msg": "Just trust the Rust" }"#)`让客户端成功:

```text
$ cargo run --bin client
   Compiling httptest v0.2.0 (file:///opt/dev/httptest)
      Running `target/debug/client`
{"msg":"Just trust the Rust"}
```

就像这样，我们在 Rust 中创建了一个 Web 服务和客户端。看来未来不远了。用[Iron]和[Hyper]去建立一些东西吧。

[crater]: https://github.com/brson/taskcluster-crater
[rust]: https://github.com/brson/taskcluster-crater
[iron]: http://ironframework.io/
[hyper]: http://hyperium.github.io/hyper/hyper/index.html
[rustc-serialize]: https://crates.io/crates/rustc-serialize
[serde]: https://crates.io/crates/serde
[serde_macros]: https://crates.io/crates/serde_macros
