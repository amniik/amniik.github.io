---
title: "Rust Notes: match, if let, while let, and let-else"
categories:
  - Learning Notes
  - Rust
tags: [rust, pattern-matching, match, if-let, while-let, ownership]
description: "A practical reference to Rust pattern matching."

toc: true
---

# Rust Pattern Matching: `match`, `if let`, `while let`, and `let-else`

Rust's `match`, `if let`, `while let`, and `let-else` are all built around the same fundamental feature:

> **pattern matching**

Once patterns make sense, these constructs become much easier to understand.

The most important idea is:

```text
value
  |
  v
pattern
  |
  +---- matches ----> execute code
  |
  +---- doesn't match -> another branch / stop
```

For example:

```rust
let value = Some(10);

if let Some(x) = value {
    println!("{x}");
}
```

Here:

```rust
Some(x)
```

is a **pattern**.

If the value is:

```rust
Some(10)
```

the pattern matches and `x` becomes `10`.

If the value is:

```rust
None
```

the pattern does not match.

---

# 1. `match`

`match` compares a value against multiple patterns.

Basic syntax:

```rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

Example:

```rust
let number = 3;

match number {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("something else"),
}
```

The patterns are:

```text
1
2
3
_
```

Rust checks the patterns from top to bottom.

The **first matching pattern** is executed.

---

# 2. `match` Must Be Exhaustive

One of the most important properties of `match` is that it must handle every possible case.

This does not compile:

```rust
let number = 3;

match number {
    1 => println!("one"),
    2 => println!("two"),
}
```

because `number` could be anything other than `1` or `2`.

Usually `_` is used to handle everything else:

```rust
match number {
    1 => println!("one"),
    2 => println!("two"),
    _ => println!("other"),
}
```

This exhaustive checking is one of the major benefits of `match`.

---

# 3. `match` Is an Expression

`match` does not only execute statements.

It can return a value.

```rust
let number = 3;

let message = match number {
    1 => "one",
    2 => "two",
    3 => "three",
    _ => "other",
};

println!("{message}");
```

Every branch must produce a compatible type.

For example:

```rust
let result = match number {
    1 => 10,
    2 => 20,
    _ => 30,
};
```

The result is:

```rust
i32
```

---

# 4. Match Arms Can Have Blocks

A match arm can contain multiple statements:

```rust
match number {
    1 => {
        println!("number is one");
        println!("special case");
    }

    2 => {
        println!("number is two");
    }

    _ => {
        println!("other");
    }
}
```

The last expression of the block becomes the value of the arm:

```rust
let result = match number {
    1 => {
        println!("one");
        100
    }

    _ => {
        println!("other");
        0
    }
};
```

---

# 5. `_` Wildcard Pattern

The `_` pattern matches anything.

```rust
match number {
    1 => println!("one"),
    _ => println!("not one"),
}
```

You can think of `_` as:

```text
anything else
```

It does not bind the value to a variable.

For example:

```rust
match value {
    Some(x) => println!("{x}"),
    _ => println!("something else"),
}
```

---

# 6. Binding a Value in a Pattern

A variable name in a pattern binds the matched value.

```rust
let value = Some(42);

match value {
    Some(x) => println!("{x}"),
    None => println!("nothing"),
}
```

When the value is:

```rust
Some(42)
```

the pattern:

```rust
Some(x)
```

matches and:

```text
x = 42
```

---

# 7. Matching `Option`

`Option<T>` is one of the most common places to use `match`.

```rust
let value: Option<i32> = Some(10);

match value {
    Some(x) => println!("value = {x}"),
    None => println!("no value"),
}
```

Because `Option` has only two variants:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

the match handles both possibilities.

---

# 8. Matching `Result`

`Result<T, E>` is another extremely common use.

```rust
let result: Result<i32, String> = Ok(10);

match result {
    Ok(value) => println!("success: {value}"),
    Err(error) => println!("error: {error}"),
}
```

This makes error handling explicit.

A common pattern:

```rust
match std::fs::read_to_string("file.txt") {
    Ok(content) => println!("{content}"),
    Err(error) => println!("failed: {error}"),
}
```

---

# 9. Matching Enums

Suppose we have:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

We can match every variant:

```rust
match message {
    Message::Quit => {
        println!("quit");
    }

    Message::Move { x, y } => {
        println!("move to ({x}, {y})");
    }

    Message::Write(text) => {
        println!("text: {text}");
    }

    Message::ChangeColor(r, g, b) => {
        println!("RGB: {r}, {g}, {b}");
    }
}
```

This is one of the biggest strengths of Rust's pattern matching.

---

# 10. Destructuring Tuples

Patterns can destructure tuples.

```rust
let point = (10, 20);

match point {
    (0, 0) => println!("origin"),
    (x, 0) => println!("on x axis: {x}"),
    (0, y) => println!("on y axis: {y}"),
    (x, y) => println!("point ({x}, {y})"),
}
```

The pattern:

```rust
(x, y)
```

extracts both values.

---

# 11. Destructuring Structs

Given:

```rust
struct User {
    name: String,
    age: u32,
}
```

we can write:

```rust
let user = User {
    name: String::from("Amir"),
    age: 29,
};

match user {
    User { name, age } => {
        println!("{name} is {age}");
    }
}
```

You can also match specific fields:

```rust
match user {
    User { age: 29, .. } => {
        println!("age is 29");
    }

    _ => {
        println!("different age");
    }
}
```

---

# 12. `..` in Patterns

`..` means:

> Ignore the remaining fields.

For structs:

```rust
match user {
    User {
        name,
        ..
    } => {
        println!("{name}");
    }
}
```

For tuples:

```rust
let tuple = (1, 2, 3, 4, 5);

match tuple {
    (first, ..) => println!("{first}"),
}
```

For arrays:

```rust
let values = [1, 2, 3, 4, 5];

match values {
    [first, ..] => println!("{first}"),
}
```

---

# 13. Matching Multiple Values with `|`

The `|` operator creates an **or-pattern**.

```rust
match number {
    1 | 2 | 3 => println!("small"),
    4 | 5 | 6 => println!("medium"),
    _ => println!("large"),
}
```

It means:

```text
1 OR 2 OR 3
```

You can use it with enum variants:

```rust
match value {
    Some(1) | Some(2) => println!("one or two"),
    Some(x) => println!("{x}"),
    None => println!("none"),
}
```

---

# 14. Binding with Or-Patterns

Bindings in an or-pattern must be consistent.

For example:

```rust
match value {
    Some(1) | Some(2) => println!("one or two"),
    Some(x) => println!("{x}"),
    None => {}
}
```

Both alternatives in:

```rust
Some(1) | Some(2)
```

do not introduce bindings, so this is straightforward.

If alternatives introduce bindings, they must bind compatible names and types.

---

# 15. Matching Ranges

You can match ranges.

For integers:

```rust
match number {
    1..=5 => println!("1 through 5"),
    6..=10 => println!("6 through 10"),
    _ => println!("other"),
}
```

`..=` is an inclusive range.

So:

```rust
1..=5
```

means:

```text
1, 2, 3, 4, 5
```

---

# 16. Character Ranges

Ranges are also useful with `char`.

```rust
match character {
    'a'..='z' => println!("lowercase"),
    'A'..='Z' => println!("uppercase"),
    '0'..='9' => println!("digit"),
    _ => println!("other"),
}
```

---

# 17. Match Guards

A match pattern can have an additional condition called a **guard**.

Syntax:

```rust
PATTERN if CONDITION => EXPRESSION
```

Example:

```rust
match number {
    x if x > 10 => println!("greater than 10"),
    x => println!("{x}"),
}
```

The pattern:

```rust
x
```

matches the value.

Then the guard checks:

```rust
x > 10
```

---

# 18. Match Guards with `Option`

```rust
match value {
    Some(x) if x > 10 => println!("large: {x}"),
    Some(x) => println!("small: {x}"),
    None => println!("none"),
}
```

The first arm only matches when:

```text
value = Some(x)
AND
x > 10
```

---

# 19. Match Guards with `|`

You can combine patterns and conditions:

```rust
match number {
    1 | 2 | 3 if number > 1 => {
        println!("2 or 3");
    }

    _ => {}
}
```

Be careful with the precedence and meaning of guards on or-patterns.

For complex conditions, separate match arms are often easier to understand.

---

# 20. `@` Bindings

The `@` operator lets you both:

1. test a pattern
2. bind the matched value

Example:

```rust
match number {
    value @ 1..=5 => {
        println!("small number: {value}");
    }

    _ => {}
}
```

Without `@`, you can match:

```rust
1..=5
```

but you don't automatically get the matched value as a named variable.

With:

```rust
value @ 1..=5
```

you get both.

---

# 21. `@` with Enums

```rust
match message {
    msg @ Message::Write(_) => {
        println!("message = {msg:?}");
    }

    _ => {}
}
```

The whole matching value is bound to:

```rust
msg
```

while still requiring it to match:

```rust
Message::Write(_)
```

---

# 22. `ref` and References

Patterns can also work with references.

For example:

```rust
let value = 10;
let reference = &value;

match reference {
    &x => println!("{x}"),
}
```

Modern Rust code often uses match ergonomics, so explicit `ref` is less common than it used to be.

For example:

```rust
let value = Some(String::from("hello"));

match &value {
    Some(text) => println!("{text}"),
    None => {}
}
```

Here `text` is automatically bound appropriately through match ergonomics.

---

# 23. `if let`

`if let` is useful when you care about only one pattern.

Instead of:

```rust
match value {
    Some(x) => {
        println!("{x}");
    }

    None => {}
}
```

you can write:

```rust
if let Some(x) = value {
    println!("{x}");
}
```

The mental model is:

```text
if the value matches this pattern
    run this block
otherwise
    do nothing
```

---

# 24. Basic `if let` Syntax

```rust
if let PATTERN = VALUE {
    // pattern matched
}
```

Example:

```rust
if let Some(value) = option {
    println!("{value}");
}
```

---

# 25. `if let` with `else`

You can handle the non-matching case:

```rust
if let Some(value) = option {
    println!("value = {value}");
} else {
    println!("no value");
}
```

Conceptually:

```rust
if let PATTERN = VALUE {
    // matched
} else {
    // didn't match
}
```

This is roughly equivalent to:

```rust
match VALUE {
    PATTERN => {
        // matched
    }

    _ => {
        // didn't match
    }
}
```

---

# 26. `if let` with `else if`

You can combine normal conditions and pattern matching:

```rust
if let Some(x) = value {
    println!("some: {x}");
} else if condition {
    println!("condition is true");
} else {
    println!("nothing");
}
```

You can also use another `if let`:

```rust
if let Some(x) = value {
    println!("some: {x}");
} else if let Some(y) = other {
    println!("other: {y}");
} else {
    println!("nothing");
}
```

---

# 27. `if let` Chains

Modern Rust also supports combining `let` conditions with other conditions.

Example:

```rust
if let Some(x) = value && x > 10 {
    println!("large value: {x}");
}
```

You can combine multiple conditions:

```rust
if let Some(x) = value
    && x > 10
    && x < 100
{
    println!("between 10 and 100");
}
```

This is useful when pattern matching is only one part of the condition.

---

# 28. `if let` with Multiple Patterns

Or-patterns can be used:

```rust
if let Some(1 | 2 | 3) = value {
    println!("small");
}
```

You can also write:

```rust
if let Some(x @ 1..=5) = value {
    println!("{x}");
}
```

---

# 29. When Should You Use `match` vs `if let`?

Use `match` when:

```text
you need to handle multiple cases
```

Example:

```rust
match result {
    Ok(value) => println!("success: {value}"),
    Err(error) => println!("error: {error}"),
}
```

Use `if let` when:

```text
you care primarily about one case
```

Example:

```rust
if let Ok(value) = result {
    println!("success: {value}");
}
```

The trade-off is important:

```text
match
    exhaustive
    explicit
    better for many cases

if let
    concise
    convenient for one case
    ignores other cases
```

---

# 30. `while let`

`while let` repeatedly matches a pattern.

Syntax:

```rust
while let PATTERN = VALUE {
    // body
}
```

The loop continues while the pattern matches.

As soon as the pattern does not match, the loop stops.

---

# 31. Basic `while let` Example

A classic example is consuming an iterator:

```rust
let mut iterator = vec![1, 2, 3].into_iter();

while let Some(value) = iterator.next() {
    println!("{value}");
}
```

Execution:

```text
next() -> Some(1) -> body
next() -> Some(2) -> body
next() -> Some(3) -> body
next() -> None    -> stop
```

This is why `while let` is especially useful with `Option`.

---

# 32. `while let` with `Option`

```rust
let mut value = Some(10);

while let Some(x) = value {
    println!("{x}");

    value = None;
}
```

The loop continues while:

```text
value == Some(...)
```

and stops when:

```text
value == None
```

---

# 33. `while let` with `VecDeque`

This is a very useful real-world pattern:

```rust
use std::collections::VecDeque;

let mut queue = VecDeque::from([
    "task1",
    "task2",
    "task3",
]);

while let Some(task) = queue.pop_front() {
    println!("processing {task}");
}
```

The loop naturally expresses:

```text
while the queue contains a task
    remove the task
    process it
```

---

# 34. `while let` with Channels

`while let` is also common when receiving messages.

For example:

```rust
while let Some(message) = receiver.recv().await {
    println!("{message}");
}
```

This is particularly common in async Rust.

The loop means:

```text
while recv() returns Some(message)
    process message

when recv() returns None
    stop
```

---

# 35. `while let` vs `loop + match`

This:

```rust
while let Some(value) = iterator.next() {
    println!("{value}");
}
```

can conceptually be written as:

```rust
loop {
    match iterator.next() {
        Some(value) => {
            println!("{value}");
        }

        None => {
            break;
        }
    }
}
```

`while let` is simply much cleaner when the loop condition itself is a pattern.

---

# 36. `while let` with Guards

`while let` is primarily pattern matching, but if additional conditions are required, a normal `loop` plus `match` or `if` is often clearer.

For example:

```rust
loop {
    match iterator.next() {
        Some(value) if value > 10 => {
            println!("{value}");
        }

        Some(_) => {
            break;
        }

        None => {
            break;
        }
    }
}
```

This can be clearer than trying to make the loop condition overly complicated.

---

# 37. `let-else`

`let-else` is another pattern-matching construct.

It is especially useful when you want to extract a value or return early if the pattern doesn't match.

Syntax:

```rust
let PATTERN = VALUE else {
    // must diverge
};
```

Example:

```rust
let Some(value) = option else {
    return;
};

println!("{value}");
```

If the pattern matches:

```text
value is available
execution continues
```

If it does not:

```text
else block executes
execution must leave the current flow
```

---

# 38. Why `let-else` Is Useful

Compare:

```rust
if let Some(value) = option {
    println!("{value}");
} else {
    return;
}
```

with:

```rust
let Some(value) = option else {
    return;
};

println!("{value}");
```

The second form avoids an extra indentation level.

This is particularly useful for validation:

```rust
fn process(value: Option<String>) -> Option<usize> {
    let Some(value) = value else {
        return None;
    };

    Some(value.len())
}
```

---

# 39. `let-else` with `Result`

A common pattern:

```rust
fn process() -> Result<(), Error> {
    let Ok(value) = get_value() else {
        return Err(Error::InvalidValue);
    };

    println!("{value}");

    Ok(())
}
```

It is useful when the successful path should remain at the main indentation level.

---

# 40. `let` Itself Uses Patterns

This is an important concept.

When you write:

```rust
let x = 10;
```

`x` is actually a pattern.

The syntax is conceptually:

```text
let PATTERN = VALUE;
```

For example:

```rust
let (x, y) = (10, 20);
```

The pattern is:

```rust
(x, y)
```

It destructures the tuple.

Another example:

```rust
let Some(value) = option;
```

This would only be valid when the compiler can guarantee the pattern cannot fail.

For potentially failing patterns, use:

```rust
if let
while let
match
let-else
```

---

# 41. Irrefutable vs Refutable Patterns

This distinction explains why these constructs exist.

An **irrefutable pattern** must always match.

Example:

```rust
let x = 10;
```

`x` can match any value.

A tuple pattern:

```rust
let (x, y) = (10, 20);
```

also always matches a two-element tuple.

A **refutable pattern** can fail.

For example:

```rust
Some(x)
```

can fail because the value could be:

```rust
None
```

Therefore:

```rust
if let Some(x) = value {
    // ...
}
```

is appropriate.

The mental model is:

```text
irrefutable
    cannot fail
    ordinary let

refutable
    may fail
    match
    if let
    while let
    let-else
```

---

# 42. Matching References

Suppose:

```rust
let value = Some(10);
let reference = &value;
```

You can match the reference:

```rust
match reference {
    Some(x) => println!("{x}"),
    None => println!("none"),
}
```

Rust's match ergonomics automatically handle the reference in many cases.

You can also explicitly match a reference:

```rust
match reference {
    &Some(x) => println!("{x}"),
    &None => println!("none"),
}
```

The explicit form makes the dereferencing behavior visible.

---

# 43. Matching Nested Structures

Patterns can be nested.

```rust
let value = Some(Ok(10));

match value {
    Some(Ok(x)) => println!("success: {x}"),
    Some(Err(e)) => println!("error: {e}"),
    None => println!("no value"),
}
```

This is one reason pattern matching is powerful.

You can inspect several layers of a data structure in one expression.

---

# 44. Nested `if let`

You can also use nested patterns:

```rust
if let Some(Ok(value)) = result {
    println!("{value}");
}
```

This matches only when:

```text
Some
  |
  +-- Ok
       |
       +-- value
```

---

# 45. Matching Strings

Rust does not normally pattern-match a `String` directly against string literals in the same way as enums or integers.

Usually you match a string slice:

```rust
match value.as_str() {
    "start" => println!("starting"),
    "stop" => println!("stopping"),
    _ => println!("unknown command"),
}
```

Or:

```rust
match value.as_str() {
    "get" | "read" => println!("read operation"),
    "set" | "write" => println!("write operation"),
    _ => println!("unknown"),
}
```

---

# 46. Matching Multiple Conditions

Sometimes `match` is cleaner than a long `if/else` chain.

Instead of:

```rust
if number == 1 {
    println!("one");
} else if number == 2 {
    println!("two");
} else if number == 3 {
    println!("three");
} else {
    println!("other");
}
```

you can write:

```rust
match number {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("other"),
}
```

And patterns can be much more expressive than equality checks.

---

# 47. Match as a State Machine

`match` is especially useful when working with state machines.

For example:

```rust
enum State {
    Starting,
    Running,
    Stopped,
}
```

Then:

```rust
match state {
    State::Starting => start(),
    State::Running => run(),
    State::Stopped => stop(),
}
```

This is a very natural Rust design because adding a new enum variant can cause the compiler to identify matches that need updating.

---

# 48. Common `Option` Patterns

## Extract only `Some`

```rust
if let Some(value) = option {
    println!("{value}");
}
```

## Handle both

```rust
match option {
    Some(value) => println!("{value}"),
    None => println!("none"),
}
```

## Loop until `None`

```rust
while let Some(value) = iterator.next() {
    println!("{value}");
}
```

## Early return on `None`

```rust
let Some(value) = option else {
    return;
};
```

These four constructs are closely related:

```text
match
    handle multiple cases

if let
    handle one matching case

while let
    repeatedly handle one matching case

let-else
    extract a value or leave the current flow
```

---

# 49. Common `Result` Patterns

## Handle both success and error

```rust
match result {
    Ok(value) => println!("{value}"),
    Err(error) => println!("{error}"),
}
```

## Handle only success

```rust
if let Ok(value) = result {
    println!("{value}");
}
```

## Early return on error

```rust
let Ok(value) = result else {
    return;
};
```

In real Rust code, the `?` operator is often even better when the goal is simply to propagate an error:

```rust
let value = result?;
```

But `let-else` is useful when you need custom handling or when the pattern is more specific than just `Ok`/`Err`.

---

# 50. Pattern Matching with Ownership

Patterns interact with ownership and borrowing.

For example:

```rust
let value = Some(String::from("hello"));

match value {
    Some(text) => println!("{text}"),
    None => {}
}
```

Here the `String` can be moved into `text`.

Afterwards, `value` may no longer be usable.

To avoid moving the inner value, match a reference:

```rust
let value = Some(String::from("hello"));

match &value {
    Some(text) => println!("{text}"),
    None => {}
}

println!("{value:?}");
```

The important question when reading a pattern is therefore:

> **Am I moving the value, borrowing it, or copying it?**

---

# 51. `match` and `ref`

Older Rust code may contain:

```rust
match value {
    Some(ref text) => {
        println!("{text}");
    }

    None => {}
}
```

The `ref` pattern binding creates a reference instead of moving the value.

Modern match ergonomics often allow simpler code:

```rust
match &value {
    Some(text) => {
        println!("{text}");
    }

    None => {}
}
```

When reading older Rust code, knowing `ref` is still useful.

---

# 52. Match Guards vs Patterns

Prefer a pattern when the condition naturally describes the structure:

```rust
match value {
    Some(x) => println!("{x}"),
    None => println!("none"),
}
```

Use a guard when the condition depends on a value:

```rust
match value {
    Some(x) if x > 10 => println!("large"),
    Some(x) => println!("small"),
    None => println!("none"),
}
```

Think:

```text
pattern
    describes structure

guard
    adds a boolean condition
```

---

# 53. `match` vs `if`

Use ordinary `if` when you are testing a boolean condition:

```rust
if x > 10 {
    println!("large");
}
```

Use `if let` when you are testing whether a value matches a pattern:

```rust
if let Some(x) = value {
    println!("{x}");
}
```

Use `match` when multiple patterns represent different cases:

```rust
match value {
    Some(x) if x > 10 => println!("large"),
    Some(x) => println!("small"),
    None => println!("none"),
}
```

---

# 54. `match` vs `if let` vs `while let` vs `let-else`

| Construct   | Purpose                                   |
| ----------- | ----------------------------------------- |
| `match`     | Handle multiple patterns                  |
| `if let`    | Handle one matching pattern conditionally |
| `while let` | Repeatedly handle a matching pattern      |
| `let-else`  | Extract a matching value or exit/diverge  |
| `if`        | Handle boolean conditions                 |

A useful mental model:

```text
                 Pattern Matching
                       |
        +--------------+--------------+
        |              |              |
      match         if let        while let
        |              |              |
    many cases      one case      repeat case
        |
     exhaustive

                 let-else
                     |
              extract or exit
```

---

# 55. Practical Decision Guide

When you see a value and need to decide what to do:

### Multiple cases?

Use:

```rust
match value {
    ...
}
```

### Only interested in one case?

Use:

```rust
if let PATTERN = value {
    ...
}
```

### Want to keep doing something while the pattern matches?

Use:

```rust
while let PATTERN = value {
    ...
}
```

### Want to extract a value and leave the function/block if it doesn't match?

Use:

```rust
let PATTERN = value else {
    return;
};
```

### Just checking a boolean?

Use:

```rust
if condition {
    ...
}
```

---

# 56. Syntax Cheat Sheet

## `match`

```rust
match value {
    PATTERN => expression,
    PATTERN => expression,
    _ => expression,
}
```

## Match with blocks

```rust
match value {
    PATTERN => {
        // statements
    }

    _ => {
        // statements
    }
}
```

## Binding

```rust
match value {
    Some(x) => println!("{x}"),
    _ => {}
}
```

## Or-pattern

```rust
match value {
    1 | 2 | 3 => {}
    _ => {}
}
```

## Range

```rust
match value {
    1..=10 => {}
    _ => {}
}
```

## Guard

```rust
match value {
    x if x > 10 => {}
    _ => {}
}
```

## `@` binding

```rust
match value {
    x @ 1..=10 => {}
    _ => {}
}
```

## Struct destructuring

```rust
match value {
    User { name, age } => {}
}
```

## Tuple destructuring

```rust
match value {
    (x, y) => {}
}
```

## Nested pattern

```rust
match value {
    Some(Ok(x)) => {}
    Some(Err(e)) => {}
    None => {}
}
```

## `if let`

```rust
if let PATTERN = value {
    // matched
}
```

## `if let` with `else`

```rust
if let PATTERN = value {
    // matched
} else {
    // not matched
}
```

## `if let` with `else if`

```rust
if let PATTERN = value {
    // ...
} else if condition {
    // ...
} else {
    // ...
}
```

## `if let` chain

```rust
if let Some(x) = value && x > 10 {
    // ...
}
```

## `while let`

```rust
while let PATTERN = value {
    // ...
}
```

## `let-else`

```rust
let PATTERN = value else {
    // must diverge
};
```

Examples:

```rust
let Some(value) = option else {
    return;
};
```

```rust
let Ok(value) = result else {
    return Err(error);
};
```

---

# 57. The Most Important Mental Model

Do not think of:

```rust
if let Some(x) = value
```

as strange syntax.

Read it as:

```text
"If value matches the pattern Some(x),
 bind the inside value to x,
 and execute the block."
```

Similarly:

```rust
while let Some(x) = iterator.next()
```

means:

```text
"Keep calling next().
If it returns Some(x), execute the body.
When it returns None, stop."
```

And:

```rust
let Some(x) = value else {
    return;
};
```

means:

```text
"Extract x if value is Some.
Otherwise leave this control-flow path."
```

Once you understand this, all three constructs become variations of the same idea:

```text
              PATTERN MATCHING
                     |
        +------------+------------+
        |            |            |
      match       if let       while let
        |            |            |
   many cases    one case     repeated case

                     +
                     
                  let-else
                     |
              extract or exit
```

---

# 58. Key Takeaways

* `match`, `if let`, `while let`, and `let-else` are all based on Rust's pattern-matching system.
* `match` is exhaustive.
* `if let` is useful when you care about one pattern.
* `while let` repeatedly executes code while a pattern matches.
* `let-else` extracts a value and handles failure by leaving the current control-flow path.
* `Some(x)` is a pattern, not just a value.
* `Ok(x)` and `Err(e)` are patterns.
* Tuple and struct destructuring are patterns.
* `_` matches anything without binding it.
* `|` creates or-patterns.
* `..` ignores remaining fields/elements.
* `if` tests boolean conditions; `if let` tests patterns.
* Match guards add additional boolean conditions to patterns.
* `@` lets you both match a pattern and bind the matched value.
* Patterns interact directly with ownership and borrowing.
* `match &value` is often useful when you want to inspect a value without moving it.
* `while let Some(x) = iterator.next()` is one of the most important iterator patterns to recognize.
* `let-else` is especially useful for early validation and keeping the successful path unindented.
* Understanding patterns is more important than memorizing the individual constructs.
* Once patterns are understood, `match`, `if let`, `while let`, and `let-else` become different ways of applying the same underlying mechanism.
