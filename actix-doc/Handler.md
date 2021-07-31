## Request Handlers

请求的处理函数是一个接受０个或多个的异步函数，参数可以从请求(例如实现了 FromRequest trait)中提取,并返回一个可以被转换为http响应的类型(实现了[Responder trait](https://docs.rs/actix-web/3/actix_web/trait.Responder.html)).

处理请求发生在两个阶段．首先处理对象被调用，返回了一个实现了*Responder*t trait的对象，然后在被返回的对象上调用`Respond_to()`将它转换为一个`HttpResponse`或者`Error`

actix-web为标准类型提供了`Responder`的实现，比如`&‘static str`,`String`等等．

查看 [*Responder documentation*](https://docs.rs/actix-web/3/actix_web/trait.Responder.html#foreign-impls)来获取完整的实现列表

下面是有效的处理函数例子

```rust
async fn index(_req: HttpRequest) -> &'static str {
    "Hello world!"
}
async fn index(_req: HttpRequest) -> String {
    "Hello world!".to_owned()
}
```

如果要返回值中有复杂的类型，可以将签名改为return `impl Responder`

```rust
async fn index(_req: HttpRequest) -> impl Responder {
    web::Bytes::from_static(b"Hello world!")
}

async fn index(req: HttpRequest) -> Box<Future<Item=HttpResponse, Error=Error>> {
    ...
}
```



## 返回自定义类型

为了直接从处理函数返回自定义类型，该类型需要实现`Responder` trait.

我们创建一个自定义类型并把它序列化为一个 `application/json`响应

```rust
use actix_web::{Error, HttpRequest, HttpResponse, Responder};
use futures::future::{ready, Ready};
use serde::Serialize;

#[derive(Serialize)]
struct MyObj {
    name: &'static str,
}

// Responder
impl Responder for MyObj {
    type Error = Error;
    type Future = Ready<Result<HttpResponse, Error>>;

    fn respond_to(self, _req: &HttpRequest) -> Self::Future {
        let body = serde_json::to_string(&self).unwrap();

        // Create response and set content type
        ready(Ok(HttpResponse::Ok()
            .content_type("application/json")
            .body(body)))
    }
}

async fn index() -> impl Responder {
    MyObj { name: "user" }
}
```



## 流式相应体

相应体可以被异步的生成.下面的例子中body必须实现`Stream<Item=Bytes, Error=Error>`

```rust
use actix_web::{get, web, App, Error, HttpResponse, HttpServer};
use futures::{future::ok, stream::once};

#[get("/stream")]
async fn stream() -> HttpResponse {
    let body = once(ok::<_, Error>(web::Bytes::from_static(b"test")));

    HttpResponse::Ok()
        .content_type("application/json")
        .streaming(body)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(stream))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```



## 返回不同的类型(Either)

有时，你需要返回不同类型的响应．例如，检查错误并返回错误，返回异步响应，活着需要两种不同类型的结果

下面的例子中，使用`Either`来将两种不同类型的响应合并为一个单独的类型．

```rust
use actix_web::{Either, Error, HttpResponse};

type RegisterResult = Either<HttpResponse, Result<&'static str, Error>>;

async fn index() -> RegisterResult {
    if is_a_variant() {
        // <- choose variant A
        Either::A(HttpResponse::BadRequest().body("Bad data"))
    } else {
        // <- variant B
        Either::B(Ok("Hello!"))
    }
}
```

