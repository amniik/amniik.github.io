---
title: "Packages, Crates and Modules in Rust"
categories:
  - Learning Notes
  - Rust
tags:
  - rust
  - cargo
  - crate
  - module
description: "A concise mental model for understanding Rust packages, crates, modules, mod, use and pub use."

toc: true
---

# Why three concepts?

Rust separates project organisation into three levels because each solves a different problem.

| Concept | Responsibility | Managed by |
|---------|----------------|------------|
| **Package** | Project, dependencies, metadata | Cargo |
| **Crate** | Compilation unit | `rustc` |
| **Module** | Code organisation and namespaces | Rust |

Think of the relationship as:

```text
Cargo
│
└── Package
     │
     ├── Binary Crate
     │      └── Modules
     │
     └── Library Crate
            └── Modules
```

# Package

A package is a **Cargo** concept.

A package contains:

- `Cargo.toml`
- One or more crates

Example:

```text
my_app/
├── Cargo.toml
└── src/
    ├── main.rs
    └── lib.rs
```

Rules:

- A package contains **at least one crate**.
- A package may contain **multiple binary crates**.
- A package may contain **at most one library crate**.


# Crate

A crate is the unit compiled by `rustc`.

There are two kinds.

## Binary crate

Entry point:

```text
src/main.rs
```

Produces an executable.

## Library crate

Entry point:

```text
src/lib.rs
```

Produces a reusable library.

A package may contain both.

## Accessing the Library Crate

When a package contains both a binary crate (`main.rs`) and a library crate (`lib.rs`), the binary crate accesses the library crate using the **package name**.

Suppose `Cargo.toml` contains:

```toml
[package]
name = "my_app"
```

Project layout:

```text
src/
├── main.rs
└── lib.rs
```

Then `main.rs` can use the library crate like this:

```rust
use my_app::run;

fn main() {
    run();
}
```

Although both the binary crate and the library crate have the package name by default, **`my_app` here refers to the library crate**, not the binary crate.

The binary crate cannot be imported. Its name is used only as the name of the generated executable.

> **💡 Note**
>
> The package name and the library crate name are the same by default.
> This is why you can write:
>
> ```rust
> use my_app::parser;
> ```
>
> even though `main.rs` belongs to a different crate.
> The binary crate becomes a client of the library crate, just like any external project would.

# Crate Root vs Root Module

These are different concepts.

## Crate root

The source file where compilation starts.

- `main.rs`
- `lib.rs`

## Root module

Every crate has an implicit root module called:

```rust
crate
```

The contents of `main.rs` or `lib.rs` become the contents of this root module.

```text
main.rs
    │
    ▼
 crate
```

# The Module Tree

Rust builds a tree of modules.

Initially:

```text
crate
```

After:

```rust
mod parser;
mod network;
```

The tree becomes:

```text
crate
├── parser
└── network
```

Every module can contain child modules.

# `mod` is **not** `#include`

`mod` does **not** copy source code.

Instead, it tells the compiler:

> This module exists, and its source code is located here.

Example:

```rust
mod parser;
```

Once the compiler processes it, every other module refers to it using paths:

```rust
crate::parser::parse();
```

There is no need to write another:

```rust
mod parser;
```

# `use` creates a local alias

`use` does **not** import code.

It simply creates a shorter path.

```rust
use std::collections::HashMap;
```

means:

> Inside this scope, `HashMap` refers to `std::collections::HashMap`.

Properties:

- Local to its scope
- Does not modify the module tree
- Not inherited by parent or child modules


# `pub use` (Re-export)

`pub use` creates a **public alias**.

```rust
pub use network::Socket;
```

Users can now write:

```rust
use mycrate::Socket;
```

instead of

```rust
use mycrate::network::Socket;
```

Useful for:

- Designing clean public APIs
- Hiding internal implementation details
- Providing stable public paths

# Binary + Library Packages

Recommended layout:

```text
src/
├── lib.rs
├── main.rs
├── parser.rs
└── network.rs
```

`lib.rs`

```rust
pub mod parser;
pub mod network;

pub fn run() {
    // application logic
}
```

`main.rs`

```rust
use my_app::run;

fn main() {
    run();
}
```

The binary crate becomes a **client** of the library crate.

Benefits:

- Reusable code
- Easier testing
- Cleaner public API

# Why Explicit Modules?

Rust intentionally requires explicit module declarations.

Advantages:

- Predictable project structure
- Explicit namespaces
- No accidental compilation
- Easier navigation
- Clear ownership of every module

# Common Misconceptions

### Every `.rs` file is compiled automatically.

❌ No.

A file is compiled only if:

- It is a crate root (`main.rs` or `lib.rs`)
- It is declared with `mod`

### `mod` imports symbols.

❌ No.

It declares a module and adds it to the module tree.

### `use` imports code.

❌ No.

It only creates a local alias.

### `pub use` copies an item.

❌ No.

It creates another public path to the same item.

### `main.rs` is a module named `main`.

❌ No.

Its contents become the root module (`crate`) of the binary crate.

### Every file can belong to the same module.

❌ No.

One source file corresponds to one module (except for the directory-based module layout).

Large modules should be split into child modules.

---

### `mod` is like C's `#include`.

❌ No.

`#include` copies source code.

`mod` builds the module tree.

# Mental Model

```text
Cargo
 │
 │ manages
 ▼
Package
 │
 │ contains
 ▼
Crates
 │
 │ compiled by rustc
 ▼
Modules
 │
 │ contain
 ▼
Functions
Structs
Enums
Traits
Constants
Macros
```

Remember these rules:

- **Package** → Managed by **Cargo**
- **Crate** → Compiled by **`rustc`**
- **Module** → Organises code
- **`mod`** → Builds the module tree
- **`use`** → Creates local aliases
- **`pub use`** → Re-exports items as part of the public API
