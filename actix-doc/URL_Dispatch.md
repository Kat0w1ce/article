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

为了捕获两个段，必须使用两个替代符号

```
foo/{name}.{ext}
```

