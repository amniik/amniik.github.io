---
title: "Memory, Ownership, and Borrowing in Rust"
categories:
  - Learning Notes
  - Rust
tags: [rust, memory-management, ownership, borrowing, lifetimes, pointers, systems-programming]
description: "A concise guide about memory in Rust."

toc: true
---
Rust's memory model is built around ownership, borrowing, and lifetimes. These concepts allow Rust to provide memory safety without requiring a garbage collector.

To understand them, it helps to distinguish three things:

- **Memory:** where values are stored.
- **Ownership:** which value or variable is responsible for a resource.
- **Borrowing:** temporarily accessing a value without taking ownership of it.

# 1. Stack and Heap

Programs use different memory regions for different purposes. Two important regions are the stack and the heap.

## 1.1 Stack

The stack stores function-call frames and many local values whose sizes are known at compile time.

```rust
fn main() {
    let x = 10;
    let y = 20;

    let result = x + y;
}
```

The local integers can typically be stored directly in the current stack frame.

The stack is efficient because allocating and releasing stack-frame space is usually straightforward.

## 1.2 Heap

The heap supports dynamically sized allocations whose storage is managed at runtime.

```rust
let text = String::from("hello");
let numbers = vec![10, 20, 30];
```

A `String` or `Vec<T>` typically contains metadata inline while owning a separate heap allocation for its elements.

Conceptually:

```text
Stack                         Heap

text                          String's bytes
+----------+                  +-------------------+
| pointer  | ---------------->| h e l l o         |
| length   |                  +-------------------+
| capacity |
+----------+
```

The exact physical placement of local variables is not guaranteed by Rust; the diagram illustrates the logical relationship.

**Important:** a heap allocation does not mean the variable owning it is itself necessarily on the heap.

# 2. Ownership

Ownership determines which value is responsible for an owned resource.

Rust's central ownership rules are:

1. Every owned value has an owner.
2. A value has one owner at a time.
3. When the owner is dropped, its owned resources are released, unless ownership has been transferred or shared through an appropriate abstraction.

Consider:

```rust
let text = String::from("hello");
```

The variable `text` owns the `String`.

When `text` goes out of scope, the `String` is dropped and its heap allocation is released.

```rust
{
    let text = String::from("hello");

    println!("{text}");
} // text is dropped here
```

This is deterministic resource management.

It applies not only to memory but also to resources such as files, locks, and sockets when their types implement appropriate cleanup behavior through `Drop`.

# 3. Move Semantics

When a non-`Copy` value is assigned to another variable, ownership is usually moved rather than duplicated.

```rust
let first = String::from("hello");
let second = first;

println!("{second}");

// println!("{first}"); // Compile error: value was moved
```

After the assignment, `second` owns the `String`.

Conceptually:

```text
Before:

first ───────> String allocation

After:

second ──────> String allocation
first         invalid for further use
```

The heap allocation is not duplicated. Ownership of the resource changes.

This prevents two independent owners from accidentally freeing the same allocation.

A moved variable may still be assigned a new value later:

```rust
let first = String::from("hello");
let second = first;

let first = String::from("world");
```

Here, the second `let first` introduces a new binding through shadowing.

# 4. Copy Types

Some types implement the `Copy` trait. Assigning them copies their value instead of invalidating the original binding.

```rust
let x = 10;
let y = x;

println!("{x}");
println!("{y}");
```

Both variables remain usable.

Common examples include integers, booleans, floating-point numbers, and suitable tuples and arrays whose element types are all `Copy`.

A struct can implement `Copy` only when all its fields are `Copy` and the other trait requirements are satisfied.

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}
```

A struct containing a `String` cannot implement `Copy`, because copying the `String` would require duplicating its ownership of a heap allocation.

# 5. `Clone` vs `Copy`

`Clone` explicitly duplicates a value according to the type's implementation.

```rust
let first = String::from("hello");
let second = first.clone();

println!("{first}");
println!("{second}");
```

For `String`, `clone()` creates a separate owned string allocation containing the same text.

Compare:

```rust
let x = 10;
let y = x; // Copy

let first = String::from("hello");
let second = first.clone(); // Explicit clone
```

| Trait | Meaning | Typical cost |
|---|---|---|
| `Copy` | Implicit value duplication | Usually small, fixed-size copy |
| `Clone` | Explicit duplication | May allocate or perform substantial work |
| Move | Transfer ownership | Often avoids copying the resource |

Do not assume every `clone()` allocates memory. The cost depends on the type's implementation.

# 6. Borrowing

Borrowing allows code to access a value without taking ownership of it.

A reference is created using `&`.

```rust
let text = String::from("hello");

let reference = &text;

println!("{reference}");
println!("{text}");
```

The reference borrows the string. The original variable remains its owner.

Conceptually:

```text
text ─────────> String allocation
                  ^
                  |
reference --------+
```

The reference does not become an independent owner of the allocation.

Borrowing is useful when a function needs to inspect a value without consuming it.

# 7. Immutable References

An immutable reference provides read-only access through that reference.

```rust
fn print_length(text: &String) {
    println!("{}", text.len());
}

fn main() {
    let text = String::from("hello");

    print_length(&text);

    println!("{text}");
}
```

The function borrows the string rather than taking ownership of it.

For functions that only need to read text, `&str` is usually more flexible than `&String`:

```rust
fn print_length(text: &str) {
    println!("{}", text.len());
}

let text = String::from("hello");

print_length(&text);
print_length("world");
```

Both a `String` borrow and a string literal can be used.

# 8. Mutable References

A mutable reference, written `&mut T`, allows mutation through the reference.

```rust
fn append_world(text: &mut String) {
    text.push_str(" world");
}

fn main() {
    let mut text = String::from("hello");

    append_world(&mut text);

    println!("{text}");
}
```

The owner remains the same; the function temporarily receives mutable access.

Two key requirements apply:

- The value must be accessed through a mutable binding when creating the mutable reference.
- Rust enforces exclusive mutable borrowing while the mutable reference is active.

# 9. The Borrowing Rules

Rust enforces these core reference rules:

- You can have multiple immutable references to a value at the same time.
- You can have a mutable reference to a value, provided there are no conflicting active references.
- References must not outlive the data they refer to.

For example, multiple immutable references are allowed:

```rust
let text = String::from("hello");

let first = &text;
let second = &text;

println!("{first} {second}");
```

But conflicting active borrows are rejected:

```rust
let mut text = String::from("hello");

let reference = &text;

// let mutable_reference = &mut text; // Error while reference is used

println!("{reference}");
```

The compiler uses the references' actual usage to determine their active lifetimes. This is why non-lexical lifetimes allow a borrow to end before the end of its enclosing block.

For example:

```rust
let mut text = String::from("hello");

let reference = &text;
println!("{reference}");

// The immutable borrow is no longer needed.
let mutable_reference = &mut text;
mutable_reference.push('!');
```

The borrow ends after its last use, allowing the mutable borrow afterward.

# 10. Why Does Rust Restrict Mutable References?

Consider a language that allows multiple unrestricted mutable references to the same memory.

One part of a program could modify a value while another part assumes it has not changed. This can cause inconsistent behavior and, in unsafe circumstances, memory-safety violations.

Rust's borrowing rules prevent such conflicting access in safe Rust.

Conceptually:

```text
Many readers OR one writer
```

This is a useful mental model, although the precise rule is about conflicting references while their borrows are active.

# 11. Functions and Ownership

A function parameter takes ownership if it accepts an owned value.

```rust
fn consume(text: String) {
    println!("{text}");
}

fn main() {
    let text = String::from("hello");

    consume(text);

    // println!("{text}"); // Error: moved into consume
}
```

If the function accepts a reference, ownership remains with the caller:

```rust
fn inspect(text: &String) {
    println!("{text}");
}

let text = String::from("hello");

inspect(&text);

println!("{text}");
```

A useful API-design guideline:

- Use `T` when the function should take ownership.
- Use `&T` when the function only needs to inspect a value.
- Use `&mut T` when the function needs to modify a borrowed value.

For strings, prefer `&str` over `&String` when you only need to read text.

# 12. Returning Ownership

A function can return an owned value to its caller.

```rust
fn create_message() -> String {
    String::from("hello")
}

let message = create_message();
```

The returned `String` becomes owned by `message`.

You can also transfer ownership into a function and return it:

```rust
fn process(text: String) -> String {
    text.to_uppercase()
}

let text = String::from("hello");
let result = process(text);
```

The original binding is moved into `process`, and the returned value is owned by `result`.

However, returning a reference to a local variable is not allowed:

```rust,ignore
fn invalid() -> &String {
    let text = String::from("hello");
    &text
}
```

The local `text` is dropped when the function returns, so its reference would be invalid.

Return an owned value instead, or return a reference to data whose lifetime is guaranteed by the caller or another valid owner.

# 13. Lifetimes

A lifetime describes how long a reference is valid.

Consider:

```rust
fn longer<'a>(first: &'a str, second: &'a str) -> &'a str {
    if first.len() >= second.len() {
        first
    } else {
        second
    }
}
```

The lifetime `'a` expresses that the returned reference is valid for a lifetime shared by the input references.

It does not extend the lifetime of either string. Instead, it restricts how the returned reference may be used.

Usage:

```rust
let first = String::from("hello");
let second = String::from("world!");

let result = longer(&first, &second);

println!("{result}");
```

The compiler ensures the result is not used beyond the validity of the data it references.

## 13.1 Lifetime annotations on structs

A struct that stores a reference usually needs a lifetime parameter:

```rust
struct User<'a> {
    name: &'a str,
}
```

This says that the reference stored in `User` must remain valid for the relevant lifetime `'a`.

```rust
let name = String::from("Amir");

let user = User {
    name: &name,
};
```

The lifetime parameter does not mean the struct owns the string. The field is still a borrow.

## 13.2 Lifetime elision

Rust can infer lifetimes in many common cases, so you do not always need to write them explicitly.

For example:

```rust
fn first_character(text: &str) -> Option<char> {
    text.chars().next()
}
```

The function returns an owned `Option<char>`, so the returned value does not borrow from `text`.

For a function returning a reference:

```rust
fn identity(text: &str) -> &str {
    text
}
```

Rust can infer the relationship between the input and output reference using lifetime elision rules.

# 14. Dangling References

A dangling reference points to data that is no longer valid.

Safe Rust prevents ordinary code from creating dangling references.

```rust,ignore
fn dangling() -> &String {
    let text = String::from("hello");
    &text
}
```

This is rejected because `text` is dropped at the end of the function.

In contrast:

```rust
fn valid() -> String {
    String::from("hello")
}
```

returns ownership of the string, so the caller can safely use it.

# 15. Slices

A slice is a borrowed view into a contiguous sequence of elements.

For arrays and vectors:

```rust
let numbers = vec![10, 20, 30, 40];

let slice: &[i32] = &numbers[1..3];

println!("{slice:?}");
```

Output:

```text
[20, 30]
```

The slice does not own the elements. It provides access to a selected range.

String slices work similarly:

```rust
let text = String::from("hello");

let slice: &str = &text[0..2];

println!("{slice}");
```

Output:

```text
he
```

String slicing uses UTF-8 byte offsets, so indices must lie on valid character boundaries. For general Unicode text, iterating with `.chars()` may be more appropriate.

# 16. Smart Pointers

Rust's standard library provides types that manage ownership in different ways.

## 16.1 `Box<T>`

`Box<T>` owns a value stored on the heap.

```rust
let value = Box::new(42);

println!("{value}");
```

A box is useful for heap allocation, recursive types, and cases where indirection is needed.

Example of a recursive type:

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

The `Box<List>` provides the indirection required for the recursive type to have a finite size.

## 16.2 `Rc<T>`

`Rc<T>` provides shared ownership through reference counting for single-threaded code.

```rust
use std::rc::Rc;

let first = Rc::new(String::from("hello"));
let second = Rc::clone(&first);

println!("{first}");
println!("{second}");
```

Both `Rc` values refer to the same allocation. The allocation is released when the last strong `Rc` owner is dropped.

`Rc<T>` is not designed for sharing ownership across threads.

## 16.3 `Arc<T>`

`Arc<T>` provides thread-safe reference-counted shared ownership.

```rust
use std::sync::Arc;
use std::thread;

let value = Arc::new(42);
let shared = Arc::clone(&value);

let handle = thread::spawn(move || {
    println!("{shared}");
});

handle.join().unwrap();
```

`Arc<T>` makes shared ownership thread-safe, but it does not automatically make arbitrary mutation of `T` safe.

For shared mutation, consider synchronization primitives such as `Mutex<T>` or `RwLock<T>`.

## 16.4 `RefCell<T>`

`RefCell<T>` enforces borrowing rules at runtime rather than entirely at compile time.

```rust
use std::cell::RefCell;

let value = RefCell::new(10);

*value.borrow_mut() += 1;

println!("{}", value.borrow());
```

If incompatible borrows occur at runtime, `RefCell` can panic.

It is useful in single-threaded code when mutation through a shared reference is necessary and the borrowing relationships cannot be expressed conveniently with ordinary references.

## 16.5 `Mutex<T>`

`Mutex<T>` provides synchronized access to a value across threads.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));

let shared = Arc::clone(&counter);

let handle = thread::spawn(move || {
    let mut value = shared.lock().unwrap();
    *value += 1;
});

handle.join().unwrap();

println!("{}", *counter.lock().unwrap());
```

`Arc` provides shared ownership; `Mutex` coordinates access to the protected value.

These types solve different problems and are often used together.

# 17. Interior Mutability

Interior mutability allows mutation through a shared reference, using abstractions that enforce their safety rules internally.

Important examples:

| Type | Thread-safe sharing | Borrow checking |
|---|---|---|
| `Cell<T>` | No | Operations designed around copying or replacing values |
| `RefCell<T>` | No | Runtime borrow checking |
| `Mutex<T>` | Yes, when its type bounds are satisfied | Lock-based synchronization |
| `RwLock<T>` | Yes, when its type bounds are satisfied | Lock-based reader/writer synchronization |
| `AtomicUsize` and other atomics | Yes | Atomic operations |

For example, `Cell<T>` is useful for small `Copy` values:

```rust
use std::cell::Cell;

let value = Cell::new(10);

value.set(20);

println!("{}", value.get());
```

The value is mutated through the `Cell` abstraction even though the `Cell` binding itself is not declared `mut`.

Interior mutability should be used when its semantics are appropriate, not as a way to bypass the borrowing model indiscriminately.

# 18. `Drop` and Deterministic Cleanup

Rust runs a value's destructor when it is dropped.

You can define cleanup behavior with the `Drop` trait:

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping {}", self.name);
    }
}

fn main() {
    let resource = Resource {
        name: String::from("socket"),
    };

    println!("Using resource");
}
```

When `resource` goes out of scope, Rust calls its `drop` method.

This is the basis of RAII-style resource management: resources are tied to owned values, and cleanup occurs when those values are dropped.

For files, locks, and many other resources, standard-library types already implement appropriate cleanup behavior.

# 19. Ownership and Collections

Collections such as `Vec<T>`, `String`, and `HashMap<K, V>` own their contents.

```rust
let first = String::from("hello");
let values = vec![first];

// first was moved into the vector.
// println!("{first}"); // Compile error
```

Borrowing lets you inspect collection elements without taking ownership:

```rust
let values = vec![String::from("hello"), String::from("world")];

for value in &values {
    println!("{value}");
}

println!("{values:?}");
```

Using a mutable reference allows mutation:

```rust
let mut values = vec![10, 20, 30];

for value in &mut values {
    *value += 1;
}

println!("{values:?}");
```

The collection continues to own its elements.

# 20. Common Ownership Patterns

## Borrow when reading

```rust
fn print_message(message: &str) {
    println!("{message}");
}
```

Use this when the function only needs temporary access to text.

## Borrow mutably when modifying

```rust
fn increment(value: &mut i32) {
    *value += 1;
}
```

Use this when the caller should retain ownership but allow mutation.

## Take ownership when storing or consuming

```rust
struct Config {
    path: String,
}

fn create_config(path: String) -> Config {
    Config { path }
}
```

The function transfers the string into the resulting configuration.

## Return ownership when creating data

```rust
fn default_name() -> String {
    String::from("guest")
}
```

This avoids returning a reference to a local value.

# 21. Common Mistakes

### Using a moved value

```rust,ignore
let text = String::from("hello");
let other = text;

println!("{text}");
```

**Fix:** borrow the value if ownership should remain with the original variable, or clone when an independent copy is actually needed.

### Returning a reference to a local variable

```rust,ignore
fn make_text() -> &str {
    let text = String::from("hello");
    &text
}
```

**Fix:** return `String`, or borrow data whose lifetime is guaranteed to outlive the returned reference.

### Borrowing mutably while an immutable borrow is active

```rust,ignore
let mut text = String::from("hello");
let reference = &text;
let mutable_reference = &mut text;

println!("{reference}");
```

**Fix:** end the immutable borrow's use before creating the mutable borrow.

### Using `clone()` to silence every ownership error

Cloning can be correct, but it may allocate memory or perform expensive work. First determine whether a borrow or an ownership transfer is more appropriate.

# 22. Practical Mental Model

Think about the relationship between ownership and borrowing this way:

```text
Ownership:
    Who is responsible for the value?

Borrowing:
    Who can access the value temporarily?

Lifetime:
    For how long is that access valid?

Mutability:
    Is the access read-only or allowed to modify the value?

Drop:
    What happens when the owner is no longer needed?
```

These concepts work together.

For example, a `String` owns its heap allocation. A `&str` borrows some of its UTF-8 contents. The borrow checker ensures the reference does not outlive the relevant data, while ownership ensures the allocation is released when its owner is dropped.

# Key Takeaways

- Rust uses ownership and borrowing to enforce memory safety without a garbage collector.
- A value has one owner at a time, except when shared-ownership abstractions are used.
- Moves transfer ownership; `Copy` duplicates eligible values implicitly, and `Clone` explicitly duplicates values according to the type's implementation.
- `&T` provides shared borrowing; `&mut T` provides exclusive mutable borrowing.
- References cannot outlive the data they reference.
- Lifetimes describe valid borrowing relationships; they do not extend the lifetime of data.
- Slices such as `&str` and `&[T]` provide borrowed views without taking ownership.
- `Box<T>`, `Rc<T>`, and `Arc<T>` provide different ownership and allocation models.
- `RefCell<T>`, `Mutex<T>`, and `RwLock<T>` support interior mutability with different safety and synchronization mechanisms.
- `Drop` enables deterministic resource cleanup.
- Good API design makes ownership requirements clear: borrow for temporary access, borrow mutably for modification, and take ownership when the function needs to consume or retain a value.
