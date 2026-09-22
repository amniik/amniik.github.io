---
title: "From and Into in Rust"
categories:
  - Learning Notes
  - Rust
tags: [rust, conversions, traits, type-system]
description: "A concise guide about From & Into traites."

toc: true
---

Rust provides the `From` and `Into` traits for converting one type into another in a consistent and type-safe way.

They are commonly used when working with custom types, error handling, APIs, and generic code.

## 1. The `From` Trait

The `From` trait defines how to create a value of one type from another type.

Its simplified definition is:

```rust
trait From<T>: Sized {
    fn from(value: T) -> Self;
}
```

The important points are:

* `T` is the source type.
* `Self` is the destination type.
* `from` creates a destination value from the source value.

### Example

```rust
let number = String::from("42");

println!("{number}");
```

Here:

```text
&str  ──From──>  String
```

The `String` type implements `From<&str>`.

You can also convert numeric types:

```rust
let number = i32::from(10_i16);

println!("{number}");
```

The conversion is valid because `i32` implements:

```rust
From<i16>
```

## 2. Implementing `From` for a Custom Type

Suppose we have a custom `UserId` type:

```rust
struct UserId(u64);
```

We can define how to create a `UserId` from a `u64`:

```rust
impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        UserId(value)
    }
}
```

Now we can write:

```rust
let user_id = UserId::from(42);

println!("{}", user_id.0);
```

The conversion is explicit and type-safe.

## 3. The `Into` Trait

The `Into` trait defines how a value is converted into another type.

Its simplified definition is:

```rust
trait Into<T>: Sized {
    fn into(self) -> T;
}
```

Example:

```rust
let text: String = "hello".into();

println!("{text}");
```

The compiler knows that the destination type is `String`, so it uses the appropriate conversion.

The type annotation is important:

```rust
let text: String = "hello".into();
```

Without the annotation, Rust may not know which destination type you want:

```rust
let text = "hello".into(); // Usually cannot infer the target type
```

## 4. `From` Automatically Provides `Into`

When you implement:

```rust
From<Source> for Destination
```

Rust automatically provides the reverse-looking conversion:

```rust
Into<Destination> for Source
```

For example:

```rust
struct UserId(u64);

impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        UserId(value)
    }
}
```

You can use both forms:

```rust
let id1 = UserId::from(42);

let id2: UserId = 42.into();
```

Both create a `UserId` from a `u64`.

Conceptually:

```text
From:

UserId::from(42)

Into:

42.into() -> UserId
```

The conversion implementation is defined only once, through `From`.

## 5. Prefer Implementing `From` Instead of `Into`

If you control both types, prefer implementing `From`:

```rust
impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        UserId(value)
    }
}
```

Avoid implementing only `Into`:

```rust
impl Into<UserId> for u64 {
    // Usually not recommended when From can be implemented
}
```

Why?

Implementing `From` automatically gives you `Into`, while implementing only `Into` does not automatically give you `From`.

Therefore, `From` is the canonical direction for defining conversions.

## 6. `From` Is Intended for Infallible Conversions

`From` should generally be used when conversion cannot fail.

For example:

```rust
impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        UserId(value)
    }
}
```

Every `u64` can become a `UserId`, so this conversion is infallible.

However, consider parsing a string into an integer:

```rust
let number: u32 = "42".parse().unwrap();
```

Parsing can fail, so it should not use `From<&str> for u32`.

Instead, Rust uses `FromStr`:

```rust
use std::str::FromStr;

let number = u32::from_str("42");
```

The result is:

```rust
Result<u32, ParseIntError>
```

This makes the possibility of failure explicit.

### General guideline

| Conversion         | Appropriate trait      |
| ------------------ | ---------------------- |
| Cannot fail        | `From` / `Into`        |
| Can fail           | `TryFrom` / `TryInto`  |
| Parsing text       | `FromStr`              |
| Formatting as text | `Display` / `ToString` |

## 7. `TryFrom` and `TryInto`

For conversions that may fail, Rust provides `TryFrom` and `TryInto`.

Simplified definitions:

```rust
trait TryFrom<T>: Sized {
    type Error;

    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

```rust
trait TryInto<T>: Sized {
    type Error;

    fn try_into(self) -> Result<T, Self::Error>;
}
```

Example:

```rust
use std::convert::TryFrom;

let value = u8::try_from(300_u16);

println!("{value:?}");
```

Output:

```text
Err(TryFromIntError(...))
```

The conversion fails because `300` cannot fit into a `u8`.

A valid conversion succeeds:

```rust
let value = u8::try_from(100_u16);

assert_eq!(value.unwrap(), 100);
```

## 8. `From` in Error Handling

One of the most important uses of `From` is converting different error types into a common application error.

Suppose we define:

```rust
use std::io;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
}
```

Implement `From`:

```rust
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError::Io(error)
    }
}
```

Now an `io::Error` can automatically become an `AppError`.

This is especially useful with the `?` operator:

```rust
fn read_file() -> Result<String, AppError> {
    let content = std::fs::read_to_string("config.txt")?;

    Ok(content)
}
```

What happens internally?

```text
read_to_string()
       |
       v
Result<String, io::Error>
       |
       | error occurs
       v
From::from(io::Error)
       |
       v
Result<String, AppError>
```

The `?` operator uses the available conversion to transform the error into the function's declared error type.

This is why implementing `From` for application errors makes error propagation much easier.

## 9. Multiple Error Conversions

An application may have several possible error sources:

```rust
use std::{io, num::ParseIntError};

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
}
```

Implement conversions:

```rust
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError::Io(error)
    }
}

impl From<ParseIntError> for AppError {
    fn from(error: ParseIntError) -> Self {
        AppError::Parse(error)
    }
}
```

Now both operations can use `?`:

```rust
fn load_number() -> Result<u32, AppError> {
    let content = std::fs::read_to_string("number.txt")?;
    let number = content.trim().parse::<u32>()?;

    Ok(number)
}
```

Each error is automatically converted into `AppError`.

## 10. Generic Functions Using `Into`

`Into` is useful when you want a function to accept several input types that can all be converted into one target type.

Example:

```rust
fn greet(name: impl Into<String>) {
    let name: String = name.into();

    println!("Hello, {name}!");
}
```

Now the function accepts different types:

```rust
greet("Amir");
greet(String::from("Amir"));
```

Both work because both arguments can be converted into `String`.

You can also write the generic form explicitly:

```rust
fn greet<T>(name: T)
where
    T: Into<String>,
{
    let name: String = name.into();

    println!("Hello, {name}!");
}
```

The `impl Into<String>` syntax is shorthand for this kind of generic bound.

## 11. `From` and API Design

Consider an API that accepts only `String`:

```rust
fn set_name(name: String) {
    println!("{name}");
}
```

Calling it with a string literal requires an explicit conversion:

```rust
set_name("Amir".to_string());
```

Alternatively, define the API using `Into<String>`:

```rust
fn set_name(name: impl Into<String>) {
    let name = name.into();

    println!("{name}");
}
```

Now both are accepted:

```rust
set_name("Amir");
set_name(String::from("Amir"));
```

This can make APIs more ergonomic.

However, `Into` should not be used automatically everywhere. If the function only needs to temporarily read text, borrowing is often better:

```rust
fn print_name(name: &str) {
    println!("{name}");
}
```

Use `Into<String>` when the function genuinely needs to own or normalize the input.

## 12. `From` vs `Into`

| Feature        | `From`                 | `Into`                               |
| -------------- | ---------------------- | ------------------------------------ |
| Direction      | Source to destination  | Self to target                       |
| Syntax         | `Target::from(value)`  | `value.into()`                       |
| Implementation | Usually implement this | Automatically provided by `From`     |
| Type inference | Usually clearer        | Destination type may be required     |
| Common use     | Defining conversions   | Generic APIs and concise conversions |

Example:

```rust
let value1 = String::from("hello");

let value2: String = "hello".into();
```

Both perform the same conceptual conversion.

## 13. `From` Is Not the Same as `as`

Rust also supports primitive casting with `as`:

```rust
let number = 10_u32 as u64;
```

However, `as` and `From` have different purposes.

### `as`

Used mainly for primitive casts:

```rust
let value = 10_u32 as u64;
```

### `From`

Used for defined semantic conversions:

```rust
let value = u64::from(10_u32);
```

`From` communicates that the conversion is supported as part of the type's API.

It also avoids some potentially surprising casts.

For example, narrowing integer conversions using `as` can truncate:

```rust
let value = 300_u16 as u8;

assert_eq!(value, 44);
```

A checked conversion is safer:

```rust
let value = u8::try_from(300_u16);

assert!(value.is_err());
```

## 14. `From` and Ownership

`Into` consumes `self`:

```rust
trait Into<T> {
    fn into(self) -> T;
}
```

Therefore, calling `.into()` may move the original value.

Example:

```rust
let text = String::from("hello");

let converted: Vec<u8> = text.into_bytes();

// `text` can no longer be used here.
```

Similarly:

```rust
let text = String::from("hello");

let converted: SomeType = text.into();

// `text` was moved into the conversion.
```

If you need to preserve the original value, use a borrowed conversion or clone when appropriate:

```rust
let text = String::from("hello");

let converted = text.clone().into();

// `text` is still available.
println!("{text}");
```

The exact available conversions depend on the implemented traits.

## 15. Common Standard Library Examples

Rust uses `From` extensively:

```rust
let text = String::from("hello");
let bytes = Vec::<u8>::from("hello".as_bytes());
let number = u64::from(10_u32);
let path = std::path::PathBuf::from("/tmp/file.txt");
```

Other examples include:

```rust
let boxed = Box::from(42);
let option = Option::from(42);
let result = Result::<i32, ()>::from(Ok(42));
```

Some conversions are also available through `Into`:

```rust
let text: String = "hello".into();
let path: std::path::PathBuf = "/tmp/file.txt".into();
let option: Option<i32> = 42.into();
```

The destination type often determines which implementation the compiler selects.

## 16. Important Coherence Rule

You cannot implement arbitrary conversions between any two types.

Rust's orphan rules restrict trait implementations when neither the trait nor the type is defined in your crate.

For example, you generally cannot implement a foreign trait for two foreign types:

```rust
// Not allowed in ordinary Rust code:
impl From<String> for Vec<u8> {
    // ...
}
```

Both `From` and `String` are defined outside your crate.

However, you can implement `From` for your own type:

```rust
struct UserId(u64);

impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        UserId(value)
    }
}
```

Here, `UserId` is local to your crate, so the implementation is allowed.

## 17. Mental Model

Think of `From` as defining a constructor-like conversion:

```text
Source value
     |
     | From
     v
Destination value
```

For example:

```text
u64
 |
 | UserId::from(...)
 v
UserId
```

`Into` is the same relationship viewed from the source value:

```text
u64
 |
 | .into()
 v
UserId
```

There is one conversion relationship, but two ways to express it.

## Key Takeaways

* `From<T> for U` defines how to convert `T` into `U`.
* `U::from(value)` performs the conversion explicitly.
* Implementing `From` automatically provides the corresponding `Into` implementation.
* `value.into()` is concise but often requires the destination type to be known.
* Prefer implementing `From` instead of implementing `Into` directly.
* Use `From` for infallible conversions.
* Use `TryFrom` and `TryInto` for fallible conversions.
* `From` is especially useful for converting errors when using `?`.
* `Into` can make generic APIs accept multiple convertible input types.
* Conversions using `Into` consume the source value unless the conversion is implemented for a reference.
* `From` is a trait-based conversion mechanism, not merely a primitive cast like `as`.

---
