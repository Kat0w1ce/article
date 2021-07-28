## The Http Server

`HttpServer`类型负责服务HTTP请求．

`HttpServer`接受app工厂函数作为参数，工厂函数必须必须是`Send`+`Sync`的．更多相关的内容参阅多线程部分．

`bind()`必须被多次调用来绑定特定的socket地址，他可能被调用多次．绑定ssl socket时必须使用`bind_openssl()`和`bind_rustls()`. 使用 `HttpServer::run()`方法来运行HTTP服务器．

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().route("/", web::get().to(|| HttpResponse::Ok()))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```



`run()`方法返回`Server`类型的实例，server类型的方法可以用来管理`HTTP`服务器

- `pause()` - 暂停接受将要到来的连接
- `resume()`-恢复接受将要到来的连接
- ` stop()`  停止处理到来的连接，停止所有worker并退出

下面的例子展示了如何在一个分离的线程里启动HTTP服务

```rust
use actix_web::{rt::System, web, App, HttpResponse, HttpServer};
use std::sync::mpsc;
use std::thread;

#[actix_web::main]
async fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let sys = System::new("http-server");

        let srv = HttpServer::new(|| {
            App::new().route("/", web::get().to(|| HttpResponse::Ok()))
        })
        .bind("127.0.0.1:8080")?
        .shutdown_timeout(60) // <- Set shutdown timeout to 60 seconds
        .run();

        let _ = tx.send(srv);
        sys.run()
    });

    let srv = rx.recv().unwrap();

    // pause accepting new connections
    srv.pause().await;
    // resume accepting new connections
    srv.resume().await;
    // stop server
    srv.stop(true).await;
}

```



## Multi-threading

`HttpServer` 自动地启动一些Http `workers`,默认worker数量与系统中逻辑CPU数量相同．这个数量可以用`HttpServer::workers()`方法重载．



```rust
use actix_web::{web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() {
    HttpServer::new(|| {
        App::new().route("/", web::get().to(|| HttpResponse::Ok()))
    })
    .workers(4); // <- Start 4 workers
}
```

workers一旦被创建后，各自接受一个不同的app实例来处理请求．应用状态各个线程间不共享．handlers可以自由的操作他们状态的复制而不用担心并发问题．

应用状态不需要是`Send`或`Sync`的，但应用工厂必须是`Send`+`Sync`．

使用`Arc`在不同线程分享状态.引入共享和同步时要特别小心．许多场景下，为了修改共享状态上锁会造成不经意的性能损失．

一些情况下，可以使用更高效的上锁测率减轻损失．使用读写锁而不是`mutex`来达到非排他性上锁，但最好的性能往往出现在完全没有上锁操作时．

因为每个工作线程有序的处理请求，会阻塞的handler将导致工作线程停止处理新的请求．

```rust
fn my_handler() -> impl Responder {
    std::thread::sleep(Duration::from_secs(5)); // <-- Bad practice! Will cause the current worker thread to hang!
    "response"
}
```

因此，任何非CPU负载的操作(比如I.O，数据库操作)应当是future或者异步函数．异步handler被工作线程并发的执行而不会阻塞执行．

```rust
async fn my_handler() -> impl Responder {
    tokio::time::delay_for(Duration::from_secs(5)).await; // <-- Ok. Worker thread will handle other requests here
    "response"
}
```

提取器也有同样的限制．当一个处理函数接收带有实现了`fromRequest`的声明，并且该实现阻塞了当前线程，工作线程在运行处理函数时也会阻塞．因此实现提取器时要特别注意，它们也需要被异步的实现．



## SSL

ssl服务器需要`rustls`和`openssl`两个特性．`rustls`特性用于集成`rustls`,`openssl`同理．

```toml
[dependencies]
actix-web = { version = "3", features = ["openssl"] }
openssl = { version = "0.10" }
```

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Responder};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

#[get("/")]
async fn index(_req: HttpRequest) -> impl Responder {
    "Welcome!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // load ssl keys
    // to create a self-signed temporary cert for testing:
    // `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'`
    let mut builder =
        SslAcceptor::mozilla_intermediate(SslMethod::tls()).unwrap();
    builder
        .set_private_key_file("key.pem", SslFiletype::PEM)
        .unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| App::new().service(index))
        .bind_openssl("127.0.0.1:8080", builder)?
        .run()
        .await
}
```

**注意:** *HTTP/2.0*协议需要 [tls alpn](https://actix.rs/docs/server/#:~:text=2.0%20protocol%20requires-,tls%20alpn,-.%20At%20the%20moment). 现在只有`openssl`有`tlpn`支持．具体例子请查看 [examle/openssl](https://github.com/actix/examples/tree/master/security/openssl).

使用以下指令创建key.pem和cert.pem，填入自己的项目中．



```shell
$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -sha256 -subj "/C=CN/ST=Fujian/L=Xiamen/O=TVlinux/OU=Org/CN=muro.lxd"
```

移除密码然后复制 nopass.pem到key.pem

```shell
$ openssl rsa -in key.pem -out nopass.pem
```



## Keep-Alive

Actix 可以在　keep-alive的连接上等待请求

*keep alive*连接行为由服务器定义

- `75`, `Some(75)`, `KeepAlive::Timeout(75)` - 设置75秒keep-alive定时器.
- `None` or `KeepAlive::Disabled` - 取消keep alive
- `KeepAlive::Tcp(75)` - 使用 SO_KEEPALIVE 套接自选项．

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let one = HttpServer::new(|| {
        App::new().route("/", web::get().to(|| HttpResponse::Ok()))
    })
    .keep_alive(75); // <- Set keep-alive to 75 seconds

    // let _two = HttpServer::new(|| {
    //     App::new().route("/", web::get().to(|| HttpResponse::Ok()))
    // })
    // .keep_alive(); // <- Use `SO_KEEPALIVE` socket option.

    let _three = HttpServer::new(|| {
        App::new().route("/", web::get().to(|| HttpResponse::Ok()))
    })
    .keep_alive(None); // <- Disable keep-alive

    one.bind("127.0.0.1:8080")?.run().await
}
```

如果选择了上述第一个选项，*keep alive*状态基于响应的*connection-type*计算．默认`HttpResponse::Connection_type`没有定义，这种情况下,*keep alive*由请求类型定义．

*keep alvie*在*HTTP*/1.0上是关闭的,在*HTTP/1.1*和*HTTP/2.0*上默认开启．

*Connection type*可以用`HttpResponseBuilder::Connection_type()`方法改变

```rust
use actix_web::{http, HttpRequest, HttpResponse};

async fn index(req: HttpRequest) -> HttpResponse {
    HttpResponse::Ok()
        .connection_type(http::ConnectionType::Close) // <- Close connection
        .force_close() // <- Alternative method
        .finish()
}
```





## Graceful shutdown

`HttpServer`支持优雅的停机．收到停止信号后，workres有一定时间来停止处理请求．超时和强制停止后工作者仍然存活．默认超时时间为30秒，你可以使用`HttpServer::shutdown_timeout()`方法改变参数．

`HttpServer`处理多种系统信号，CLRT-C在所有系统上都可用，其他信号在unix系统上可用

- SIGINT - Force shutdown workers
- SIGTERM - Graceful shutdown workers
- SIGQUIT - Force shutdown workers

 可以通过 `Http::disable_signals()`方法禁用信号．