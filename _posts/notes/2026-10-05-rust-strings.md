---
title: "Strings and String-Like Types in Rust"
categories:
  - Learning Notes
  - Rust
tags: [rust, strings, utf8, osstring, cstring, path, ffi, systems-programming]
description: "A concise guide about Strings and String-Like Types in Rust."

toc: true
---

Rust has several string and string-like types.

This can initially feel confusing because languages such as C usually have one primary representation for strings:

```c
char *str;
```

Rust separates strings based on important properties:

* Does the value own its memory?
* Is the text guaranteed to be valid UTF-8?
* Is it intended for the operating system?
* Is it intended for C APIs?
* Is it actually a filesystem path rather than text?

The most important types are:

```text
String       owned UTF-8 text
&str         borrowed UTF-8 text

OsString     owned OS-native string
&OsStr       borrowed OS-native string

CString      owned C-compatible string
&CStr        borrowed C-compatible string

PathBuf      owned filesystem path
&Path        borrowed filesystem path
```

Understanding these types is particularly important for systems programming because operating systems and C APIs do not necessarily use UTF-8 strings.

# 1. `String`

`String` is Rust's owned, growable, UTF-8 encoded string.

```rust
let mut name = String::from("Amir");

name.push('!');
name.push_str(" Nikpour");

println!("{name}");
```

Conceptually:

```text
String
 ├── pointer ────────> heap allocation
 ├── length
 └── capacity
```

A `String` owns the underlying memory.

Therefore, it can grow:

```rust
let mut text = String::new();

text.push_str("hello");
text.push(' ');
text.push_str("world");
```

## UTF-8

`String` must always contain valid UTF-8.

```rust
let text = String::from("hello");
```

The bytes inside the string are guaranteed to represent valid UTF-8.

For example:

```rust
let text = String::from("é");

println!("{}", text.len());
```

The result is:

```text
2
```

because `é` occupies two bytes in UTF-8.

Therefore:

```rust
text.len()
```

returns the number of **bytes**, not the number of Unicode characters.

Use:

```rust
text.chars().count()
```

if you want the number of Unicode scalar values.

# 2. `&str`

`&str` is a borrowed string slice.

It does not own the string data.

```rust
let text = String::from("hello");

let slice: &str = &text;
```

Conceptually:

```text
String
   |
   | borrow
   v
 &str
```

A string literal is also a `&str`:

```rust
let text: &str = "hello";
```

The literal is stored in the program's binary and normally has a `'static` lifetime:

```rust
let text: &'static str = "hello";
```

# 3. `String` vs `&str`

This is one of the most important distinctions in Rust.

```text
String
 └── owns the string

&str
 └── borrows string data
```

Example:

```rust
fn print_name(name: &str) {
    println!("{name}");
}
```

You can pass both:

```rust
let name = String::from("Amir");

print_name(&name);
print_name("Amir");
```

This is one reason `&str` is commonly used for function parameters when the function only needs to read the text.

If a function needs to store or own the text:

```rust
fn create_user(name: String) {
    // owns name
}
```

you may use `String`.

# 4. `str` Is Unsized

`str` is a dynamically sized type (DST).

You normally cannot have:

```rust
let text: str;
```

because the compiler does not know the size of `str` at compile time.

Instead, you use a pointer to it:

```rust
let text: &str;
```

A `&str` is a **fat pointer** containing information such as:

```text
&str
 ├── pointer
 └── length
```

This allows Rust to know which bytes belong to the string slice.

# 5. `OsString`

`OsString` is an owned string intended for communication with the operating system.

It is important because operating-system strings are not necessarily valid UTF-8.

For example, Unix allows arbitrary byte sequences except `NUL` in many pathname contexts.

Windows has different native string representations.

Therefore, Rust provides:

```rust
std::ffi::OsString
```

Example:

```rust
use std::ffi::OsString;

let name = OsString::from("config.txt");
```

The important idea is:

```text
String
 └── guaranteed UTF-8

OsString
 └── OS-compatible representation
```

You should use `OsString` when dealing with values coming from OS interfaces where UTF-8 is not guaranteed.

# 6. `&OsStr`

`&OsStr` is the borrowed version of `OsString`.

```rust
use std::ffi::{OsStr, OsString};

let name = OsString::from("config.txt");

let name_ref: &OsStr = &name;
```

Conceptually:

```text
OsString
   |
   | borrow
   v
 &OsStr
```

This is similar to:

```text
String
   |
   | borrow
   v
 &str
```

but the important difference is that `OsStr` does not promise UTF-8.

# 7. Why `OsString` Exists

Consider an environment variable:

```rust
use std::env;

for (key, value) in env::vars_os() {
    println!("{key:?} = {value:?}");
}
```

Notice the `_os` version:

```rust
env::vars_os()
```

instead of:

```rust
env::vars()
```

The normal version returns:

```rust
String
```

and therefore requires valid Unicode.

The OS version returns:

```rust
OsString
```

because the underlying operating-system data might not be valid Unicode.

Similarly:

```rust
std::env::args()
```

returns Unicode strings and can fail when arguments aren't valid Unicode.

While:

```rust
std::env::args_os()
```

returns:

```rust
OsString
```

and preserves the underlying OS representation.

This distinction is very important in system programs.

# 8. Converting `OsString` to `String`

You cannot simply assume an `OsString` contains valid UTF-8.

You can explicitly request a lossy conversion:

```rust
let os_string = std::ffi::OsString::from("hello");

let string = os_string.to_string_lossy();

println!("{string}");
```

`to_string_lossy()` returns:

```rust
Cow<'_, str>
```

If the content is valid UTF-8, it can borrow it.

If it is not valid UTF-8, invalid sequences are replaced with the Unicode replacement character:

```text
�
```

You can also consume the `OsString`:

```rust
let result = os_string.into_string();

match result {
    Ok(string) => println!("{string}"),
    Err(original) => {
        println!("Not valid UTF-8: {original:?}");
    }
}
```

This does not silently lose information.

# 9. `CString`

`CString` is an owned string designed for interoperability with C.

```rust
use std::ffi::CString;

let text = CString::new("hello").unwrap();
```

C strings are traditionally represented as:

```text
char *
```

and terminated by a NUL byte:

```text
'h' 'e' 'l' 'l' 'o' '\0'
```

Rust's `String` does **not** have this guarantee.

Therefore, when calling a C API that expects a C string, use `CString`.

# 10. Why Can't We Pass `String` Directly to C?

Suppose a C function expects:

```c
void print_name(const char *name);
```

You cannot simply pass:

```rust
let name = String::from("Amir");

some_c_function(name);
```

because a Rust `String` is not a C string.

A Rust `String` is represented approximately as:

```text
pointer + length + capacity
```

while a C string is:

```text
pointer
   |
   v
'a' 'm' 'i' 'r' '\0'
```

The C API determines the string length by finding the NUL terminator.

Rust instead stores the length separately.

# 11. Creating a `CString`

Use:

```rust
use std::ffi::CString;

let name = CString::new("Amir").unwrap();
```

`CString::new()` can fail.

Why?

Because C strings cannot contain an embedded NUL byte.

This is invalid:

```rust
let name = CString::new("Amir\0Nikpour");
```

The result is:

```rust
Err(NulError)
```

because C would interpret the first NUL as the end of the string.

# 12. Passing `CString` to C

To get a raw pointer:

```rust
let name = CString::new("Amir").unwrap();

let ptr = name.as_ptr();
```

The pointer has type:

```rust
*const c_char
```

For example:

```rust
unsafe {
    libc::puts(name.as_ptr());
}
```

The `CString` must remain alive while C uses the pointer:

```rust
let name = CString::new("Amir").unwrap();

unsafe {
    libc::puts(name.as_ptr());
}
```

Do not do something like:

```rust
let ptr = CString::new("Amir").unwrap().as_ptr();
```

and then use `ptr`.

The temporary `CString` is immediately dropped, leaving a dangling pointer.

# 13. `CStr`

`CStr` is the borrowed counterpart of `CString`.

```text
CString
   |
   | borrow
   v
 &CStr
```

Example:

```rust
use std::ffi::{CStr, CString};

let owned = CString::new("hello").unwrap();

let borrowed: &CStr = owned.as_c_str();
```

`CStr` represents an existing NUL-terminated C string without owning it.

# 14. `CString` vs `CStr`

The relationship is similar to:

```text
String      -> &str
OsString    -> &OsStr
CString     -> &CStr
PathBuf     -> &Path
```

The owned types own their underlying data.

The borrowed types provide a view into existing data.

# 15. Converting `CStr` to Rust Text

You can attempt to interpret a C string as UTF-8:

```rust
let c_string = CString::new("hello").unwrap();

let c_str = c_string.as_c_str();

let text = c_str.to_str().unwrap();

println!("{text}");
```

`to_str()` returns:

```rust
Result<&str, Utf8Error>
```

because a C string is not necessarily valid UTF-8.

You can also use:

```rust
let text = c_str.to_string_lossy();
```

which returns:

```rust
Cow<'_, str>
```

# 16. `PathBuf`

`PathBuf` is an owned filesystem path.

```rust
use std::path::PathBuf;

let path = PathBuf::from("/home/amir/file.txt");
```

It may look like a string, but semantically it represents a **path**, not arbitrary text.

Therefore:

```text
PathBuf
 └── owned filesystem path

&Path
 └── borrowed filesystem path
```

# 17. `&Path`

`&Path` is the borrowed form of `PathBuf`.

```rust
use std::path::Path;

let path = Path::new("/tmp/file.txt");
```

Or:

```rust
let path_buf = PathBuf::from("/tmp/file.txt");

let path: &Path = &path_buf;
```

# 18. Why Use `Path` Instead of `String`?

Consider:

```rust
fn read_file(path: String) {
    // ...
}
```

This says the function needs to own an arbitrary string.

But:

```rust
fn read_file(path: &Path) {
    // ...
}
```

communicates the actual meaning:

> This parameter represents a filesystem path.

It also gives access to path-specific operations:

```rust
path.exists();
path.is_file();
path.is_dir();
path.parent();
path.file_name();
path.extension();
```

This is much better than manually manipulating strings.

# 19. `PathBuf` and `Path`

Like the other owned/borrowed pairs:

```text
PathBuf
   |
   | borrow
   v
 &Path
```

Example:

```rust
let mut path = PathBuf::from("/tmp");

path.push("file.txt");

println!("{}", path.display());
```

Notice the `.display()`.

Unlike `String`, `PathBuf` does not implement `Display` directly.

Instead:

```rust
path.display()
```

returns a display adapter.

This is why:

```rust
println!("{}", path);
```

doesn't work, while:

```rust
println!("{}", path.display());
```

does.

# 20. `Path` Is Closely Related to `OsStr`

One of the most useful relationships to remember is:

```text
PathBuf  ──> OsString
Path     ──> OsStr
```

A filesystem path is essentially an OS string with path semantics.

You can access its OS representation:

```rust
let path = Path::new("/tmp/file.txt");

let os_str = path.as_os_str();
```

And convert a path from an OS string:

```rust
let os_string = std::ffi::OsString::from("/tmp/file.txt");

let path = PathBuf::from(os_string);
```

# 21. The Main String Family

The relationships can be summarized as:

```text
                 Owned              Borrowed
                 ─────              ────────

UTF-8 text       String             &str

OS string        OsString           &OsStr

C string         CString            &CStr

Filesystem path  PathBuf            &Path
```

This is a very useful mental model.

# 22. UTF-8 vs OS Strings vs C Strings

The biggest conceptual difference is what representation the type guarantees.

```text
String / &str
    |
    +-- valid UTF-8


OsString / &OsStr
    |
    +-- OS-native representation
    +-- no UTF-8 guarantee


CString / &CStr
    |
    +-- NUL-terminated
    +-- no UTF-8 guarantee


PathBuf / &Path
    |
    +-- filesystem path semantics
    +-- OS-native representation
```

# 23. Rust String vs C String

This distinction is especially important when writing system software.

### Rust

```rust
let text = String::from("hello");
```

Conceptually:

```text
pointer
length
capacity
```

The length is known directly.

### C

```c
char *text = "hello";
```

Conceptually:

```text
pointer
  |
  v
h e l l o \0
```

The string length is determined by searching for `\0`.

Therefore Rust needs:

```rust
CString
```

when interacting with APIs expecting this representation.

# 24. Rust String vs OS String

Do not assume that everything returned by the operating system is UTF-8.

For example:

```rust
use std::env;

let args = std::env::args_os();

for arg in args {
    println!("{arg:?}");
}
```

The arguments are:

```rust
OsString
```

because the OS may provide values that cannot be represented as valid UTF-8.

When you know that UTF-8 is required and acceptable:

```rust
let args = std::env::args();

for arg in args {
    println!("{arg}");
}
```

Here the arguments are:

```rust
String
```

# 25. `Cow<str>`

Another useful type when working with strings is:

```rust
std::borrow::Cow<'a, str>
```

`Cow` means:

> Clone on Write.

It can either borrow existing string data or own a new `String`.

For example:

```rust
let value = os_string.to_string_lossy();
```

returns:

```rust
Cow<'_, str>
```

Conceptually:

```text
Cow<str>
   |
   +── Borrowed(&str)
   |
   └── Owned(String)
```

If conversion does not require modification, it can borrow.

If conversion requires replacement of invalid UTF-8, it can create an owned `String`.

This avoids unnecessary allocation when possible.

# 26. `String` and `Vec<u8>`

A `String` is closely related to:

```rust
Vec<u8>
```

because UTF-8 text is ultimately stored as bytes.

You can get the bytes:

```rust
let text = String::from("hello");

let bytes = text.as_bytes();
```

The result is:

```rust
&[u8]
```

You can also consume a `String` and obtain its underlying bytes:

```rust
let text = String::from("hello");

let bytes: Vec<u8> = text.into_bytes();
```

This does not copy the allocation.

The ownership is transferred:

```text
String
   |
   | into_bytes()
   v
Vec<u8>
```

# 27. Creating a `String` From Bytes

If you have bytes that you know are valid UTF-8:

```rust
let bytes = vec![104, 101, 108, 108, 111];

let text = String::from_utf8(bytes).unwrap();

println!("{text}");
```

`from_utf8()` returns:

```rust
Result<String, FromUtf8Error>
```

because arbitrary bytes are not necessarily valid UTF-8.

For a byte slice:

```rust
let bytes = &[104, 101, 108, 108, 111];

let text = std::str::from_utf8(bytes).unwrap();
```

This returns:

```rust
Result<&str, Utf8Error>
```

The distinction is:

```text
Vec<u8>  -> String
          owned conversion

&[u8]    -> &str
          borrowed conversion
```

# 28. `String` Conversion Cheat Sheet

Common conversions include:

```rust
// &str -> String
let s = String::from("hello");

// &str -> String
let s = "hello".to_string();

// String -> &str
let s = String::from("hello");
let r: &str = &s;

// String -> Vec<u8>
let s = String::from("hello");
let bytes = s.into_bytes();

// &[u8] -> &str
let bytes = b"hello";
let s = std::str::from_utf8(bytes).unwrap();

// Vec<u8> -> String
let bytes = vec![104, 105];
let s = String::from_utf8(bytes).unwrap();
```

# 29. OS String Conversion Cheat Sheet

```rust
use std::ffi::{OsStr, OsString};

// &str -> OsString
let owned = OsString::from("hello");

// &OsStr -> OsString
let borrowed = OsStr::new("hello");
let owned = borrowed.to_os_string();

// OsString -> &OsStr
let owned = OsString::from("hello");
let borrowed: &OsStr = &owned;

// OsString -> String
let owned = OsString::from("hello");

match owned.into_string() {
    Ok(text) => println!("{text}"),
    Err(original) => println!("{original:?}"),
}
```

# 30. C String Conversion Cheat Sheet

```rust
use std::ffi::{CStr, CString};

// &str -> CString
let owned = CString::new("hello").unwrap();

// CString -> &CStr
let borrowed: &CStr = owned.as_c_str();

// &CStr -> &str
let text = borrowed.to_str().unwrap();

// CString -> raw pointer
let ptr = owned.as_ptr();
```

Remember:

```rust
CString::new(...)
```

can fail if the input contains an embedded NUL byte.

# 31. Path Conversion Cheat Sheet

```rust
use std::path::{Path, PathBuf};

// &str -> PathBuf
let owned = PathBuf::from("/tmp/file.txt");

// &str -> &Path
let borrowed = Path::new("/tmp/file.txt");

// PathBuf -> &Path
let path = PathBuf::from("/tmp/file.txt");
let borrowed: &Path = &path;

// Path -> &OsStr
let os_str = borrowed.as_os_str();

// PathBuf -> String-like display
println!("{}", path.display());
```

# 32. Which Type Should I Use?

A practical rule:

### Normal application text

Use:

```rust
String
&str
```

For example:

```rust
fn greet(name: &str) {
    println!("Hello {name}");
}
```

### Operating-system data

Use:

```rust
OsString
&OsStr
```

For example:

```rust
std::env::args_os()
```

### Filesystem paths

Use:

```rust
PathBuf
&Path
```

For example:

```rust
fn open_config(path: &Path) {
    // ...
}
```

### C FFI

Use:

```rust
CString
&CStr
```

For example:

```rust
unsafe {
    libc::puts(c_string.as_ptr());
}
```

# 33. A Useful Mental Model

Think about strings according to the **boundary they are crossing**.

```text
                         Rust application
                               |
                               |
                    UTF-8 text |
                               v
                         String / &str
                               |
              +----------------+----------------+
              |                                 |
              v                                 v
       Operating system                    C library
              |                                 |
              v                                 v
       OsString / &OsStr                 CString / &CStr
              |
              v
        PathBuf / &Path
        filesystem paths
```

This is much more useful than memorizing the types independently.

# 34. Systems Programming Perspective

When writing systems software, avoid converting everything to `String`.

For example, this is often unnecessary:

```rust
fn open_file(path: String) {
    // ...
}
```

A better API is:

```rust
fn open_file(path: &Path) {
    // ...
}
```

If you only need OS-level text:

```rust
fn process_argument(arg: &OsStr) {
    // ...
}
```

If you only need normal UTF-8 text:

```rust
fn process_name(name: &str) {
    // ...
}
```

If you need to call C:

```rust
fn call_c_api(name: &CStr) {
    // ...
}
```

Choosing the type according to the boundary makes invalid conversions and unnecessary allocations less likely.

# 35. Summary Table

| Type       | Owns data | UTF-8 guaranteed | NUL terminated | Main purpose                 |
| ---------- | --------: | ---------------: | -------------: | ---------------------------- |
| `String`   |       Yes |              Yes |             No | Owned Rust text              |
| `&str`     |        No |              Yes |             No | Borrowed Rust text           |
| `OsString` |       Yes |               No |             No | Owned OS string              |
| `&OsStr`   |        No |               No |             No | Borrowed OS string           |
| `CString`  |       Yes |               No |            Yes | Owned C string               |
| `&CStr`    |        No |               No |            Yes | Borrowed C string            |
| `PathBuf`  |       Yes |               No |             No | Owned filesystem path        |
| `&Path`    |        No |               No |             No | Borrowed filesystem path     |
| `Cow<str>` | Sometimes |              Yes |             No | Borrowed-or-owned UTF-8 text |

# Key Takeaways

* `String` is an owned, growable, valid UTF-8 string.
* `&str` is a borrowed UTF-8 string slice.
* `OsString` and `&OsStr` are for OS-native strings where UTF-8 is not guaranteed.
* `CString` and `&CStr` are for C-compatible NUL-terminated strings.
* `PathBuf` and `&Path` represent filesystem paths and are built on OS-string representations.
* `String` and `OsString` are not interchangeable because OS strings may not be valid UTF-8.
* `CString` is different from `String` because C uses NUL termination rather than Rust's explicit length.
* `str`, `OsStr`, `CStr`, and `Path` are borrowed/unsized views; their owned counterparts are `String`, `OsString`, `CString`, and `PathBuf`.
* Use `&str` for ordinary borrowed UTF-8 text.
* Use `&OsStr` when interacting with OS APIs.
* Use `&Path` for filesystem paths.
* Use `&CStr` when receiving borrowed C strings from FFI.
* Do not convert everything to `String just because it is convenient; preserve the representation appropriate to the system boundary.

---
