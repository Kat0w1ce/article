## Errors

Actix-web 使用自己的[`actix_web::error::Error`](https://docs.rs/actix-web/3/actix_web/error/struct.Error.html)类型和 [`actix_web::error::ResponseError`](https://docs.rs/actix-web/3/actix_web/error/trait.ResponseError.html)trait来处理响应函数的错误

如果一个处理函数返回一个Result中的`Error`（指一般的Rust trait [`std::error::Error`](https://doc.rust-lang.org/std/error/trait.Error.html)]，比且也实现了`ResponseError`trait,actix-web将会将http响应渲染为对应的[`actix_web::http::StatusCode`](https://docs.rs/actix-web/3.0.0/actix_web/http/struct.StatusCode.html)，默认生成的服务器内部错误如下:

```rust
pub trait ResponseError {
    fn error_response(&self) -> Response<Body>;
    fn status_code(&self) -> StatusCode;
}
```

`Responder`强制把兼容的`Result`转换为 HTTP响应

```rust
impl<T: Responder, E: Into<Error>> Responder for Result<T, E>
```

代码中的Error`高于Actix-web定义的错误，任何实现了`ResponseError`的错误可以被自动转换为Error

Actix-web提供实现了一些非actix的错误的`ResponseError`.举例来说,如果处理函数相应了一个`io::Error`,这个错误被转化为`HttpInternalServerError` 

```rust
use std::io;
use actix_files::NamedFile;

fn index(_req: HttpRequest) -> io::Result<NamedFile> {
    Ok(NamedFile::open("static/index.html")?)
}
```

`ResponseError`所有外部实现请查阅[api文档](https://docs.rs/actix-web/3/actix_web/error/trait.ResponseError.html#foreign-impls)



## 返回自定义类型错误的例子

下面是一个实现`RespondError`的例子，使用 [derive_more](https://crates.io/crates/derive_more) crate来继承错误枚举

```rust
use actix_web::{error, Result};
use derive_more::{Display, Error};

#[derive(Debug, Display, Error)]
#[display(fmt = "my error: {}", name)]
struct MyError {
    name: &'static str,
}

// Use default implementation for `error_response()` method
impl error::ResponseError for MyError {}

async fn index() -> Result<&'static str, MyError> {
    Err(MyError { name: "test" })
}
```

`ResponseError`有一个将会成为500(服务器内部错误)的`error_response()`的默认实现．上面的index函数执行时将会发生这种情况．

重载`error_response`将会提供更多有用的结果

```rust
use actix_web::{
    dev::HttpResponseBuilder, error, get, http::header, http::StatusCode, App, HttpResponse,
};
use derive_more::{Display, Error};

#[derive(Debug, Display, Error)]
enum MyError {
    #[display(fmt = "internal error")]
    InternalError,

    #[display(fmt = "bad request")]
    BadClientData,

    #[display(fmt = "timeout")]
    Timeout,
}

impl error::ResponseError for MyError {
    fn error_response(&self) -> HttpResponse {
        HttpResponseBuilder::new(self.status_code())
            .set_header(header::CONTENT_TYPE, "text/html; charset=utf-8")
            .body(self.to_string())
    }

    fn status_code(&self) -> StatusCode {
        match *self {
            MyError::InternalError => StatusCode::INTERNAL_SERVER_ERROR,
            MyError::BadClientData => StatusCode::BAD_REQUEST,
            MyError::Timeout => StatusCode::GATEWAY_TIMEOUT,
        }
    }
}

#[get("/")]
async fn index() -> Result<&'static str, MyError> {
    Err(MyError::BadClientData)
}
```



## Error helpers

Actix-web提供了一系列错误帮助函数，它们生成对于从其他错误上生成特定的HTTP错误码很有用．

下面是使用`map_err`将没有实现`ResponseError` trait的`MyError`转化为400类型的例子

```rust
use actix_web::{error, get, App, HttpServer, Result};

#[derive(Debug)]
struct MyError {
    name: &'static str,
}

#[get("/")]
async fn index() -> Result<&'static str> {
    let result: Result<&'static str, MyError> = Err(MyError { name: "test error" });

    Ok(result.map_err(|e| error::ErrorBadRequest(e.name))?)
}
```

可用的error helpers列表请查阅 [API documentation for actix-web’s `error` module](https://docs.rs/actix-web/3/actix_web/error/struct.Error.html) ．



## Error logging

Actixs 记录了所有`WARN`级别错误的日志，如果应用的日志等级被设置为`DEBUG`并且`RUST_BACKTRACE`可用., backtrace也会被记录．下面是环境变量的配置

```shell
>> RUST_BACKTRACE=1 RUST_LOG=actix_web=debug cargo run
```

如果可用`Error`类型使用使用诱因的error backtrace .　如果潜在的错误没有提供backtrace,新的backtrace将会被构建并指向转换发生的地方(而不是错误的源头)

## Recommended practices in error handling

考虑将app产生的错误分为两类是很有用处的，一些是面向用户的，一些则不是．

先前的例子中，我可能使用指定了`UserError`枚举的错误，该枚举封装了`ValidationError`当用户发送了非法的输入．

```rust
use actix_web::{
    dev::HttpResponseBuilder, error, get, http::header, http::StatusCode, App, HttpResponse,
    HttpServer,
};
use derive_more::{Display, Error};

#[derive(Debug, Display, Error)]
enum UserError {
    #[display(fmt = "Validation error on field: {}", field)]
    ValidationError { field: String },
}

impl error::ResponseError for UserError {
    fn error_response(&self) -> HttpResponse {
        HttpResponseBuilder::new(self.status_code())
            .set_header(header::CONTENT_TYPE, "text/html; charset=utf-8")
            .body(self.to_string())
    }
    fn status_code(&self) -> StatusCode {
        match *self {
            UserError::ValidationError { .. } => StatusCode::BAD_REQUEST,
        }
    }
}
```

这将会完全符合预期，因为带有`display`定义的错误信息明确想要被用户可读．

但是，发回的错误信息不能描述所有错误，很多错误可能发生在服务器环境，我们可能想要这些细节对用户隐藏．举个例子，如果数据库挂了，client库开始产生连接超时错误，或者html模板没有被合适的渲染，在渲染时发生错误．这些情况下，可能更适宜将错误映射为一个通用的错误，它是适用于被用户消费．

下面是使用自定义消息将内部错误映射为面向用户的`InternalError`的例子

```rust
use actix_web::{
    dev::HttpResponseBuilder, error, get, http::header, http::StatusCode, App, HttpResponse,
    HttpServer,
};
use derive_more::{Display, Error};

#[derive(Debug, Display, Error)]
enum UserError {
    #[display(fmt = "An internal error occurred. Please try again later.")]
    InternalError,
}

impl error::ResponseError for UserError {
    fn error_response(&self) -> HttpResponse {
        HttpResponseBuilder::new(self.status_code())
            .set_header(header::CONTENT_TYPE, "text/html; charset=utf-8")
            .body(self.to_string())
    }
    fn status_code(&self) -> StatusCode {
        match *self {
            UserError::InternalError => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}

#[get("/")]
async fn index() -> Result<&'static str, UserError> {
    do_thing_that_fails().map_err(|_e| UserError::InternalError)?;
    Ok("success!")
}
```

通过区分错误是否面向用户，我们可以确保不会意外将用户不想看见的app内部错误暴露给用户．



## Error Logging

下面是一个基础的例子，他使用了依赖的`env_logger`和`log`的`Middleware::log`

```toml
[dependencies]
env_logger = "0.8"
log = "0.4"
```



```rust
use actix_web::{error, get, middleware::Logger, App, HttpServer, Result};
use derive_more::{Display, Error};
use log::info;

#[derive(Debug, Display, Error)]
#[display(fmt = "my error: {}", name)]
pub struct MyError {
    name: &'static str,
}

// Use default implementation for `error_response()` method
impl error::ResponseError for MyError {}

#[get("/")]
async fn index() -> Result<&'static str, MyError> {
    let err = MyError { name: "test error" };
    info!("{}", err);
    Err(err)
}

#[rustfmt::skip]
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "info");
    std::env::set_var("RUST_BACKTRACE", "1");
    env_logger::init();

    HttpServer::new(|| {
        let logger = Logger::default();

        App::new()
            .wrap(logger)
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

