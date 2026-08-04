---
title: "Rust Notes: Box, Rc, RefCell and Interior Mutability"
categories:
  - Learning Notes
  - Rust
tags: [rust, smart-pointers, ownership, box, rc, refcell]
description: "Quick reference for Box<T>, Rc<T>, RefCell<T>, and interior mutability."

toc: true
---

# Rust Notes: Box, Rc, RefCell and Interior Mutability

> **Mental model**
>
> - `Box<T>` → Heap allocation + single ownership
> - `Rc<T>` → Shared ownership
> - `RefCell<T>` → Runtime borrow checking (interior mutability)

---

# Box<T>

## Purpose

- Allocate data on the heap.
- Own the value.
- Zero-cost abstraction (except heap allocation).

```rust
let x = Box::new(5);
```

## Common use cases

- Recursive data structures.

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

- Large objects without copying ownership.
- Trait objects (`Box<dyn Trait>`).

---

# Rc<T>

## Purpose

Allow **multiple owners** of the same value.

```rust
use std::rc::Rc;

let a = Rc::new(String::from("Rust"));
let b = Rc::clone(&a);
```

```
a ----\
       \
      Rc(count=2)
         |
      "Rust"
         |
       /
b -----/
```

## Important

`Rc::clone()` **does not clone the data**.

It only increments the reference count.

```rust
let b = Rc::clone(&a);
```

instead of

```rust
let b = (*a).clone();
```

---

## Rc only provides immutable access

```rust
let value = Rc::new(5);

// OK
println!("{}", value);

// Not allowed
// *value = 10;
```

Reason:

Multiple owners may exist.

---

## Rc is **not** thread-safe

For multi-threaded shared ownership, use:

```rust
Arc<T>
```

---

# RefCell<T>

## Purpose

Move borrow checking from **compile time** to **runtime**.

```rust
use std::cell::RefCell;

let value = RefCell::new(5);

*value.borrow_mut() += 1;
```

Notice:

```rust
let value = ...
```

is **not mutable**.

---

# Interior Mutability

Mutating the inside of an immutable object.

```rust
let counter = RefCell::new(0);

*counter.borrow_mut() += 1;
```

The outer object is immutable.

The inner value changes.

---

# Borrow Rules

Exactly the same rules as normal references:

- Many immutable borrows

OR

- One mutable borrow

The difference is **when** they're checked.

| Normal references | RefCell |
|-------------------|----------|
| Compile time | Runtime |
| Compile error | Panic |

---

# Why does RefCell exist?

Because sometimes the compiler cannot prove the code is safe.

The rules do **not** change.

Only the enforcement changes.

```
Compiler cannot prove safety
            ↓
Use RefCell
            ↓
Runtime checks borrowing
```

---

# Classic Example: Mock Objects

Library API:

```rust
trait Messenger {
    fn send(&self, msg: &str);
}
```

Need to record messages:

```rust
struct MockMessenger {
    messages: RefCell<Vec<String>>,
}
```

```rust
fn send(&self, msg: &str) {
    self.messages.borrow_mut().push(msg.to_string());
}
```

The API requires `&self`.

Without `RefCell`, mutation is impossible.

---

# Production Example: Cache

```rust
struct Calculator {
    cache: RefCell<Option<i32>>,
}
```

```rust
fn result(&self) -> i32
```

looks immutable,

but internally updates the cache.

This is **interior mutability**.

---

# Rc + RefCell

Very common combination.

```rust
Rc<RefCell<T>>
```

Meaning:

- shared ownership
- mutable value
- runtime borrow checking

Useful for:

- Trees
- Graphs
- GUI state
- Shared mutable data (single thread)

---

# Comparison

| Feature | Box<T> | Rc<T> | RefCell<T> |
|----------|---------|--------|-------------|
| Owners | One | Many | One |
| Heap allocation | ✅ | ✅ | ❌* |
| Mutable access | ✅ | ❌ | ✅ |
| Immutable access | ✅ | ✅ | ✅ |
| Borrow checks | Compile time | Compile time | Runtime |
| Can panic | ❌ | ❌ | ✅ |

> `RefCell<T>` itself doesn't allocate on the heap. It can live on the stack or inside another allocation.

---

# Compile-Time vs Runtime Borrow Checking

## Compile Time

Pros

- No runtime cost
- Errors found before execution
- No borrow panics

Cons

- Compiler may reject safe programs.

---

## Runtime (`RefCell`)

Pros

- More flexible
- Works when compile-time analysis is too restrictive

Cons

- Runtime overhead
- May panic if borrow rules are violated

---

# Choosing the Right Smart Pointer

Need heap allocation?

→ `Box<T>`

Need multiple owners?

→ `Rc<T>`

Need mutation through `&self`?

→ `RefCell<T>`

Need shared ownership **and** mutation?

→ `Rc<RefCell<T>>`

Need shared ownership across threads?

→ `Arc<T>`

Need shared mutable ownership across threads?

→ `Arc<Mutex<T>>`

---

# Mental Models

## Box

One owner.

```
Owner
  |
 Box
  |
Value
```

---

## Rc

Many owners.

```
A ----\
       \
        Rc ---> Value
       /
B ----/
```

---

## RefCell

One owner.

Runtime borrow checking.

```
Owner
  |
RefCell
  |
Value
```

---

# Key Takeaways

- `Box<T>` solves **where** the value lives (heap).
- `Rc<T>` solves **who owns** the value (shared ownership).
- `RefCell<T>` solves **when borrow rules are checked** (runtime instead of compile time).
- `RefCell` does **not** weaken Rust's borrowing rules; it enforces the same rules at runtime.
- Prefer compile-time borrowing (`&`, `&mut`) whenever possible. Use `RefCell` only when compile-time analysis is too restrictive.
