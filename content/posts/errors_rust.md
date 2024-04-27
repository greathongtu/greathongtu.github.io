+++
title = 'Rust Error Handling'
date = 2024-04-27T19:11:57+08:00
draft = false
+++

从第一性原理推导Rust Error:

One API error, three stakeholders:
Users, Programmers, machines.

### model errors

#### first step
```rust
enum Falliable<T> {
  Ok(T),
  Err
}
```

#### second step
improve error reporting capabilities
```rust
enum Falliable<T> {
  Ok(T),
  Err(String),
}
```

#### third step
provide different error messages to users and programmers
```rust
enum Falliable<T> {
  Ok(T),
  Err {
    user_report: String,
    programmer_report: String,
  }
}
```

#### forth step
provide a stable contract to enable control flow

增加泛型参数Error
```rust
enum Falliable<T, Error> {
  Ok(T),
  Err {
    user_report: String,
    programmer_report: String,
    error: Error,
  }
}
```

for example:
```rust
enum CreateArticleError {
  RateLimited,
  InvalidInputs {},
  GenericError,
}

fn create_article() -> Falliable<T,CreateArticleError>
// inside function we can match the result to enable control flow
```

#### fifth step
avoid duplication

Both reports are generated even if might never used.
```rust
enum Falliable<T, Error> 
where
  Error: Debug + Display
{
  Ok(T),
  Err(Error)
}
```
因为泛型参数Error实现了Debug(用来给programmer看)和Display(用来给User看)

#### sixth step
in a layered application:

fn create_article() -> Falliable<ArticleId, CreateArticleError> 会调用send_http_request() -> Falliable<HttpResponse, HttpError>

这时在create_article()里我们会丢失错误的context
```rust
trait Error: Debug + Display {}
enum CreateArticleError {
  RateLimited,
  InvalidInputs {},
  GenericError{
    source: Box<dyn Error>
  },
}
```

#### final step
make it recursive
```rust
trait Error: Debug + Display {
  fn source(&self) -> Option<&(dyn Error + 'static)>
}
```

### some patterns:
impl std::error::Error for all error types:
derive Debug, impl Display and fn source().

Tip: Use thiserror to do that:
```rust
#[derive(thiserror::Error, Debug)]
pub enum CreateArticleError {
    #[error("failed due to rate limiting")]
    RateLimited,
    #[error("failed: {0}")]
    InvalidInputs {},
    #[error("something wrong")]
    GenericError(#[source] HTTPError)
}
```

### best practise:
use my_module::Error;

use my_module::Result<T>;

* 测试场景：
```rust
pub type Result<T> = core::result::Result<T, Error>;
pub type Error = Box<dyn std::error::Error>;
```
* production 场景：
再细分，写error enum
