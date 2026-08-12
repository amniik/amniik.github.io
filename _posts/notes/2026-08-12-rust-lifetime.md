---
title: "Rust Notes: Lifetime"
categories:
  - Learning Notes
  - Rust
tags: [rust, lifetimes, ownership, borrowing, references]
description: "A review of Lifetime in Rust."

toc: true
---

# Lifetimes in Rust

Lifetimes are one of the most important parts of Rust's ownership system.

At first, lifetimes can feel strange because Rust introduces syntax such as:

```rust
&'a str
```

and:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str
```

It is tempting to think that `'a` represents a duration of time.

It does not.

A lifetime is primarily a **relationship between references**.

The compiler uses lifetimes to make sure that a reference never outlives the data it refers to.

The most important idea is:

> **A lifetime describes how long a reference is valid, and lifetime annotations describe relationships between references.**

---

# 1. Why Do We Need Lifetimes?

Consider:

```rust
fn main() {
    let r;

    {
        let x = 10;
        r = &x;
    }

    println!("{r}");
}
```

This is invalid.

Why?

Because `x` is destroyed at the end of the inner scope:

```text
{
    let x = 10;
    r = &x;
}   ← x is dropped here

println!("{r}");  ← r would point to dead data
```

Rust prevents this at compile time.

Conceptually:

```text
x:
|----------------|
        scope ends

r:
|------------------------|
        would outlive x
```

Therefore:

```text
reference lifetime > data lifetime
        ↓
      invalid
```

Rust rejects the program.

---

# 2. Lifetimes Are About References

Suppose:

```rust
let x = String::from("hello");

let r = &x;
```

There are two things involved:

```text
x
└── owns String data

r
└── borrows x
```

The lifetime is associated with the reference:

```rust
r: &'a String
```

It describes the period during which `r` is valid as a reference to `x`.

It does not mean that `x` itself has an `'a` lifetime.

---

# 3. Lifetime Is Not a Runtime Timer

This is one of the most important things to understand.

A lifetime is not:

```text
5 seconds
10 seconds
100 milliseconds
```

Rust does not insert a timer into your program.

Instead, lifetimes are mainly a **compile-time concept**.

For example:

```rust
let x = String::from("hello");
let r = &x;

println!("{r}");
```

The compiler analyzes the scopes and determines that:

```text
x is alive while r is used
```

Therefore the reference is valid.

---

# 4. Lifetime as a Region of Validity

It is useful to imagine a lifetime as a region in the program where a reference is valid.

For example:

```rust
let x = String::from("hello");

{
    let r = &x;
    println!("{r}");
}
```

Conceptually:

```text
x lifetime:
|-----------------------------|

r lifetime:
      |-------------|
```

The lifetime of `r` is shorter than the lifetime of `x`.

That is perfectly valid.

The important rule is:

> **A reference cannot be used outside the lifetime of the value it refers to.**

---

# 5. The Borrow Checker Checks Lifetimes

The borrow checker uses ownership, borrowing, and lifetime analysis together.

For example:

```rust
fn main() {
    let x = String::from("hello");

    let r = &x;

    println!("{r}");
}
```

The compiler knows:

```text
x owns the String
r borrows x
r must not outlive x
```

Because `x` remains alive while `r` is used, the program is valid.

---

# 6. The Most Important Example

Consider:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

You might expect this to work.

But Rust cannot determine the lifetime of the returned reference.

Why?

Because the function can return either:

```text
x
```

or:

```text
y
```

The compiler needs to know the relationship between:

```text
lifetime of x
lifetime of y
lifetime of returned reference
```

We express that relationship using a lifetime parameter.

---

# 7. Lifetime Parameters

The function becomes:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

Here:

```rust
'a
```

is a lifetime parameter.

It says:

```text
x is valid for lifetime 'a
y is valid for lifetime 'a
returned reference is valid for lifetime 'a
```

The important relationship is:

```text
'a
 │
 ├── x: &'a str
 ├── y: &'a str
 └── return: &'a str
```

This does NOT mean:

> `x`, `y`, and the return value live for exactly the same amount of time.

Instead, it expresses a relationship between them.

---

# 8. What Does `'a` Actually Mean?

Consider:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str
```

A useful way to read it is:

> For some lifetime `'a`, both input references are valid for `'a`, and the returned reference is also valid for `'a`.

The compiler chooses an appropriate lifetime when the function is called.

For example:

```rust
let result;

let s1 = String::from("long string");

{
    let s2 = String::from("short");

    result = longest(&s1, &s2);

    println!("{result}");
}
```

The returned reference cannot remain valid after `s2` disappears if the function might return `s2`.

Therefore the compiler effectively chooses a lifetime constrained by both references.

Conceptually:

```text
s1:
|-----------------------------|

s2:
       |-------------|

'a:
       |-------------|
```

The lifetime must fit within the validity of both inputs.

---

# 9. Lifetime Parameters Do Not Extend Lifetimes

This is a common misunderstanding.

Writing:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str
```

does NOT make `x` and `y` live longer.

It does not say:

> Keep these strings alive for `'a`.

Instead, it says:

> The returned reference cannot be valid for longer than the lifetime represented by `'a`.

The caller still controls how long the actual values live.

---

# 10. Lifetime Annotation Syntax

The basic syntax is:

```rust
'a
```

A lifetime name starts with an apostrophe:

```rust
'a
'b
'c
```

For example:

```rust
&'a str
```

A reference with a mutable borrow:

```rust
&'a mut String
```

A lifetime parameter:

```rust
fn foo<'a>() {
}
```

Multiple lifetime parameters:

```rust
fn foo<'a, 'b>() {
}
```

---

# 11. Lifetime on Immutable References

Syntax:

```rust
&'a T
```

Example:

```rust
fn print<'a>(value: &'a String) {
    println!("{value}");
}
```

Here:

```text
&'a String
```

means:

> A reference to a `String` that is valid for lifetime `'a`.

---

# 12. Lifetime on Mutable References

Syntax:

```rust
&'a mut T
```

Example:

```rust
fn modify<'a>(value: &'a mut String) {
    value.push_str(" world");
}
```

Here:

```rust
&'a mut String
```

is a mutable reference with lifetime `'a`.

Remember that lifetime and mutability are separate concepts:

```rust
&'a T
```

means:

```text
immutable reference
+
lifetime 'a
```

while:

```rust
&'a mut T
```

means:

```text
mutable reference
+
lifetime 'a
```

---

# 13. Lifetime Parameters on Functions

Syntax:

```rust
fn function<'a>(x: &'a T) -> &'a T {
    x
}
```

Example:

```rust
fn identity<'a>(x: &'a str) -> &'a str {
    x
}
```

This expresses:

```text
input lifetime
      ↓
     'a
      ↓
output lifetime
```

The returned reference is tied to the input reference.

---

# 14. Multiple Lifetimes

A function can have multiple lifetime parameters:

```rust
fn choose<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x
}
```

Here:

```text
'a → lifetime of x
'b → lifetime of y
```

The returned reference is tied specifically to `'a`.

This makes sense because the function always returns `x`.

---

# 15. Multiple Lifetimes with Different Relationships

Consider:

```rust
fn first<'a, 'b>(
    x: &'a str,
    y: &'b str,
) -> &'a str {
    x
}
```

The compiler knows:

```text
return → x
```

Therefore:

```text
return lifetime → 'a
```

It does not need to be related to `'b`.

---

# 16. Lifetime Relationships

You can express relationships between lifetimes.

For example:

```rust
fn foo<'a, 'b>(x: &'a str, y: &'b str)
where
    'a: 'b,
{
}
```

The syntax:

```rust
'a: 'b
```

means:

> Lifetime `'a` outlives lifetime `'b`.

Conceptually:

```text
'a:
|----------------------|

'b:
     |-----------|
```

So:

```text
'a: 'b
```

means `'a` lasts at least as long as `'b`.

---

# 17. Lifetime Bounds

A lifetime can also appear in a type bound.

For example:

```rust
T: 'a
```

This means:

> `T` contains no references that are invalid before `'a`.

It is often easier to think of it as:

> `T` is valid for at least lifetime `'a`.

Example:

```rust
fn foo<'a, T: 'a>(value: T) {
}
```

---

# 18. `where` Lifetime Bounds

The same constraint can be written using `where`:

```rust
fn foo<'a, T>(value: T)
where
    T: 'a,
{
}
```

This is often easier to read when there are many bounds.

---

# 19. Structs Containing References

A struct can contain a reference.

For example:

```rust
struct Important<'a> {
    part: &'a str,
}
```

The lifetime parameter belongs to the struct:

```text
Important<'a>
       │
       └── part: &'a str
```

Example:

```rust
let text = String::from("hello");

let important = Important {
    part: &text,
};
```

The struct cannot outlive `text`.

---

# 20. Why Does the Struct Need a Lifetime?

Consider:

```rust
struct Important {
    part: &str,
}
```

This does not tell Rust how the reference relates to the lifetime of the struct.

Instead:

```rust
struct Important<'a> {
    part: &'a str,
}
```

means:

> `Important<'a>` contains a reference that must remain valid for `'a`.

Therefore:

```text
text
│
├── owns String
│
└── must outlive
        ↓
Important<'a>
        │
        └── part: &'a str
```

---

# 21. Multiple References in a Struct

You can have multiple lifetime parameters:

```rust
struct Pair<'a, 'b> {
    first: &'a str,
    second: &'b str,
}
```

Here:

```text
first  → 'a
second → 'b
```

The two references do not have to have the same lifetime.

---

# 22. One Lifetime for Multiple References

You can also use one lifetime:

```rust
struct Pair<'a> {
    first: &'a str,
    second: &'a str,
}
```

This expresses a relationship between the two references.

It means both references are valid for `'a`.

Again, this does not mean they necessarily have identical real-world lifetimes.

---

# 23. Lifetime on `impl`

If a struct has a lifetime parameter:

```rust
struct Important<'a> {
    part: &'a str,
}
```

its `impl` usually includes the lifetime:

```rust
impl<'a> Important<'a> {
    fn part(&self) -> &str {
        self.part
    }
}
```

The two occurrences have different roles:

```text
impl<'a>
     ↑
declares lifetime parameter

Important<'a>
         ↑
uses that lifetime parameter
```

---

# 24. Lifetime on Methods

Example:

```rust
impl<'a> Important<'a> {
    fn get_part(&self) -> &'a str {
        self.part
    }
}
```

Here the returned reference uses the lifetime stored in the struct.

You can also introduce a separate lifetime for the method:

```rust
impl<'a> Important<'a> {
    fn choose<'b>(&self, other: &'b str) -> &'b str {
        other
    }
}
```

Now:

```text
'a → struct lifetime
'b → method lifetime
```

---

# 25. Lifetime Elision

You will often see functions without lifetime annotations:

```rust
fn first(s: &str) -> &str {
    s
}
```

You might wonder:

> Where is the lifetime?

Rust has **lifetime elision rules**.

The compiler can infer the lifetime relationship.

Conceptually, the compiler treats the function approximately like:

```rust
fn first<'a>(s: &'a str) -> &'a str {
    s
}
```

Therefore you usually don't need to write the lifetime explicitly.

---

# 26. Lifetime Elision Rule #1

Every reference parameter gets its own lifetime parameter.

For example:

```rust
fn foo(x: &str, y: &str) {
}
```

is conceptually treated like:

```rust
fn foo<'a, 'b>(x: &'a str, y: &'b str) {
}
```

Conceptually, not literally in the source code.

---

# 27. Lifetime Elision Rule #2

If there is exactly one input lifetime, that lifetime is assigned to all output references.

For example:

```rust
fn identity(x: &str) -> &str {
    x
}
```

is approximately:

```rust
fn identity<'a>(x: &'a str) -> &'a str {
    x
}
```

This is why many simple functions don't need lifetime annotations.

---

# 28. Lifetime Elision and Methods

Methods have an additional rule.

The lifetime of `&self` or `&mut self` is assigned to output references.

For example:

```rust
impl User {
    fn name(&self) -> &str {
        &self.name
    }
}
```

Conceptually:

```rust
impl User {
    fn name<'a>(&'a self) -> &'a str {
        &self.name
    }
}
```

This is extremely common in Rust.

---

# 29. Why `longest` Needs an Explicit Lifetime

Now compare:

```rust
fn first(x: &str) -> &str {
    x
}
```

with:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

The first has one input reference.

Rust can infer:

```text
input → output
```

The second has two input references:

```text
x → ?
y → ?
```

and the output can come from either one.

Therefore Rust needs us to specify the relationship:

```rust
fn longest<'a>(
    x: &'a str,
    y: &'a str,
) -> &'a str
```

---

# 30. Lifetimes Do Not Determine Which Value Is Returned

This function:

```rust
fn longest<'a>(
    x: &'a str,
    y: &'a str,
) -> &'a str
```

does not mean:

> Return the value that has lifetime `'a`.

It means:

> The returned reference is connected to the lifetime represented by `'a`.

The actual returned value is determined by:

```rust
if x.len() > y.len() {
    x
} else {
    y
}
```

---

# 31. Lifetime Annotations Do Not Change Runtime Behavior

Adding:

```rust
'a
```

does not make your program:

* slower
* faster
* allocate memory
* create a runtime object
* keep a variable alive
* extend a variable's scope

Lifetimes are primarily used by the compiler.

For example:

```rust
fn identity<'a>(x: &'a str) -> &'a str {
    x
}
```

does not create a runtime `'a` object.

---

# 32. Lifetime and Ownership

Lifetimes make sense when combined with ownership.

Suppose:

```rust
fn get_name() -> &str {
    let name = String::from("Amir");
    &name
}
```

This is invalid.

Why?

```text
name created
   ↓
&name returned
   ↓
function ends
   ↓
name dropped
   ↓
returned reference would point to dead data
```

Rust rejects it.

---

# 33. You Cannot Return a Reference to a Local Variable

This is one of the most common lifetime errors.

Invalid:

```rust
fn create() -> &String {
    let s = String::from("hello");
    &s
}
```

The correct solution is usually to return ownership:

```rust
fn create() -> String {
    String::from("hello")
}
```

Then:

```rust
let s = create();
```

The caller owns the returned `String`.

This is an important Rust design principle:

> **If the created data needs to outlive the function, return ownership instead of a reference to local data.**

---

# 34. Lifetime vs Ownership

Compare:

```rust
fn create() -> String {
    String::from("hello")
}
```

with:

```rust
fn view<'a>(s: &'a String) -> &'a str {
    &s[..]
}
```

The first transfers ownership:

```text
function
   │
   └── creates String
          │
          └── ownership returned to caller
```

The second borrows:

```text
caller owns String
       │
       └── function temporarily borrows it
```

Lifetimes describe the second relationship.

---

# 35. Lifetime vs Scope

These concepts are related but not identical.

Scope is about where a variable name is accessible:

```rust
{
    let x = 10;
    println!("{x}");
}
```

The scope of `x` is the block.

A lifetime describes how long a reference is valid.

For example:

```rust
let x = 10;

{
    let r = &x;
    println!("{r}");
}
```

Here:

```text
x scope:
|------------------------|

r scope:
      |----------|
```

The reference `r` has a shorter lifetime than `x`.

---

# 36. Non-Lexical Lifetimes

Modern Rust uses **non-lexical lifetimes (NLL)**.

This means a borrow can end when the compiler determines that the reference is no longer used, rather than necessarily waiting until the end of the lexical scope.

For example:

```rust
let mut x = String::from("hello");

let r = &x;

println!("{r}");

x.push_str(" world");
```

The immutable borrow can end after the last use of `r`.

Conceptually:

```text
x:
|-----------------------------|

r:
     |--------|
              ↑
           last use
```

After that point, `x` can be mutably borrowed.

This makes Rust's borrowing system much more flexible.

---

# 37. Lifetime of a Mutable Reference

Consider:

```rust
let mut x = String::from("hello");

let r = &mut x;

r.push_str(" world");

println!("{r}");
```

While `r` is actively borrowed:

```text
x
│
└── exclusively borrowed by r
```

You cannot simultaneously use `x` in a conflicting way.

This is related to the rule:

> At any point, you can have either one mutable reference or any number of immutable references.

Lifetimes describe how long those borrows are active.

---

# 38. Lifetime and Borrowing

Think of a borrow as having a lifetime.

```rust
let x = String::from("hello");

let r = &x;

println!("{r}");
```

Conceptually:

```text
x owns data
│
├─────────────── lifetime of x
│
└── r borrows x
    └──────── lifetime of borrow
```

The borrow checker ensures the borrowing rules remain valid throughout the lifetime of the borrow.

---

# 39. `'static`

One special lifetime is:

```rust
'static
```

It means the reference can remain valid for the entire duration of the program.

For example:

```rust
let s: &'static str = "hello";
```

String literals have `'static` lifetime because they are embedded in the program's binary.

---

# 40. String Literals and `'static`

Consider:

```rust
let s = "hello";
```

The type is:

```rust
&'static str
```

conceptually.

The data exists for the entire program.

Therefore:

```rust
let s: &'static str = "hello";
```

is valid.

---

# 41. `'static` Does Not Always Mean "Lives Forever"

There is another important use:

```rust
T: 'static
```

This does not necessarily mean:

> T itself lives forever.

It means:

> `T` does not contain references that are shorter-lived than `'static`.

For example:

```rust
String
```

is `'static` in this sense because it owns its data.

This can therefore satisfy:

```rust
T: 'static
```

even though a particular `String` value can be dropped normally.

---

# 42. `'static` and Owned Types

Consider:

```rust
fn spawn<T: 'static>(value: T) {
}
```

A `String` can satisfy this:

```rust
let s = String::from("hello");

spawn(s);
```

because `String` owns its data.

But this may not:

```rust
let s = String::from("hello");
let r = &s;

spawn(r);
```

because `r` refers to `s`, which may not live for `'static`.

The important distinction is:

```text
String
└── owns data
    └── no borrowed data

&String
└── borrows data
    └── lifetime matters
```

---

# 43. Why Thread APIs Often Require `'static`

You will often see:

```rust
T: Send + 'static
```

when working with threads.

For example:

```rust
thread::spawn(move || {
    println!("{value}");
});
```

The spawned thread may continue running after the current scope ends.

Therefore Rust cannot allow the thread to hold a reference to a local variable that might disappear.

For example:

```rust
let s = String::from("hello");

thread::spawn(|| {
    println!("{s}");
});
```

A normal spawned thread cannot safely assume that `s` remains alive.

Using:

```rust
move
```

moves ownership into the thread:

```rust
thread::spawn(move || {
    println!("{s}");
});
```

Now the thread owns the `String`.

This is one reason why ownership is often preferable to borrowing for spawned threads.

---

# 44. Scoped Threads and Lifetimes

Scoped threads are different.

For example:

```rust
let s = String::from("hello");

thread::scope(|scope| {
    scope.spawn(|| {
        println!("{s}");
    });
});
```

The scope guarantees that the thread finishes before the borrowed data can go away.

Conceptually:

```text
s:
|---------------------------|

scope:
     |-----------------|

thread:
       |-----------|
```

The thread cannot outlive the scope.

Therefore borrowing `s` is safe.

This is a great example of lifetimes being used to express relationships between concurrency and data.

---

# 45. Lifetime Bounds and Threads

This distinction is important:

```text
thread::spawn
    ↓
thread may outlive current scope
    ↓
borrowed references generally cannot escape
    ↓
often requires 'static
```

while:

```text
thread::scope
    ↓
threads must finish before scope ends
    ↓
borrowing local data is possible
```

Lifetimes are therefore deeply connected to Rust's concurrency safety.

---

# 46. Lifetime on Trait Objects

You may encounter:

```rust
Box<dyn Trait>
```

and:

```rust
Box<dyn Trait + 'a>
```

The latter explicitly specifies a lifetime bound.

Example:

```rust
Box<dyn Display + 'a>
```

means the trait object may contain references valid for `'a`.

You can also see:

```rust
&'a dyn Trait
```

which means:

> A reference with lifetime `'a` to a trait object.

---

# 47. `dyn Trait + 'static`

A common form is:

```rust
Box<dyn Trait + 'static>
```

This means the object does not contain non-`'static` borrowed references.

For example:

```rust
struct User {
    name: String,
}
```

can generally be stored in:

```rust
Box<dyn SomeTrait + 'static>
```

because `String` owns its data.

But a type containing a short-lived reference may not satisfy that bound.

---

# 48. Lifetime Bounds on Traits

You can define:

```rust
trait Trait<'a> {
    fn get(&self) -> &'a str;
}
```

Here the trait itself has a lifetime parameter.

A type can implement it:

```rust
struct Data<'a> {
    value: &'a str,
}

impl<'a> Trait<'a> for Data<'a> {
    fn get(&self) -> &'a str {
        self.value
    }
}
```

This means the trait's behavior is explicitly associated with lifetime `'a`.

---

# 49. Generic Types and Lifetimes Together

You can have both type parameters and lifetime parameters:

```rust
struct Container<'a, T> {
    value: &'a T,
}
```

Here:

```text
'a → lifetime parameter
T  → type parameter
```

Example:

```rust
let value = 10;

let container = Container {
    value: &value,
};
```

---

# 50. Lifetime Bounds with Generic Types

Example:

```rust
fn process<'a, T>(value: &'a T)
where
    T: 'a,
{
}
```

This says:

```text
T must be valid for at least 'a
```

This becomes especially relevant when `T` itself might contain references.

---

# 51. Higher-Ranked Trait Bounds

An advanced lifetime syntax is:

```rust
for<'a>
```

For example:

```rust
F: for<'a> Fn(&'a str) -> &'a str
```

This means:

> `F` must work for every possible lifetime `'a`.

This is called a **higher-ranked trait bound (HRTB)**.

It is useful when a function or closure must accept references with arbitrary lifetimes.

---

# 52. Understanding `for<'a>`

Consider:

```rust
for<'a> Fn(&'a str) -> &'a str
```

Read it as:

> For every lifetime `'a`, this function can accept a `&'a str` and return a `&'a str`.

Conceptually:

```text
'a = lifetime #1 → works
'a = lifetime #2 → works
'a = lifetime #3 → works
...
```

It is not one particular lifetime.

It means **all possible lifetimes**.

---

# 53. Anonymous Lifetimes

Rust also has anonymous lifetime syntax in certain contexts:

```rust
'_
```

For example:

```rust
fn foo(x: &'_ str) {
}
```

`'_` means:

> Let the compiler infer this lifetime.

You may also encounter:

```rust
impl Iterator<Item = &'_ str>
```

where the lifetime is intentionally left for inference.

---

# 54. `'_` vs `'a`

Compare:

```rust
&'a str
```

and:

```rust
&'_ str
```

`'a` is a named lifetime parameter.

For example:

```rust
fn foo<'a>(x: &'a str) -> &'a str {
    x
}
```

`'_` is an anonymous lifetime:

```rust
fn foo(x: &'_ str) {
}
```

It tells Rust:

> Infer this lifetime rather than giving it a name.

---

# 55. Lifetime Elision vs `'_`

These are related but not identical concepts.

This:

```rust
fn foo(x: &str) -> &str
```

uses lifetime elision rules.

This:

```rust
fn foo(x: &'_ str) -> &'_ str
```

explicitly uses anonymous lifetime syntax.

Usually, the first form is preferred when lifetime elision makes the relationship obvious.

---

# 56. Lifetime on References in Type Aliases

You can define a lifetime-aware type alias:

```rust
type StrRef<'a> = &'a str;
```

Then:

```rust
fn foo<'a>(value: StrRef<'a>) -> StrRef<'a> {
    value
}
```

This is useful when a reference type appears repeatedly.

---

# 57. Lifetime on Type Aliases with Generic Types

For example:

```rust
type Ref<'a, T> = &'a T;
```

Then:

```rust
fn get<'a, T>(value: Ref<'a, T>) -> Ref<'a, T> {
    value
}
```

---

# 58. Lifetime Parameters in Enums

Enums can contain references too:

```rust
enum Message<'a> {
    Text(&'a str),
    Number(i32),
}
```

Example:

```rust
let text = String::from("hello");

let message = Message::Text(&text);
```

The enum cannot outlive the referenced data.

---

# 59. Lifetime Parameters in Traits

Traits can have lifetime parameters:

```rust
trait Parser<'a> {
    fn parse(&self, input: &'a str);
}
```

Implementation:

```rust
struct MyParser;

impl<'a> Parser<'a> for MyParser {
    fn parse(&self, input: &'a str) {
        println!("{input}");
    }
}
```

---

# 60. Lifetime Parameters in Functions Returning References

Common pattern:

```rust
fn get<'a>(value: &'a String) -> &'a str {
    value.as_str()
}
```

This means:

```text
input:
&'a String

output:
&'a str
```

The returned reference cannot outlive the input reference.

---

# 61. Lifetime on `self`

You can explicitly name the lifetime of `self`:

```rust
impl User {
    fn name<'a>(&'a self) -> &'a str {
        &self.name
    }
}
```

Usually this is unnecessary because Rust's lifetime elision handles it:

```rust
impl User {
    fn name(&self) -> &str {
        &self.name
    }
}
```

The second form is idiomatic.

---

# 62. Lifetime on `&mut self`

Similarly:

```rust
impl User {
    fn name_mut<'a>(&'a mut self) -> &'a mut String {
        &mut self.name
    }
}
```

Usually written:

```rust
impl User {
    fn name_mut(&mut self) -> &mut String {
        &mut self.name
    }
}
```

Again, lifetime elision handles the relationship.

---

# 63. Lifetime Subtyping

Rust allows lifetime relationships where one lifetime outlives another.

For example:

```rust
'a: 'b
```

means:

```text
'a lasts at least as long as 'b
```

This is sometimes described as lifetime subtyping.

Conceptually:

```text
'a
|-------------------------|

'b
     |---------------|
```

The longer lifetime can be used where a shorter lifetime is required.

---

# 64. Reborrowing

Lifetimes also help explain **reborrowing**.

Consider:

```rust
let mut value = String::from("hello");

let r = &mut value;

r.push_str(" world");

r.push_str("!");
```

Rust can create shorter mutable borrows from `r`.

Conceptually:

```text
original mutable borrow
|---------------------------|

first reborrow:
   |------|

second reborrow:
           |------|
```

This is one reason Rust can safely support complex mutable-reference code.

---

# 65. Lifetime and `&mut`

A mutable reference is exclusive for its active lifetime.

For example:

```rust
let mut x = 10;

let r = &mut x;
```

While that borrow is active, another conflicting borrow cannot be used.

This:

```rust
let r1 = &mut x;
let r2 = &mut x;
```

is invalid if both are simultaneously active.

The borrow checker tracks the relevant lifetimes.

---

# 66. Lifetimes and Interior Mutability

Types such as:

```rust
RefCell<T>
```

allow borrowing rules to be checked at runtime rather than compile time.

For example:

```rust
let value = RefCell::new(10);

let borrow = value.borrow();
```

The reference returned by the borrow still has a lifetime.

The difference is that `RefCell` manages the borrowing rules dynamically.

So:

```text
normal reference
    ↓
borrow rules checked at compile time

RefCell
    ↓
borrow rules checked at runtime
```

Lifetimes remain part of the type system even though some borrow checking is moved to runtime.

---

# 67. Lifetimes and `Rc`

`Rc<T>` provides shared ownership.

For example:

```rust
use std::rc::Rc;

let value = Rc::new(String::from("hello"));

let a = Rc::clone(&value);
let b = Rc::clone(&value);
```

There is no lifetime parameter required for the `Rc` itself because `Rc` owns the data.

Conceptually:

```text
Rc
 ├── owner #1
 ├── owner #2
 └── data
```

The data remains alive while at least one `Rc` exists.

This is different from:

```rust
&'a T
```

which is a borrowed reference and therefore needs lifetime reasoning.

---

# 68. Lifetimes and `Box`

Similarly:

```rust
Box<T>
```

owns its value.

For example:

```rust
let value = Box::new(String::from("hello"));
```

`Box<T>` does not need a lifetime parameter for the owned value.

Compare:

```text
Box<T>
└── owns T

&'a T
└── borrows T
    └── lifetime required
```

This distinction is fundamental.

---

# 69. Owned vs Borrowed Data

A useful rule:

```text
Owned:
String
Vec<T>
Box<T>
Rc<T>
Arc<T>

Borrowed:
&T
&mut T
```

Borrowed data requires lifetime reasoning.

Owned data generally does not require explicit lifetime parameters because ownership determines how long the data remains valid.

---

# 70. Lifetimes and `Arc`

`Arc<T>` is shared ownership across threads.

```rust
use std::sync::Arc;

let value = Arc::new(String::from("hello"));

let a = Arc::clone(&value);
let b = Arc::clone(&value);
```

Again, the `Arc` owns the data.

This is different from sharing:

```rust
&'a T
```

The `Arc` keeps the data alive through ownership.

---

# 71. Lifetimes and `Send` / `Sync`

Lifetimes also appear indirectly when working with concurrency.

For example, a spawned thread often requires:

```text
Send + 'static
```

`Send` answers:

> Can this value be transferred to another thread?

`'static` answers:

> Does this value contain references that could become invalid before the thread is finished?

These are different properties.

For example:

```rust
String
```

can be `Send` and `'static`.

But:

```rust
&String
```

may have a short lifetime and therefore cannot necessarily be sent to an independently running thread.

---

# 72. A Lifetime Does Not Mean Ownership

This is especially important.

Consider:

```rust
let s = String::from("hello");

let r = &s;
```

`r` has a lifetime.

But `r` does not own `s`.

The ownership relationship is:

```text
s
└── owns String

r
└── borrows s
```

The lifetime describes:

```text
how long r can safely be used
```

It does not transfer ownership.

---

# 73. Can We Talk About Ownership of `&T`?

Yes, but carefully.

A reference itself is a value.

Therefore:

```rust
let r1 = &s;
let r2 = r1;
```

moves or copies the **reference value**, depending on its type semantics.

But this does not mean:

```text
r1 owns s
```

It means:

```text
r1 contains a reference to s
```

References are themselves values, but they do not own the data they point to.

This distinction is important when reasoning about `Send`, `Sync`, ownership, and lifetimes.

---

# 74. Lifetime of a Reference vs Lifetime of Referenced Data

Suppose:

```rust
let s = String::from("hello");

{
    let r = &s;
    println!("{r}");
}
```

There are two different lifetimes:

```text
s:
|-----------------------------|

r:
     |-----------|
```

The data's lifetime is longer.

The reference's lifetime is shorter.

This is normal.

A lifetime parameter does not mean:

> The data itself lives exactly for `'a`.

It means:

> The reference is valid for `'a`.

---

# 75. Lifetime of a Reference Can Be Shorter Than the Named Lifetime

When you write:

```rust
fn foo<'a>(x: &'a str) {
}
```

`'a` represents a lifetime chosen by the compiler for that call.

The actual borrow may be shorter than the entire lifetime of the underlying object.

This is why lifetime annotations should be understood as **constraints and relationships**, not timers.

---

# 76. Common Lifetime Syntax Cheat Sheet

## Reference with lifetime

```rust
&'a T
```

## Mutable reference with lifetime

```rust
&'a mut T
```

## Function lifetime parameter

```rust
fn foo<'a>() {
}
```

## Multiple lifetime parameters

```rust
fn foo<'a, 'b>() {
}
```

## Input and output relationship

```rust
fn foo<'a>(x: &'a T) -> &'a T {
    x
}
```

## Multiple inputs

```rust
fn foo<'a, 'b>(
    x: &'a T,
    y: &'b T,
) {
}
```

## Struct with lifetime

```rust
struct Foo<'a> {
    value: &'a T,
}
```

## Multiple lifetimes in a struct

```rust
struct Foo<'a, 'b> {
    first: &'a T,
    second: &'b T,
}
```

## `impl` with lifetime

```rust
impl<'a> Foo<'a> {
}
```

## Lifetime bound

```rust
T: 'a
```

## Lifetime outlives relationship

```rust
'a: 'b
```

## `where` lifetime bound

```rust
where
    T: 'a,
```

## Trait with lifetime

```rust
trait Foo<'a> {
}
```

## Trait implementation with lifetime

```rust
impl<'a> Foo<'a> for Bar<'a> {
}
```

## Trait object with lifetime

```rust
dyn Trait + 'a
```

## Reference to trait object

```rust
&'a dyn Trait
```

## Boxed trait object

```rust
Box<dyn Trait + 'a>
```

## Static lifetime

```rust
'static
```

## Anonymous lifetime

```rust
'_
```

## Higher-ranked trait bound

```rust
for<'a>
```

Example:

```rust
F: for<'a> Fn(&'a str) -> &'a str
```

---

# 77. A Practical Decision Process

When you see a lifetime error, ask these questions in order.

### 1. Who owns the data?

```text
String?
Vec?
Box?
Rc?
Arc?
```

If something owns the data, determine whether ownership can be returned or moved instead of borrowing.

### 2. Who is borrowing?

Look for:

```rust
&T
&mut T
```

### 3. How long does the owner live?

Find the scope where the owned value is valid.

### 4. How long does the reference need to live?

Find where the reference is used.

### 5. Does the reference outlive its owner?

If yes, the program is invalid.

### 6. Does the function return a reference?

If yes, ask:

> Which input does the returned reference come from?

### 7. Can lifetime elision express the relationship?

If yes, you probably don't need explicit lifetime syntax.

---

# 78. The Most Common Lifetime Pattern

You will encounter this pattern constantly:

```rust
fn get<'a>(value: &'a T) -> &'a U {
    // ...
}
```

The key idea is:

```text
input borrow
    │
    │ 'a
    ▼
function
    │
    │ 'a
    ▼
output borrow
```

The output cannot outlive the input reference.

---

# 79. Another Common Pattern: Struct Holding a Reference

```rust
struct Parser<'a> {
    input: &'a str,
}
```

Think:

```text
Parser<'a>
    │
    └── borrows input
            │
            └── input must remain alive
```

This is common in parsers and zero-copy data structures.

---

# 80. Zero-Copy Programming

Lifetimes are especially important in zero-copy programming.

Instead of creating a new `String`:

```rust
fn get_name(input: &str) -> String {
    input[..5].to_string()
}
```

you can return a slice:

```rust
fn get_name<'a>(input: &'a str) -> &'a str {
    &input[..5]
}
```

Now the result borrows the original data.

Conceptually:

```text
input:
"Amir Nikpour"
 │
 └────────────── owned data

result:
&input[..5]
 │
 └── points into existing data
```

No new string allocation is necessary.

This is one of the major practical benefits of lifetimes.

---

# 81. Lifetimes Enable Safe Zero-Copy APIs

Consider:

```rust
struct User<'a> {
    name: &'a str,
}
```

The `User` does not own the name.

It borrows it:

```text
input buffer
┌──────────────────────────┐
│ Amir Nikpour             │
└──────────────────────────┘
          ▲
          │
          │ &'a str
          │
       User<'a>
```

Rust ensures the `User` cannot outlive the input buffer.

This allows efficient zero-copy designs while maintaining memory safety.

---

# 82. Lifetime Errors Are Often Design Errors

When Rust gives you a lifetime error, don't immediately try to "fight the borrow checker."

First ask:

> What ownership relationship am I trying to express?

For example, if you write:

```rust
fn create() -> &str {
    let s = String::from("hello");
    &s
}
```

adding random lifetime annotations cannot fix the problem.

The actual problem is:

```text
function owns data
        ↓
function ends
        ↓
data destroyed
        ↓
reference would become invalid
```

The correct design is probably:

```rust
fn create() -> String {
    String::from("hello")
}
```

---

# 83. When Should You Add an Explicit Lifetime?

Do not add lifetime annotations everywhere.

Usually add them when:

* a function returns a reference
* multiple input references are involved
* a struct stores references
* a trait has lifetime-dependent behavior
* you need to express an outlives relationship
* the compiler cannot infer the relationship
* you are implementing advanced generic abstractions

For simple borrowing:

```rust
fn print(value: &str) {
    println!("{value}");
}
```

is better than:

```rust
fn print<'a>(value: &'a str) {
    println!("{value}");
}
```

The explicit lifetime adds no useful information here.

---

# 84. When Should You Avoid Lifetimes?

Sometimes the easiest solution is to use ownership.

Instead of:

```rust
struct User<'a> {
    name: &'a str,
}
```

you can use:

```rust
struct User {
    name: String,
}
```

Now:

```text
User
 └── owns name
```

rather than:

```text
User<'a>
 └── borrows name
```

Owned data often makes APIs simpler.

Borrowed data can be useful when:

* performance matters
* allocations should be avoided
* data is naturally borrowed
* zero-copy parsing is desired
* the lifetime relationship is clear

---

# 85. Lifetimes Are Part of Rust's Memory Safety Model

Rust's memory safety comes from several concepts working together:

```text
Ownership
    +
Borrowing
    +
Lifetimes
    +
Type system
    +
Borrow checker
```

Ownership answers:

> Who is responsible for the data?

Borrowing answers:

> Who can temporarily access the data?

Lifetimes answer:

> How long is that access valid?

The borrow checker uses these rules to prevent problems such as:

* use-after-free
* dangling references
* invalid mutable aliases
* references to destroyed stack variables

---

# 86. Lifetimes and Dangling References

Languages with manual memory management can accidentally create:

```text
pointer → freed memory
```

Rust prevents this.

For example:

```rust
fn bad() -> &String {
    let value = String::from("hello");
    &value
}
```

The compiler detects that:

```text
value dies
    ↓
returned reference would remain
    ↓
dangling reference
```

Therefore the program is rejected before it runs.

---

# 87. A Good Mental Model

Don't think:

> `'a` means this object lives for some fixed amount of time.

Think:

> `'a` is a name for a region of validity of a reference, and annotations tell Rust how different references are related.

For example:

```rust
fn longest<'a>(
    x: &'a str,
    y: &'a str,
) -> &'a str
```

Think:

```text
          'a
           │
     ┌─────┴─────┐
     │           │
   x: &'a      y: &'a
     │           │
     └─────┬─────┘
           │
        returned
         &'a str
```

The annotation tells the compiler:

> The returned reference is tied to the lifetime of the input references.

---

# 88. Final Mental Model

The easiest way to remember lifetimes is:

```text
Ownership
    │
    └── Who owns the data?

Borrowing
    │
    └── Who temporarily accesses it?

Lifetime
    │
    └── How long is that reference valid?

Borrow checker
    │
    └── Are all these relationships safe?
```

And when reading Rust code:

```rust
'a
```

think:

> "A named lifetime relationship."

```rust
&'a T
```

think:

> "A reference to `T` valid for `'a`."

```rust
fn foo<'a>(x: &'a T) -> &'a T
```

think:

> "The returned reference is tied to the input reference."

```rust
'a: 'b
```

think:

> "`'a` outlives `'b`."

```rust
T: 'a
```

think:

> "`T` is valid for at least `'a`."

```rust
'static
```

think:

> "Valid for the entire program, or, for a type bound, contains no shorter-lived borrowed references."

```rust
'_
```

think:

> "Let the compiler infer this lifetime."

```rust
for<'a>
```

think:

> "For every possible lifetime `'a`."

The central idea is:

> **Lifetimes do not control how long data lives. Ownership and scope determine that. Lifetimes describe and constrain how long references can safely be used in relation to the data they borrow.**

Once you see lifetimes as **relationships between references rather than timers**, most lifetime syntax becomes much easier to understand.
