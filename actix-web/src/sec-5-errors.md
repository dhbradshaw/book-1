# Errors

Actix uses the [*Error*](../../actix-web/actix_web/error/struct.Error.html) type
and [*ResponseError*](../../actix-web/actix_web/error/trait.ResponseError.html)
trait for handling handler's errors.

Any error that implements the `ResponseError` trait can be returned as an error value.
`Handler` can return an `Result` object. By default, actix provides a
`Responder` implementation for compatible result types. Here is the implementation
definition:

```rust,ignore
impl<T: Responder, E: Into<Error>> Responder for Result<T, E>
```

Any error that implements `ResponseError` can be converted into an `Error` object.

For example, if the *handler* function returns `io::Error`, it would be converted
into an `HttpInternalServerError` response. Implementation for `io::Error` is provided
by default.

```rust
# extern crate actix_web;
# use actix_web::*;
use std::io;

fn index(req: HttpRequest) -> io::Result<fs::NamedFile> {
    Ok(fs::NamedFile::open("static/index.html")?)
}
#
# fn main() {
#     App::new()
#         .resource(r"/a/index.html", |r| r.f(index))
#         .finish();
# }
```

## Custom error response

To add support for custom errors, all we need to do is implement the `ResponseError` trait
for the custom error type. The `ResponseError` trait has a default implementation
for the `error_response()` method: it generates a *500* response.

```rust
# extern crate actix_web;
#[macro_use] extern crate failure;
use actix_web::*;

#[derive(Fail, Debug)]
#[fail(display="my error")]
struct MyError {
   name: &'static str
}

/// Use default implementation for `error_response()` method
impl error::ResponseError for MyError {}

fn index(req: HttpRequest) -> Result<&'static str, MyError> {
    Err(MyError{name: "test"})
}
#
# fn main() {
#     App::new()
#         .resource(r"/a/index.html", |r| r.f(index))
#         .finish();
# }
```

In this example the *index* handler always returns a *500* response. But it is easy
to return different responses for different types of errors.

```rust
# extern crate actix_web;
#[macro_use] extern crate failure;
use actix_web::{App, HttpRequest, HttpResponse, http, error};

#[derive(Fail, Debug)]
enum MyError {
   #[fail(display="internal error")]
   InternalError,
   #[fail(display="bad request")]
   BadClientData,
   #[fail(display="timeout")]
   Timeout,
}

impl error::ResponseError for MyError {
    fn error_response(&self) -> HttpResponse {
       match *self {
          MyError::InternalError => HttpResponse::new(
              http::StatusCode::INTERNAL_SERVER_ERROR),
          MyError::BadClientData => HttpResponse::new(
              http::StatusCode::BAD_REQUEST),
          MyError::Timeout => HttpResponse::new(
              http::StatusCode::GATEWAY_TIMEOUT),
       }
    }
}

fn index(req: HttpRequest) -> Result<&'static str, MyError> {
    Err(MyError::BadClientData)
}
#
# fn main() {
#     App::new()
#         .resource(r"/a/index.html", |r| r.f(index))
#         .finish();
# }
```

## Error helpers

Actix provides a set of error helper types. It is possible to use them for generating
specific error responses. We can use the helper types for the first example with a custom error.

```rust
# extern crate actix_web;
#[macro_use] extern crate failure;
use actix_web::*;

#[derive(Debug)]
struct MyError {
   name: &'static str
}

fn index(req: HttpRequest) -> Result<&'static str> {
    let result: Result<&'static str, MyError> = Err(MyError{name: "test"});

    Ok(result.map_err(|e| error::ErrorBadRequest(e.name))?)
}
# fn main() {
#     App::new()
#         .resource(r"/a/index.html", |r| r.f(index))
#         .finish();
# }
```

In this example, a *BAD REQUEST* response is generated for the `MyError` error.

## Error logging

Actix logs all errors with the log level `WARN`. If log level set to `DEBUG`
and `RUST_BACKTRACE` is enabled, the backtrace gets logged. The Error type uses
the cause's error backtrace if available. If the underlying failure does not provide
a backtrace, a new backtrace is constructed pointing to that conversion point
(rather than the origin of the error). This construction only happens if there
is no underlying backtrace; if it does have a backtrace, no new backtrace is constructed.

You can enable backtrace and debug logging with following command:

```
>> RUST_BACKTRACE=1 RUST_LOG=actix_web=debug cargo run
```
