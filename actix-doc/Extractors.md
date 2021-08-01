

## 类型安全的信息提取器(Type-safe information extraction)

Actix-web 提供了叫做提取器的类型安全的请求信息获取方法(`impl FromRequet)．默认actix-web提供了几种提取器实现.

提取器可以由处理函数的声明参数访问．Actix-web每个处理函数支持12个提取器．声明的位置并不重要．

```rust
async fn index(path: web::Path<(String, String)>, json: web::Json<MyInfo>) -> impl Responder {
    let path = path.into_inner();
    format!("{} {} {} {}", path.0, path.1, json.id, json.username)
}
```



## Path

*Path*提供的信息可以又请求的路径提取．你可以从路径的变量部分反序列化．

例如，对于注册了`/users/{user_id}/{friend}`的路径的资源，`user_id`和`friend`两部分可以被反序列化．这些部分可以提取到一个元组(`tuple`)中比如(`Path<(u32, String)>`)或任何实现了*serde* crate中`Deserialize`　trait的结构

```rust
use actix_web::{get, web, Result};

/// extract path info from "/users/{user_id}/{friend}" url
/// {user_id} - deserializes to a u32
/// {friend} - deserializes to a String
#[get("/users/{user_id}/{friend}")] // <- define path parameters
async fn index(web::Path((user_id, friend)): web::Path<(u32, String)>) -> Result<String> {
    Ok(format!("Welcome {}, user_id {}!", friend, user_id))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

也可以提取路径信息到一个实现了*serde* crate中`Deserialize`trait的结构体．下面的例子与刚才使用元组的相等的实现．

```rust
use actix_web::{get, web, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    user_id: u32,
    friend: String,
}

/// extract path info using serde
#[get("/users/{user_id}/{friend}")] // <- define path parameters
async fn index(info: web::Path<Info>) -> Result<String> {
    Ok(format!(
        "Welcome {}, user_id {}!",
        info.friend, info.user_id
    ))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```



## Query

`Query` 类型提供了从请求query参数的功能．底层他使用了`serde_urlencoded` crate

```rust
use actix_web::{get, web, App, HttpServer};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

// this handler gets called if the query deserializes into `Info` successfully
// otherwise a 400 Bad Request error response is returned
#[get("/")]
async fn index(info: web::Query<Info>) -> String {
    format!("Welcome {}!", info.username)
}
```



## Json

`Json`允许将请求body反序列化到结构题的功能，为了从请求body中提取类型信息，Type T必须实现*serde* crate中`Deserialize`trait．

```rust
use actix_web::{get, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body
#[get("/")]
async fn index(info: web::Json<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}
```

一些提取器提供了方法来配置提前过程．为了配置提取器，将他的配置对象传递给资源的data方法．如果是json提取器，他返回一个`JsonConfig`可以配置json的有效负载的最大大小和自定义错误处理函数．

下面的例子限制负载的大小为4kb并使用了一个自定义错误处理函数．

```rust
use actix_web::{error, web, App, HttpResponse, HttpServer, Responder};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body, max payload size is 4kb
async fn index(info: web::Json<Info>) -> impl Responder {
    format!("Welcome {}!", info.username)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let json_config = web::JsonConfig::default()
            .limit(4096)
            .error_handler(|err, _req| {
                // create custom error response
                error::InternalError::from_response(err, HttpResponse::Conflict().finish()).into()
            });

        App::new().service(
            web::resource("/")
                // change json extractor configuration
                .app_data(json_config)
                .route(web::post().to(index)),
        )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```



Form 

现在暂时只支持url-encoded编码的表单．url-encoded编码的类型可以被提取到特定的类型．该类型必须实现*serde* crate中`Deserialize`trait．

`FormConfig`允许配置提取过程．

```rust
use actix_web::{post, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct FormData {
    username: String,
}

/// extract form data using serde
/// this handler gets called only if the content type is *x-www-form-urlencoded*
/// and the content of the request could be deserialized to a `FormData` struct
#[post("/")]
async fn index(form: web::Form<FormData>) -> Result<String> {
    Ok(format!("Welcome {}!", form.username))
}
```



## Other

Actix-web提供了几种其他提取器．

- `Data`-如果你需要获取app状态
- *HttpRequest*-HttpRequest本身也是一个返回self的提取器，如果你需要获取请求．
- *String*-可以把请求的负载转化为String.  [*例子*](https://docs.rs/actix-web/3/actix_web/trait.FromRequest.html#example-2) 可以在文档中查阅．
- *actix-web::web::Bytes*-可以将请求的payload转化为Bytes.[*Example*](https://docs.rs/actix-web/3/actix_web/trait.FromRequest.html#example-4) 可以查看文档．
- Payload-可以访问请求的payload..[*Example*](https://docs.rs/actix-web/3/actix_web/web/struct.Payload.html)



## 应用状态提取器

应用的状态可以从处理函数中的`web::Data`提取器访问．但是状态只能作为只读引用访问．如果需要可变的方法访问状态，它(可变的方法)必须被实现．

谨记，actix创建应用状态和处理函数的多个副本．每个线程创建一份副本．

下面是一个例子，它存储了处理过的请求的数量

```rust
use actix_web::{web, Responder};
use std::cell::Cell;

#[derive(Clone)]
struct AppState {
    count: Cell<usize>,
}

async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!("count: {}", data.count.get())
}

async fn add_one(data: web::Data<AppState>) -> impl Responder {
    let count = data.count.get();
    data.count.set(count + 1);

    format!("count: {}", data.count.get())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    let data = AppState {
        count: Cell::new(0),
    };

    HttpServer::new(move || {
        App::new()
            .data(data.clone())
            .route("/", web::to(show_count))
            .route("/add", web::to(add_one))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

尽管这个处理函数可以工作，`data.count`只会记录每个线程的处理的请求数．

为了记录所有线程的总请求数，必须使用Arc和atomic

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use std::cell::Cell;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    local_count: Cell<usize>,
    global_count: Arc<AtomicUsize>,
}

#[get("/")]
async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[get("/add")]
async fn add_one(data: web::Data<AppState>) -> impl Responder {
    data.global_count.fetch_add(1, Ordering::Relaxed);

    let local_count = data.local_count.get();
    data.local_count.set(local_count + 1);

    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = AppState {
        local_count: Cell::new(0),
        global_count: Arc::new(AtomicUsize::new(0)),
    };

    HttpServer::new(move || {
        App::new()
            .data(data.clone())
            .service(show_count)
            .service(add_one)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

**注意**，如果你想要整个状态在线程间共享状态，按之前[共享可变状态](https://actix.rs/docs/application#shared-mutable-state)部分中描述的使用`Web::Data`和`app_data`.

小心像Mutex或Rwlock之类的同步原语．`actix-web`框架异步的处理请求．通过阻塞线程执行，所有的并发请求都会阻塞．如果在多线程间需要分享或更新状态，考虑使用tokio中的同步原语．