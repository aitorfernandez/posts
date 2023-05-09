# Rust Custom Errors

Rust doesn't utilize exceptions like try/catch or begin/rescue commonly found in other languages. Instead, the outcome of functions or methods is returned as a [Result](https://doc.rust-lang.org/std/result/) enum type, indicating either successful execution (Ok) or an error (Err) along with more details.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Errors in systems and applications are inevitable, so it's a good idea to anticipate potential problems and plan for proper handling.

Suppose we are creating a public API, it's beneficial to define custom error types and provide detailed information to help us in error resolution.

Some minimal code to handle error details could look like this:

```rs
pub struct ApiError {
    pub code: u16,
    pub message: String,
}
```

And when we want to raise an API error, we can return it easily.

```rust
fn upload_file() -> Result<(), ApiError> {
    // ...

    Err(ApiError {
        code: 403,
        message: "Quota has been exceeded.".to_string(),
    })
}
```

To have more control, we can implement a [constructor](https://en.wikipedia.org/wiki/Constructor_%28object-oriented_programming%29) function to assist in returning the errors.

```rust
impl ApiError {
    pub fn new(code: u16, message: String) -> Self {
        Self { code, message }
    }
}

fn upload_file(...) -> Result<(), ApiError> {
    // ...

    Err(ApiError::new(403, "Quota has been exceeded.".to_string()))
}
```

This works well but there are a few guidelines to follow to make our custom error works like the Rust standard error types.

Firstly, we must ensure that the errors are printable by adding the [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) directive to the custom error structure.

```rust
#[derive(Debug)]
pub struct ApiError {
    // [...]
}
```

Secondly, we must manually implement the [Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) trait to enable error printing and use the [write!](https://doc.rust-lang.org/std/macro.write.html) macro for formatting the arguments we want.

```rust
impl std::fmt::Display for ApiError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        write!(f, "API error: {}, {}", self.code, self.message)
    }
}
```

Thirdly, we must implement the [Error](https://doc.rust-lang.org/std/error/trait.Error.html) trait. Luckily, the default methods are sufficient, and we just need to declare the implementation.

```rust
impl std::error::Error for ApiError { }
```

However, if we are not happy with the default methods we can implement custom versions.

```rust
impl std::error::Error for ApiError {
    fn description(&self) -> &str {
        &self.message
    }

    // ...
}
```

We can simplify things by using a type alias when multiple functions or methods return the `ApiError`.

```rust
type ApiResult<T> = Result<T, ApiError>;
```

Now, instead of indicating both types in the Result enum `Result<(), ApiError>`, as we did previously, we can simplify it further by using the ApiResult type alias.

```rust
fn upload_file(...) -> ApiResult<()> {
    // ...
}
```

The `ApiError` works well but in a complex system, many things can go wrong. For example, in another layer of the service, we may have a database that returns its custom errors.

```rs
#[derive(Debug)]
pub enum DatabaseError {
    RowNotFound,
    // ...
}

impl std::fmt::Display for DatabaseError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        // ...
    }
}

impl std::error::Error for DatabaseError {}
```

Transitioning from one error to another causes the Rust compiler to produce an error. The following code would result in an error.

```rust
fn find_in_database(...) -> Result<(), DatabaseError> {
    // ...

    Err(DatabaseError::RowNotFound)
}

fn find_file(...) -> ApiResult<()> {
    let file = find_in_database()?;

    // ...
}
```

Before dealing with the compiler error, let's see the `?` operator used in the previous code. This operator provides an easier way to handle errors without repeatedly using [match](https://doc.rust-lang.org/std/keyword.match.html) statements. By using the `?` operator, we can either receive the return value or immediately return the error terminating the function flow.

The previous code without the`?` operator can be expressed as:

```rust
fn find_file(...) -> ApiResult<()> {
    let file = mathc find_in_database() {
        Ok(result) => result,
        Err(err) => return Err(From::from(err)),
    }

    // ...
}
```

Going back to the compiler error, the compiler informs us that it can't convert from `DatabaseError` to `ApiError`.

```rust
error[E0277]: `?` couldn't convert the error to `ApiError`
--> src/main.rs:22:35
|
| fn upload_file() -> ApiResult<()> {
|                     ------------- expected `ApiError` because of this
...
| let _file = find_in_database()?;
|                               ^ the trait `From<DatabaseError>` is not implemented for `ApiError`
|
```

Following the compiler's suggestions, we can implement the [From](https://doc.rust-lang.org/std/convert/trait.From.html) trait to convert values from one type to another.

```rust
impl From<DatabaseError> for ApiError {
    fn from(err: DatabaseError) -> Self {
        match err {
            DatabaseError::RowNotFound => Self::new(404, "File not found".to_string()),
            // ...
        }
    }
}
```

And now the error can be propagated without encountering any issues.

```rust
fn find_in_database(...) -> Result<(), DatabaseError> {
    // ...

    Err(DatabaseError::RowNotFound)
}

fn find_file(...) -> ApiResult<()> {
    let file = find_in_database()?;

    // ...
}
```

Custom errors in Rust can be easily implemented but it requires following certain rules, such as printing the error and implementing the Display and Error traits. By following these easy rules, we can make sure that our custom errors work similarly to the standard Rust error types.
