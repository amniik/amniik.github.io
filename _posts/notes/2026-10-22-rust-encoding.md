---
title: "Unicode, UTF-8, and Rust: From Text to Operating-System Paths"
categories:
  - Learning Notes
  - Rust
tags: [rust, unicode, utf8, encoding, strings, filesystem, linux, windows, paths]
description: "A concise guide about encoding in Rust."

toc: true
---

# Unicode, UTF-8, and Rust: From Text to Operating-System Paths

When working with Rust's strings and filesystem APIs, it is important to understand the difference between bytes, characters, Unicode, and character encodings.

For example, Rust's `String` guarantees that its contents are valid UTF-8, but a `PathBuf` can represent a Linux filename containing bytes that are not valid UTF-8.

Why is that possible? Does Linux use a different encoding? Why is `0xFF` invalid UTF-8? And how does Windows differ from Linux?

This article explains these concepts from the perspective of systems programming.

## 1. Bytes, text, and character encodings

Computers store data as bytes. A byte contains eight bits and can represent a value from `0x00` to `0xFF`.

For example:

```text
'A'  -> 0x41
'B'  -> 0x42
'0'  -> 0x30
```

For ASCII characters, the numerical byte values are straightforward. However, the world contains many more characters than ASCII can represent.

Consider these characters:

```text
A
é
中
😀
```

How should a computer represent them as bytes?

A **character encoding** defines how text characters are represented as bytes. UTF-8 and UTF-16 are examples of Unicode encodings.

The important distinction is:

- Bytes are numerical data.
- Characters are abstract textual elements.
- An encoding defines how Unicode code points are represented as bytes.

The bytes themselves do not always tell you which encoding was intended.

## 2. What is Unicode?

Unicode is a standard that assigns code points to characters and other textual elements.

A Unicode code point is written using the prefix `U+`.

For example:

| Character | Unicode code point |
|---|---|
| `A` | `U+0041` |
| `é` | `U+00E9` |
| `中` | `U+4E2D` |
| `😀` | `U+1F600` |

A code point identifies a Unicode value. It does not, by itself, specify the bytes used to store that value.

For example, the character `é` has code point `U+00E9`, but it can be encoded differently depending on the encoding.

Unicode also distinguishes code points from user-perceived characters. For example, an accented character can be represented as one precomposed code point or as a base character followed by a combining mark.

Therefore, the number of Unicode code points is not always the same as the number of characters a user perceives on the screen.

## 3. What is UTF-8?

UTF-8 is a variable-width encoding for Unicode code points. It uses one to four bytes per code point.

| Character | Unicode code point | UTF-8 bytes |
|---|---|---|
| `A` | `U+0041` | `41` |
| `é` | `U+00E9` | `C3 A9` |
| `中` | `U+4E2D` | `E4 B8 AD` |
| `😀` | `U+1F600` | `F0 9F 98 80` |

ASCII characters retain their original byte values in UTF-8. This makes UTF-8 backward-compatible with ASCII.

The variable-width representation is important: a single Unicode code point may occupy multiple bytes.

### 3.1 UTF-8 byte patterns

UTF-8 uses specific bit patterns to identify the role of each byte.

| Binary pattern | Meaning |
|---|---|
| `0xxxxxxx` | A one-byte sequence |
| `110xxxxx` | Start of a two-byte sequence |
| `1110xxxx` | Start of a three-byte sequence |
| `11110xxx` | Start of a four-byte sequence |
| `10xxxxxx` | Continuation byte |

These patterns describe the structure of UTF-8. There are also additional validity rules, including restrictions on overlong encodings, surrogate code points, and values beyond the Unicode range.

A byte beginning with `11111` is not a valid UTF-8 byte pattern.

### 3.2 Why is `0xFF` invalid UTF-8?

Consider the hexadecimal value:

```text
0xFF = 11111111
```

Its binary representation begins with `11111`, which is not a permitted UTF-8 sequence pattern.

Therefore, `0xFF` is invalid in UTF-8 regardless of what bytes follow it.

However, `0xFF` is still a perfectly valid byte value. It is invalid only when interpreted as part of UTF-8 text.

This distinction matters in systems programming because many APIs operate on arbitrary bytes rather than text.

### 3.3 Incomplete sequences versus invalid bytes

Consider two examples:

```text
C3
```

`0xC3` can begin a two-byte UTF-8 sequence, but the continuation byte is missing. This is an incomplete sequence.

Now consider:

```text
FF
```

`0xFF` is not allowed in any valid UTF-8 sequence. It is an invalid byte for this encoding.

Rust can detect both conditions when validating byte data as UTF-8.

```rust
fn main() {
    assert!(std::str::from_utf8(&[0x41]).is_ok());
    assert!(std::str::from_utf8(&[0xC3, 0xA9]).is_ok());

    assert!(std::str::from_utf8(&[0xC3]).is_err());
    assert!(std::str::from_utf8(&[0xFF]).is_err());
}
```

## 4. UTF-8 is not the only Unicode encoding

Unicode defines code points, while encodings define how those code points become bytes or code units.

UTF-8, UTF-16, and UTF-32 are different Unicode encoding forms.

For example, the code point `U+1F600` (😀) is represented as:

```text
UTF-8:  F0 9F 98 80

UTF-16: D83D DE00
        (two 16-bit code units)

UTF-32: 0001F600
        (one 32-bit code unit)
```

The byte representation of UTF-16 depends on byte order when stored as bytes. UTF-16 commonly uses little-endian or big-endian representations.

UTF-8 and UTF-16 can represent the same Unicode text, but their byte sequences differ.

This is why it is important to know which encoding a file, protocol, API, or application expects.

## 5. Unicode, UTF-8, and Rust

Rust's standard string types are designed around UTF-8.

### 5.1 `String`

`String` is an owned, growable string whose contents must always be valid UTF-8 when accessed through safe Rust APIs.

```rust
fn main() {
    let text = String::from("Hello, 世界!");

    println!("{text}");
    println!("Bytes: {}", text.len());
    println!("Unicode scalar values: {}", text.chars().count());
}
```

The output is:

```text
Hello, 世界!
Bytes: 13
Unicode scalar values: 9
```

The length is 13 bytes because ASCII letters and punctuation use one byte each, while each Chinese character uses three bytes.

The character count is nine Unicode scalar values.

`String::len()` measures bytes, not Unicode code points or user-perceived characters.

### 5.2 `str` and `&str`

`str` is a string slice type containing valid UTF-8 text. It is commonly accessed through a reference, `&str`.

```rust
fn greet(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let owned = String::from("Amir");

    greet("Amir");
    greet(&owned);
    greet(owned.as_str());
}
```

`&str` is useful for function parameters because it accepts string literals, borrowed `String` values through deref coercion, and slices of string data.

### 5.3 How does Rust guarantee valid UTF-8?

Safe Rust APIs prevent creating a `String` containing invalid UTF-8.

For example:

```rust
fn main() {
    let bytes = vec![0xC3];

    match String::from_utf8(bytes) {
        Ok(text) => println!("{text}"),
        Err(error) => {
            println!("Invalid UTF-8: {error}");
        }
    }
}
```

The conversion fails because `0xC3` is an incomplete UTF-8 sequence.

If you have a byte buffer that might contain arbitrary binary data, keep it as `Vec<u8>` or `&[u8]` until you know that interpreting it as UTF-8 is appropriate.

Unsafe Rust can violate ordinary invariants if used incorrectly, so the guarantee relies on using the APIs and invariants correctly.

### 5.4 String slicing and UTF-8 boundaries

Rust string indexes are byte offsets. Slicing is allowed only at UTF-8 character boundaries.

```rust
fn main() {
    let text = String::from("é");

    println!("{}", &text[0..2]);

    // Panics at runtime because byte index 1 is not
    // a UTF-8 character boundary.
    // println!("{}", &text[0..1]);
}
```

The character `é` occupies two bytes in UTF-8. Cutting the slice after its first byte would produce an invalid UTF-8 sequence, so Rust rejects that slice at runtime.

When you need to iterate over Unicode scalar values, use `chars()`. When you need the underlying UTF-8 bytes, use `as_bytes()`.

## 6. Operating systems do not all handle filenames the same way

A filename is not necessarily a Unicode string. Its representation depends on the operating system and its filesystem interfaces.

### 6.1 Linux and Unix-like systems

On Linux, filename components are generally handled as byte sequences.

The two important restrictions are:

- A filename component cannot contain `/`, because it separates path components.
- A filename component cannot contain a NUL byte (`0x00`), because the underlying pathname interfaces use NUL-terminated strings.

Linux does not generally require filename bytes to form valid UTF-8.

For example, a filename could contain:

```text
66 69 6C 65 FF 2E 74 78 74
```

These bytes correspond to:

```text
file<0xFF>.txt
```

The byte `0xFF` is invalid UTF-8, but it can still appear in a Linux filename.

This does not mean Linux automatically converts the filename to a different character encoding. Linux generally treats the filename bytes as bytes rather than interpreting them as UTF-8, UTF-16, or another text encoding.

Applications and terminal environments may interpret or display those bytes according to their own conventions.

### 6.2 Windows

Windows differs from Linux in this area.

Windows filesystem APIs generally represent filenames as UTF-16 strings. Rust's Windows-specific operating-system string implementation accommodates this native representation.

This does not mean every Windows filename is a sequence of Unicode scalar values in the strictest sense: Windows UTF-16 strings can contain unpaired surrogate code units, which cannot be converted into valid Unicode scalar values.

Rust's `OsString` and `OsStr` are designed to represent operating-system strings without requiring them to be valid UTF-8.

### 6.3 Does every operating system have its own encoding?

Not necessarily.

An operating system can manage data without assigning it a character encoding. On Linux, a filename may simply be a sequence of bytes.

Encoding becomes relevant when an application interprets those bytes as text.

For example, the bytes `C3 A9` can represent `é` in UTF-8, but a different encoding could interpret the same bytes differently.

The bytes themselves do not specify their intended encoding. The application, protocol, or data format establishes that interpretation.

## 7. Why does Rust have `OsStr`, `OsString`, and `PathBuf`?

If Rust's `String` is always valid UTF-8, how can Rust represent a Linux filename containing `0xFF`?

Rust provides operating-system string types that do not impose the same UTF-8 requirement.

| Type | Purpose |
|---|---|
| `String` | Owned, valid UTF-8 text |
| `&str` | Borrowed, valid UTF-8 text |
| `OsString` | Owned operating-system string |
| `&OsStr` | Borrowed operating-system string |
| `PathBuf` | Owned filesystem path |
| `&Path` | Borrowed filesystem path |
| `Vec<u8>` | Owned arbitrary bytes |
| `&[u8]` | Borrowed arbitrary bytes |

`OsString` and `OsStr` are useful for operating-system strings. `PathBuf` and `Path` build on these concepts for filesystem paths.

The exact native representation differs by platform. On Unix, `OsStr` can be accessed as bytes through Unix-specific extension traits. On Windows, the native representation is based on wide strings.

### 7.1 Creating a non-UTF-8 Linux path

The following example is specific to Unix-like systems:

```rust
use std::ffi::OsStr;
use std::os::unix::ffi::OsStrExt;
use std::path::PathBuf;

fn main() {
    let filename = OsStr::from_bytes(b"file-\xFF.txt");
    let path = PathBuf::from(filename);

    assert!(path.to_str().is_none());

    println!("{path:?}");
}
```

Here is what happens:

1. `b"file-\xFF.txt"` creates a byte string containing `0xFF`.
2. `OsStr::from_bytes()` constructs an operating-system string from those bytes without requiring UTF-8 validity.
3. `PathBuf::from()` creates an owned filesystem path.
4. `path.to_str()` returns `None` because the path is not valid UTF-8.

The example creates a path value; it does not create an actual file on disk.

If you want to create a file using this path, you can use standard filesystem APIs on Unix:

```rust
use std::ffi::OsStr;
use std::fs::File;
use std::io;
use std::os::unix::ffi::OsStrExt;
use std::path::PathBuf;

fn main() -> io::Result<()> {
    let filename = OsStr::from_bytes(b"file-\xFF.txt");
    let path = PathBuf::from(filename);

    let _file = File::create(&path)?;

    println!("Created file: {path:?}");

    Ok(())
}
```

This creates the file in the current working directory if the operation succeeds and the process has the required permissions.

### 7.2 Why does `PathBuf::to_str()` return an `Option`?

Consider:

```rust
let path = std::path::PathBuf::from("hello.txt");

let text = path.to_str();

assert_eq!(text, Some("hello.txt"));
```

The `to_str()` method returns `Option<&str>` because the path might not contain valid UTF-8.

If the path contains invalid UTF-8, the method returns `None` rather than constructing an invalid `&str`.

If you need a lossy textual representation for logging, use:

```rust
println!("{}", path.display());
```

Or:

```rust
println!("{}", path.to_string_lossy());
```

These methods are useful for display, but they are not a way to recover the original non-UTF-8 bytes from the displayed text. Invalid data may be replaced during conversion.

### 7.3 Recovering the original bytes on Unix

If you need to inspect or preserve the original path bytes, use Unix-specific APIs:

```rust
use std::os::unix::ffi::OsStrExt;
use std::path::Path;

fn inspect_path(path: &Path) {
    let bytes = path.as_os_str().as_bytes();

    println!("{bytes:?}");
}
```

For a path created from `b"file-\xFF.txt"`, the original bytes can be accessed without converting them to UTF-8.

This is useful for filesystem tools, archive utilities, backup programs, and other systems software that must handle arbitrary Linux filenames.

## 8. Why does `PathBuf` need `.display()` when printed?

You may encounter code like this:

```rust
use std::path::PathBuf;

fn main() {
    let path = PathBuf::from("/tmp/example.txt");

    println!("{}", path.display());
    println!("{path:?}");
}
```

`PathBuf` implements `Debug`, but it does not directly implement `Display`.

The formatting placeholders in Rust have different meanings:

| Format | Trait or behavior |
|---|---|
| `{}` | Uses `Display` |
| `{:?}` | Uses `Debug` |
| `{:x}` | Uses `LowerHex` |

The `.display()` method returns a formatting adapter that implements `Display` for a path.

It does not change the path's encoding or guarantee that the path is valid UTF-8. It only provides a way to format the path for human-readable output.

Use `Debug` when you want a diagnostic representation. Use `.display()` when you want a convenient, human-readable path representation.

## 9. When should you use `String` versus `PathBuf`?

Use the type that matches the data's purpose.

### Example: parsing textual configuration

If a configuration field contains a human-readable name, `String` is usually appropriate:

```rust
fn set_username(username: &str) {
    println!("Username: {username}");
}
```

### Example: opening a file

If a value represents a filesystem path, use `Path` or `PathBuf`:

```rust
use std::fs;
use std::path::Path;

fn read_config(path: &Path) -> std::io::Result<String> {
    fs::read_to_string(path)
}
```

`read_to_string()` requires the file contents to be valid UTF-8. The path itself does not need to be valid UTF-8 on Unix.

This distinction is important: **the encoding requirements of a file's contents are separate from the encoding requirements of its filename.**

### Example: reading arbitrary binary data

If a file can contain arbitrary binary data, use byte-oriented APIs:

```rust
use std::fs;
use std::io;
use std::path::Path;

fn read_binary(path: &Path) -> io::Result<Vec<u8>> {
    fs::read(path)
}
```

The returned `Vec<u8>` can contain any byte values, including `0xFF` and `0x00`.

### Example: converting bytes into text

If you receive bytes from a network socket or file, validate them before interpreting them as UTF-8:

```rust
fn decode_utf8(bytes: &[u8]) -> Result<&str, std::str::Utf8Error> {
    std::str::from_utf8(bytes)
}
```

This avoids assuming that arbitrary binary data is valid text.

## 10. Common misconceptions

**Misconception 1: `0xFF` is invalid data.**

No. It is a valid byte value, but it is not valid in UTF-8.

**Misconception 2: Every Linux filename is UTF-8.**

No. Linux generally allows filename components containing arbitrary bytes except for `/` and NUL.

**Misconception 3: Every operating system uses a different character encoding.**

Not necessarily. Operating systems differ in their native APIs and string representations. Some interfaces handle byte sequences without requiring a text encoding.

**Misconception 4: `PathBuf` always contains valid UTF-8.**

No. `PathBuf` is designed to represent filesystem paths in a platform-compatible way.

**Misconception 5: `String::len()` counts characters.**

No. It counts bytes. Use `chars().count()` to count Unicode scalar values, remembering that these are not always user-perceived characters.

**Misconception 6: `.display()` converts a path to valid UTF-8 without data loss.**

No. It provides a formatting adapter. Invalid path data may not be faithfully represented in the displayed output.

## 11. A practical mental model

When writing systems software, keep these layers separate:

1. **Bytes:** numerical values, such as `0x41` or `0xFF`.
2. **Unicode:** a standard assigning code points to characters.
3. **Encoding:** a representation of Unicode code points as bytes or code units, such as UTF-8 or UTF-16.
4. **Operating-system strings:** native string representations used by OS APIs.
5. **Rust types:** abstractions that enforce the guarantees appropriate for each kind of data.

The relationship can be summarized as follows:

- `String` and `&str` represent valid UTF-8 text.
- `Vec<u8>` and `&[u8]` represent arbitrary byte data.
- `OsString` and `OsStr` represent operating-system strings.
- `PathBuf` and `Path` represent filesystem paths.

These types are not interchangeable in meaning, even when conversions between them are possible.

## Conclusion

Unicode defines code points for textual data, while UTF-8 defines one way to encode those code points as bytes. Rust's `String` and `str` guarantee valid UTF-8, which allows Rust to provide safe string operations.

Filesystem paths have different requirements. On Linux, filenames can contain non-UTF-8 bytes, so Rust provides `OsStr`, `OsString`, `Path`, and `PathBuf` to represent them without imposing a UTF-8 requirement.

For systems programming, the most important lesson is to distinguish **text from bytes and paths from text**. That distinction helps prevent encoding bugs when working with filesystems, networking, process execution, and operating-system interfaces.
