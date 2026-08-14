---
title: "Rust Notes: Option and Result"
categories:
  - Learning Notes
  - Rust
tags: [rust, option, rust-lang, result, error-handling]
description: "A practical guide to Rust's Option and Result types."

toc: true
---

# Rust `Option` and `Result`

Rust does not use `null` and exceptions as its primary way of representing absence and recoverable errors.

Instead, Rust gives us two fundamental enums:

```rust
Option<T>
Result<T, E>
```

They are everywhere in Rust code.

A good mental model is:

```text
Option<T>
    |
    +-- Some(T)   -> a value exists
    |
    +-- None      -> no value

Result<T, E>
    |
    +-- Ok(T)     -> operation succeeded
    |
    +-- Err(E)    -> operation failed
```

`Option` is mainly about **absence**.

`Result` is about **success or failure with an error value**.

The standard library defines `Option<T>` as an optional value and `Result<T, E>` as the type used for returning and propagating errors.

---

# 1. `Option<T>`

The definition is essentially:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

For example:

```rust
let name: Option<String> = Some(String::from("Amir"));

let missing_name: Option<String> = None;
```

The important point is that the compiler forces us to acknowledge that the value might not exist.

---

# 2. Why `Option` Instead of `null`?

In languages with `null`, this is possible:

```text
String
  |
  +-- actual string
  |
  +-- null
```

The type itself does not necessarily tell you that `null` is possible.

Rust instead represents this explicitly:

```rust
Option<String>
```

which tells us:

```text
String may exist
      OR
String may not exist
```

This makes absence part of the type system.

---

# 3. Handling `Option` with `match`

The most explicit way to handle an `Option` is:

```rust
let name = Some("Amir");

match name {
    Some(name) => println!("Name: {name}"),
    None => println!("No name"),
}
```

The compiler makes sure both cases are handled.

---

# 4. `if let`

When you only care about one variant:

```rust
let name = Some("Amir");

if let Some(name) = name {
    println!("Name: {name}");
}
```

You can also handle the other case:

```rust
if let Some(name) = name {
    println!("Name: {name}");
} else {
    println!("No name");
}
```

This is useful when a full `match` would be unnecessarily verbose.

---

# 5. `while let`

`while let` repeatedly matches a pattern:

```rust
let mut values = vec![1, 2, 3];

while let Some(value) = values.pop() {
    println!("{value}");
}
```

The loop continues while the expression produces `Some`.

When it produces `None`, the loop ends.

---

# 6. `unwrap()`

One of the most recognizable methods is:

```rust
let value = Some(42);

let x = value.unwrap();

println!("{x}");
```

Result:

```text
42
```

But:

```rust
let value: Option<i32> = None;

let x = value.unwrap();
```

causes a panic.

Conceptually:

```rust
Some(value).unwrap()
    -> value

None.unwrap()
    -> panic!
```

`unwrap()` should therefore be used when you genuinely know that the value cannot be absent, or when a panic is acceptable. The standard library documentation explicitly warns that `unwrap` panics on `None`.

---

# 7. `expect()`

`expect()` is similar to `unwrap()` but lets you provide a useful panic message:

```rust
let config = Some("config.toml");

let path = config.expect("configuration path must exist");
```

If the value is `None`, the panic message explains the assumption that was violated.

Prefer:

```rust
expect("configuration must exist")
```

over:

```rust
unwrap()
```

when a panic is genuinely appropriate.

---

# 8. `unwrap_or()`

Instead of panicking, provide a default:

```rust
let name = Some("Amir");

let value = name.unwrap_or("Unknown");
```

Result:

```text
Amir
```

With `None`:

```rust
let name: Option<&str> = None;

let value = name.unwrap_or("Unknown");
```

Result:

```text
Unknown
```

Conceptually:

```text
Some(x).unwrap_or(default)
    -> x

None.unwrap_or(default)
    -> default
```

---

# 9. `unwrap_or_default()`

If the contained type implements `Default`:

```rust
let name: Option<String> = None;

let name = name.unwrap_or_default();
```

Result:

```rust
String::new()
```

For example:

```rust
let number: Option<i32> = None;

let number = number.unwrap_or_default();
```

Result:

```text
0
```

---

# 10. `unwrap_or_else()`

Use a closure to calculate the fallback:

```rust
let name: Option<String> = None;

let name = name.unwrap_or_else(|| {
    String::from("Unknown")
});
```

This is particularly useful when calculating the default is expensive.

Compare:

```rust
value.unwrap_or(expensive_operation())
```

with:

```rust
value.unwrap_or_else(|| expensive_operation())
```

The second version evaluates the fallback only when needed.

The standard library documents `unwrap_or` as eagerly evaluating its argument and `unwrap_or_else` as lazily evaluating the fallback closure.

---

# 11. `map()` on `Option`

`map()` transforms the value inside `Some`.

```rust
let name = Some("Amir");

let length = name.map(|name| name.len());
```

The result is:

```rust
Some(4)
```

The important property is:

```text
Some(x)
   |
   | map(f)
   v
Some(f(x))

None
   |
   | map(f)
   v
None
```

So:

```rust
Some(10)
    .map(|x| x * 2)
```

becomes:

```rust
Some(20)
```

while:

```rust
None
    .map(|x| x * 2)
```

remains:

```rust
None
```

This is one of the most useful patterns in Rust.

---

# 12. `map()` Does Not Handle Failure

Suppose:

```rust
fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.parse()
}
```

You cannot use `Option::map` to flatten a function that itself returns an `Option`:

```rust
let result = Some("42")
    .map(|s| find_number(s));
```

If `find_number()` returns `Option<i32>`, the result becomes:

```rust
Option<Option<i32>>
```

This is where `and_then()` becomes useful.

---

# 13. `and_then()` on `Option`

Suppose:

```rust
fn get_user(id: u32) -> Option<User> {
    // ...
}

fn get_address(user: User) -> Option<Address> {
    // ...
}
```

You can chain them:

```rust
let address = get_user(10)
    .and_then(get_address);
```

The important difference is:

```rust
map:
T -> U

and_then:
T -> Option<U>
```

For example:

```rust
Some(10)
    .map(|x| Some(x * 2))
```

produces:

```rust
Some(Some(20))
```

while:

```rust
Some(10)
    .and_then(|x| Some(x * 2))
```

produces:

```rust
Some(20)
```

`and_then()` is therefore often described as **flat-mapping** an `Option`.

---

# 14. `and_then()` Creates a Pipeline

This pattern is extremely useful:

```rust
let result = get_user()
    .and_then(get_account)
    .and_then(get_permissions)
    .and_then(get_configuration);
```

Conceptually:

```text
get_user()
    |
    v
Option<User>
    |
    | and_then
    v
Option<Account>
    |
    | and_then
    v
Option<Permissions>
    |
    | and_then
    v
Option<Configuration>
```

If any step returns `None`, the remaining operations are skipped.

This gives us a clean "stop if unavailable" pipeline.

---

# 15. `filter()`

`filter()` keeps a `Some` only if the predicate returns `true`.

```rust
let number = Some(10);

let result = number.filter(|x| *x > 5);
```

Result:

```rust
Some(10)
```

But:

```rust
let number = Some(3);

let result = number.filter(|x| *x > 5);
```

produces:

```rust
None
```

Conceptually:

```text
Some(x) + predicate true
    -> Some(x)

Some(x) + predicate false
    -> None

None
    -> None
```

---

# 16. `flatten()`

Suppose you have:

```rust
let value: Option<Option<i32>> = Some(Some(42));
```

You can remove one level:

```rust
let value = value.flatten();
```

Result:

```rust
Some(42)
```

Examples:

```rust
Some(Some(42)).flatten()
    -> Some(42)

Some(None).flatten()
    -> None

None.flatten()
    -> None
```

`and_then()` is often preferable when the nesting comes directly from a function call.

---

# 17. `ok_or()`

Sometimes you have an `Option` but need a `Result`.

For example:

```rust
let user = find_user(id);
```

where:

```rust
find_user(id) -> Option<User>
```

You can convert it:

```rust
let user = find_user(id)
    .ok_or(MyError::UserNotFound)?;
```

The conversion is:

```text
Some(value)
    -> Ok(value)

None
    -> Err(error)
```

This is extremely useful when moving from optional data into error handling.

---

# 18. `ok_or_else()`

`ok_or_else()` is the lazy version:

```rust
let user = find_user(id)
    .ok_or_else(|| MyError::UserNotFound)?;
```

The error is constructed only if the `Option` is `None`.

Use:

```rust
ok_or(...)
```

when the error is cheap to construct.

Use:

```rust
ok_or_else(|| ...)
```

when constructing the error requires work.

---

# 19. `Option` and `?`

The `?` operator works with `Option`.

For example:

```rust
fn get_first_name(user: &User) -> Option<&str> {
    let name = user.name.as_ref()?;
    Some(name)
}
```

The behavior is:

```text
Some(value)?
    -> value

None?
    -> return None
```

So:

```rust
fn get_name(user: &User) -> Option<&str> {
    Some(user.name.as_str())
}
```

can use `?` to propagate absence through a chain.

The Rust standard library describes `?` for `Option` as returning `None` early when the value is absent.

---

# 20. `?` Is Not the Same as `unwrap()`

This distinction is extremely important.

```rust
let x = value.unwrap();
```

means:

```text
If failure:
    panic
```

while:

```rust
let x = value?;
```

means:

```text
If failure:
    return the failure from this function
```

For `Option`:

```text
unwrap()
    None -> panic

?
    None -> return None
```

For `Result`:

```text
unwrap()
    Err -> panic

?
    Err -> return Err(...)
```

This is one of the most important differences in Rust error handling.

---

# 21. `Result<T, E>`

The definition is:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

For example:

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}
```

Now:

```rust
let result = divide(10, 2);
```

gives:

```rust
Ok(5)
```

while:

```rust
let result = divide(10, 0);
```

gives:

```rust
Err("division by zero")
```

---

# 22. `Result` vs `Option`

A useful rule:

```text
Option<T>
    "There may or may not be a value."

Result<T, E>
    "The operation may succeed or fail, and I want to know why."
```

Examples:

```rust
Vec::get(0)
```

returns:

```rust
Option<&T>
```

because the element may simply not exist.

But:

```rust
"123".parse::<i32>()
```

returns:

```rust
Result<i32, ParseIntError>
```

because parsing can fail for a meaningful reason.

---

# 23. Handling `Result` with `match`

```rust
match divide(10, 2) {
    Ok(value) => println!("Result: {value}"),
    Err(error) => println!("Error: {error}"),
}
```

This explicitly handles both outcomes.

---

# 24. `unwrap()` on `Result`

```rust
let value = "42".parse::<i32>().unwrap();
```

If parsing succeeds:

```text
42
```

If parsing fails:

```rust
"hello".parse::<i32>().unwrap();
```

the program panics.

Again:

```text
Ok(value).unwrap()
    -> value

Err(error).unwrap()
    -> panic
```

---

# 25. `expect()` on `Result`

```rust
let value = "42"
    .parse::<i32>()
    .expect("input must be a number");
```

This is useful when the failure indicates a programming error or violates an invariant that you intentionally want to treat as unrecoverable.

---

# 26. `map()` on `Result`

`Result::map()` transforms the `Ok` value.

```rust
let result: Result<i32, &str> = Ok(10);

let result = result.map(|x| x * 2);
```

Result:

```rust
Ok(20)
```

But:

```rust
let result: Result<i32, &str> = Err("failed");

let result = result.map(|x| x * 2);
```

remains:

```rust
Err("failed")
```

Conceptually:

```text
Ok(x)
   |
   | map(f)
   v
Ok(f(x))

Err(e)
   |
   | map(f)
   v
Err(e)
```

---

# 27. `map_err()`

This is one of the most important methods when working with errors.

`map()` transforms the success:

```rust
Result<T, E>
    -> Result<U, E>
```

`map_err()` transforms the error:

```rust
Result<T, E>
    -> Result<T, F>
```

Example:

```rust
let result = std::fs::read_to_string("config.toml")
    .map_err(|error| MyError::Config(error));
```

The success value stays unchanged.

Only the error is transformed.

Conceptually:

```text
Ok(value)
    |
    | map_err(...)
    v
Ok(value)

Err(error)
    |
    | map_err(...)
    v
Err(transformed_error)
```

The standard library specifically defines `map_err` as transforming the contained `Err` value while leaving `Ok` unchanged.

---

# 28. Why `map_err()` Is Useful

Suppose:

```rust
fn load_config() -> Result<Config, ConfigError> {
    std::fs::read_to_string("config.toml")
        .map_err(ConfigError::Io)
}
```

The file operation might produce:

```rust
std::io::Error
```

but your function wants:

```rust
ConfigError
```

`map_err()` converts between the error types.

This is especially common when building your own error hierarchy.

---

# 29. `and_then()` on `Result`

`Result::and_then()` is used to chain operations that themselves return `Result`.

Suppose:

```rust
fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.parse()
}

fn double(x: i32) -> Result<i32, MyError> {
    Ok(x * 2)
}
```

You can chain fallible operations:

```rust
let result = parse_number("42")
    .and_then(double);
```

Conceptually:

```text
Ok(x)
   |
   | and_then(f)
   v
f(x)

Err(e)
   |
   | and_then(f)
   v
Err(e)
```

The function passed to `and_then()` must itself return a `Result`.

---

# 30. `map()` vs `and_then()`

This distinction is extremely important.

Suppose:

```rust
fn double(x: i32) -> Result<i32, MyError> {
    Ok(x * 2)
}
```

Using `map()`:

```rust
let result = Ok(10).map(double);
```

produces:

```rust
Ok(Ok(20))
```

because:

```text
map:
T -> U

i32 -> Result<i32, MyError>
```

Therefore:

```text
Result<i32, E>
    -> Result<Result<i32, MyError>, E>
```

But with:

```rust
let result = Ok(10).and_then(double);
```

we get:

```rust
Ok(20)
```

because:

```text
and_then:
T -> Result<U, E>
```

and it flattens the nested `Result`.

---

# 31. A Useful Pipeline

This is a very common Rust pattern:

```rust
let value = read_file()
    .map_err(MyError::Io)
    .and_then(parse_config)
    .and_then(validate_config);
```

Think of it as:

```text
read_file()
     |
     v
Result<String, IoError>
     |
     | map_err
     v
Result<String, MyError>
     |
     | and_then
     v
Result<Config, MyError>
     |
     | and_then
     v
Result<ValidConfig, MyError>
```

If any operation produces `Err`, later operations are skipped.

---

# 32. `or_else()`

`or_else()` is useful when you want an alternative operation after failure.

For example:

```rust
let result = read_primary_config()
    .or_else(|_| read_backup_config());
```

Conceptually:

```text
Ok(value)
    -> Ok(value)

Err(error)
    -> call fallback function
```

Unlike `and_then()`, which continues on success, `or_else()` provides a path for failure.

---

# 33. `map_or()`

`map_or()` converts a `Result` into another value.

```rust
let result: Result<i32, &str> = Ok(10);

let value = result.map_or(0, |x| x * 2);
```

Result:

```text
20
```

For:

```rust
let result: Result<i32, &str> = Err("failed");

let value = result.map_or(0, |x| x * 2);
```

Result:

```text
0
```

Conceptually:

```text
Ok(x)
    -> f(x)

Err(_)
    -> default
```

---

# 34. `map_or_else()`

Use `map_or_else()` when the fallback needs computation:

```rust
let value = result.map_or_else(
    |error| {
        println!("error: {error}");
        0
    },
    |value| value * 2,
);
```

It provides:

```text
Err -> error handler
Ok  -> success handler
```

---

# 35. `ok()` and `err()`

You can convert a `Result` into an `Option`.

```rust
let result: Result<i32, &str> = Ok(42);

let value = result.ok();
```

Result:

```rust
Some(42)
```

For an error:

```rust
let result: Result<i32, &str> = Err("failed");

let value = result.ok();
```

Result:

```rust
None
```

`err()` does the opposite side:

```rust
let result: Result<i32, &str> = Err("failed");

let error = result.err();
```

Result:

```rust
Some("failed")
```

The standard library defines `ok()` as converting `Ok(v)` to `Some(v)` and `Err(_)` to `None`, while `err()` extracts the error into an `Option`.

---

# 36. `Result` and `Option` Conversion

These operations are useful:

```rust
Option<T>
    |
    +-- ok_or(...)
    +-- ok_or_else(...)
    |
    v
Result<T, E>
```

And:

```rust
Result<T, E>
    |
    +-- ok()
    +-- err()
    |
    v
Option<T>
```

This allows you to move between:

```text
absence semantics
```

and:

```text
error semantics
```

depending on what your function needs.

---

# 37. `?` with `Result`

This is probably the most important Rust error-handling syntax.

Without `?`:

```rust
fn load_config() -> Result<String, std::io::Error> {
    let result = std::fs::read_to_string("config.toml");

    match result {
        Ok(value) => Ok(value),
        Err(error) => Err(error),
    }
}
```

With `?`:

```rust
fn load_config() -> Result<String, std::io::Error> {
    let value = std::fs::read_to_string("config.toml")?;

    Ok(value)
}
```

The `?` operator propagates the error.

The standard library documentation describes this as returning the `Err` early from the enclosing function and unwrapping the `Ok` value otherwise.

---

# 38. What `?` Conceptually Does

This:

```rust
let value = operation()?;
```

can be thought of roughly as:

```rust
let value = match operation() {
    Ok(value) => value,
    Err(error) => return Err(error),
};
```

So:

```text
operation()
     |
     +-- Ok(value) -----> continue
     |
     +-- Err(error) ----> return Err(error)
```

This is why `?` makes error propagation so concise.

---

# 39. `?` Can Convert Error Types

One of the powerful aspects of `?` is that error conversion can happen when the target error type supports the necessary conversion.

For example:

```rust
fn load() -> Result<Config, MyError> {
    let text = std::fs::read_to_string("config.toml")?;
    // ...
}
```

If:

```text
std::io::Error
```

can be converted into:

```text
MyError
```

then `?` can perform that conversion.

A common approach is to implement:

```rust
impl From<std::io::Error> for MyError {
    fn from(error: std::io::Error) -> Self {
        MyError::Io(error)
    }
}
```

Then:

```rust
let text = std::fs::read_to_string("config.toml")?;
```

can propagate the converted error.

---

# 40. `?` Does Not Mean "Ignore Errors"

This is an important misconception.

This:

```rust
let value = operation()?;
```

does **not** mean:

```text
ignore the error
```

It means:

```text
if successful:
    continue with the value

if failed:
    return the error from this function
```

So `?` is better understood as:

> **propagate failure upward.**

---

# 41. Chaining `?`

This is where Rust error handling becomes very clean:

```rust
fn load_config() -> Result<Config, MyError> {
    let text = std::fs::read_to_string("config.toml")?;

    let config: Config = toml::from_str(&text)?;

    validate_config(&config)?;

    Ok(config)
}
```

The flow is:

```text
read file
   |
   +-- error -> return
   |
   v
parse
   |
   +-- error -> return
   |
   v
validate
   |
   +-- error -> return
   |
   v
Ok(config)
```

This is one of the main reasons Rust error handling can remain readable even when many operations can fail.

---

# 42. `?` with `Option`

The same idea works with `Option`:

```rust
fn get_port(config: &Config) -> Option<u16> {
    let network = config.network.as_ref()?;
    let port = network.port?;

    Some(port)
}
```

The flow is:

```text
Some(value)
    -> continue

None
    -> return None
```

---

# 43. `?` with Both `Option` and `Result`

Sometimes you need to convert between them.

For example:

```rust
fn get_port(config: &Config) -> Result<u16, MyError> {
    let network = config
        .network
        .as_ref()
        .ok_or(MyError::MissingNetwork)?;

    let port = network
        .port
        .ok_or(MyError::MissingPort)?;

    Ok(port)
}
```

Here:

```text
Option
   |
   | ok_or(...)
   v
Result
   |
   | ?
   v
value
```

This pattern is extremely common.

---

# 44. `transpose()`

Another useful conversion is `transpose()`.

Suppose:

```rust
Option<Result<T, E>>
```

You can turn it into:

```rust
Result<Option<T>, E>
```

For example:

```rust
let value: Option<Result<i32, &str>> = Some(Ok(42));

let value = value.transpose();
```

Result:

```rust
Ok(Some(42))
```

This is useful when working with nested `Option` and `Result` values.

---

# 45. `Result::transpose()`

The opposite structure:

```rust
Result<Option<T>, E>
```

can become:

```rust
Option<Result<T, E>>
```

using:

```rust
transpose()
```

The key idea is that `transpose()` swaps the outer structure:

```text
Option<Result<T, E>>
        |
        v
Result<Option<T>, E>
```

or:

```text
Result<Option<T>, E>
        |
        v
Option<Result<T, E>>
```

---

# 46. `inspect()`

Sometimes you want to look at a successful value without changing it.

```rust
let result = "42"
    .parse::<i32>()
    .inspect(|value| {
        println!("parsed value: {value}");
    });
```

The original `Result` continues through the pipeline.

Similarly for `Option`:

```rust
let value = Some(42)
    .inspect(|value| {
        println!("value = {value}");
    });
```

`inspect()` is useful for debugging pipelines.

---

# 47. `inspect_err()`

For `Result`, you can inspect an error without transforming it:

```rust
let result = operation()
    .inspect_err(|error| {
        eprintln!("operation failed: {error}");
    });
```

The error remains unchanged.

This is useful for logging or debugging without breaking the pipeline.

---

# 48. `Option` Boolean-Style Combinators

`Option` has:

```rust
and()
or()
xor()
```

For example:

```rust
Some(10).and(Some(20))
```

produces:

```rust
Some(20)
```

while:

```rust
Some(10).and(None)
```

produces:

```rust
None
```

---

# 49. `and_then()` vs `and()`

The difference is that `and()` takes an already-created `Option`:

```rust
Some(10).and(Some(20))
```

while `and_then()` takes a closure:

```rust
Some(10).and_then(|x| {
    Some(x * 2)
})
```

`and_then()` is lazy because the closure is only called when the original value is `Some`.

---

# 50. `or()` and `or_else()`

`or()` provides another `Option`:

```rust
None.or(Some(10))
```

produces:

```rust
Some(10)
```

`or_else()` computes the fallback lazily:

```rust
None.or_else(|| {
    Some(10)
})
```

The pattern is:

```text
and / and_then
    -> alternative on success

or / or_else
    -> alternative on absence/failure
```

---

# 51. `Result` Boolean-Style Combinators

`Result` also has:

```rust
and()
or()
and_then()
or_else()
```

For example:

```rust
Ok(10).and(Ok(20))
```

produces:

```rust
Ok(20)
```

while:

```rust
Ok(10).and(Err("failed"))
```

produces:

```rust
Err("failed")
```

And:

```rust
Err("primary failed")
    .or_else(|_| Ok(42))
```

can recover using an alternative operation.

---

# 52. `map` vs `map_err` vs `and_then` vs `or_else`

This is one of the most important tables to remember:

| Method     | Operates on   | Function returns             | Main purpose              |
| ---------- | ------------- | ---------------------------- | ------------------------- |
| `map`      | success/value | `U`                          | transform success         |
| `map_err`  | error         | `F`                          | transform error           |
| `and_then` | success/value | `Result<U, E>` / `Option<U>` | chain fallible operations |
| `or_else`  | error/absence | `Result<T, F>` / `Option<T>` | provide fallback          |

For `Result`:

```text
              Result<T, E>
                   |
       +-----------+-----------+
       |                       |
      Ok                      Err
       |                       |
       v                       v
     map()                 map_err()
       |                       |
       v                       v
   transform               transform
    success                  error
```

And:

```text
              Result<T, E>
                   |
       +-----------+-----------+
       |                       |
      Ok                      Err
       |                       |
       v                       v
   and_then()              or_else()
       |                       |
       v                       v
 continue with             fallback
 fallible operation        operation
```

---

# 53. A Realistic Error Pipeline

Suppose we want to:

1. read a file
2. parse it
3. validate it

We can write:

```rust
fn load_config() -> Result<Config, MyError> {
    std::fs::read_to_string("config.toml")
        .map_err(MyError::Io)
        .and_then(|text| parse_config(&text))
        .and_then(|config| validate_config(config))
}
```

Or, often more readably:

```rust
fn load_config() -> Result<Config, MyError> {
    let text = std::fs::read_to_string("config.toml")
        .map_err(MyError::Io)?;

    let config = parse_config(&text)?;

    let config = validate_config(config)?;

    Ok(config)
}
```

Both approaches are valid.

The second version is often easier to read when the operations are substantial.

---

# 54. When Should I Use Combinators?

Combinators such as:

```rust
map()
map_err()
and_then()
or_else()
```

are particularly nice when the operation is naturally a pipeline:

```rust
let result = input
    .map(parse)
    .and_then(validate)
    .map(transform)
    .map_err(convert_error);
```

But don't force everything into method chains.

This:

```rust
let config = read_file()?;
let config = parse(config)?;
validate(&config)?;
Ok(config)
```

is often clearer than a very long chain.

Rust gives you both styles.

---

# 55. `unwrap()` vs `?`

A useful rule:

Use:

```rust
unwrap()
```

when:

> "If this fails, something is fundamentally wrong and I intentionally want to panic."

Use:

```rust
?
```

when:

> "If this fails, this function cannot continue, so I want to return the error to my caller."

For example:

```rust
let config = read_config().unwrap();
```

means:

```text
I expect this can never fail.
If I'm wrong -> panic.
```

while:

```rust
let config = read_config()?;
```

means:

```text
This operation can fail.
I don't handle the error here.
I'll propagate it to my caller.
```

---

# 56. `unwrap()` vs `expect()`

Prefer:

```rust
expect("config must exist")
```

over:

```rust
unwrap()
```

when you have a meaningful invariant.

For example:

```rust
let first = values
    .first()
    .expect("values must not be empty");
```

This communicates the assumption to someone reading the code.

---

# 57. Don't Use `unwrap()` Everywhere

This is tempting:

```rust
let file = File::open("config.toml").unwrap();
let text = read_to_string(file).unwrap();
let config = parse(text).unwrap();
```

But now any failure causes a panic.

For application code, a better pattern is:

```rust
let file = File::open("config.toml")?;
let text = read_to_string(file)?;
let config = parse(text)?;
```

Then the caller decides how to handle the error.

---

# 58. Error Handling Philosophy

A useful principle is:

> **Handle an error where you have enough context to make a meaningful decision.**

For example, a low-level function might return:

```rust
Result<Data, IoError>
```

instead of printing the error itself.

A higher-level function might decide:

```rust
match load_data() {
    Ok(data) => process(data),
    Err(error) => {
        eprintln!("Failed to load configuration: {error}");
    }
}
```

This keeps responsibilities separated.

---

# 59. Library vs Application Error Handling

For a library:

```rust
pub fn load() -> Result<Data, Error>
```

is often appropriate.

The library should usually return useful errors rather than immediately printing or terminating the process.

The application can then decide:

```rust
fn main() {
    if let Err(error) = run() {
        eprintln!("error: {error}");
        std::process::exit(1);
    }
}
```

Or:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    run()?;
    Ok(())
}
```

---

# 60. `if let Err(...)`

A convenient way to handle only the error case:

```rust
if let Err(error) = operation() {
    eprintln!("operation failed: {error}");
}
```

You don't need to write a complete `match` when you don't care about the successful value.

---

# 61. `Result<(), E>`

You will frequently see:

```rust
fn start_server() -> Result<(), ServerError>
```

Why `()`?

Because the operation has no meaningful success value.

It means:

```text
Ok(())
    -> succeeded

Err(error)
    -> failed
```

For example:

```rust
fn save_config(config: &Config) -> Result<(), IoError> {
    // ...
    Ok(())
}
```

---

# 62. `Result<T, E>` as a Contract

A function such as:

```rust
fn read_config() -> Result<Config, ConfigError>
```

communicates a lot:

```text
The function may succeed:
    -> Config

or fail:
    -> ConfigError
```

This is much more informative than:

```rust
fn read_config() -> Config
```

and then unexpectedly panicking.

---

# 63. `Option` as a Contract

Similarly:

```rust
fn find_user(id: u64) -> Option<User>
```

communicates:

```text
User exists
    -> Some(User)

User does not exist
    -> None
```

There is no need for an exception or `null`.

---

# 64. A Practical Decision Tree

When designing a function, ask:

```text
Can the value legitimately be absent?
        |
       yes
        |
        v
    Option<T>
```

If:

```text
Can the operation fail?
        |
       yes
        |
        v
   Result<T, E>
```

If you need to distinguish why it failed:

```text
Result<T, E>
```

is usually preferable to:

```text
Option<T>
```

---

# 65. Common Method Cheat Sheet

## `Option<T>`

### Inspect

```rust
is_some()
is_none()
is_some_and(...)
```

### Extract

```rust
unwrap()
expect(...)
unwrap_or(...)
unwrap_or_else(...)
unwrap_or_default()
```

### Transform

```rust
map(...)
map_or(...)
map_or_else(...)
filter(...)
flatten()
```

### Chain

```rust
and(...)
and_then(...)
or(...)
or_else(...)
```

### Convert

```rust
ok_or(...)
ok_or_else(...)
transpose()
```

### Debug

```rust
inspect(...)
```

### Propagate

```rust
?
```

---

# 66. `Result<T, E>` Cheat Sheet

### Inspect

```rust
is_ok()
is_err()
is_ok_and(...)
is_err_and(...)
```

### Extract success

```rust
unwrap()
expect(...)
unwrap_or(...)
unwrap_or_else(...)
```

### Extract error

```rust
unwrap_err()
expect_err(...)
```

### Transform success

```rust
map(...)
map_or(...)
map_or_else(...)
```

### Transform error

```rust
map_err(...)
inspect_err(...)
```

### Chain

```rust
and(...)
and_then(...)
or(...)
or_else(...)
```

### Convert

```rust
ok()
err()
transpose()
```

### Propagate

```rust
?
```

---

# 67. The Most Important Five Methods

If you are learning Rust, don't try to memorize every method immediately.

Start with these:

```rust
map()
map_err()
and_then()
or_else()
?
```

Understand these deeply.

For example:

```rust
operation()
    .map(transform_success)
    .map_err(transform_error)
    .and_then(next_operation)
    .or_else(fallback_operation)?;
```

These methods form a large part of Rust's functional-style error-handling patterns.

---

# 68. The Big Picture

`Option`:

```text
              Option<T>
                  |
          +-------+-------+
          |               |
       Some(T)           None
          |               |
       success          absence
```

`Result`:

```text
              Result<T,E>
                  |
          +-------+-------+
          |               |
        Ok(T)           Err(E)
          |               |
       success           failure
```

Transform success:

```text
map()
```

Transform failure:

```text
map_err()
```

Continue with another fallible operation:

```text
and_then()
```

Try an alternative after failure:

```text
or_else()
```

Propagate failure:

```text
?
```

Extract with a panic:

```text
unwrap()
expect()
```

Extract with a fallback:

```text
unwrap_or()
unwrap_or_else()
unwrap_or_default()
```

Convert between `Option` and `Result`:

```text
Option -> Result
    ok_or()
    ok_or_else()

Result -> Option
    ok()
    err()
```

---

# 69. A Final Example

Consider a configuration loader:

```rust
fn load_port(path: &str) -> Result<u16, ConfigError> {
    let text = std::fs::read_to_string(path)
        .map_err(ConfigError::Io)?;

    let config: Config = toml::from_str(&text)
        .map_err(ConfigError::Parse)?;

    let port = config
        .server
        .and_then(|server| server.port)
        .ok_or(ConfigError::MissingPort)?;

    if port == 0 {
        return Err(ConfigError::InvalidPort);
    }

    Ok(port)
}
```

Notice how several concepts work together:

```text
read_to_string()
       |
       | map_err()
       v
ConfigError
       |
       | ?
       v
text
       |
       v
toml::from_str()
       |
       | map_err()
       v
ConfigError
       |
       | ?
       v
Config
       |
       | and_then()
       v
Option<u16>
       |
       | ok_or()
       v
Result<u16, ConfigError>
       |
       v
     Ok(port)
```

This is the kind of code where Rust's `Option`, `Result`, combinators, and `?` really shine.

---

# 70. Key Takeaways

The most important mental model is:

```text
Option<T>
    = value may be absent

Result<T, E>
    = operation may fail and provides an error
```

For transformations:

```text
map()
    transform the success value

map_err()
    transform the error
```

For chaining:

```text
and_then()
    continue when successful

or_else()
    provide a fallback when failed
```

For extraction:

```text
unwrap()
    panic on failure

expect()
    panic with a useful message

unwrap_or()
    use a fallback

unwrap_or_else()
    calculate a fallback lazily
```

For propagation:

```text
?
    success -> continue
    failure -> return failure
```

And the most important distinction to remember:

```text
unwrap()
    "I expect this cannot fail."

?
    "This can fail, and my caller should handle it."
```

Rust's `Option` and `Result` are not just containers. Together with methods such as `map`, `map_err`, `and_then`, `or_else`, and the `?` operator, they form a composable model for representing, transforming, and propagating absence and failure without relying on exceptions or null values.
