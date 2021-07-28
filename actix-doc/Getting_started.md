## 安装Rust

如果你还没有安装rust, 推荐你使用rustup来管理rust的安装．[rust官方指南](https://doc.rust-lang.org/book/ch01-01-installation.html)上有很棒的上手教程．

actix Web 现在支持rust 1.42及以上版本．运行`rustup update`确保你使用最新最好的rust版本．因此这篇教程假设你在使用`Rust`1.42及之后的版本．

## Hello, world!

Cargo创建一个二进制项目，进入新的文件夹

```shell
cargo new hello-world
cd hello-world
```

将`actix-web`作为依赖加入你项目中的`Cargo.toml`文件

```toml
[dependencies]
actix-web = "3"
```

Request handler 使用异步函数接收0个或多个参数．这些参数从请求(见`FromRequest trait`)中读取并返回一个能被转换为`HttpResponse`的类型(见`Responder trait`)

```rust
use actix_web::{get, post, web, App, HttpResponse, HttpServer, Responder};

#[get("/")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().body("Hello world!")
}

#[post("/echo")]
async fn echo(req_body: String) -> impl Responder {
    HttpResponse::Ok().body(req_body)
}

async fn manual_hello() -> impl Responder {
    HttpResponse::Ok().body("Hey there!")
}
```

注意到上面的一些handler直接使用内建宏附加路由信息.这允许你指明路径和方法响应的请求．之后你会看到如何不使用宏注册其他路由．

接着，创建`App`实例并注册`request hanlders`．使用`App::Service`对使用宏注册的路由声明方法和路径，使用`App::route()`手动注册路由．最终,App在一个`httpServer`中启动，服务器将作为一个应用工厂(Application Fatory)相应即将到来的请求．

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(hello)
            .service(echo)
            .route("/hey", web::get().to(manual_hello))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

完成了！使用`cargo run`编译运行．`#[actix_web::main]`宏在actix运行时内执行异步的main函数.现在你可以使用`http://127.0.0.1:8080/”或者其他你定义的路由查看结果．