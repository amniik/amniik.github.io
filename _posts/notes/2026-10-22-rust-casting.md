---
title: "Type Casting and Conversions in Rust"
date: 2026-10-22
categories:
  - Learning Notes
  - Rust
tags: [rust, memory-management, ownership, borrowing, lifetimes, pointers, systems-programming]
description: "A concise guide about casting in Rust."

toc: true
---

# Type Casting and Conversions in Rust

Rust provides several mechanisms for converting values, changing pointer types, borrowing data, and exposing memory to other programming languages.

Understanding the differences is especially important in systems programming, where code often interacts with C libraries, Linux APIs, and low-level memory representations.

This article covers:

- The `as` keyword.
- Numeric casts and their potential pitfalls.
- `From`, `Into`, `TryFrom`, and `TryInto`.
- `as_ptr()`, `as_mut_ptr()`, `as_ref()`, and `as_mut()`.
- Raw pointer casts and the `.cast()` method.
- Reference-to-pointer conversions.
- `String`, `CString`, and C FFI.
- How to choose the right conversion mechanism.

## 1. The different kinds of conversions

Rust does not automatically perform arbitrary type conversions. Instead, it provides explicit casts, conversion traits, borrowing operations, and type-specific methods.

| Mechanism | Purpose | Example |
|---|---|---|
| `as` | Explicit primitive conversions and permitted casts | `42_i32 as i64` |
| `From` | Infallible value conversion | `String::from("hello")` |
| `Into` | Infallible conversion to an inferred target type | `"hello".into()` |
| `TryFrom` | Fallible value conversion | `u8::try_from(number)` |
| `TryInto` | Fallible conversion using the source value | `number.try_into()` |
| `as_ptr()` | Obtain a raw pointer to data exposed by a type | `vec.as_ptr()` |
| `as_mut_ptr()` | Obtain a mutable raw pointer to data | `vec.as_mut_ptr()` |
| `as_ref()` | Borrow or convert to a reference, depending on the type | `boxed.as_ref()` |
| `as_mut()` | Borrow or convert to a mutable reference, depending on the type | `boxed.as_mut()` |
| `.cast::<T>()` | Change the static type of a raw pointer | `ptr.cast::<u8>()` |

These operations are not interchangeable. In particular, `as_ptr()` is a method, whereas `as` is a language keyword.

## 2. The `as` keyword

The `as` keyword performs explicit conversions permitted by Rust's casting rules.

### 2.1 Numeric casts

```rust
fn main() {
    let number: i32 = 42;

    let wider: i64 = number as i64;
    let floating: f64 = number as f64;

    println!("{wider}");
    println!("{floating}");
}
```

Here, `i32` is converted into `i64` and `f64`.

However, numeric casts can lose information.

```rust
fn main() {
    let number: i32 = 300;
    let result: u8 = number as u8;

    assert_eq!(result, 44);
}
```

Since `u8` can represent only values from `0` through `255`, the cast retains the low eight bits of the integer.

Casting a floating-point number to an integer also has specific rules: finite values are truncated toward zero, and out-of-range values saturate to the target integer's minimum or maximum. `NaN` converts to zero.

The behavior is defined by Rust, but it may not be what your application requires.

### 2.2 Prefer checked conversions when necessary

If a conversion might fail, use `TryFrom` or `TryInto`.

```rust
use std::convert::TryFrom;

fn main() {
    let number: i32 = 300;

    let result = u8::try_from(number);

    assert!(result.is_err());
}
```

This makes the failure explicit instead of silently truncating the value.

For infallible conversions, `From` and `Into` are generally preferable when an appropriate implementation exists.

### 2.3 Casting a `char` or `bool`

Rust also permits certain primitive casts:

```rust
fn main() {
    let letter = 'A';
    let code_point = letter as u32;

    assert_eq!(code_point, 65);

    let enabled = true;
    let number = enabled as u8;

    assert_eq!(number, 1);
}
```

Casting a Rust `char` to an integer returns its Unicode code point. This is not necessarily the same as the character's UTF-8 byte representation.

## 3. `From`, `Into`, `TryFrom`, and `TryInto`

These conversion traits express conversions between types without requiring a low-level cast.

### 3.1 `From` and `Into`

`From<T>` defines how to construct a value of one type from another type.

```rust
fn main() {
    let text = String::from("Rust");

    let another: String = "systems programming".into();

    println!("{text}");
    println!("{another}");
}
```

`Into<T>` is the corresponding conversion interface from the source type's perspective. Implementing `From<T>` for the destination type automatically provides the corresponding `Into` implementation.

### 3.2 `TryFrom` and `TryInto`

Some conversions cannot always succeed.

```rust
use std::convert::TryFrom;

fn main() {
    let value: i64 = 1_000;

    let result = u8::try_from(value);

    match result {
        Ok(number) => println!("Converted: {number}"),
        Err(error) => println!("Conversion failed: {error}"),
    }
}
```

Here, conversion fails because `1000` cannot fit in a `u8`.

### 3.3 Which should you use?

- Use `From`/`Into` for supported infallible value conversions.
- Use `TryFrom`/`TryInto` when conversion can fail.
- Use `as` when you specifically need one of Rust's permitted cast operations and understand its semantics.

Do not assume that every pair of Rust types can be converted using `as`.

## 4. What are `as_ptr()` and `as_mut_ptr()`?

These are methods implemented by particular types. They are not language keywords.

For example, `String`, `Vec<T>`, and slices expose pointers to their underlying elements.

```rust
fn main() {
    let text = String::from("Rust");
    let text_ptr: *const u8 = text.as_ptr();

    let numbers = vec![10_i32, 20, 30];
    let numbers_ptr: *const i32 = numbers.as_ptr();

    println!("{text_ptr:p}");
    println!("{numbers_ptr:p}");
}
```

The pointers refer to the underlying data, not necessarily to the objects that manage that data.

### 4.1 Pointer to a `String` versus pointer to its bytes

```rust
fn main() {
    let text = String::from("Rust");

    let object_ptr: *const String = &text;
    let bytes_ptr: *const u8 = text.as_ptr();

    println!("{object_ptr:p}");
    println!("{bytes_ptr:p}");
}
```

These pointers have different meanings:

- `object_ptr` points to the `String` object.
- `bytes_ptr` points to the first byte of its UTF-8 buffer.

This distinction is important when interacting with C. A C function expecting character data usually needs a pointer to bytes, not a pointer to Rust's `String` object.

### 4.2 Mutable pointers

`as_mut_ptr()` returns a mutable raw pointer to the underlying elements when the type provides that method.

```rust
fn main() {
    let mut numbers = vec![10_i32, 20, 30];

    let ptr: *mut i32 = numbers.as_mut_ptr();

    unsafe {
        *ptr = 100;
    }

    assert_eq!(numbers[0], 100);
}
```

This example modifies the first element. The pointer must remain valid, and the mutation must respect the vector's allocation, initialization, aliasing, and lifetime requirements.

## 5. What do `as_ref()` and `as_mut()` do?

These methods do not generally produce raw pointers. Their behavior depends on the type.

For `Box<T>`, `as_ref()` borrows the contained value:

```rust
fn main() {
    let boxed = Box::new(42_i32);

    let reference: &i32 = boxed.as_ref();

    println!("{reference}");
}
```

For a mutable `Box<T>`, `as_mut()` provides a mutable reference:

```rust
fn main() {
    let mut boxed = Box::new(42_i32);

    let reference: &mut i32 = boxed.as_mut();

    *reference += 1;

    assert_eq!(*boxed, 43);
}
```

Other types can implement these methods differently. For example, `Option<T>::as_ref()` returns an `Option<&T>`, while `Option<T>::as_mut()` returns an `Option<&mut T>`.

Always check the particular type's method signature rather than inferring its behavior from the name alone.

## 6. Raw pointer casts

Rust allows certain conversions between raw pointer types.

```rust
fn main() {
    let value: i32 = 42;

    let ptr: *const i32 = &value;
    let byte_ptr: *const u8 = ptr.cast();

    assert_eq!(ptr as usize, byte_ptr as usize);
}
```

The `.cast::<u8>()` method changes the pointer's static type. It does not change the underlying bytes or validate that the pointer is safe to dereference as a `u8`.

The equivalent explicit cast is:

```rust
let byte_ptr: *const u8 = ptr as *const u8;
```

For raw pointers, `.cast::<T>()` is often clearer when changing the pointee type.

### 6.1 Casting between `*const T` and `*mut T`

A raw pointer can be cast from a mutable pointer to a const pointer:

```rust
let mut value = 42_i32;

let mutable_ptr: *mut i32 = &mut value;
let const_ptr: *const i32 = mutable_ptr.cast();
```

This conversion does not mutate the value.

However, casting a `*const T` to `*mut T` does not make the underlying data mutable. Never use such a cast to bypass Rust's mutation rules.

### 6.2 Casting a pointer to an integer

You can obtain a pointer's address as an integer:

```rust
let value = 42_i32;
let ptr: *const i32 = &value;

let address: usize = ptr as usize;

println!("{address:#x}");
```

`usize` is the appropriate integer type for representing machine-sized addresses.

An address is not ownership of the allocation. Converting a pointer to an integer and later reconstructing a pointer also requires care: pointer provenance and validity matter, not merely whether two addresses have the same numerical representation.

## 7. Reference-to-pointer conversions

You do not always need an explicit cast to obtain a raw pointer.

```rust
fn main() {
    let value = 42_i32;

    let reference: &i32 = &value;
    let ptr: *const i32 = reference;

    println!("{ptr:p}");
}
```

Rust can coerce the reference into a raw pointer.

For mutable data:

```rust
fn main() {
    let mut value = 42_i32;

    let ptr: *mut i32 = &mut value;

    unsafe {
        *ptr = 100;
    }

    assert_eq!(value, 100);
}
```

The mutable reference is taken from a mutable variable, and the raw pointer can then be used under the appropriate safety conditions.

## 8. Casting does not convert the underlying representation

Suppose you have a Rust string:

```rust
use std::ffi::c_char;

fn main() {
    let text = String::from("Hello");

    let ptr: *const c_char = text.as_ptr().cast();

    println!("{ptr:p}");
}
```

This is a pointer-type conversion. It does not convert `String` into a C string.

The UTF-8 bytes remain the same, and the buffer is not guaranteed to have a trailing NUL byte.

If a C function expects `const char *` as a NUL-terminated string, use `CString`:

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn print_message(message: *const c_char);
}

fn main() {
    let text = CString::new("Hello").unwrap();

    unsafe {
        print_message(text.as_ptr());
    }
}
```

`CString` owns NUL-terminated bytes and rejects interior NUL bytes. Its allocation must remain alive while the C function uses the pointer.

If a C function expects a pointer and an explicit length, a pointer from `String::as_ptr()` can instead be appropriate, provided the foreign function follows the documented buffer and lifetime contract.

## 9. A practical decision guide

| Requirement | Recommended approach |
|---|---|
| Convert `i32` to `i64` | `i64::from(value)` |
| Convert an integer with range checking | `u8::try_from(value)` |
| Get a pointer to a `String`'s bytes | `string.as_ptr()` |
| Get a mutable pointer to a vector's elements | `vec.as_mut_ptr()` |
| Change a raw pointer's pointee type | `ptr.cast::<T>()` |
| Borrow a value inside a `Box<T>` | `boxed.as_ref()` |
| Pass a NUL-terminated string to C | `CString` followed by `.as_ptr()` |
| Get a raw pointer to an ordinary value | `&value` or `&mut value` |
| Convert a pointer to a numerical address | `ptr as usize` |

## 10. Safety considerations

Casting can make low-level code possible, but it does not eliminate memory-safety requirements.

Before dereferencing a raw pointer, establish that it is valid, properly aligned, points to initialized data of the correct type, and is accessed in accordance with Rust's aliasing rules.

When passing pointers to C, also determine whether the foreign function:

- Reads or modifies the buffer.
- Requires a NUL terminator.
- Receives an explicit buffer length.
- Retains the pointer after returning.
- Expects the caller to allocate or free memory.
- Requires a specific ABI or C-compatible data type.

A pointer cast is not a substitute for meeting these requirements.

## Further reading

- [Rust Reference — Type Cast Expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html)
- [Rust `as` keyword](https://doc.rust-lang.org/std/keyword.as.html)
- [Rust `From` and `Into`](https://doc.rust-lang.org/std/convert/index.html)
- [Rust raw pointer documentation](https://doc.rust-lang.org/std/primitive.pointer.html)
- [The Rustonomicon — Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html)

## Conclusion

Rust provides different mechanisms for different kinds of conversions. The `as` keyword performs permitted casts, conversion traits handle value conversions, and methods such as `as_ptr()` expose pointers to data managed by particular types.

For systems programming, the most important distinction is between changing a type and changing the underlying representation. Casting a pointer does not change the memory it points to, establish ownership, guarantee validity, or automatically make data compatible with a C API.
