---
title: "Rust Data Structures: Arrays, Vectors, Slices, Strings, Tuples, HashMaps, and More"
categories:
  - Learning Notes
  - Rust

tags:
    - rust
    - rust-lang
    - data-structures
    - collections
    - programming
description: "A concise practical guide to Rust's core data structures, including arrays, vectors, slices, tuples, unit type, strings, string slices, HashMaps, and common collection patterns."

toc: true
---

# Rust Data Structures

Rust has a relatively small set of built-in data structures, plus a rich standard-library collection API.

The most important ones to understand are:

```text
Array       [T; N]
Vector      Vec<T>
Slice       &[T]
Tuple       (T1, T2, ...)
Unit        ()
String      String
Str slice   &str
HashMap     HashMap<K, V>
HashSet     HashSet<T>
```

A useful first distinction is:

```text
Fixed size:
    [T; N]
    (T1, T2, ...)
    
Dynamic size:
    Vec<T>
    String
    HashMap<K, V>
    HashSet<T>

Borrowed views:
    &[T]
    &str
```

---

# 1. Array

An array contains a fixed number of values of the same type.

```rust
let numbers = [1, 2, 3, 4, 5];
```

Its type is:

```rust
[i32; 5]
```

The `5` is part of the type.

Therefore:

```rust
let a: [i32; 3] = [1, 2, 3];
let b: [i32; 5] = [1, 2, 3, 4, 5];
```

These are different types:

```text
[i32; 3]
[i32; 5]
```

---

# 2. Creating Arrays

You can explicitly specify every element:

```rust
let numbers = [1, 2, 3, 4, 5];
```

Or repeat the same value:

```rust
let zeros = [0; 5];
```

This means:

```text
[0, 0, 0, 0, 0]
```

You can also specify the type:

```rust
let numbers: [i32; 5] = [1, 2, 3, 4, 5];
```

---

# 3. Accessing Array Elements

Array indexing starts at zero:

```rust
let numbers = [10, 20, 30];

println!("{}", numbers[0]);
println!("{}", numbers[2]);
```

Output:

```text
10
30
```

An invalid index causes a panic:

```rust
let numbers = [10, 20, 30];

println!("{}", numbers[10]);
```

For safe access, use `get()`:

```rust
let value = numbers.get(10);
```

The result is:

```rust
None
```

because `get()` returns:

```rust
Option<&T>
```

This is a good example of how Rust's data structures interact with `Option`.

---

# 4. Arrays Are Stored Inline

An array's elements are stored directly as part of the array.

Conceptually:

```text
let numbers = [10, 20, 30];

stack:
+----+----+----+
| 10 | 20 | 30 |
+----+----+----+
```

There is no separate heap allocation required just to store the array.

This makes arrays useful when the number of elements is known at compile time.

---

# 5. Array Length

Use:

```rust
let numbers = [1, 2, 3, 4];

println!("{}", numbers.len());
```

Result:

```text
4
```

The length is also part of the type:

```rust
[i32; 4]
```

---

# 6. Array Iteration

You can iterate over an array:

```rust
let numbers = [1, 2, 3];

for number in numbers {
    println!("{number}");
}
```

You can also borrow the array:

```rust
for number in &numbers {
    println!("{number}");
}
```

The distinction matters when ownership is involved.

---

# 7. Vector: `Vec<T>`

A vector is Rust's growable array.

```rust
let numbers = vec![1, 2, 3];
```

Its type is:

```rust
Vec<i32>
```

Unlike:

```rust
[i32; 3]
```

a vector can grow or shrink at runtime.

```rust
let mut numbers = vec![1, 2, 3];

numbers.push(4);
numbers.push(5);
```

Now:

```text
[1, 2, 3, 4, 5]
```

---

# 8. Array vs Vector

The basic difference:

```text
Array:
    [T; N]

    fixed size
    size is part of the type
    stored inline

Vector:
    Vec<T>

    dynamic size
    heap allocation
    can grow and shrink
```

For example:

```rust
let array = [1, 2, 3];

let mut vector = vec![1, 2, 3];
vector.push(4);
```

---

# 9. Creating a Vector

Using the `vec!` macro:

```rust
let values = vec![1, 2, 3];
```

Using `Vec::new()`:

```rust
let mut values = Vec::new();

values.push(1);
values.push(2);
values.push(3);
```

Using `Vec::with_capacity()`:

```rust
let mut values = Vec::with_capacity(100);
```

This allocates enough capacity for approximately 100 elements, but the vector's length is still zero.

```rust
values.len();      // 0
values.capacity(); // at least 100
```

---

# 10. Length vs Capacity

This distinction is important.

```rust
let mut values = Vec::with_capacity(10);
```

Initially:

```text
length   = 0
capacity >= 10
```

After:

```rust
values.push(42);
```

we have:

```text
length   = 1
capacity >= 10
```

Think of:

```text
length
    = number of elements currently stored

capacity
    = number of elements that can currently fit
      before another allocation may be required
```

---

# 11. Vector Methods

Common methods include:

```rust
push()
pop()
insert()
remove()
get()
first()
last()
len()
is_empty()
clear()
contains()
sort()
reverse()
```

Example:

```rust
let mut values = vec![1, 2, 3];

values.push(4);

let last = values.pop();

println!("{:?}", last);
```

Result:

```text
Some(4)
```

Notice that `pop()` returns:

```rust
Option<T>
```

because the vector might be empty.

---

# 12. Slice

A slice is a dynamically sized view into a contiguous sequence.

The most common form is:

```rust
&[T]
```

For example:

```rust
let numbers = [1, 2, 3, 4, 5];

let slice = &numbers[1..4];
```

The slice contains:

```text
2, 3, 4
```

The important thing is:

> A slice does not own the data.

It is a borrowed view.

---

# 13. Slice From a Vector

Slices can also borrow a vector:

```rust
let numbers = vec![1, 2, 3, 4, 5];

let slice = &numbers[1..4];
```

Now:

```text
Vec
+---+---+---+---+---+
| 1 | 2 | 3 | 4 | 5 |
+---+---+---+---+---+
    ^
    |
    +--- slice [2,3,4]
```

The vector owns the data.

The slice only borrows it.

---

# 14. Why Use Slices?

Suppose we write:

```rust
fn sum(values: &[i32]) -> i32 {
    values.iter().sum()
}
```

Now we can pass both arrays:

```rust
let array = [1, 2, 3];

sum(&array);
```

and vectors:

```rust
let vector = vec![1, 2, 3];

sum(&vector);
```

This is extremely useful.

Instead of writing separate functions for:

```rust
fn sum_array(...)
fn sum_vector(...)
```

we can write:

```rust
fn sum(values: &[i32])
```

because both arrays and vectors can be viewed as slices.

---

# 15. Slice Syntax

The general syntax is:

```rust
&collection[start..end]
```

The `end` index is exclusive.

```rust
let values = [0, 1, 2, 3, 4];

let slice = &values[1..4];
```

gives:

```text
1, 2, 3
```

because:

```text
start = 1
end   = 4
```

and index `4` is excluded.

---

# 16. Slice Range Syntax

You can omit either side.

From the beginning:

```rust
&values[..3]
```

means:

```text
0..3
```

From an index to the end:

```rust
&values[2..]
```

means:

```text
2..len
```

The entire collection:

```rust
&values[..]
```

means the entire slice.

---

# 17. Mutable Slices

You can also borrow a mutable slice:

```rust
fn double(values: &mut [i32]) {
    for value in values {
        *value *= 2;
    }
}
```

Then:

```rust
let mut values = vec![1, 2, 3];

double(&mut values);
```

Now:

```text
[2, 4, 6]
```

The function doesn't care whether the original data came from:

```text
Vec
Array
another Slice
```

It only requires:

```rust
&mut [i32]
```

---

# 18. `str` and `&str`

This is one of the most confusing parts of Rust.

There are two important types:

```rust
String
&str
```

`String` owns its string data.

`&str` is a borrowed string slice.

Conceptually:

```text
String
    |
    +-- owns UTF-8 bytes

&str
    |
    +-- borrows UTF-8 bytes
```

---

# 19. `String`

`String` is an owned, growable UTF-8 string.

```rust
let mut name = String::from("Amir");

name.push_str(" Nikpour");
```

Now:

```text
Amir Nikpour
```

Because `String` owns its data, it can grow.

---

# 20. Creating a `String`

Common ways:

```rust
let s = String::new();

let s = String::from("hello");

let s = "hello".to_string();

let s = format!("Hello, {}", name);
```

For example:

```rust
let mut message = String::new();

message.push_str("Hello ");
message.push('A');
```

---

# 21. `&str`

A string literal:

```rust
"hello"
```

has type:

```rust
&'static str
```

It is a reference to string data embedded in the program.

For example:

```rust
let name: &str = "Amir";
```

The variable does not own the string data.

---

# 22. `String` vs `&str`

A useful mental model:

```text
String
+------------------------+
| pointer                |
| length                 |
| capacity               |
+------------------------+
          |
          v
      heap bytes
      "hello"
```

While:

```text
&str
+----------------+
| pointer        |
| length         |
+----------------+
        |
        v
    UTF-8 bytes
```

`&str` is essentially a **fat pointer** containing:

```text
pointer + length
```

It does not contain capacity because it does not own or manage the allocation.

---

# 23. Converting Between `String` and `&str`

From `String` to `&str`:

```rust
let name = String::from("Amir");

let slice: &str = &name;
```

You can also use:

```rust
let slice = name.as_str();
```

From `&str` to `String`:

```rust
let slice = "Amir";

let name = slice.to_string();
```

or:

```rust
let name = String::from(slice);
```

This creates owned data.

---

# 24. Function Parameters: Prefer `&str` When You Only Need to Read

Instead of:

```rust
fn greet(name: String) {
    println!("Hello {name}");
}
```

if the function does not need ownership, use:

```rust
fn greet(name: &str) {
    println!("Hello {name}");
}
```

Now both work:

```rust
let name = String::from("Amir");

greet(&name);
greet("Amir");
```

This is an important Rust API-design pattern.

If a function only needs to read a string, `&str` is often more flexible than `String`.

---

# 25. String Slices

A `String` can be sliced:

```rust
let hello = String::from("Hello world");

let hello_slice = &hello[0..5];
```

Result:

```text
Hello
```

But remember that Rust strings are UTF-8.

---

# 26. Rust Strings Are UTF-8

A `String` is not an array of characters.

For example:

```rust
let s = String::from("hello");
```

contains UTF-8 bytes.

This is why:

```rust
s[0]
```

is not allowed.

Instead, you can iterate over characters:

```rust
for c in s.chars() {
    println!("{c}");
}
```

Or bytes:

```rust
for b in s.bytes() {
    println!("{b}");
}
```

These are different concepts:

```text
bytes
    -> UTF-8 encoded bytes

chars()
    -> Unicode scalar values
```

---

# 27. `String` Methods

Common methods include:

```rust
push()
push_str()
pop()
len()
is_empty()
clear()
contains()
starts_with()
ends_with()
replace()
replace_range()
split()
chars()
bytes()
as_str()
```

Example:

```rust
let mut s = String::from("hello");

s.push('!');
s.push_str(" Rust");

println!("{s}");
```

---

# 28. Tuple

A tuple groups values of potentially different types.

```rust
let user = ("Amir", 29, true);
```

Its type is:

```rust
(&str, i32, bool)
```

Unlike arrays:

```text
Array:
    same type

Tuple:
    different types allowed
```

---

# 29. Accessing Tuple Values

Tuple fields are accessed using indexes:

```rust
let user = ("Amir", 29, true);

println!("{}", user.0);
println!("{}", user.1);
println!("{}", user.2);
```

Output:

```text
Amir
29
true
```

Tuple indexes are known at compile time.

---

# 30. Tuple Destructuring

You can destructure a tuple:

```rust
let user = ("Amir", 29, true);

let (name, age, active) = user;

println!("{name}");
println!("{age}");
println!("{active}");
```

This is called pattern destructuring.

You can ignore values:

```rust
let (name, _, active) = user;
```

---

# 31. Nested Tuples

Tuples can contain other tuples:

```rust
let value = (1, ("hello", true));
```

Type:

```rust
(i32, (&str, bool))
```

---

# 32. Unit Type `()`

The unit type is:

```rust
()
```

It represents exactly one value:

```rust
()
```

It is used when there is no meaningful value to return.

For example:

```rust
fn print_message() {
    println!("hello");
}
```

is essentially:

```rust
fn print_message() -> () {
    println!("hello");
}
```

---

# 33. `()` Is a Real Type

This is important:

```text
()
```

is not "nothing".

It is a value of a real type:

```text
unit type: ()
unit value: ()
```

For example:

```rust
let value: () = ();
```

This is why:

```rust
Result<(), Error>
```

means:

```text
Ok(())
    -> operation succeeded

Err(error)
    -> operation failed
```

---

# 34. Struct

Structs are one of Rust's main ways of creating custom data structures.

```rust
struct User {
    name: String,
    age: u32,
}
```

Create one:

```rust
let user = User {
    name: String::from("Amir"),
    age: 29,
};
```

Access fields:

```rust
println!("{}", user.name);
println!("{}", user.age);
```

---

# 35. Tuple Struct

A tuple struct has unnamed fields:

```rust
struct Point(i32, i32);
```

Create:

```rust
let point = Point(10, 20);
```

Access:

```rust
println!("{}", point.0);
println!("{}", point.1);
```

Tuple structs are useful when the fields have meaning but don't need names.

---

# 36. Enum

An enum represents one of several possible variants.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}
```

Create:

```rust
let message = Message::Write(String::from("hello"));
```

Enums are especially powerful in Rust because variants can contain different data.

---

# 37. `Option` Is an Enum

For example:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

So:

```rust
Option<String>
```

can contain either:

```text
Some(String)
```

or:

```text
None
```

---

# 38. `Result` Is an Enum

Similarly:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

This is why `Option` and `Result` are closely related to Rust's general enum system.

---

# 39. HashMap

`HashMap<K, V>` stores key-value pairs.

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert("Alice", 10);
scores.insert("Bob", 20);
```

Conceptually:

```text
HashMap

"Alice" -> 10
"Bob"   -> 20
```

Keys are used to find values.

---

# 40. Accessing a HashMap

Use `get()`:

```rust
let score = scores.get("Alice");
```

The result is:

```rust
Option<&i32>
```

Why?

Because the key might not exist.

```text
key exists
    -> Some(&value)

key doesn't exist
    -> None
```

This is another common use of `Option`.

---

# 41. HashMap Indexing

You can also write:

```rust
let score = scores["Alice"];
```

But if the key does not exist, indexing can panic.

Therefore:

```rust
scores.get("Alice")
```

is safer when the key might be missing.

---

# 42. Updating a HashMap

Insert a value:

```rust
scores.insert("Alice", 10);
```

If the key already exists:

```rust
scores.insert("Alice", 20);
```

the previous value is replaced.

The return value of `insert()` is:

```rust
Option<V>
```

It returns the old value if there was one.

---

# 43. `entry()` API

The `entry()` API is extremely useful.

Suppose we want to count words:

```rust
use std::collections::HashMap;

let mut counts = HashMap::new();

for word in ["hello", "world", "hello"] {
    let count = counts.entry(word).or_insert(0);
    *count += 1;
}
```

Result:

```text
hello -> 2
world -> 1
```

The important pattern is:

```rust
map.entry(key).or_insert(default)
```

If the key exists, it returns a mutable reference to the existing value.

If it doesn't exist, it inserts the default.

---

# 44. `or_insert()` and `or_insert_with()`

```rust
map.entry(key).or_insert(value);
```

uses an already-created value.

For expensive initialization:

```rust
map.entry(key).or_insert_with(|| expensive_operation());
```

The closure is evaluated only when the entry does not exist.

This follows the same eager/lazy pattern seen elsewhere in Rust.

---

# 45. HashMap Iteration

Iterate over keys and values:

```rust
for (key, value) in &scores {
    println!("{key}: {value}");
}
```

Keys only:

```rust
for key in scores.keys() {
    println!("{key}");
}
```

Values only:

```rust
for value in scores.values() {
    println!("{value}");
}
```

Mutable values:

```rust
for value in scores.values_mut() {
    *value += 1;
}
```

---

# 46. HashSet

A `HashSet<T>` stores unique values.

```rust
use std::collections::HashSet;

let mut users = HashSet::new();

users.insert("Alice");
users.insert("Bob");
users.insert("Alice");
```

The second `"Alice"` is not added as a duplicate.

Conceptually:

```text
Alice
Bob
```

---

# 47. HashSet Membership

Use:

```rust
if users.contains("Alice") {
    println!("Alice exists");
}
```

`contains()` returns:

```rust
bool
```

---

# 48. HashSet vs HashMap

Think of:

```text
HashSet<T>
    = collection of unique values

HashMap<K, V>
    = collection of key -> value pairs
```

For example:

```text
HashSet:
    Alice
    Bob
    Charlie

HashMap:
    Alice -> 10
    Bob   -> 20
    Charlie -> 30
```

---

# 49. `VecDeque`

For queue-like data structures:

```rust
use std::collections::VecDeque;

let mut queue = VecDeque::new();

queue.push_back(1);
queue.push_back(2);
queue.push_front(0);
```

Result:

```text
0 1 2
```

Remove from the front:

```rust
let value = queue.pop_front();
```

Remove from the back:

```rust
let value = queue.pop_back();
```

`VecDeque` is useful when you need efficient insertion/removal from both ends.

---

# 50. `LinkedList`

Rust also provides:

```rust
std::collections::LinkedList<T>
```

but it is much less commonly useful than `Vec` or `VecDeque`.

A linked list stores nodes connected through links:

```text
Node -> Node -> Node -> Node
```

Compared with `Vec`:

```text
Vec:
[1][2][3][4]
```

For most ordinary workloads, a contiguous `Vec` is often preferable because of cache locality and simpler memory access.

Use `LinkedList` only when its particular characteristics actually solve a problem.

---

# 51. `BTreeMap`

Rust also provides:

```rust
use std::collections::BTreeMap;
```

It stores key-value pairs in an ordered tree structure.

```rust
let mut users = BTreeMap::new();

users.insert("Alice", 10);
users.insert("Bob", 20);
```

Unlike a `HashMap`, iteration is ordered by key.

Conceptually:

```text
HashMap:
    unordered iteration

BTreeMap:
    ordered by key
```

---

# 52. `BTreeSet`

Similarly:

```rust
use std::collections::BTreeSet;
```

stores unique values in sorted order.

```rust
let mut values = BTreeSet::new();

values.insert(30);
values.insert(10);
values.insert(20);
```

Iteration produces:

```text
10
20
30
```

---

# 53. Choosing Between `HashMap` and `BTreeMap`

A simple mental model:

```text
HashMap<K, V>
    -> fast average lookup
    -> no sorted iteration requirement

BTreeMap<K, V>
    -> ordered keys
    -> range queries
    -> sorted iteration
```

Don't choose based only on theoretical complexity. The actual workload, memory layout, cache behavior, and API requirements matter.

---

# 54. Collection Overview

The most important standard collections are:

```text
Vec<T>
    Growable contiguous sequence

VecDeque<T>
    Double-ended queue

LinkedList<T>
    Doubly linked list

HashMap<K, V>
    Hash table

HashSet<T>
    Hash set

BTreeMap<K, V>
    Ordered map

BTreeSet<T>
    Ordered set
```

---

# 55. Contiguous vs Non-Contiguous Collections

This distinction is important for systems programming.

`Vec<T>` stores elements contiguously:

```text
+----+----+----+----+----+
| T  | T  | T  | T  | T  |
+----+----+----+----+----+
```

This gives good cache locality.

A linked list is conceptually:

```text
+------+       +------+       +------+
| Node | ----> | Node | ----> | Node |
+------+       +------+       +------+
```

The nodes can be located in different areas of memory.

This affects performance significantly.

---

# 56. Slices as Abstractions

One of the most useful Rust patterns is to accept slices in APIs.

Instead of:

```rust
fn process(values: Vec<i32>) {
    // ...
}
```

you can often write:

```rust
fn process(values: &[i32]) {
    // ...
}
```

Now the caller doesn't need to give up ownership.

It can pass:

```rust
let values = vec![1, 2, 3];

process(&values);
```

or:

```rust
let values = [1, 2, 3];

process(&values);
```

This makes the API more general.

---

# 57. `&[T]` Is Similar to `&str`

There is a useful analogy:

```text
Vec<T>       -> &[T]
String       -> &str
```

In both cases:

```text
owned value
    |
    | borrow
    v
borrowed view
```

For example:

```text
Vec<T>
    -> &[T]

String
    -> &str
```

The borrowed versions do not own the underlying data.

---

# 58. Ownership Comparison

A useful summary:

```text
[T; N]
    owns fixed-size data

Vec<T>
    owns growable data

&[T]
    borrows a sequence

String
    owns growable UTF-8 data

&str
    borrows UTF-8 string data

HashMap<K, V>
    owns key-value entries

Tuple
    owns its fields

Struct
    owns its fields
```

---

# 59. Copying vs Borrowing

Consider:

```rust
let values = vec![1, 2, 3];

let other = values;
```

`values` is moved.

You cannot use:

```rust
println!("{:?}", values);
```

afterward.

But:

```rust
let values = vec![1, 2, 3];

let slice = &values[..];

println!("{:?}", values);
println!("{:?}", slice);
```

works because `slice` only borrows the vector.

This is one of the fundamental reasons slices are so useful.

---

# 60. Passing Collections to Functions

Ownership:

```rust
fn consume(values: Vec<i32>) {
    // takes ownership
}
```

Borrow:

```rust
fn read(values: &[i32]) {
    // borrows
}
```

Mutable borrow:

```rust
fn modify(values: &mut [i32]) {
    // mutably borrows
}
```

The same idea applies to strings:

```rust
fn consume(name: String) {}

fn read(name: &str) {}

fn modify(name: &mut String) {}
```

---

# 61. A Useful Type Hierarchy

For sequences:

```text
                 sequence
                    |
        +-----------+-----------+
        |                       |
    owned data              borrowed view
        |                       |
    +---+---+               +---+---+
    |       |               |       |
 [T; N]   Vec<T>          &[T]    &str
                             |
                           strings
```

More precisely, `&str` is specifically a borrowed UTF-8 string slice, while `&[T]` is a generic borrowed slice.

---

# 62. `str` Is Unsized

This is a useful connection to Rust's `Sized` concept.

You normally don't have:

```rust
let s: str;
```

because `str` does not have a known size at compile time.

Instead you use:

```rust
&str
```

A `&str` is a reference to dynamically sized string data.

Similarly:

```rust
[T]
```

is an unsized slice type.

You normally use:

```rust
&[T]
```

---

# 63. Arrays, Slices, and `Sized`

Compare:

```text
[i32; 5]
```

The compiler knows:

```text
size = 5 * size_of::<i32>()
```

So the type has a known size.

But:

```text
[i32]
```

does not specify how many elements exist.

Therefore it is dynamically sized.

This is why we usually use:

```rust
&[i32]
```

instead of:

```rust
[i32]
```

---

# 64. Indexing vs Iterators

You can access elements using indexing:

```rust
let values = vec![10, 20, 30];

let value = values[1];
```

But Rust code often prefers iterators for processing:

```rust
let sum: i32 = values.iter().sum();
```

Or:

```rust
let doubled: Vec<i32> =
    values.iter()
        .map(|x| x * 2)
        .collect();
```

This works naturally with slices and many other collections.

---

# 65. `iter()` vs `iter_mut()` vs `into_iter()`

This is important when working with collections.

### `iter()`

Borrow elements:

```rust
for value in values.iter() {
    println!("{value}");
}
```

Conceptually:

```text
&T
```

### `iter_mut()`

Mutably borrow elements:

```rust
for value in values.iter_mut() {
    *value += 1;
}
```

Conceptually:

```text
&mut T
```

### `into_iter()`

Consume the collection:

```rust
for value in values.into_iter() {
    println!("{value}");
}
```

Conceptually:

```text
T
```

So:

```text
iter()
    -> &T

iter_mut()
    -> &mut T

into_iter()
    -> T
```

This is closely connected to Rust's ownership system.

---

# 66. `String` and Collection Iteration

For strings:

```rust
let text = String::from("hello");
```

You can iterate over Unicode scalar values:

```rust
for character in text.chars() {
    println!("{character}");
}
```

Or UTF-8 bytes:

```rust
for byte in text.bytes() {
    println!("{byte}");
}
```

Don't confuse:

```text
bytes()
    -> UTF-8 bytes

chars()
    -> Unicode scalar values
```

---

# 67. Formatting Collections

For debugging:

```rust
let values = vec![1, 2, 3];

println!("{:?}", values);
```

For pretty debugging:

```rust
println!("{:#?}", values);
```

Many Rust collections implement `Debug`, making this very useful during development.

---

# 68. `String` Is Not `Vec<char>`

This is an important misconception.

It is tempting to think:

```text
String = Vec<char>
```

but that's not how Rust strings work.

A `String` stores UTF-8 encoded bytes.

For example, one visible character can require multiple bytes.

Therefore:

```rust
let s = String::from("é");

println!("{}", s.len());
```

does not necessarily equal:

```text
1
```

because `len()` reports the number of bytes, not the number of Unicode characters.

---

# 69. `len()` Means Different Things

Be careful with `len()`.

For:

```rust
Vec<T>
```

it means:

```text
number of elements
```

For:

```rust
String
```

it means:

```text
number of UTF-8 bytes
```

For:

```rust
&str
```

it also means:

```text
number of UTF-8 bytes
```

For:

```rust
HashMap
```

it means:

```text
number of key-value entries
```

---

# 70. Core Data Structures Cheat Sheet

```text
┌─────────────────────┬──────────────────────────────┐
│ Type                │ Meaning                      │
├─────────────────────┼──────────────────────────────┤
│ [T; N]              │ Fixed-size array             │
│ Vec<T>              │ Growable array                │
│ &[T]                │ Borrowed slice                │
│ &mut [T]             │ Mutable borrowed slice       │
│ String              │ Owned growable UTF-8 string   │
│ &str                │ Borrowed UTF-8 string slice   │
│ (T1, T2, ...)        │ Tuple                        │
│ ()                  │ Unit type                    │
│ struct              │ Custom data structure        │
│ enum                │ One of several variants      │
│ HashMap<K, V>        │ Hash table                   │
│ HashSet<T>           │ Hash set                     │
│ BTreeMap<K, V>       │ Ordered map                  │
│ BTreeSet<T>          │ Ordered set                  │
│ VecDeque<T>          │ Double-ended queue           │
│ LinkedList<T>        │ Doubly linked list           │
└─────────────────────┴──────────────────────────────┘
```

---

# 71. The Most Important Relationships

For Rust development, especially systems programming, remember these relationships:

```text
Array
    [T; N]
       |
       | borrow
       v
    &[T]
```

```text
Vec<T>
    |
    | borrow
    v
&[T]
```

```text
String
    |
    | borrow
    v
&str
```

And:

```text
Vec<T>
    |
    +-- iter()      -> &T
    |
    +-- iter_mut()  -> &mut T
    |
    +-- into_iter() -> T
```

---

# 72. Choosing the Right Structure

A practical decision tree:

```text
Do I know the size at compile time?
        |
       yes
        |
        v
     [T; N]

       no
        |
        v
Do I need a contiguous growable sequence?
        |
       yes
        |
        v
     Vec<T>
```

For a borrowed sequence:

```text
Do I need to own the elements?
        |
       no
        |
        v
     &[T]
```

For strings:

```text
Do I need an owned growable UTF-8 string?
        |
       yes
        |
        v
     String

Do I only need to read a string?
        |
       yes
        |
        v
     &str
```

For key-value lookup:

```text
Do I need key -> value lookup?
        |
       yes
        |
        +-- no ordering needed -> HashMap
        |
        +-- ordered keys        -> BTreeMap
```

For unique values:

```text
HashSet
    -> uniqueness

BTreeSet
    -> uniqueness + ordering
```

For queue behavior:

```text
VecDeque
    -> efficient front/back operations
```

---

# 73. A Systems Programming Perspective

When working with low-level Rust, the most important distinction is often not simply "which collection?"

It is:

```text
Who owns the memory?
        |
        v
Who can mutate it?
        |
        v
How long can the reference live?
        |
        v
Is the data contiguous?
        |
        v
Is its size known at compile time?
```

For example:

```rust
fn process(buffer: &[u8]) {
    // ...
}
```

This function says:

```text
I don't own the buffer.
I only need read access.
The buffer is contiguous.
I don't care whether it came from an array or Vec.
```

While:

```rust
fn process(buffer: Vec<u8>) {
    // ...
}
```

says:

```text
I take ownership of the vector.
```

And:

```rust
fn process(buffer: &mut [u8]) {
    // ...
}
```

says:

```text
I don't own the buffer.
I need mutable access.
The data is contiguous.
```

This distinction becomes particularly important when working with networking, storage, virtualization, device I/O, and memory management.

---

# 74. Key Takeaways

The most important types to remember are:

```rust
[T; N]
```

Fixed-size array.

```rust
Vec<T>
```

Owned, growable contiguous collection.

```rust
&[T]
```

Borrowed view over contiguous elements.

```rust
String
```

Owned, growable UTF-8 string.

```rust
&str
```

Borrowed UTF-8 string slice.

```rust
(T1, T2, ...)
```

Heterogeneous fixed-size tuple.

```rust
()
```

Unit type representing one empty value.

```rust
HashMap<K, V>
```

Key-value lookup.

```rust
HashSet<T>
```

Unique values.

And remember the two especially useful API relationships:

```text
Vec<T>  -> &[T]
String  -> &str
```

The owned type owns the data; the slice provides a borrowed view.

For functions that only need to inspect data, accepting a borrowed slice such as:

```rust
fn process(values: &[T])
```

or a string slice:

```rust
fn process(text: &str)
```

## often produces a more flexible API than taking ownership of `Vec<T>` or `String`.
