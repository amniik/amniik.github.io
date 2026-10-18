---
title: "Strings and String-Like Types in Rust"
categories:
  - Learning Notes
  - Rust
tags: [rust, structs, enums, pattern-matching, methods, data-types]
description: "A concise guide Enum and Structs in Rust."

toc: true
---

Rust uses `struct` and `enum` to define custom data types. They are fundamental building blocks for modeling application state, configuration, errors, commands, and operating-system resources.

- A `struct` groups related fields into one type.
- An `enum` defines a type that can represent one of several variants.

# 1. Structs

A struct groups related data into a single value.

## 1.1 Named-field structs

The most common form is a struct with named fields.

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    sign_in_count: u64,
}
```

Create an instance using field names:

```rust
let user = User {
    username: String::from("amir"),
    email: String::from("amir@example.com"),
    active: true,
    sign_in_count: 1,
};

println!("{}", user.username);
println!("{}", user.active);
```

Each field has a declared type. Every instance of `User` has the same fields and field types.

## 1.2 Mutable structs

Rust distinguishes between mutable and immutable bindings.

```rust
struct Point {
    x: i32,
    y: i32,
}

let mut point = Point { x: 10, y: 20 };

point.x = 30;
point.y = 40;
```

If the binding is immutable, you cannot modify its fields:

```rust
let point = Point { x: 10, y: 20 };

// point.x = 30; // Compile error
```

Field-level mutability does not exist in ordinary Rust structs. Mutability is determined by the binding, unless interior mutability types such as `Cell` or `RefCell` are used.

## 1.3 Field initialization shorthand

When a variable has the same name as a field, you can omit the repeated field name.

```rust
let username = String::from("amir");
let email = String::from("amir@example.com");

let user = User {
    username,
    email,
    active: true,
    sign_in_count: 1,
};
```

This is equivalent to:

```rust
let user = User {
    username: username,
    email: email,
    active: true,
    sign_in_count: 1,
};
```

## 1.4 Struct update syntax

You can create a new instance by reusing fields from an existing instance.

```rust
let user1 = User {
    username: String::from("amir"),
    email: String::from("amir@example.com"),
    active: true,
    sign_in_count: 1,
};

let user2 = User {
    email: String::from("new@example.com"),
    ..user1
};
```

The fields not specified explicitly are taken from `user1`.

**Ownership matters:** if a reused field is a non-`Copy` value such as `String`, that field may be moved into the new instance. You may no longer be able to use the original instance as a whole.

## 1.5 Tuple structs

Tuple structs have fields without names.

```rust
struct Color(u8, u8, u8);

struct Coordinates(f64, f64, f64);

let red = Color(255, 0, 0);
let origin = Coordinates(0.0, 0.0, 0.0);

println!("{}", red.0);
println!("{}", origin.2);
```

Tuple structs are useful when you want a distinct type without named fields.

Although `Color` and `(u8, u8, u8)` have similar contents, they are different types.

## 1.6 Unit-like structs

A unit-like struct has no fields.

```rust
struct Marker;

let marker = Marker;
```

These can represent marker types, stateless types, or types used to implement traits.

For example:

```rust
struct JsonFormat;

trait Format {
    fn content_type(&self) -> &'static str;
}

impl Format for JsonFormat {
    fn content_type(&self) -> &'static str {
        "application/json"
    }
}
```

`JsonFormat` carries no data but represents a particular behavior or configuration.

# 2. Methods and Associated Functions

Rust defines methods and associated functions inside an `impl` block.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn is_square(&self) -> bool {
        self.width == self.height
    }
}
```

Usage:

```rust
let rectangle = Rectangle {
    width: 10,
    height: 5,
};

println!("{}", rectangle.area());
println!("{}", rectangle.is_square());
```

## 2.1 The `self` parameter

Rust methods commonly use three receiver forms.

| Receiver | Meaning |
|---|---|
| `&self` | Borrows the instance immutably |
| `&mut self` | Borrows the instance mutably |
| `self` | Takes ownership of the instance |

### Immutable borrow

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}
```

The method can read fields but cannot directly modify them.

### Mutable borrow

```rust
impl Rectangle {
    fn resize(&mut self, width: u32, height: u32) {
        self.width = width;
        self.height = height;
    }
}
```

Usage:

```rust
let mut rectangle = Rectangle {
    width: 10,
    height: 5,
};

rectangle.resize(20, 10);
```

### Ownership

```rust
impl Rectangle {
    fn destroy(self) {
        println!("Rectangle consumed");
    }
}
```

Calling `rectangle.destroy()` moves the value into the method. If the type is not `Copy`, the original binding cannot be used afterward.

## 2.2 Associated functions

An associated function does not require an instance receiver such as `self`.

```rust
impl Rectangle {
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }
}

let rectangle = Rectangle::new(10, 5);
```

`Self` refers to the type being implemented.

Associated functions are often used as constructors, although Rust does not have special constructor syntax.

Other examples include helper functions and factory functions.

# 3. Struct Visibility and Encapsulation

Struct fields are private by default outside their defining module.

```rust
pub struct User {
    username: String,
    active: bool,
}
```

Here, the type `User` is public, but its fields are private.

External modules cannot directly access those fields. Instead, expose methods:

```rust
pub struct User {
    username: String,
    active: bool,
}

impl User {
    pub fn new(username: String) -> Self {
        Self {
            username,
            active: true,
        }
    }

    pub fn username(&self) -> &str {
        &self.username
    }

    pub fn deactivate(&mut self) {
        self.active = false;
    }
}
```

This lets the type control how its state is created and modified.

The constructor also provides a place to enforce invariants, such as validating a username before storing it.

# 4. Enums

An enum defines a type that can represent one of several variants.

```rust
enum Direction {
    North,
    South,
    East,
    West,
}
```

Create a value by choosing a variant:

```rust
let direction = Direction::North;
```

At a given moment, a `Direction` value represents one of the four variants.

Enums are particularly useful when a value must belong to a defined set of alternatives.

## 4.1 Matching enum variants

Use `match` to handle each possible variant.

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn describe(direction: Direction) {
    match direction {
        Direction::North => println!("Moving north"),
        Direction::South => println!("Moving south"),
        Direction::East => println!("Moving east"),
        Direction::West => println!("Moving west"),
    }
}
```

Rust checks that a `match` is exhaustive. Every possible variant must be handled, unless a wildcard or another catch-all pattern covers the remaining cases.

This makes enums useful for representing states that must be handled explicitly.

## 4.2 Enums with data

Each variant can contain data.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}
```

This enum represents four kinds of messages:

- `Quit` contains no data.
- `Move` contains named `x` and `y` fields.
- `Write` contains a `String`.
- `ChangeColor` contains three `u8` values.

Create values:

```rust
let quit = Message::Quit;

let movement = Message::Move {
    x: 10,
    y: 20,
};

let write = Message::Write(String::from("hello"));

let color = Message::ChangeColor(255, 0, 0);
```

An enum can carry different data shapes while remaining one concrete type.

## 4.3 Matching data-bearing variants

```rust
fn process_message(message: Message) {
    match message {
        Message::Quit => {
            println!("Quit requested");
        }
        Message::Move { x, y } => {
            println!("Move to ({x}, {y})");
        }
        Message::Write(text) => {
            println!("{text}");
        }
        Message::ChangeColor(r, g, b) => {
            println!("RGB: {r}, {g}, {b}");
        }
    }
}
```

The patterns extract data from the selected variant.

For a named-field variant, use a struct-style pattern. For a tuple variant, use parentheses.

# 5. `if let` and `let else`

When you care about only one enum variant, `if let` can be more concise than `match`.

```rust
let message = Message::Write(String::from("hello"));

if let Message::Write(text) = message {
    println!("{text}");
}
```

To handle the other cases explicitly, you can use `else`:

```rust
if let Message::Write(text) = message {
    println!("{text}");
} else {
    println!("Not a write message");
}
```

You can also use `let else` when a pattern must match to continue:

```rust
fn extract_text(message: Message) -> String {
    let Message::Write(text) = message else {
        return String::new();
    };

    text
}
```

The `else` block must diverge, for example by returning, breaking, or panicking.

# 6. The `Option` Enum

Rust uses `Option<T>` to represent a value that may or may not exist.

Conceptually, its definition is:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

Examples:

```rust
let name: Option<String> = Some(String::from("Amir"));

let missing_name: Option<String> = None;
```

Handle both possibilities:

```rust
match name {
    Some(value) => println!("{value}"),
    None => println!("No name"),
}
```

Unlike a nullable pointer in some languages, `Option<T>` makes the absence of a value explicit in the type.

It is commonly used for optional configuration, lookup results, and values that may not exist.

# 7. The `Result` Enum

`Result<T, E>` represents either success or failure.

Conceptually:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Example:

```rust
let result = "42".parse::<u32>();

match result {
    Ok(number) => println!("Number: {number}"),
    Err(error) => println!("Invalid number: {error}"),
}
```

The successful variant contains a `T`; the error variant contains an `E`.

This is the foundation of Rust's explicit error handling and the `?` operator.

# 8. Enums and State Machines

Enums are especially useful for modeling state machines.

```rust
enum ConnectionState {
    Disconnected,
    Connecting,
    Connected { peer_id: u64 },
    Failed(String),
}
```

A connection can be in one of four states, with extra data attached to the connected and failed states.

You can handle these states:

```rust
fn report_state(state: &ConnectionState) {
    match state {
        ConnectionState::Disconnected => {
            println!("Disconnected");
        }
        ConnectionState::Connecting => {
            println!("Connecting");
        }
        ConnectionState::Connected { peer_id } => {
            println!("Connected to peer {peer_id}");
        }
        ConnectionState::Failed(reason) => {
            println!("Connection failed: {reason}");
        }
    }
}
```

This is useful in networking, virtualization, storage, and asynchronous systems because the state and its associated data are represented together.

It also prevents unrelated state fields from accidentally representing contradictory combinations.

# 9. Methods on Enums

Enums can have `impl` blocks just like structs.

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

impl Direction {
    fn as_str(&self) -> &'static str {
        match self {
            Self::North => "north",
            Self::South => "south",
            Self::East => "east",
            Self::West => "west",
        }
    }
}
```

Usage:

```rust
let direction = Direction::East;

println!("{}", direction.as_str());
```

`Self::East` refers to a variant of the enum currently being implemented.

Methods can be used to expose behavior without exposing the internal representation.

# 10. Generics in Structs

Structs can be generic over types.

```rust
struct Pair<T> {
    first: T,
    second: T,
}
```

Both fields must have the same type `T`.

```rust
let numbers = Pair {
    first: 10,
    second: 20,
};

let names = Pair {
    first: String::from("Amir"),
    second: String::from("Rust"),
};
```

To allow different types for each field:

```rust
struct Pair<T, U> {
    first: T,
    second: U,
}

let pair = Pair {
    first: 10,
    second: "hello",
};
```

Here, `T` and `U` are independent type parameters.

# 11. Generics in Enums

Enums can also be generic.

```rust
enum Message<T> {
    Value(T),
    Empty,
}
```

Examples:

```rust
let number = Message::Value(42);
let text = Message::Value(String::from("hello"));
let empty: Message<i32> = Message::Empty;
```

The `Value` variant contains a value of type `T`.

The `Empty` variant contains no value, but the enum type still has a concrete type parameter. That is why the example annotates `Message<i32>` for `Empty`.

Standard-library types such as `Option<T>` and `Result<T, E>` use this pattern.

# 12. Lifetimes in Structs

A struct can borrow data rather than own it.

```rust
struct User<'a> {
    name: &'a str,
}
```

The lifetime `'a` expresses the relationship between the reference stored in `User` and the data it refers to.

```rust
let name = String::from("Amir");

let user = User {
    name: &name,
};

println!("{}", user.name);
```

The referenced string must remain valid for as long as the borrow stored in `user` is used.

If you do not need to borrow, an owned field is often simpler:

```rust
struct User {
    name: String,
}
```

Use borrowed fields when the lifetime relationship is appropriate and avoiding ownership or allocation is valuable.

# 13. Structs vs Enums

| Feature | Struct | Enum |
|---|---|---|
| Purpose | Groups fields | Represents alternatives |
| Value shape | Has all its fields | Has one active variant |
| Can hold data | Yes | Yes, per variant |
| Methods | Yes | Yes |
| Generics | Yes | Yes |
| Pattern matching | Possible | Especially useful |
| Typical examples | `User`, `Point`, `Config` | `Option`, `Result`, `State` |

A struct is a good fit when the value has a consistent set of properties.

An enum is a good fit when the value represents one of several alternatives.

You can combine them:

```rust
struct User {
    id: u64,
    status: UserStatus,
}

enum UserStatus {
    Active,
    Suspended { reason: String },
    Deleted,
}
```

This models a user with a stable identity and a status that varies.

# 14. Tuple Structs vs Tuples

A tuple is a built-in compound type:

```rust
let point = (10, 20);

println!("{}", point.0);
```

A tuple struct introduces a distinct named type:

```rust
struct Point(i32, i32);

let point = Point(10, 20);

println!("{}", point.0);
```

Use a tuple for temporary grouping. Use a tuple struct when the values have a specific meaning and should not be confused with other tuples of the same shape.

# 15. Deriving Traits

Custom structs and enums do not automatically implement every useful formatting or comparison trait.

You can ask the compiler to generate certain implementations with `derive`.

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}
```

Now you can debug-print, clone, and compare points:

```rust
let point = Point { x: 10, y: 20 };

println!("{point:?}");

let copy = point.clone();

assert_eq!(point, copy);
```

Common derived traits include:

- `Debug`: supports `{:?}` and `{:#?}`.
- `Clone`: provides an explicit cloning operation.
- `Copy`: allows implicit bitwise copying for eligible types.
- `PartialEq` and `Eq`: enable equality comparisons.
- `PartialOrd` and `Ord`: enable ordering comparisons.
- `Hash`: enables use in hash-based collections when the other requirements are met.
- `Default`: provides a default value when derivation is supported by the fields.

`Copy` is not appropriate for every type. For example, a struct containing a `String` cannot implement `Copy` because `String` owns heap memory.

# 16. Structs, Enums, and Trait Implementations

Traits define behavior that types can implement.

```rust
trait Describe {
    fn describe(&self) -> String;
}

struct Server {
    hostname: String,
}

impl Describe for Server {
    fn describe(&self) -> String {
        format!("Server: {}", self.hostname)
    }
}

enum ServiceState {
    Running,
    Stopped,
}

impl Describe for ServiceState {
    fn describe(&self) -> String {
        match self {
            Self::Running => String::from("Running"),
            Self::Stopped => String::from("Stopped"),
        }
    }
}
```

Both the struct and the enum implement the same trait, but provide different behavior.

This is one of Rust's main forms of polymorphism.

# 17. Memory and Representation

A struct typically stores its fields together as one value, subject to padding and alignment.

```rust
struct Point {
    x: i32,
    y: i32,
}
```

An enum must be able to represent its largest variant, plus enough information to identify the active variant. Its exact layout is generally determined by the compiler unless a specific representation is requested.

```rust
enum Value {
    Integer(i64),
    Float(f64),
    Text(String),
}
```

This allows a single `Value` variable to hold any one of those alternatives.

An enum is not automatically a heap allocation. In this example, the enum value itself can store its active variant inline; the `String` variant owns a separate string allocation.

Use `#[repr(C)]` or another explicit representation only when you have a concrete representation requirement, such as a C ABI. Do not assume the default layout of an ordinary Rust struct or enum is stable across compiler versions.

# 18. Practical Systems Programming Example

A command-line shell can represent parsed commands with enums and structs.

```rust
use std::path::PathBuf;

struct Shell {
    current_dir: PathBuf,
}

enum Command {
    Pwd,
    Cd(PathBuf),
    Echo(Vec<String>),
    Exit,
}
```

The struct represents the shell's state. The enum represents the possible commands.

You can dispatch commands with pattern matching:

```rust
fn execute(command: Command, shell: &mut Shell) {
    match command {
        Command::Pwd => {
            println!("{}", shell.current_dir.display());
        }
        Command::Cd(path) => {
            shell.current_dir = path;
        }
        Command::Echo(args) => {
            println!("{}", args.join(" "));
        }
        Command::Exit => {
            println!("Exiting");
        }
    }
}
```

This is a simplified example: a real shell should validate paths and handle errors when changing directories.

The main idea is that an enum prevents a command from simultaneously being `Pwd` and `Exit`, while each variant carries only the data relevant to that command.

# Key Takeaways

- Use a `struct` to group related fields into one type.
- Use an `enum` to represent one of several variants.
- Structs can have named fields, tuple fields, or no fields.
- Both structs and enums support methods, associated functions, generics, and trait implementations.
- `&self`, `&mut self`, and `self` represent borrowing, mutable borrowing, and ownership respectively.
- Rust's private-by-default fields and module visibility support encapsulation.
- `Option<T>` and `Result<T, E>` are enums that make absence and failure explicit.
- Pattern matching works especially well with enums because Rust checks exhaustiveness.
- Lifetimes are needed when structs store references.
- `derive` can generate common trait implementations.
- Rust uses composition and traits rather than classical class inheritance.
- In systems programming, structs often represent resources and state, while enums represent commands, events, and state-machine alternatives.
