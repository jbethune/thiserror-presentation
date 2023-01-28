---
theme: gaia
paginate: true
---

# Abstracting Error handling in Rust with `thiserror`

---

# Once upon a time ...

the young Rust community was debating on how to do error handling in the language.

![width:200px height:200px](https://rustacean.net/assets/rustacean-flat-gesture.svg)![width:200px height:200px](https://rustacean.net/assets/rustacean-flat-happy.svg)

---

# And they came up with the `std::error::Error` trait

```rust
// Rust 1.0 from 2015:
pub trait Error: Debug + Display {
    fn description(&self) -> &str
    fn cause(&self) -> Option<&dyn Error>
}
```
(the `dyn` was actually not part of the signature back in the day)

* `description`: Get a human-readable error message
* `cause`: Get the original error (if any)

---

# 2018: [RFC 2504](https://github.com/rust-lang/rfcs/pull/2504) **Fix the error trait**

![width: 100px height:100px](https://www.rust-lang.org/logos/error.png)

* Rust 1.30.0: Deprecate `cause` and introduce `fn source(&self) -> Option<&(dyn std::error::Error + 'static)>`
* Rust 1.42.0: Deprecate `description` in favor of the `std::fmt::Display` trait which gives us owned `String`s for error messages.
* (also some excluded ideas about backtraces)

---

# The `std::error::Error` trait today (version 1.69.0)

```rust
// we ignore the deprecated and unstable methods
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
}
```

* `std::fmt::Debug` can be `derive`d for your types
* `std::fmt::Display` is annoying to implement
* `source()` is also tricky to implement due to lifetimes and static typing.

---

# Reminder: How to implement `std::fmt::Display`

```rust
impl fmt::Display for MyType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "imagine some meaningful user-facing text here")
    }
}
```
The text that is shown to the user is typically the only thing that differs between different implementations. The rest is just boilerplate.

---

# Meanwhile: The quest for the perfect error-handling library

* People wanted to make implementing errors easier
* Many different error handling libraries were created
* And based on the experiences with those libraries, the official `std::error::Error` trait was also improved
* But the need for error libraries still exists, because the standard library prefers to only offer basic functionality to stay flexible

---

# Two sides of error handling: `thiserror` and `anyhow`

Both can be used to quickly create error types

* `thiserror`: Typically used in library code
* `anyhow`: Typically used in application code
* The code produced by `thiserror` does not appear in your public API. It's only there to reduce boilerplate.

---

# How to get started with `thiserror`

In your `Cargo.toml`
```rust
[dependencies]
thiserror = "1.0"
```

In your code:
```rust
use thiserror::Error as ThisError;

// your error types here
```

---

# Implementing `Display` for an error `struct` with `thiserror`

```rust
use thiserror::Error as ThisError;

#[derive(Debug, ThisError)]
#[error("bad things happened: {message}")]
pub struct MyError {
    message: String,
}
```

---

# Implementing `Display` for an error `enum` with `thiserror`

```rust
use thiserror::Error as ThisError;
const MIN_TEMP: usize = 10;

#[derive(Debug, ThisError)]
pub enum MyEnumError {
    #[error("things simply didn't work")]
    SimpleVariant,
    #[error("found only {0} entries")]
    NotEnoughEntries(usize),
    #[error("Not warm enough: {temperature} {unit} <= {}", MIN_TEMP)]
    Temperature { temperature: usize, unit: String },
}
```

---

# what about the `source` method?

```rust
fn foo() -> Result<(), FooError> {
  Ok(bar()?)
}

fn bar() -> Result<(), BarError> {
  Ok(baz()?)
}

fn baz() -> Result<(), BazError> {
  Err(BazError{})
}
```

call chain: `foo() -> bar() -> baz()`
error chain: `BazError <- BarError <- FooError`

---

<!--
# Error type conversions and error chains

* Type conversion via the `std::convert::From` trait consumes a value of one type and produces a value of another type
* Error chaining takes an error of one type and appends it to a value (typically) of another error type
* Error chaining can be seen as a special case of type conversion.

---
-->

# Implementing the `source` method with `thiserror`

There are two options:
- Use `#[from]` if you have an error type or variant __that contains only the inner error__ (and possibly a backtrace). `#[from]` generates automatic `?` operator conversion code for you.
- Use `#[source]` if you need to store more data in the error. You need to write the conversion code yourself, because `thiserror` cannot guess how to fill the extra values in your error type.

---

# Using `#[from]`

```rust
use thiserror::Error as ThisError;

#[derive(ThisError, Debug)]
#[error("something on the inside went wrong")]
pub struct MyInnerError {} // empty error type for demonstration purposes

#[derive(ThisError, Debug)]
pub enum MyError {
    #[error("things broke on the inside")]
    InnerProblem {
        #[from] inner: MyInnerError,
    },
    #[error("IO troubles")]
    IO(#[from] std::io::Error),
}
```

---

# Using `#[source]`

```rust
use thiserror::Error as ThisError;

#[derive(ThisError, Debug)]
#[error("Terrible things happened: {message}")]
pub struct MyError {
    #[source]  // optional if field name is `source`
    source: MyInnerError, // if we know it is always this error type
    message: String, // we can't use #[from] because of this extra field
}
```

flexible option: `source: Option<Box<dyn std::error::Error>>`
flexible easy option: `source: anyhow::Error` (from the `anyhow` crate)

---

# Let the inner error do the work: `#[transparent]`

You can forward the `source` and `std::fmt::Display` implementations from the inner error to the outer error.

```rust
#[derive(Debug, ThisError)]
pub enum MyError {
    #[error("IO failed")] // we provide the error message ourselves
    IO(#[from] std::io::Error),
    #[error(transparent)] // use source and Display from inner error
    Other(#[from] anyhow::Error), // catch-all for any other inner error type
}
```

---

# Summary

- Error handling in Rust can require writing a lot of boilerplate code, because the standard library only provides basic functionality
- Error libraries like `thiserror` can reduce the boilerplate
- `thiserror` uses the `#[error(text)]` macro to generate `std::fmt::Display` implementations with format strings.
- `thiserror` can generate `std::convert::From<SomeInnerError>` implementations when you use the `#[from]` macro.
- `thiserror` can generate the `std::error::Error::source` method for you, when you use the `#[source]` macro or when you name the inner error field `source` (or when you use `#[from]`)
