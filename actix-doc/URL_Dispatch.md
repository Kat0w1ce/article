## URL Dispatch

URL分发使用简单的模式匹配语言提供了将URL映射到处理代码的简单方法．如果一个模式匹配到了一个与路径信息相关的请求，特定的函数处理对象就会被调用．

请求的处理函数是一个接受０个或多个的函数，参数可以从请求(例如实现了 FromRequest trait)中提取,并返回一个可以被转换为http响应的类型(实现了[Responder trait](https://docs.rs/actix-web/3/actix_web/trait.Responder.html)).详情请查看[handler](https://actix.rs/docs/handlers/)部分．



## Resource configuration

资源配置是将新资源加入到应用的行为．资源有一个作为url生成标识的名字．这个名字也允许开发者向已经存在的资源加入路由．资源爷爷一个为了批评url 路径部分的模式(该部分在schema和端口之后,例如  http://localhost:8080/foo/bar?q=value中的*/foo/bar*).他不匹配*query*部分(该部分在?之后，比如 *q=value* in *http://localhost:8080/foo/bar?q=value*)

 [*App::route()*](https://docs.rs/actix-web/3/actix_web/struct.App.html#method.route) 方法提供了注册路由的简单方式．这个方法向应用路由表加入一个简单路由．这个方法接受一个路径模式,*HTTP*方法和一个处理函数．`route()`方法可以向同一路径调用多次，这种情况下，多个路径注册到了单个资源路径

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Hello")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(index))
            .route("/user", web::post().to(index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

当*App::route()*提供了简单的方法 来获取完整的资源的配置，使用了一个不同的方法． [*App::service()*](https://docs.rs/actix-web/3/actix_web/struct.App.html?search=#method.service) 方法想路由表中加入单个[resource](https://docs.rs/actix-web/3/actix_web/struct.Resource.html).这个方法接受一个路径模式，守卫和更多的路径．

```rust
use actix_web::{guard, web, App, HttpResponse};

fn index() -> HttpResponse {
    HttpResponse::Ok().body("Hello")
}

pub fn main() {
    App::new()
        .service(web::resource("/prefix").to(index))
        .service(
            web::resource("/user/{name}")
                .name("user_detail")
                .guard(guard::Header("content-type", "application/json"))
                .route(web::get().to(|| HttpResponse::Ok()))
                .route(web::put().to(|| HttpResponse::Ok())),
        );
}
```

如果一个资源没有包含路径或者没有匹配的路径，将会回复一个*NOT FOUND*　HTTP相应.



## 配置路由

资源包含了一系列路径的集合．一个路径反过来有一系列`guards`和一个处理函数．新的路由可以由`Resource::route()`方法创建，它返回一个*Route*实例的引用．默认*route*不包含任何守卫，所以匹配所有的请求，默认处理函数是`HttpNotFound`

应用为到来的请求提供计入在注册资源和路由时定义的路由规则．资源按路由经过`Resource::route()`注册的顺序匹配所有路由．

一个路径可以包涵任意数量的守卫和唯一的处理函数

```rust
App::new().service(
    web::resource("/path").route(
        web::route()
            .guard(guard::Get())
            .guard(guard::Header("content-type", "text/plain"))
            .to(|| HttpResponse::Ok()),
    ),
)
```

例子中，`HttpResponse::OK()`返回给包含`Content-Type`头的*Get*请求，这个头的类型是*text/palin*路径等同于`/path`

如果资源没有匹配任何路径，“NOT FOUND”`响应被返回．

[*ResourceHandler::route()*](https://docs.rs/actix-web/3/actix_web/struct.Resource.html#method.route) 返回一个 [*Route*](https://docs.rs/actix-web/3/actix_web/struct.Route.html)对象．路径可以由一个builder-like模式配置．下面是考验的配置方法:

- [*Route::guard()*](https://docs.rs/actix-web/3/actix_web/struct.Route.html#method.guard) 注册一个新的守卫，任何数量的守卫注册给每个路径.
- [*Route::method()*](https://docs.rs/actix-web/3/actix_web/struct.Route.html#method.method) 注册一个方法守卫，任何数量的守卫可以注册每个路径.
- [*Route::to()*](https://docs.rs/actix-web/3/actix_web/struct.Route.html#method.to) 为这个路由注册一个一部的处理函数，只有一个处理函数可以被注册，通常处理函数是配置的最后一个操作．

## 路由匹配

配置路由的主要目的是匹配(或不匹配)URL的路径模式．`path`代表URL被请求的路径部分．

*actix-web*实现这样的方法非常简单．当一个请求到达系统，对于出现在系统中的每个资源配置声明，acitx对于请求的路径检查声明的模式．按通过`App::service()`声明的路径顺序检查．如果资源没有被找到，默认的资源用来批评资源．

当一个路径配置被声明，它可能包含路由守卫生命．检查中，对于被给予的请求的路由配置，与路径相关的所有路由守卫必须为真．如果检查中提供给路由配置的路由守卫集合中任何一个守卫返回`false`，这个路径会被跳过，路由继续按路径集合的顺序批评．

如果任何路径批评，路由匹配过程停止，与路径相关的处理函数被唤醒．如果所有路径模式耗尽后没有路由批评，*NOT FOUND*响应被返回．



## 资源模式语法

axtix在模式论证中使用的模式匹配语言语法是直接了当的．

路径配置中的模式可能由一个‘/’开始,如果没有由’/‘开始，在匹配时会前置一个隐式的‘/’，例如下面两个模式是相等的

```
{foo}/bar/baz
```

和

```
/{foo}/bar/baz
```

变量部分有{identifier}形式指定，意味着接受任意字符直到下一个‘/’，并在`HttpRequest.match_info()`对象中作为名字使用它．

模式匹配中的替代符号符合正则表达式`[^{}/]+`．

`Params`对象中的match_info代表了从基于路径模式的URL中提取的动态部分．它作为 *request.match_info*可用．例如，下面的模式定义了一个字面段(foo)和两个可变符合(baz 和 bar)

```
foo/{baz}/{bar}
```

上面的模式可以被下面的url匹配，并生成匹配信息

```
foo/1/2        -> Params {'baz':'1', 'bar':'2'}
foo/abc/def    -> Params {'baz':'abc', 'bar':'def'}
```

但它不会匹配下面的模式

```
foo/1/2/        -> No match (trailing slash)
bar/abc/def     -> First segment literal mismatch
```

一个部分中可变部分的匹配直到模式段中第一个非字母数字的字符结束．例如如果使用下面这个路由模式

```
foo/{name}.html
```

字面路径 */foo/blz.html*将会匹配上面的字符模式，匹配结果将会是 `Params{'name': 'biz'}`，但字面路径“/foo/blz”不会匹配．因为他不包含段由*{name}.html*最后的文字 *.html*(它只包含 blz 不是 blz.html)

为了捕获两个段，必须使用两个替身符号

```
foo/{name}.{ext}
```

字面路径 */foo/blz.html*将会匹配上面的路径模式，匹配结果为参数Params *{name’: ‘biz’, ‘ext’: ‘html’}*,因为两个替身符号 *{name}*和*{ext}*之间有一个句号

替身符合可以可选的制定一个正则表达式来决定路径段是否匹配符号．为了指定匹配如正则表达式定义的特定字符集合的替身符号，必须使用替身符合的小小的拓展形式语法．在大括号内替身符号名字必须紧跟一个冒号，之后直接是一个正则表达式．默认的正则表达式与替身符号 *[^/]+*相关联，匹配一个或多个不为 /的字符．例如，下面的占位符 *{foo}*可以被更详细的拼为*{foo:^/+}* .可以把这个改为任意的正则表达式来匹配任意的字符序列，比如 *{foo:\d+}* 只匹配数字．

为了匹配一段占位符，片段必须包括至少一个字符．例如对与URL /abc/

- */abc/{foo}*不匹配
- */{foo}/*会批评

**注意:**路径是不带符合号的URL并在模式匹配前被解码为有效的unicode字符串，代表匹配的路径的值也是无符号的URL

对于下面的模式

```
foo/{bar}
```

当匹配如下的URL时

```
http://example.com/foo/La%20Pe%C3%B1a
```

匹配的文件夹看起来像下面这样(解码的URL值)

```
Params{'bar': 'La Pe\xf1a'}
```

路径段中的字符串必须代表提供给actix解码后的路径．你不会想在模式中使用一个URL编码的值.例如，相比这个

```
/Foo%20Bar/{baz}
```

你更想使用像这样的

```
/Foo Bar/{baz}
```

尾匹配是可能的．为了实现这个可以使用自定义的正则

```
foo/{bar}/{tail:.*}
```

上面的模式可以批评这些URL,生成如下的匹配信息．

```
foo/1/2/           -> Params{'bar':'1', 'tail': '2/'}
foo/abc/def/a/b/c  -> Params{'bar':u'abc', 'tail': 'def/a/b/c'}
```



## 块路由

Scope帮助你组织共享根的路径，你可以在块内嵌套块．假设你想组织用于查看“Users”的端点路径，比如下面的这些路径

- /users
- /users/show
- /users/show/{id}

一个块布局会如下所示

```rust
#[get("/show")]
async fn show_users() -> HttpResponse {
    HttpResponse::Ok().body("Show users")
}

#[get("/show/{id}")]
async fn user_detail(path: web::Path<(u32,)>) -> HttpResponse {
    HttpResponse::Ok().body(format!("User detail: {}", path.into_inner().0))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            web::scope("/users")
                .service(show_users)
                .service(user_detail),
        )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

一个路径块包涵可变的路径段作为资源．和没有被块包裹的是一致的．

可以通过`HttpRequest::match_info()`提取可变路径段. [`Path` 提取器](https://actix.rs/docs/extractors)也可以用来提取块级变量段．



## 匹配信息

所有代表匹配到的路径段的值可以在 [`HttpRequest::match_info`](https://docs.rs/actix-web/3/actix_web/struct.HttpRequest.html#method.match_info)获取．特定的值可以被 [`Path::get()`](https://docs.rs/actix-web/3/actix_web/dev/struct.Path.html#method.get)取回．

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Result};

#[get("/a/{v1}/{v2}/")]
async fn index(req: HttpRequest) -> Result<String> {
    let v1: u8 = req.match_info().get("v1").unwrap().parse().unwrap();
    let v2: u8 = req.match_info().query("v2").parse().unwrap();
    let (v3, v4): (u8, u8) = req.match_info().load().unwrap();
    Ok(format!("Values {} {} {} {}", v1, v2, v3, v4))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

对于例子中的路径‘/a/1/2’，v1和v2被解析为1和２．

## 路径信息提取器

Actix提供类型安全的路径信息提取功能． [*Path*](https://docs.rs/actix-web/3/actix_web/dev/struct.Path.html) 提取信息，目标类型可以被定义为多种形式．最简单的方式是使用`tuple`类型．元组中的每一个元素与路径模式中一一对应．例如你可以用`Path<(u32, String)>`匹配路径模式`/{id}/{username}/`,但`Path<(String, String, String)>`类型会失败

```rust
use actix_web::{get, web, App, HttpServer, Result};

#[get("/{username}/{id}/index.html")] // <- define path parameters
async fn index(info: web::Path<(String, u32)>) -> Result<String> {
    let info = info.into_inner();
    Ok(format!("Welcome {}! id: {}", info.0, info.1))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

也可以用结构提取信息，这种情况下，该结构必须实现 serde的Deserialize trait

```rust
use actix_web::{get, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

// extract path info using serde
#[get("/{username}/index.html")] // <- define path parameters
async fn index(info: web::Path<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

[*Query*](https://docs.rs/actix-web/3/actix_web/web/struct.Query.html) 对请求查询参数提供相似的功能．



## 生成资源URL

使用[*HttpRequest.url_for()*](https://docs.rs/actix-web/3/actix_web/struct.HttpRequest.html#method.url_for) 方法生成基于资源模式的URLS.例如你可以配置资源名为“foo”模式为”{a}/{b}/{c}“,可以这样做

```rust
use actix_web::{get, guard, http::header, HttpRequest, HttpResponse, Result};

#[get("/test/")]
async fn index(req: HttpRequest) -> Result<HttpResponse> {
    let url = req.url_for("foo", &["1", "2", "3"])?; // <- generate url for "foo" resource

    Ok(HttpResponse::Found()
        .header(header::LOCATION, url.as_str())
        .finish())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| {
        App::new()
            .service(
                web::resource("/test/{a}/{b}/{c}")
                    .name("foo") // <- set resource name, then it could be used in `url_for`
                    .guard(guard::Get())
                    .to(|| HttpResponse::Ok()),
            )
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

这将会返回类似于字符串 *http://example.com/test/1/2/3*(至少如果当前的协议和主机名暗示*http://example.com*). `url_for()`方法返回 [*Url object*](https://docs.rs/url/1.7.2/url/struct.Url.html)，所以你可以修改这个url(加入query参数，锚点等等).`url_for()`只能在具明资源上调用否则会返回错误．

## 外部资源

为有效url的资源可以被注册为外部资源．他们只对生成的目的URL有效，在请求时不会被考虑．

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Responder};

#[get("/")]
async fn index(req: HttpRequest) -> impl Responder {
    let url = req.url_for("youtube", &["oHg5SJYRHA0"]).unwrap();
    assert_eq!(url.as_str(), "https://youtube.com/watch/oHg5SJYRHA0");

    url.into_string()
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(index)
            .external_resource("youtube", "https://youtube.com/watch/{video_id}")
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

