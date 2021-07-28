`actix-web`提供了多种用rust构建web服务器和应用的原语，包括路由，中间件，http请求预处理和http响应的后期处理等等.

所有的`acitx-web`服务器都围绕着`App`实例构建．它被用于为资源和中间件注册路由，也存放了同一个域下的handlers共享的应用状态．

一个应用的`scope`作为所有路由的命名空间，例如一个具体应用scope的所有路由有同样的前缀．应用的前缀总是有前置的“/”．‘/‘将会自动插入到路由起始位置, 如果被提供的前缀不包含起始的”/“．前缀必须由值路径段(value path segments ?)组成 .

对于有`/app`域的应用来说，所有带有`/app`,`/app/`,或者`/app/test`路径的请求将会被匹配; 但路径｀/application`不会匹配．

```rust
use actix_web::{web, App, HttpServer, Responder};

async fn index() -> impl Responder {
    "Hello world!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            // prefixes all resources and routes attached to it...
            web::scope("/app")
                // ...so this handles requests for `GET /app/index.html`
                .route("/index.html", web::get().to(index)),
        )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

在这个例子里，创建了一个带有`/app`前缀和`index.html`资源的应用. 可以通过｀/app/index.html`访问资源．

更多的信息请查看URL Dispatch部分．

## State

同一个域内的资源和路由分享应用状态．状态可以使用`web::Data<T>`提取器获得，T是状态类型．状态也可以被中间件获得．

让我们写一个简单应用并把应用名存储到状态中．

```rust
use actix_web::{get, web, App, HttpServer};

// This struct represents state
struct AppState {
    app_name: String,
}

#[get("/")]
async fn index(data: web::Data<AppState>) -> String {
    let app_name = &data.app_name; // <- get app_name

    format!("Hello {}!", app_name) // <- response with app_name
}
```

在应用初始化时传递状态，启动应用

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .data(AppState {
                app_name: String::from("Actix-web"),
            })
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```



## Shared Muteble State

`HttpServer`接受一个应用工厂而不是应用实例．`HttpServer`为每个线程构造一个应用实例．因此应用必须被构造多次．如果你想在不同的线程中共享数据，必须使用可变共享对象，例如`Send`+`Sync`.

`web::Data`在内部使用`Arc`．因此为了避免创建两个`Arc`,我们必须在注册前使用`App::app_data()`使用创建它．

在下面的例子中，我们将写一个有共享可变状态的应用．首先，定义我们的状态并创建handler

```rust
use actix_web::{web, App, HttpServer};
use std::sync::Mutex;

struct AppStateWithCounter {
    counter: Mutex<i32>, // <- Mutex is necessary to mutate safely across threads
}

async fn index(data: web::Data<AppStateWithCounter>) -> String {
    let mut counter = data.counter.lock().unwrap(); // <- get counter's MutexGuard
    *counter += 1; // <- access counter inside MutexGuard

    format!("Request number: {}", counter) // <- response with count
}
```

然后把数据注册到`App`

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let counter = web::Data::new(AppStateWithCounter {
        counter: Mutex::new(0),
    });

    HttpServer::new(move || {
        // move counter into the closure
        App::new()
            // Note: using app_data instead of data
            .app_data(counter.clone()) // <- register the created data
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```



## Using an Application Scope to Compose Applications

`web::scope()`方法允许设置资源组前缀．这个域代表了一个资源前缀，它将会被前置到所有由resource configuration添加的资源模式．这能被用于将一系列路由安装在与原作者本意不同的位置，同时保持下共同的资源名．

例如

```rust
#[actix_web::main]
async fn main() {
    let scope = web::scope("/users").service(show_users);
    App::new().service(scope);
}

```

在上面的例子中，`show_users`路由有一个`/users/show`的有效路由模式而不是`show`,因为这个应用的域声明将被放到模式之前．路由只会匹配`/users/show`的URL路径.当`HttpRequest.url_for()`函数被带有`show_user`路由路径调用时, 将会产生有相同路径的URL.

## Application guards and virtual hosting

你可以把guard当作一个接受请求对象的引用并返回true或者false的简单函数. 正式的来说，guard可以是任何实现了`Guard`trait的对象. Actix-web 提供了一些guards.　你可以在查看API文档的[函数部分](https://docs.rs/actix-web/3/actix_web/guard/index.html#functions)．

`Header`是一个被提供的guard．它可被用做基于请求头信息的过滤器.

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(
                web::scope("/")
                    .guard(guard::Header("Host", "www.rust-lang.org"))
                    .route("", web::to(|| HttpResponse::Ok().body("www"))),
            )
            .service(
                web::scope("/")
                    .guard(guard::Header("Host", "users.rust-lang.org"))
                    .route("", web::to(|| HttpResponse::Ok().body("user"))),
            )
            .route("/", web::to(|| HttpResponse::Ok()))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```



## Configure

为了简单和可复用性，`App`和`web::Scope`提供了`Configure`方法.这个函数在迁移一部分配置到不同的模块和库时很有用．例如，一些资源配置可以被移动到不同的模块．

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

// this function could be located in a different module
fn scoped_config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/test")
            .route(web::get().to(|| HttpResponse::Ok().body("test")))
            .route(web::head().to(|| HttpResponse::MethodNotAllowed())),
    );
}

// this function could be located in a different module
fn config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/app")
            .route(web::get().to(|| HttpResponse::Ok().body("app")))
            .route(web::head().to(|| HttpResponse::MethodNotAllowed())),
    );
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .configure(config)
            .service(web::scope("/api").configure(scoped_config))
            .route("/", web::get().to(|| HttpResponse::Ok().body("/")))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

```

上面例子的结果是

```
/         -> "/"
/app      -> "app"
/api/test -> "test"
```

###### 每个`ServiceConfig`可以有属于自己的`data`,`routes`和`services`