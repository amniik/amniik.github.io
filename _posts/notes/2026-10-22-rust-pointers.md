---
title: "Understanding Pointers in Rust: References, Raw Pointers, and C FFI"
categories:
  - Learning Notes
  - Rust
tags: [rust, pointers, references, raw-pointers, ffi, c, linux, memory-safety, operating-systems]
description: "A concise guide about pointers in Rust."

toc: true
---

# Understanding Pointers in Rust: References, Raw Pointers, and C FFI

Pointers are fundamental to systems programming. They allow programs to access memory, pass data between functions, work with dynamically allocated objects, and interact with operating systems and C libraries.

Rust provides several abstractions for managing memory safely, but systems programming sometimes requires direct access to memory addresses. Understanding the difference between references, raw pointers, and smart pointers is essential when working with Linux APIs, C libraries, virtual machines, and low-level infrastructure software.

## 1. What is a pointer?

A pointer is a value that identifies a memory location.

Consider this example:

```rust
fn main() {
    let value: i32 = 42;
    let ptr: *const i32 = &value;

    println!("Value: {value}");
    println!("Pointer address: {:p}", ptr);
}
```

The variable `value` contains the integer `42`. The variable `ptr` contains a pointer to the memory location where `value` resides.

Conceptually:

```text
Stack
+-----------------------+
| value = 42            |  <-- memory address 0x1000
+-----------------------+
             ^
             |
             | ptr
             |
+-----------------------+
| ptr = 0x1000           |
+-----------------------+
```

The address shown here is illustrative; actual addresses depend on the program and execution environment.

A pointer is not the same thing as the value it points to. The integer `42` is the data, while `ptr` identifies where that data can be accessed.

## 2. The main pointer types in Rust

Rust has several pointer-related types. They provide different ownership, borrowing, and memory-safety guarantees.

| Type | Purpose |
|---|---|
| `&T` | Shared reference to a value |
| `&mut T` | Mutable reference to a value |
| `*const T` | Raw pointer intended for read-only access |
| `*mut T` | Raw pointer that can be used for mutation |
| `Box<T>` | Owns a heap-allocated value |
| `Rc<T>` | Shared ownership using reference counting in a single-threaded context |
| `Arc<T>` | Shared ownership using atomic reference counting across threads |
| `Weak<T>` | Non-owning reference associated with `Rc` or `Arc` |
| `NonNull<T>` | Raw pointer type that guarantees the pointer is non-null |
| `Pin<P>` | Restricts moving a pinned value through that pinned pointer abstraction |

These types are not interchangeable. Some enforce ownership and borrowing rules, while others provide lower-level access.

## 3. References: `&T` and `&mut T`

References are Rust's primary mechanism for borrowing data without transferring ownership.

### 3.1 Shared reference: `&T`

A shared reference lets you read a value without taking ownership.

```rust
fn print_value(value: &i32) {
    println!("{value}");
}

fn main() {
    let number = 42;

    print_value(&number);

    println!("{number}");
}
```

The function borrows `number`. It does not own the integer, and the caller can continue using it afterward.

Shared references generally follow Rust's borrowing rule: you can have multiple shared references to a value at the same time, provided the borrowing rules are respected.

### 3.2 Mutable reference: `&mut T`

A mutable reference allows mutation through the reference.

```rust
fn increment(value: &mut i32) {
    *value += 1;
}

fn main() {
    let mut number = 42;

    increment(&mut number);

    println!("{number}"); // 43
}
```

The `*` operator dereferences the reference, accessing the value it refers to.

Rust generally permits either multiple shared references or one exclusive mutable reference at a time for a particular borrow.

### 3.3 Why use references instead of pointers?

References are appropriate when you want to borrow data within safe Rust code.

The compiler checks important properties, including:

- The reference must be valid when used.
- A reference must not outlive the data it refers to.
- Mutable borrowing must respect Rust's aliasing rules.
- Safe Rust code cannot directly create an ordinary reference from an arbitrary invalid address.

These guarantees make references the preferred choice for ordinary application logic.

## 4. Raw pointers: `*const T` and `*mut T`

Raw pointers provide lower-level memory access.

Unlike references, raw pointers can be null, can be dangling, and can point to memory that is not valid to access. Dereferencing them requires an `unsafe` context.

### 4.1 `*const T`

A `*const T` is a raw pointer intended for reading a value of type `T`.

```rust
fn main() {
    let number = 42;
    let ptr: *const i32 = &number;

    unsafe {
        println!("{}", *ptr);
    }
}
```

The expression `*ptr` dereferences the pointer.

Why is `unsafe` needed? Rust cannot generally verify that an arbitrary raw pointer is valid, properly aligned, and points to initialized data of the expected type.

Although the pointer type is `*const i32`, the type alone does not guarantee that the pointer is valid.

### 4.2 `*mut T`

A `*mut T` is a raw pointer that can be used to modify its pointee, provided the memory is valid and the operation respects Rust's safety requirements.

```rust
fn main() {
    let mut number = 42;
    let ptr: *mut i32 = &mut number;

    unsafe {
        *ptr = 100;
    }

    println!("{number}"); // 100
}
```

The raw pointer does not itself enforce exclusive access. The programmer must ensure that mutation does not violate Rust's aliasing and memory-safety requirements.

### 4.3 `*const T` versus `*mut T`

| Property | `*const T` | `*mut T` |
|---|---|---|
| Intended access | Read | Read and write |
| Can be null | Yes | Yes |
| Can be dangling | Yes | Yes |
| Can be dereferenced safely without `unsafe` | No | No |
| Guarantees valid memory | No | No |
| Guarantees exclusive access | No | No |

`*const T` does not guarantee that the underlying memory can never change. Another pointer, thread, or external component may modify it under appropriate conditions. It only prevents mutation *through that pointer itself*.

Both raw pointer types can be copied freely, so copying one does not transfer ownership of the pointed-to data.

## 5. Why do raw pointers require `unsafe`?

Suppose a pointer contains an address, but the memory at that address has already been freed.

```rust
let ptr: *const i32 = std::ptr::null();
```

This is valid Rust: creating a null raw pointer is allowed.

However, dereferencing it is invalid:

```rust
let ptr: *const i32 = std::ptr::null();

unsafe {
    // Undefined behavior: never dereference a null pointer.
    println!("{}", *ptr);
}
```

The `unsafe` keyword does not make the operation safe. It tells the compiler that the programmer is responsible for satisfying the operation's safety requirements.

Before dereferencing a raw pointer, the programmer must establish the relevant guarantees, including:

- The pointer is non-null.
- The address is properly aligned for `T`.
- The pointed-to memory is valid and accessible.
- The memory contains an initialized value of the expected type.
- The access respects aliasing and mutation rules.
- The pointer remains valid for the duration of the access.

The exact requirements depend on the operation.

## 6. Smart pointers: `Box`, `Rc`, and `Arc`

Smart pointers provide ownership or sharing semantics beyond ordinary references.

### 6.1 `Box<T>`: owned heap allocation

`Box<T>` owns a value allocated on the heap.

```rust
fn main() {
    let number = Box::new(42);

    println!("{number}");
}
```

When the `Box` is dropped, it deallocates the heap allocation and drops the contained value.

Use `Box<T>` when you need a heap-allocated value with a single owner, such as a recursive data structure or an object whose size should not be stored directly inside its parent.

### 6.2 `Rc<T>`: shared ownership in one thread

`Rc<T>` lets multiple owners share the same allocation.

```rust
use std::rc::Rc;

fn main() {
    let first = Rc::new(String::from("Rust"));
    let second = Rc::clone(&first);

    println!("{first}");
    println!("{second}");
}
```

Cloning an `Rc` increases its reference count. The allocation is dropped when the last strong owner disappears.

`Rc<T>` is not thread-safe and should not be used for shared ownership across threads.

### 6.3 `Arc<T>`: shared ownership across threads

`Arc<T>` provides thread-safe reference counting.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let value = Arc::new(42);
    let other = Arc::clone(&value);

    let handle = thread::spawn(move || {
        println!("{other}");
    });

    println!("{value}");

    handle.join().unwrap();
}
```

`Arc<T>` makes shared ownership thread-safe, but it does not automatically make arbitrary interior mutation safe. For mutable shared state, types such as `Mutex<T>` or `RwLock<T>` may be needed.

### 6.4 `Weak<T>`

A `Weak<T>` is a non-owning reference associated with `Rc<T>` or `Arc<T>`. It can help prevent reference cycles from keeping allocations alive indefinitely.

To access the allocation, call `upgrade()`, which returns an `Option` because the allocation might already have been dropped.

### 6.5 `NonNull<T>` and `Pin<P>`

`NonNull<T>` represents a raw pointer that is guaranteed to be non-null. It does not guarantee that the pointer is otherwise valid or safe to dereference.

`Pin<P>` is a wrapper that restricts moving a pinned value through that pointer abstraction when the relevant pinning contract is upheld. It is useful for some self-referential structures and asynchronous state machines.

Neither type removes the need to understand memory safety.

## 7. What is a C pointer in Rust?

A C pointer is not a separate Rust pointer category. C pointers are typically represented in Rust using raw pointers.

For example, C might declare:

```c
void process(int *value);
```

The corresponding Rust declaration is:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn process(value: *mut c_int);
}
```

Here:

- `extern "C"` specifies the C calling convention.
- `*mut c_int` corresponds to the C `int *` parameter.
- The function is declared `unsafe` because calling an arbitrary external function may have requirements the Rust compiler cannot verify.

The `unsafe extern "C"` syntax is used by current Rust editions and toolchains; older Rust versions may use different syntax for declaring external blocks.

The C function might modify the integer through the pointer. Rust cannot automatically enforce the implementation's behavior.

### 7.1 C pointers versus Rust references

A C function often accepts pointers that may be nullable, point to buffers, or follow API-specific lifetime rules.

A Rust reference, by contrast, promises validity and obeys Rust's aliasing rules.

Therefore, a C pointer should not automatically be treated as a Rust reference.

For example, this is a valid declaration of a nullable C pointer:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn find_value() -> *mut c_int;
}
```

The function might return null to indicate that no value was found. The Rust caller must check the result before dereferencing it.

## 8. Why use `*const c_int` instead of `*const i32`?

This is an important distinction when writing portable FFI bindings.

- `i32` is a Rust integer type that is always a signed 32-bit integer.
- `c_int` is Rust's type corresponding to the target platform's C `int`.

On common Linux targets, including Linux x86-64, `c_int` is typically an alias for `i32`. However, that correspondence should not be assumed for every possible target.

If a C API declares an argument as `const int *`, use:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn inspect_value(value: *const c_int);
}
```

If a protocol or file format explicitly specifies a signed 32-bit integer, `i32` is appropriate:

```rust
fn decode_integer(bytes: [u8; 4]) -> i32 {
    i32::from_le_bytes(bytes)
}
```

The choice is about matching the API contract, not about which type looks more low-level.

### 8.1 Common C-to-Rust type mappings

| C type | Rust FFI type |
|---|---|
| `int` | `c_int` |
| `unsigned int` | `c_uint` |
| `long` | `c_long` |
| `unsigned long` | `c_ulong` |
| `void *` | `*mut c_void` |
| `const void *` | `*const c_void` |
| `char *` | `*mut c_char` |
| `const char *` | `*const c_char` |

These C-compatible types can be imported from `std::ffi` on current Rust versions. `core::ffi` provides the corresponding types for environments that need to avoid the standard library.

Always match the actual C declaration and its platform ABI.

## 9. How pointers connect Rust to the operating system

Operating systems expose APIs that let applications interact with files, sockets, memory, devices, and processes.

Some APIs accept buffers through pointers.

For example, the POSIX `read()` function has a signature conceptually equivalent to:

```c
ssize_t read(int fd, void *buf, size_t count);
```

The arguments mean:

- `fd`: the file descriptor to read from.
- `buf`: the address of a memory buffer.
- `count`: the maximum number of bytes to read.

A Rust FFI declaration can use C-compatible types:

```rust
use std::ffi::{c_int, c_void};
use std::os::raw::c_long;

unsafe extern "C" {
    fn read(
        fd: c_int,
        buf: *mut c_void,
        count: usize,
    ) -> c_long;
}
```

**Note:** This is illustrative, not a portable, correct declaration of POSIX `read()` on every target. The return type should match the platform's `ssize_t`, which can be represented through the target's appropriate C-compatible signed-size type. In real projects, prefer a maintained binding or the Rust standard library's safe I/O APIs when possible.

The key point is that the operating system or C library needs an address at which it can read from or write to a buffer.

### 9.1 A safer Rust approach

Ordinary Rust code should usually use `std::io::Read` rather than call a raw system interface directly:

```rust
use std::io::{self, Read};

fn main() -> io::Result<()> {
    let mut buffer = [0_u8; 128];

    let bytes_read = io::stdin().read(&mut buffer)?;

    println!("Read {bytes_read} bytes");

    Ok(())
}
```

The safe API manages the unsafe details internally.

Raw pointers remain useful when building bindings or implementing lower-level abstractions.

## 10. How Rust passes a buffer to C

Suppose a C library provides this function:

```c
int transform(unsigned char *buffer, size_t length);
```

It receives a mutable byte buffer and its length.

A Rust declaration might be:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn transform(
        buffer: *mut u8,
        length: usize,
    ) -> c_int;
}
```

A call could look like this:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn transform(
        buffer: *mut u8,
        length: usize,
    ) -> c_int;
}

fn main() {
    let mut buffer = vec![1_u8, 2, 3, 4];

    let result = unsafe {
        transform(buffer.as_mut_ptr(), buffer.len())
    };

    println!("Result: {result}");
    println!("Buffer: {buffer:?}");
}
```

This is only safe if the C function follows its documented contract. For example, it must not access beyond the supplied length, retain the pointer after the buffer is dropped, or violate the requirements for concurrent access.

A raw pointer does not carry a length. The length is passed separately, which is why buffer APIs often accept both a pointer and a size.

In production code, you must also handle the C function's error conventions and any constraints on buffer alignment, lifetime, and ownership.

## 11. C strings: `CString` and `CStr`

C strings are different from Rust strings.

A traditional C string is a sequence of bytes terminated by a NUL byte (`\0`). C functions often expect this representation.

Rust's `String` stores valid UTF-8 text but does not automatically add a trailing NUL byte. To pass a string to a C API, use `CString` when you own the string data.

```rust
use std::ffi::CString;

fn main() {
    let message = CString::new("Hello from Rust")
        .expect("string must not contain an interior NUL");

    let ptr = message.as_ptr();

    unsafe {
        // Pass ptr to a C function that expects a valid
        // NUL-terminated string, if such a function is declared.
        let _ = ptr;
    }
}
```

`CString` rejects interior NUL bytes because they would terminate the C string early.

`CStr` is a borrowed view of a NUL-terminated C string. It does not own the underlying bytes.

Important differences:

- `String` owns UTF-8 text.
- `&str` borrows UTF-8 text.
- `CString` owns NUL-terminated bytes without interior NULs.
- `&CStr` borrows a NUL-terminated C string.
- `*const c_char` is a raw pointer to C character data; it does not itself guarantee validity or termination.

C strings are not necessarily valid UTF-8. Convert them to Rust text only when the encoding is known and valid.

## 12. Pointer lifetimes and ownership across FFI

One of the most dangerous FFI mistakes is allowing C code to retain a pointer after the underlying Rust allocation has been dropped or moved.

Consider:

```rust
use std::ffi::CString;

fn create_pointer() -> *const std::ffi::c_char {
    let value = CString::new("hello").unwrap();
    value.as_ptr()
}
```

This function is incorrect as a way to return a usable pointer. `value` is dropped at the end of the function, so the returned pointer is dangling.

The pointer value may still exist, but its target is no longer valid.

A correct design must keep the allocation alive for the entire period in which the C code uses the pointer. Depending on the API, this may mean:

- Keeping the owning `CString` in scope.
- Storing the owner in a structure whose lifetime matches the C operation.
- Using a C API that copies the string instead of retaining its pointer.
- Releasing memory through the allocator required by the API.

Never assume that C will copy a buffer unless its documentation explicitly says so.

## 13. Pointer casts and type compatibility

Raw pointers can be cast between pointer types:

```rust
let value: i32 = 42;
let ptr: *const i32 = &value;

let byte_ptr: *const u8 = ptr.cast();
```

This cast changes the pointer's static type. It does not convert the integer value `42` into a byte or validate the memory.

If you read through `byte_ptr`, you are accessing the memory representation of the integer. The interpretation depends on the target's byte order and the access requirements of the type.

For converting a number to bytes, prefer an explicit conversion:

```rust
let value: i32 = 42;
let bytes = value.to_ne_bytes();

println!("{bytes:?}");
```

For a specified byte order, use `to_le_bytes()` or `to_be_bytes()`.

Pointer casts are useful in FFI and low-level code, but they must not be confused with value conversions.

## 14. Practical rules for systems programming

When writing Rust systems software, use the following guidelines:

1. **Prefer references in ordinary Rust code.** Use `&T` and `&mut T` when borrowing data.
2. **Use `Box<T>` for single ownership of heap data.** Avoid introducing raw pointers without a clear reason.
3. **Use `Rc<T>` and `Arc<T>` for shared ownership.** Choose `Arc<T>` when shared ownership must work across threads.
4. **Use raw pointers at low-level boundaries.** C libraries, kernel interfaces, device APIs, and foreign runtimes may require them.
5. **Match the C ABI exactly.** Use `c_int` for C `int`, and the appropriate C-compatible type for other ABI-defined types.
6. **Treat `unsafe` as a responsibility.** Validate pointer validity, alignment, initialization, aliasing, and lifetime requirements.
7. **Never return a pointer to a dropped local allocation.**
8. **Distinguish bytes from text.** `u8` buffers can hold arbitrary binary data; `String` and `str` require valid UTF-8.
9. **Keep ownership explicit across FFI.** Document who allocates, who frees, and how long pointers remain valid.
10. **Prefer established bindings and safe wrappers.** Avoid reimplementing system interfaces unnecessarily.

## 15. Further reading

- [The Rust Book — References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- [The Rust Book — Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
- [The Rust Reference — Types](https://doc.rust-lang.org/reference/types.html)
- [The Rust Reference — External Blocks](https://doc.rust-lang.org/reference/items/external-blocks.html)
- [Rust standard library — `std::ffi`](https://doc.rust-lang.org/std/ffi/index.html)
- [Rustonomicon — Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html)

## Conclusion

Rust provides a spectrum of pointer abstractions. References express borrowing under compiler-enforced rules, smart pointers represent ownership and sharing, and raw pointers enable low-level memory access when those guarantees cannot be expressed directly.

For systems engineering, the key skill is knowing where to use each abstraction. In ordinary Rust code, prefer safe references and ownership types. At a C or operating-system boundary, use raw pointers and C-compatible types that match the API, and carefully document the safety assumptions that Rust cannot verify.
