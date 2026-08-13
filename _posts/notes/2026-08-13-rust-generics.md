---
title: "Rust Notes: Generics"
categories:
  - Learning Notes
  - Rust
tags: [rust, generics, traits, lifetimes, type-system, monomorphization]
description: "A review of Generics in Rust."

toc: true
---

# Generics in Rust

Generics are Rust's mechanism for writing code that works with **multiple types without duplicating the implementation**.

Instead of writing:

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    // ...
}
```

and another function for `f64`:

```rust
fn largest_f64(list: &[f64]) -> &f64 {
    // ...
}
```

we can write one generic function:

```rust
fn largest<T>(list: &[T]) -> &T {
    // ...
}
```

Here `T` is a **generic type parameter**.

Generics are one of the foundations of Rust's type system and appear together with:

* ownership
* borrowing
* lifetimes
* traits
* trait bounds
* associated types
* `impl Trait`
* trait objects
* const generics
* static dispatch
* monomorphization

The most important idea is:

> **Generics let us write code in terms of a type that will be specified later.**

---

# 1. Why Do We Need Generics?

Imagine we want a function that returns the first element of a slice.

Without generics:

```rust
fn first_i32(values: &[i32]) -> &i32 {
    &values[0]
}

fn first_string(values: &[String]) -> &String {
    &values[0]
}

fn first_f64(values: &[f64]) -> &f64 {
    &values[0]
}
```

The implementations are identical.

Only the type changes.

Generics allow us to write:

```rust
fn first<T>(values: &[T]) -> &T {
    &values[0]
}
```

Now the same function works with:

```rust
let numbers = vec![1, 2, 3];
let strings = vec![
    String::from("hello"),
    String::from("world"),
];

let n = first(&numbers);
let s = first(&strings);
```

The compiler determines `T` from the arguments.

---

# 2. What Is `T`?

In:

```rust
fn first<T>(values: &[T]) -> &T {
    &values[0]
}
```

`T` is a **type parameter**.

Think of it as a placeholder:

```text
T
│
└── some type
```

For example, when calling:

```rust
first(&numbers)
```

the compiler can infer:

```text
T = i32
```

When calling:

```rust
first(&strings)
```

it can infer:

```text
T = String
```

Conceptually:

```text
first<i32>(&[i32])
first<String>(&[String])
```

---

# 3. Generic Syntax

The basic syntax is:

```rust
<T>
```

For example:

```rust
fn foo<T>(value: T) {
}
```

Multiple type parameters:

```rust
fn foo<T, U>(x: T, y: U) {
}
```

A generic struct:

```rust
struct Pair<T, U> {
    first: T,
    second: U,
}
```

A generic enum:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

A generic trait:

```rust
trait Container<T> {
}
```

---

# 4. Generic Functions

The most common syntax is:

```rust
fn identity<T>(value: T) -> T {
    value
}
```

Usage:

```rust
let x = identity(10);
let y = identity("hello");
```

The compiler infers:

```text
T = i32
```

for the first call and:

```text
T = &str
```

for the second.

---

# 5. Explicit Generic Arguments

Sometimes you can specify the generic type explicitly.

For example:

```rust
fn identity<T>(value: T) -> T {
    value
}
```

You can write:

```rust
let x = identity::<i32>(10);
```

The syntax:

```rust
::<T>
```

is called **turbofish syntax**.

For multiple types:

```rust
fn pair<T, U>(x: T, y: U) -> (T, U) {
    (x, y)
}

let value = pair::<i32, String>(10, String::from("hello"));
```

---

# 6. Why Is It Called Turbofish?

This:

```rust
::<T>
```

is commonly called the **turbofish**.

For example:

```rust
foo::<i32>();
```

The `::` followed by `<...>` makes the syntax visually resemble a fish.

You will frequently see it with generic methods and functions:

```rust
"42".parse::<i32>();
```

Here:

```text
parse::<i32>()
       ↑
       explicitly choose T = i32
```

---

# 7. Type Inference

Rust usually does not require explicit generic arguments.

For example:

```rust
fn identity<T>(value: T) -> T {
    value
}

let x = identity(10);
```

Rust sees:

```text
10 → i32
```

and therefore infers:

```text
T = i32
```

You can explicitly specify it:

```rust
let x = identity::<i32>(10);
```

Both are valid.

Idiomatic Rust generally relies on inference when it is obvious.

---

# 8. Generic Structs

A struct can have generic type parameters:

```rust
struct Point<T> {
    x: T,
    y: T,
}
```

Now:

```rust
let integer_point = Point {
    x: 10,
    y: 20,
};

let float_point = Point {
    x: 1.5,
    y: 2.5,
};
```

Conceptually:

```text
Point<i32>
Point<f64>
```

---

# 9. Different Types in the Same Struct

You can use multiple generic parameters:

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

Now:

```rust
let point = Point {
    x: 10,
    y: 3.14,
};
```

Conceptually:

```text
Point<i32, f64>
```

This is useful when the fields do not need to have the same type.

---

# 10. Generic Enums

Rust's standard library uses generics extensively.

For example:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

This allows:

```rust
Option<i32>
Option<String>
Option<&str>
```

For example:

```rust
let x: Option<i32> = Some(10);

let y: Option<String> = Some(String::from("hello"));
```

The enum itself does not need separate implementations for every type.

---

# 11. `Result<T, E>`

Another important generic type is:

```rust
Result<T, E>
```

It represents:

```text
T → success value
E → error value
```

For example:

```rust
Result<String, std::io::Error>
```

or:

```rust
Result<i32, ParseIntError>
```

This is a powerful example of why generics are useful.

The same `Result` abstraction works with many combinations of success and error types.

---

# 12. Generic Methods

A struct can be generic and its methods can also be generic.

For example:

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn value(&self) -> &T {
        &self.value
    }
}
```

Notice:

```rust
impl<T> Container<T>
```

The first `T` declares the generic parameter.

The second `T` uses it.

---

# 13. Why Do We Need `impl<T>`?

Suppose:

```rust
struct Container<T> {
    value: T,
}
```

To implement methods for all `T`:

```rust
impl<T> Container<T> {
    fn value(&self) -> &T {
        &self.value
    }
}
```

Read it as:

> For every type `T`, implement these methods for `Container<T>`.

---

# 14. Implementing Only for One Concrete Type

You can also specialize the implementation for one concrete type:

```rust
struct Container<T> {
    value: T,
}

impl Container<i32> {
    fn double(&self) -> i32 {
        self.value * 2
    }
}
```

Now `double()` exists only for:

```text
Container<i32>
```

It does not exist for:

```text
Container<String>
```

---

# 15. Generic Implementation with a Different Type Parameter

You can introduce additional generic parameters inside `impl`.

For example:

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn pair<U>(self, other: U) -> (T, U) {
        (self.value, other)
    }
}
```

Usage:

```rust
let c = Container { value: 10 };

let result = c.pair("hello");
```

The types are:

```text
T = i32
U = &str
```

---

# 16. Generic Functions and Generic Methods Are Different

Consider:

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn get(&self) -> &T {
        &self.value
    }

    fn convert<U>(self, value: U) -> (T, U) {
        (self.value, value)
    }
}
```

Here:

```text
T
```

belongs to the struct.

While:

```text
U
```

belongs only to the method.

---

# 17. Generic Traits

Traits can also be generic:

```rust
trait Converter<T> {
    fn convert(&self) -> T;
}
```

A type can implement the trait for a particular `T`:

```rust
struct Number;

impl Converter<i32> for Number {
    fn convert(&self) -> i32 {
        42
    }
}
```

---

# 18. Trait Generic Parameters vs Associated Types

This is an important Rust distinction.

A generic trait:

```rust
trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

would allow one type to implement the trait multiple times for different `T`.

Rust's actual `Iterator` uses an associated type:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

This means a particular implementation chooses one `Item` type.

For example:

```rust
impl Iterator for MyIterator {
    type Item = i32;

    fn next(&mut self) -> Option<Self::Item> {
        // ...
    }
}
```

---

# 19. Why Associated Types Matter

With:

```rust
trait Iterator {
    type Item;
}
```

we can write:

```rust
fn sum<I>(iter: I) -> i32
where
    I: Iterator<Item = i32>,
{
    // ...
}
```

The `Item = i32` syntax constrains the associated type.

This is different from:

```rust
I: Iterator<i32>
```

because `Iterator` does not have a generic parameter.

---

# 20. Trait Bounds

A generic type can have restrictions.

For example:

```rust
fn print<T: std::fmt::Display>(value: T) {
    println!("{value}");
}
```

The:

```rust
T: Display
```

is a **trait bound**.

It says:

> `T` must implement `Display`.

Without this bound, Rust cannot assume that `T` supports `{value}` formatting.

---

# 21. Why Do Generic Types Need Bounds?

Consider:

```rust
fn add<T>(a: T, b: T) -> T {
    a + b
}
```

This does not compile.

Why?

Because Rust cannot assume that every type supports:

```text
+
```

For example:

```text
String
File
TcpStream
MyCustomType
```

do not automatically have addition semantics.

We need to specify the required capability.

---

# 22. Using a Trait Bound

For a type that supports addition:

```rust
use std::ops::Add;

fn add<T>(a: T, b: T) -> T
where
    T: Add<Output = T>,
{
    a + b
}
```

Now the compiler knows that `T` implements `Add`.

This demonstrates an important idea:

> **Generics describe what can vary; trait bounds describe what the generic type must be able to do.**

---

# 23. Multiple Trait Bounds

You can require multiple traits:

```rust
fn process<T>(value: T)
where
    T: Clone + std::fmt::Debug + Send,
{
}
```

This means `T` must implement:

```text
Clone
Debug
Send
```

You can also write:

```rust
fn process<T: Clone + std::fmt::Debug + Send>(value: T) {
}
```

Both forms are valid.

---

# 24. `where` Clauses

When bounds become complicated, use `where`:

```rust
fn process<T, U>(x: T, y: U)
where
    T: Clone + Send,
    U: std::fmt::Debug,
{
}
```

This is often more readable than:

```rust
fn process<T: Clone + Send, U: std::fmt::Debug>(
    x: T,
    y: U,
) {
}
```

A good rule:

> Use inline bounds for simple constraints and `where` for complex ones.

---

# 25. Bounds on Structs

You can put bounds directly on a struct declaration:

```rust
struct Container<T>
where
    T: Clone,
{
    value: T,
}
```

However, it is often better to put bounds on the specific `impl` or function that needs them.

For example:

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T>
where
    T: Clone,
{
    fn duplicate(&self) -> T {
        self.value.clone()
    }
}
```

This keeps the struct usable with any `T`.

---

# 26. Bounds on `impl`

Very common:

```rust
impl<T> Container<T>
where
    T: Clone,
{
    fn duplicate(&self) -> T {
        self.value.clone()
    }
}
```

The implementation exists only for `T` types that implement `Clone`.

---

# 27. Conditional Methods

You can use trait bounds to make methods available only when required.

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn value(&self) -> &T {
        &self.value
    }
}

impl<T> Container<T>
where
    T: Clone,
{
    fn clone_value(&self) -> T {
        self.value.clone()
    }
}
```

Therefore:

```text
Container<T>
    │
    ├── value()
    │
    └── clone_value()
          ↑
          only if T: Clone
```

---

# 28. Generic Trait Implementations

You can implement a trait for every type satisfying a bound:

```rust
trait Describe {
    fn describe(&self);
}

impl<T> Describe for T
where
    T: std::fmt::Debug,
{
    fn describe(&self) {
        println!("{self:?}");
    }
}
```

This is a **blanket implementation**.

It says:

> Every `T` that implements `Debug` also gets this `Describe` implementation.

---

# 29. Blanket Implementations

The general syntax is:

```rust
impl<T> Trait for T
where
    T: SomeBound,
{
}
```

For example:

```rust
impl<T> MyTrait for T
where
    T: Clone,
{
}
```

This can be extremely powerful.

It is also subject to Rust's **orphan rules**, which prevent arbitrary implementations of foreign traits for foreign types.

---

# 30. The Orphan Rule

Suppose both are defined in another crate:

```text
Trait
Type
```

You cannot arbitrarily write:

```rust
impl ForeignTrait for ForeignType {
}
```

This restriction prevents conflicting implementations across crates.

Generally, you need to own at least one of:

```text
trait
type
```

This is part of Rust's coherence rules.

---

# 31. Generic Type Parameters Can Be Lifetimes Too

Generics and lifetimes can appear together.

For example:

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

You can also have multiple lifetimes:

```rust
struct Pair<'a, 'b, T> {
    first: &'a T,
    second: &'b T,
}
```

---

# 32. Lifetime Bounds on Generic Types

You can write:

```rust
fn process<'a, T>(value: &'a T)
where
    T: 'a,
{
}
```

Here:

```text
T: 'a
```

means `T` is valid for at least lifetime `'a`.

This matters when `T` itself could contain references.

---

# 33. Generic Functions with Lifetimes

Example:

```rust
fn first<'a, T>(items: &'a [T]) -> &'a T {
    &items[0]
}
```

Here:

```text
'a → lifetime
T  → type
```

The returned reference is tied to the input slice.

---

# 34. Generic Struct + Lifetime + Trait Bound

You can combine everything:

```rust
struct Container<'a, T>
where
    T: Clone,
{
    value: &'a T,
}
```

You can also put the bound on the implementation instead:

```rust
struct Container<'a, T> {
    value: &'a T,
}

impl<'a, T> Container<'a, T>
where
    T: Clone,
{
    fn clone_value(&self) -> T {
        self.value.clone()
    }
}
```

This demonstrates how Rust's generic system combines:

```text
lifetimes
+
types
+
traits
+
bounds
```

---

# 35. `impl Trait`

Rust provides another generic-looking syntax:

```rust
fn create() -> impl Iterator<Item = i32> {
    0..10
}
```

`impl Trait` means:

> The function returns some concrete type that implements this trait.

The caller does not need to know the concrete type.

---

# 36. `impl Trait` in Function Arguments

You can write:

```rust
fn print(value: impl std::fmt::Display) {
    println!("{value}");
}
```

This is roughly equivalent to:

```rust
fn print<T: std::fmt::Display>(value: T) {
    println!("{value}");
}
```

Both use static dispatch.

However, the syntax and type-system behavior are not identical in every context.

---

# 37. `impl Trait` in Return Position

Example:

```rust
fn numbers() -> impl Iterator<Item = i32> {
    0..10
}
```

This hides the concrete iterator type.

The caller only knows:

```text
Iterator<Item = i32>
```

The compiler still knows the exact concrete type.

This is important:

> `impl Trait` is not the same as `dyn Trait`.

---

# 38. `impl Trait` vs `dyn Trait`

Compare:

```rust
fn foo() -> impl Trait
```

with:

```rust
fn foo() -> Box<dyn Trait>
```

`impl Trait` generally uses static dispatch.

```text
impl Trait
    ↓
compiler knows concrete type
    ↓
static dispatch
```

`dyn Trait` uses dynamic dispatch:

```text
dyn Trait
    ↓
trait object
    ↓
vtable
    ↓
dynamic dispatch
```

---

# 39. Generic Function = Static Dispatch

Consider:

```rust
fn print<T: std::fmt::Display>(value: T) {
    println!("{value}");
}
```

When called with:

```rust
print(10);
print(3.14);
```

the compiler can generate specialized versions conceptually like:

```text
print::<i32>
print::<f64>
```

This is called **monomorphization**.

---

# 40. Monomorphization

Rust generics normally use monomorphization.

Suppose:

```rust
fn print<T: Display>(value: T) {
    println!("{value}");
}
```

and:

```rust
print(10);
print(20.0);
```

The compiler can generate specialized machine-code versions:

```text
print<i32>
print<f64>
```

Conceptually:

```text
generic source
     │
     ▼
monomorphization
     │
     ├── T = i32
     └── T = f64
```

This is one reason generic Rust code can be both type-safe and fast.

---

# 41. Generic Code Is Not Necessarily Runtime Generic

When you write:

```rust
fn foo<T>(value: T) {
}
```

you might imagine one runtime function that somehow handles every possible type.

That is not generally how Rust implements it.

Rust normally specializes generic code during compilation.

Therefore the runtime usually operates on concrete machine-code versions.

---

# 42. Static Dispatch

With:

```rust
fn process<T: Trait>(value: T) {
}
```

the compiler knows the concrete type.

The call can therefore be statically dispatched.

Conceptually:

```text
process::<TypeA>()
        ↓
known at compile time

process::<TypeB>()
        ↓
known at compile time
```

Advantages include:

* optimization opportunities
* no vtable lookup
* no dynamic dispatch overhead
* easier inlining

The tradeoff is potentially larger binaries due to multiple monomorphized copies.

---

# 43. Dynamic Dispatch

With:

```rust
fn process(value: &dyn Trait) {
}
```

the concrete type is not known by the function body.

Instead Rust uses a trait object and a vtable.

Conceptually:

```text
&dyn Trait
    │
    ├── pointer to data
    │
    └── pointer to vtable
              │
              ├── method A
              ├── method B
              └── ...
```

The method is selected at runtime.

---

# 44. Generic vs Trait Object

Generic:

```rust
fn process<T: Trait>(value: T) {
}
```

Trait object:

```rust
fn process(value: &dyn Trait) {
}
```

Generic:

```text
compile time
static dispatch
monomorphization
```

Trait object:

```text
runtime
dynamic dispatch
vtable
```

---

# 45. Generic Collections

A normal `Vec<T>` contains one concrete type:

```rust
let values: Vec<i32> = vec![1, 2, 3];
```

You cannot do:

```rust
let values = vec![
    1,
    "hello",
    3.14,
];
```

because a `Vec<T>` has one `T`.

If you need different types, you need an abstraction such as:

```rust
enum Value {
    Integer(i32),
    Float(f64),
    Text(String),
}
```

or trait objects:

```rust
Vec<Box<dyn Trait>>
```

This distinction is important:

> **Generics do not mean "a collection can contain arbitrary types."**

They mean:

> **A generic abstraction can be instantiated with different concrete types.**

---

# 46. Generic Type vs Trait Object

Compare:

```rust
Vec<Box<dyn Display>>
```

with:

```rust
Vec<T>
```

`Vec<T>`:

```text
one concrete T
```

`Vec<Box<dyn Display>>`:

```text
different concrete types
        ↓
all implement Display
        ↓
trait objects
```

This is a major difference between static and dynamic polymorphism.

---

# 47. Generic Associated Types

Rust also supports **generic associated types (GATs)**.

For example:

```rust
trait Iterable {
    type Item<'a>
    where
        Self: 'a;

    fn item(&self) -> Self::Item<'_>;
}
```

Here:

```text
Item<'a>
```

is an associated type that itself has a lifetime parameter.

GATs are particularly useful for abstractions involving borrowing.

---

# 48. Generic Associated Types with Types

Associated types can also have type parameters in supported contexts.

Conceptually:

```rust
trait Factory {
    type Output<T>;
}
```

The associated type itself becomes parameterized.

GATs are especially common when building advanced iterators, parsers, and borrowing abstractions.

---

# 49. Const Generics

Generics are not limited to types.

Rust also supports **const generics**.

Example:

```rust
struct Array<T, const N: usize> {
    data: [T; N],
}
```

Here:

```text
T → type parameter
N → const parameter
```

Usage:

```rust
let a: Array<i32, 10>;
let b: Array<f64, 20>;
```

Conceptually:

```text
Array<i32, 10>
Array<f64, 20>
```

---

# 50. Why Const Generics Are Useful

Before const generics, expressing an array type with a generic length was much more difficult.

Now we can write:

```rust
fn sum<const N: usize>(values: [i32; N]) -> i32 {
    values.iter().sum()
}
```

The same function works with:

```rust
sum([1, 2, 3]);
sum([1, 2, 3, 4, 5]);
```

The array length becomes part of the type.

---

# 51. Const Generic Syntax

The basic syntax is:

```rust
<const N: usize>
```

For example:

```rust
fn foo<const N: usize>() {
}
```

Multiple const parameters:

```rust
fn foo<const N: usize, const M: usize>() {
}
```

Combined with types:

```rust
fn foo<T, const N: usize>(value: [T; N]) {
}
```

---

# 52. Const Generic Expressions

You may encounter expressions involving const parameters:

```rust
struct Buffer<T, const N: usize> {
    data: [T; N],
}
```

The exact set of allowed generic const expressions depends on Rust's current const-generic capabilities.

Simple uses such as array lengths are common and stable.

---

# 53. Generic Parameters Can Include Lifetimes, Types, and Consts

A generic declaration can contain all three:

```rust
struct Buffer<'a, T, const N: usize> {
    data: &'a [T; N],
}
```

Here:

```text
'a → lifetime parameter
T  → type parameter
N  → const parameter
```

This is the complete conceptual family:

```text
Generic parameters
├── lifetimes
├── types
└── const values
```

---

# 54. Parameter Ordering

A common generic declaration is:

```rust
struct Foo<'a, T, const N: usize> {
}
```

Generally, Rust syntax places:

```text
lifetimes
    ↓
type parameters
    ↓
const parameters
```

For example:

```rust
fn foo<'a, T, U, const N: usize>(
    value: &'a [T; N],
    other: U,
) {
}
```

---

# 55. Generic Type Defaults

Generic parameters can have default types in certain declarations.

For example:

```rust
struct Container<T = String> {
    value: T,
}
```

Then:

```rust
let value = Container {
    value: String::from("hello"),
};
```

can use the default type when inference or context permits.

Defaults are more commonly encountered in library APIs and standard-library types.

---

# 56. Default Generic Type Parameters

A well-known example is:

```rust
struct Point<T = i32> {
    x: T,
    y: T,
}
```

Then:

```rust
Point {
    x: 10,
    y: 20,
}
```

can use:

```text
T = i32
```

by default.

But:

```rust
Point::<f64> {
    x: 1.0,
    y: 2.0,
}
```

can explicitly choose another type.

---

# 57. Generic Type Parameters in `enum`

Example:

```rust
enum Either<A, B> {
    Left(A),
    Right(B),
}
```

Usage:

```rust
let a: Either<i32, String> = Either::Left(10);

let b: Either<i32, String> =
    Either::Right(String::from("hello"));
```

The two variants can carry different types.

---

# 58. Generic Functions with Multiple Type Parameters

Example:

```rust
fn make_pair<T, U>(first: T, second: U) -> (T, U) {
    (first, second)
}
```

Usage:

```rust
let pair = make_pair(10, "hello");
```

The compiler infers:

```text
T = i32
U = &str
```

---

# 59. Generic Bounds Between Multiple Types

You can constrain relationships between generic types.

For example:

```rust
fn compare<T, U>(a: T, b: U)
where
    T: PartialEq<U>,
{
    println!("{}", a == b);
}
```

This says:

```text
T can be compared with U
```

The generic relationship is not necessarily:

```text
T == U
```

Instead:

```text
T: PartialEq<U>
```

means `T` knows how to compare itself with `U`.

---

# 60. Associated Type Constraints

You can constrain an associated type:

```rust
fn sum<I>(iter: I) -> i32
where
    I: Iterator<Item = i32>,
{
    iter.sum()
}
```

Here:

```rust
I: Iterator
```

is the trait bound.

And:

```rust
Item = i32
```

constrains the associated type.

---

# 61. Fully Qualified Syntax

Generics become especially interesting when traits have associated functions or methods.

For example:

```rust
trait Animal {
    fn name() -> &'static str;
}

struct Dog;

impl Animal for Dog {
    fn name() -> &'static str {
        "Dog"
    }
}
```

You can call:

```rust
Dog::name();
```

or use fully qualified syntax:

```rust
<Type as Trait>::method()
```

For example:

```rust
<DOG as Animal>::name()
```

This syntax is useful when multiple traits provide methods with the same name.

---

# 62. Generic Type Syntax with Paths

Generic arguments can appear in type paths:

```rust
Vec<i32>
Option<String>
Result<T, E>
Box<dyn Trait>
Arc<T>
```

Nested generics:

```rust
Vec<Option<String>>
```

More complex:

```rust
HashMap<String, Vec<i32>>
```

---

# 63. Nested Generic Types

Generics compose naturally.

For example:

```rust
Vec<Option<Result<String, Error>>>
```

Read from the inside out:

```text
String
    ↓
Result<String, Error>
    ↓
Option<Result<String, Error>>
    ↓
Vec<Option<Result<String, Error>>>
```

This is one of the reasons generic abstractions are powerful.

---

# 64. Generic Type Parameters and Ownership

A generic parameter can represent an owned value:

```rust
fn take<T>(value: T) {
}
```

When called:

```rust
let s = String::from("hello");

take(s);
```

ownership of `s` moves into `T`.

The generic parameter does not change Rust's ownership rules.

It simply allows the function to work with different concrete types.

---

# 65. Generic References

You can borrow a generic type:

```rust
fn inspect<T>(value: &T) {
}
```

This means:

```text
&T
```

for any `T`.

With a lifetime:

```rust
fn inspect<'a, T>(value: &'a T) {
}
```

Now:

```text
'a → reference lifetime
T  → referenced type
```

---

# 66. Generic Mutable References

You can also write:

```rust
fn modify<T>(value: &mut T) {
}
```

Or explicitly:

```rust
fn modify<'a, T>(value: &'a mut T) {
}
```

Again:

```text
'a → lifetime
T  → type
```

---

# 67. Generic Ownership vs Borrowing

Compare:

```rust
fn consume<T>(value: T) {
}
```

with:

```rust
fn borrow<T>(value: &T) {
}
```

The first takes ownership:

```text
caller
  │
  └── moves value
          ↓
       function
```

The second borrows:

```text
caller
  │
  └── owns value
       │
       └── temporary reference
                ↓
             function
```

Generics do not change this behavior.

---

# 68. Generic Trait Bounds and Ownership

You can combine ownership and trait bounds:

```rust
fn process<T>(value: T)
where
    T: Send + Sync + 'static,
{
}
```

This is common in concurrent Rust.

The generic parameter `T` is constrained by several properties.

---

# 69. `Send` and `Sync` as Generic Bounds

For example:

```rust
fn spawn<T>(value: T)
where
    T: Send + 'static,
{
    // ...
}
```

This expresses:

```text
T can be transferred to another thread
+
T does not contain short-lived borrowed references
```

This is why generic APIs can encode concurrency requirements directly into the type system.

---

# 70. `Sized`

One important implicit bound is:

```rust
T: Sized
```

By default, generic type parameters are assumed to be `Sized`.

Conceptually:

```rust
fn foo<T>(value: T) {
}
```

means approximately:

```rust
fn foo<T: Sized>(value: T) {
}
```

unless you explicitly opt out.

---

# 71. `?Sized`

To allow dynamically sized types, use:

```rust
T: ?Sized
```

Example:

```rust
fn print<T: ?Sized + std::fmt::Display>(value: &T) {
    println!("{value}");
}
```

`?Sized` means:

> `T` does not have to be `Sized`.

This allows types such as:

```rust
str
dyn Trait
```

which are dynamically sized types.

---

# 72. Why `str` Needs `?Sized`

`str` is a dynamically sized type.

You cannot normally have:

```rust
let x: str;
```

But you can have:

```rust
let x: &str;
```

A generic function taking `&T` can therefore be written:

```rust
fn foo<T: ?Sized>(value: &T) {
}
```

Now `T` can be:

```text
Sized type
or
unsized type
```

---

# 73. `Sized` and `?Sized`

Remember:

```rust
T
```

implicitly means:

```rust
T: Sized
```

while:

```rust
T: ?Sized
```

means:

```text
T may or may not be Sized
```

This is particularly important for:

```text
str
dyn Trait
[T]
```

---

# 74. Trait Bounds with `?Sized`

You can combine them:

```rust
fn foo<T: ?Sized + Trait>(value: &T) {
}
```

or:

```rust
fn foo<T>(value: &T)
where
    T: ?Sized + Trait,
{
}
```

---

# 75. `impl Trait` and `?Sized`

You will often encounter:

```rust
&impl Trait
```

or:

```rust
Box<dyn Trait>
```

The important distinction remains:

```text
generic type parameter
    ↓
T

opaque return/argument type
    ↓
impl Trait

trait object
    ↓
dyn Trait
```

---

# 76. Generic Closures

Closures themselves have compiler-generated anonymous types.

For example:

```rust
let add = |a: i32, b: i32| a + b;
```

The closure has a unique compiler-generated type.

You usually interact with it through traits such as:

```text
Fn
FnMut
FnOnce
```

A generic function can accept a closure:

```rust
fn apply<F>(f: F)
where
    F: Fn(i32) -> i32,
{
    println!("{}", f(10));
}
```

---

# 77. Generic Functions Accepting Closures

Example:

```rust
fn apply<F>(function: F, value: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    function(value)
}
```

Usage:

```rust
let result = apply(|x| x * 2, 10);
```

Here:

```text
F
│
└── closure type
```

The compiler knows the concrete closure type.

Therefore this is normally statically dispatched.

---

# 78. Higher-Ranked Trait Bounds

Advanced generic code may use:

```rust
for<'a>
```

For example:

```rust
F: for<'a> Fn(&'a str) -> &'a str
```

This means:

> For every lifetime `'a`, `F` can accept a `&'a str` and return a `&'a str`.

This is useful when writing generic code over borrowing closures and functions.

---

# 79. Generic Associated Types

GAT syntax can look like:

```rust
trait Iterable {
    type Item<'a>
    where
        Self: 'a;

    fn item(&self) -> Self::Item<'_>;
}
```

This combines:

```text
associated types
+
generic parameters
+
lifetimes
```

GATs are especially useful for APIs that return values borrowing from `self`.

---

# 80. Generic Trait Objects

Trait objects can contain generic associated types and associated type constraints where supported.

For example:

```rust
Box<dyn Iterator<Item = i32>>
```

Here:

```text
Iterator
    ↓
associated type Item
    ↓
Item = i32
```

The trait object itself hides the concrete iterator type.

---

# 81. Generic Return Types

A function can return a generic type:

```rust
fn make<T>(value: T) -> T {
    value
}
```

The caller determines `T`.

But a function cannot normally say:

```rust
fn make<T>() -> T {
    // ...
}
```

without having some way to determine or construct the required `T`.

For example:

```rust
fn make<T: Default>() -> T {
    T::default()
}
```

Now the trait bound gives the function a way to create `T`.

---

# 82. Generic Associated Functions

A type can have a generic associated function:

```rust
struct Factory;

impl Factory {
    fn create<T: Default>() -> T {
        T::default()
    }
}
```

Usage:

```rust
let x: i32 = Factory::create();
```

or:

```rust
let x = Factory::create::<i32>();
```

---

# 83. Generic Methods with Turbofish

For:

```rust
impl Factory {
    fn create<T: Default>() -> T {
        T::default()
    }
}
```

you can write:

```rust
Factory::create::<i32>()
```

The `::<i32>` specifies the generic parameter.

---

# 84. Generic Constructors

Rust does not have constructors as a special language feature.

But you can create generic constructor-like associated functions:

```rust
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn new(value: T) -> Self {
        Self { value }
    }
}
```

Usage:

```rust
let x = Container::new(10);
let y = Container::new(String::from("hello"));
```

---

# 85. `Self` with Generics

Inside an implementation:

```rust
impl<T> Container<T> {
    fn new(value: T) -> Self {
        Self { value }
    }
}
```

`Self` means:

```text
Container<T>
```

So:

```rust
fn new(value: T) -> Self
```

is effectively:

```rust
fn new(value: T) -> Container<T>
```

---

# 86. Generic Type Aliases

You can define:

```rust
type Pair<T> = (T, T);
```

Then:

```rust
let point: Pair<i32> = (10, 20);
```

You can combine this with lifetimes:

```rust
type StrRef<'a> = &'a str;
```

Or const generics:

```rust
type Array<T, const N: usize> = [T; N];
```

---

# 87. Generic Newtype Pattern

You can wrap a type:

```rust
struct Wrapper<T>(T);
```

Usage:

```rust
let x = Wrapper(10);
let y = Wrapper(String::from("hello"));
```

Conceptually:

```text
Wrapper<i32>
Wrapper<String>
```

Newtypes are useful for:

* type safety
* encapsulation
* implementing traits
* distinguishing otherwise identical types

---

# 88. Generic Recursive Types

You can create recursive generic structures:

```rust
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```

Now:

```rust
List<i32>
List<String>
```

are possible.

The `Box` is necessary because recursive types need a known size.

---

# 89. Generics and `Box`

Consider:

```rust
Box<T>
```

`Box` is itself generic.

Conceptually:

```text
Box<T>
  │
  ├── Box<i32>
  ├── Box<String>
  └── Box<MyType>
```

The same abstraction works with many types.

---

# 90. Generics and `Vec`

Likewise:

```rust
Vec<T>
```

is generic.

Examples:

```rust
Vec<i32>
Vec<String>
Vec<MyStruct>
Vec<Box<dyn Trait>>
```

The vector stores one concrete `T`.

---

# 91. Generic APIs in the Standard Library

Rust's standard library is full of generics:

```text
Option<T>
Result<T, E>
Vec<T>
Box<T>
Rc<T>
Arc<T>
Cell<T>
RefCell<T>
Mutex<T>
RwLock<T>
HashMap<K, V>
HashSet<T>
```

Understanding generics is therefore essential for reading Rust code.

---

# 92. Generics and `RefCell`

For example:

```rust
RefCell<T>
```

works for any `T`.

```rust
let value = RefCell::new(10);
let text = RefCell::new(String::from("hello"));
```

Conceptually:

```text
RefCell<i32>
RefCell<String>
```

The borrowing behavior is implemented once, generically.

---

# 93. Generics and `Arc`

Similarly:

```rust
Arc<T>
```

can contain many types:

```rust
Arc<String>
Arc<Vec<u8>>
Arc<MyStruct>
```

The `Arc` implementation does not need to be rewritten for each type.

---

# 94. Generics and Smart Pointers

A useful mental model:

```text
Box<T>
Rc<T>
Arc<T>
Cell<T>
RefCell<T>
Mutex<T>
RwLock<T>
```

all follow the pattern:

```text
Wrapper<T>
```

where `T` represents the value being wrapped.

This is one of the most common uses of generics in Rust.

---

# 95. Generic Trait Bounds Define Capabilities

Consider:

```rust
fn save<T>(value: T)
where
    T: Serialize + Send + Sync,
{
}
```

The generic parameter says:

```text
T can vary
```

The bounds say:

```text
T must have these capabilities
```

This is a useful way to think about Rust generics:

```text
Generic
    ↓
what can vary

Trait bound
    ↓
what must be possible
```

---

# 96. Generics and Polymorphism

Rust supports polymorphism primarily through:

### Parametric polymorphism

Generics:

```rust
fn foo<T>(value: T) {}
```

### Ad-hoc polymorphism

Traits:

```rust
trait Foo {
}
```

### Dynamic polymorphism

Trait objects:

```rust
dyn Foo
```

These mechanisms solve different problems.

---

# 97. Generic Polymorphism

For:

```rust
fn print<T: Display>(value: T) {
    println!("{value}");
}
```

the same source code works with many types.

This is parametric polymorphism constrained by a trait.

For example:

```text
T = i32
T = f64
T = String
```

as long as each type implements `Display`.

---

# 98. Generics vs Inheritance

Rust does not use class inheritance for generic reuse.

Instead, Rust commonly combines:

```text
generics
+
traits
+
composition
```

For example:

```rust
fn process<T: Storage>(storage: T) {
}
```

The function does not care which concrete storage implementation it receives.

This provides polymorphism without requiring an inheritance hierarchy.

---

# 99. Generics and Composition

Instead of:

```text
BaseClass
   ↑
DerivedClass
   ↑
MoreDerivedClass
```

Rust commonly uses:

```text
struct Service<S> {
    storage: S,
}
```

with:

```rust
trait Storage {
}
```

Then:

```rust
Service<FileStorage>
Service<DatabaseStorage>
Service<MemoryStorage>
```

This is a powerful design pattern in Rust.

---

# 100. Generic Dependency Injection

A common pattern is:

```rust
trait Storage {
    fn get(&self, key: &str);
}

struct Service<S> {
    storage: S,
}

impl<S: Storage> Service<S> {
    fn new(storage: S) -> Self {
        Self { storage }
    }
}
```

Now:

```rust
let service = Service::new(MyStorage);
```

The service is generic over its storage implementation.

This gives static dispatch and compile-time checking.

---

# 101. Generic State Machines

Generics can encode state in the type system.

For example:

```rust
struct Connection<State> {
    state: State,
}

struct Connected;
struct Disconnected;
```

Then methods can be implemented only for certain states:

```rust
impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connected> {
        Connection {
            state: Connected,
        }
    }
}
```

This allows invalid state transitions to become compile-time errors.

---

# 102. Phantom Types

Sometimes a generic parameter is used only at the type level.

For example:

```rust
use std::marker::PhantomData;

struct Id<T> {
    value: u64,
    _marker: PhantomData<T>,
}
```

Now:

```rust
struct User;
struct Device;

type UserId = Id<User>;
type DeviceId = Id<Device>;
```

Even though both contain a `u64`, the types are different:

```text
UserId
≠
DeviceId
```

This is useful for type-level safety.

---

# 103. `PhantomData` and Generics

`PhantomData<T>` tells the compiler that a type logically involves `T` even though it does not physically store a `T`.

Example:

```rust
struct Token<T> {
    id: u64,
    _marker: PhantomData<T>,
}
```

The generic parameter participates in type checking without requiring an actual `T` field.

This can also affect:

* variance
* ownership reasoning
* auto traits
* drop checking

---

# 104. Generic Associated Constants

Traits and types can also have associated constants that work with generic types.

For example:

```rust
trait Limits {
    const MAX: usize;
}
```

A generic type can implement the trait:

```rust
impl<T> Limits for Vec<T> {
    const MAX: usize = 1024;
}
```

The implementation applies to every `Vec<T>`.

---

# 105. Generic `Drop`

You can implement `Drop` for a generic type:

```rust
struct Container<T> {
    value: T,
}

impl<T> Drop for Container<T> {
    fn drop(&mut self) {
        println!("dropping");
    }
}
```

The implementation applies to every `Container<T>`.

---

# 106. Generic Trait Bounds on Associated Types

You can combine bounds:

```rust
fn process<I>(iter: I)
where
    I: Iterator,
    I::Item: std::fmt::Debug,
{
    // ...
}
```

Here:

```text
I implements Iterator
+
I::Item implements Debug
```

This pattern is very common in generic Rust code.

---

# 107. Nested `where` Constraints

Complex APIs may look like:

```rust
fn process<T, U>(value: T, other: U)
where
    T: Clone + Send,
    U: Iterator,
    U::Item: std::fmt::Debug,
{
}
```

Read the bounds one at a time.

Do not try to understand the whole signature at once.

---

# 108. Generic Function with `where` + Lifetime

Example:

```rust
fn process<'a, T>(value: &'a T)
where
    T: 'a + std::fmt::Debug,
{
    println!("{value:?}");
}
```

Break it down:

```text
'a
    ↓
lifetime

T
    ↓
generic type

T: 'a
    ↓
T is valid for 'a

T: Debug
    ↓
T supports Debug
```

---

# 109. Generic Function with Lifetime + Trait Object

You may encounter:

```rust
fn process<'a>(value: &'a dyn Trait) {
}
```

This means:

```text
'a
    ↓
lifetime of the reference

dyn Trait
    ↓
trait object
```

Compare:

```rust
fn process<T: Trait>(value: &T)
```

with:

```rust
fn process(value: &dyn Trait)
```

The first is generic/static dispatch.

The second is dynamic dispatch.

---

# 110. Generic Function Returning Trait Objects

You can write:

```rust
fn create() -> Box<dyn Trait> {
    Box::new(MyType)
}
```

The caller does not know the concrete type.

Compare with:

```rust
fn create() -> impl Trait {
    MyType
}
```

Both hide the concrete return type, but they use different mechanisms.

---

# 111. Generic Function Returning `impl Trait`

Example:

```rust
fn numbers() -> impl Iterator<Item = i32> {
    0..10
}
```

The compiler knows the concrete type.

The caller only knows the trait interface.

This is an **opaque type**.

---

# 112. Generic Function Accepting `impl Trait`

Example:

```rust
fn process(value: impl Display) {
    println!("{value}");
}
```

This is approximately equivalent to:

```rust
fn process<T: Display>(value: T) {
    println!("{value}");
}
```

Use whichever syntax makes the API clearer.

---

# 113. Generic Type Parameter vs `impl Trait`

Compare:

```rust
fn foo<T: Display>(x: T) -> T
```

and:

```rust
fn foo(x: impl Display) -> impl Display
```

The first explicitly names `T`, so you can use the same type parameter elsewhere:

```rust
fn foo<T: Display>(x: T, y: T) -> T
```

With:

```rust
fn foo(x: impl Display, y: impl Display)
```

the two parameters may have different concrete types.

This is an important difference.

---

# 114. `impl Trait` Parameters Can Represent Different Types

For:

```rust
fn foo(x: impl Display, y: impl Display) {
}
```

conceptually:

```text
x: T
y: U

T: Display
U: Display
```

So:

```rust
foo(10, "hello");
```

can work.

But:

```rust
fn foo<T: Display>(x: T, y: T) {
}
```

requires:

```text
x and y have the same T
```

---

# 115. Generic Parameters Express Equality of Types

For:

```rust
fn pair<T>(x: T, y: T) -> (T, T) {
    (x, y)
}
```

both arguments must have the same type.

But:

```rust
fn pair(x: impl Display, y: impl Display) {
}
```

does not require the same type.

This makes generic syntax semantically important.

---

# 116. Generic Return Types Must Be Determined

Consider:

```rust
fn create<T>() -> T {
    // ?
}
```

The compiler cannot magically create an arbitrary `T`.

A trait bound can provide the necessary operation:

```rust
fn create<T: Default>() -> T {
    T::default()
}
```

This is a common generic design pattern.

---

# 117. Generic Bounds as Interfaces

A generic function:

```rust
fn save<T: Write>(value: T) {
}
```

does not care whether `T` is:

```text
File
TcpStream
Vec<u8>
CustomWriter
```

It only requires:

```text
T: Write
```

This is similar to programming against an interface.

But unlike inheritance-based interfaces, Rust traits are independent capabilities that types can implement.

---

# 118. Static Dispatch and Performance

Generic code:

```rust
fn process<T: Trait>(value: T) {
}
```

can often be optimized aggressively.

Because the compiler knows:

```text
exact type
exact implementation
```

it can potentially:

* inline calls
* remove abstraction overhead
* specialize code
* optimize data layout

This is one of the major strengths of Rust's generics.

---

# 119. The Cost of Monomorphization

Monomorphization is not free.

If a generic function is instantiated for many different types:

```text
T = A
T = B
T = C
T = D
...
```

the binary may contain multiple specialized versions.

Potential consequences:

* larger binaries
* longer compilation times
* more generated machine code

This is one reason dynamic dispatch can sometimes be useful.

---

# 120. Generics vs Dynamic Dispatch

A useful comparison:

| Feature                         | Generics     | `dyn Trait`   |
| ------------------------------- | ------------ | ------------- |
| Dispatch                        | Static       | Dynamic       |
| Concrete type known by compiler | Yes          | No            |
| Monomorphization                | Yes          | No            |
| Vtable                          | No           | Yes           |
| Runtime dispatch cost           | Usually none | Usually small |
| Binary size                     | Can increase | Often smaller |
| Heterogeneous collection        | Not directly | Yes           |
| Compile-time optimization       | Strong       | More limited  |

Neither approach is universally better.

Choose based on the API and requirements.

---

# 121. Generic Code and Zero-Cost Abstractions

Rust often aims for abstractions that compile down to code comparable to handwritten specialized code.

For example:

```rust
fn process<T: Trait>(value: T) {
    value.method();
}
```

The generic abstraction itself does not necessarily introduce runtime overhead.

The compiler can specialize it.

This is one of the reasons generics are heavily used throughout Rust's standard library.

---

# 122. Generics and Type Safety

Generics prevent many classes of mistakes.

For example:

```rust
struct UserId(u64);
struct DeviceId(u64);
```

Even though both contain:

```text
u64
```

they are different types.

Generic wrappers can extend this idea:

```rust
struct Id<T> {
    value: u64,
    _marker: PhantomData<T>,
}
```

Now:

```text
Id<User>
Id<Device>
```

cannot accidentally be mixed.

---

# 123. Generics and API Design

When designing a generic API, ask:

### What varies?

Use a generic parameter:

```rust
<T>
```

### What must the type be able to do?

Use a trait bound:

```rust
T: Trait
```

### Does the API need to return an abstract implementation?

Consider:

```rust
impl Trait
```

### Does the API need heterogeneous values?

Consider:

```rust
dyn Trait
```

### Does the type need to encode a number?

Consider:

```rust
const N: usize
```

### Does the type borrow data?

Consider:

```rust
'a
```

---

# 124. Generic Syntax Cheat Sheet

## One type parameter

```rust
<T>
```

## Multiple type parameters

```rust
<T, U>
```

## Generic function

```rust
fn foo<T>(value: T) {
}
```

## Generic function with return type

```rust
fn foo<T>(value: T) -> T {
    value
}
```

## Generic struct

```rust
struct Foo<T> {
    value: T,
}
```

## Generic enum

```rust
enum Foo<T> {
    Value(T),
}
```

## Generic trait

```rust
trait Foo<T> {
}
```

## Generic implementation

```rust
impl<T> Foo<T> {
}
```

## Generic trait implementation

```rust
impl<T> Trait for Foo<T> {
}
```

## Trait bound

```rust
T: Trait
```

## Multiple bounds

```rust
T: TraitA + TraitB
```

## `where`

```rust
fn foo<T, U>(x: T, y: U)
where
    T: TraitA,
    U: TraitB,
{
}
```

## Associated type constraint

```rust
T: Iterator<Item = i32>
```

## `impl Trait` parameter

```rust
fn foo(value: impl Trait) {
}
```

## `impl Trait` return

```rust
fn foo() -> impl Trait {
}
```

## Trait object

```rust
dyn Trait
```

## Boxed trait object

```rust
Box<dyn Trait>
```

## Lifetime + generic type

```rust
fn foo<'a, T>(value: &'a T) {
}
```

## Lifetime bound

```rust
T: 'a
```

## Lifetime outlives

```rust
'a: 'b
```

## Anonymous lifetime

```rust
'_
```

## Higher-ranked trait bound

```rust
for<'a>
```

## `Sized`

```rust
T: Sized
```

## Allow unsized types

```rust
T: ?Sized
```

## Const generic

```rust
<const N: usize>
```

## Type + const generic

```rust
<T, const N: usize>
```

## Lifetime + type + const

```rust
<'a, T, const N: usize>
```

## Turbofish

```rust
::<T>
```

## Multiple turbofish arguments

```rust
::<T, U>
```

---

# 125. A Complete Generic Example

Here is an example combining many of the concepts:

```rust
use std::fmt::Debug;

struct Container<'a, T, const N: usize> {
    values: &'a [T; N],
}

impl<'a, T, const N: usize> Container<'a, T, N>
where
    T: Debug,
{
    fn print(&self) {
        println!("{:?}", self.values);
    }

    fn first(&self) -> &T {
        &self.values[0]
    }
}
```

This contains:

```text
'a
    lifetime parameter

T
    type parameter

N
    const parameter

T: Debug
    trait bound

&'a [T; N]
    borrowed array

impl<'a, T, const N: usize>
    generic implementation
```

This is a good example of how Rust's generic system combines multiple concepts.

---

# 126. How to Read Complex Generic Signatures

When you see a complicated Rust signature, don't try to understand it all at once.

For example:

```rust
fn process<'a, T, U, const N: usize>(
    value: &'a [T; N],
    other: U,
) -> &'a T
where
    T: Clone + Debug,
    U: Send + 'static,
{
    &value[0]
}
```

Read it in layers.

### Layer 1: Lifetime

```rust
'a
```

There is a lifetime parameter.

### Layer 2: Types

```rust
T
U
```

There are two type parameters.

### Layer 3: Const

```rust
const N: usize
```

There is a compile-time integer parameter.

### Layer 4: Inputs

```rust
&'a [T; N]
```

A reference to an array of `N` values of type `T`.

```rust
U
```

An owned value of type `U`.

### Layer 5: Output

```rust
&'a T
```

A reference to `T` tied to `'a`.

### Layer 6: Bounds

```rust
T: Clone + Debug
```

`T` must implement `Clone` and `Debug`.

```rust
U: Send + 'static
```

`U` must satisfy `Send` and `'static`.

This layered approach makes complex Rust signatures much easier to understand.

---

# 127. Generics Are Compile-Time Abstraction

A useful mental model is:

```text
Generic source code
       │
       ▼
Compiler
       │
       ├── T = i32
       ├── T = String
       └── T = MyType
       │
       ▼
Concrete machine code
```

The generic abstraction exists primarily at compile time.

This is different from dynamic dispatch, where the runtime may need to determine which implementation to call.

---

# 128. The Three Main Generic Parameter Categories

Rust's generic parameters can be understood as three categories:

```text
Generic parameters
│
├── Lifetime
│     'a
│
├── Type
│     T
│
└── Const
      N
```

For example:

```rust
struct Buffer<'a, T, const N: usize> {
    data: &'a [T; N],
}
```

contains all three.

---

# 129. Generics, Traits, and Lifetimes Work Together

These three concepts are often learned separately, but real Rust code combines them.

For example:

```rust
fn process<'a, T>(value: &'a T)
where
    T: Clone + Send + 'a,
{
}
```

Here:

```text
'a
    lifetime

T
    generic type

Clone
    capability

Send
    concurrency capability

T: 'a
    lifetime relationship
```

Rust's type system allows all these constraints to be expressed together.

---

# 130. The Big Picture

Generics answer:

> **What type can vary?**

Traits answer:

> **What behavior does that type provide?**

Trait bounds answer:

> **What behavior must the generic type provide?**

Lifetimes answer:

> **How long are borrowed references valid?**

Const generics answer:

> **What compile-time value can vary?**

`impl Trait` answers:

> **Can I hide the concrete type while exposing its capabilities?**

`dyn Trait` answers:

> **Can I use runtime polymorphism through a trait object?**

Monomorphization answers:

> **How can generic code become concrete machine code?**

---

# 131. Final Mental Model

When you see:

```rust
fn process<T: Trait>(value: T)
```

think:

```text
T
│
└── some concrete type
       │
       └── must implement Trait
```

When you see:

```rust
fn process<'a, T>(value: &'a T)
```

think:

```text
'a
 │
 └── lifetime of the reference

T
 │
 └── type being referenced
```

When you see:

```rust
fn process<T, U>(x: T, y: U)
```

think:

```text
T and U may be different types
```

When you see:

```rust
fn process<T>(x: T, y: T)
```

think:

```text
x and y must have the same type T
```

When you see:

```rust
fn process(x: impl Trait)
```

think:

```text
some concrete type implementing Trait
```

When you see:

```rust
fn process(x: &dyn Trait)
```

think:

```text
trait object
+
dynamic dispatch
```

When you see:

```rust
T: ?Sized
```

think:

```text
T may be dynamically sized
```

When you see:

```rust
<const N: usize>
```

think:

```text
N is a compile-time value
```

When you see:

```rust
for<'a>
```

think:

```text
for every possible lifetime 'a
```

And the most important idea is:

> **Generics allow Rust to abstract over types, lifetimes, and compile-time values while preserving strong compile-time type checking and, in the common case, enabling static dispatch and zero-cost abstractions.**

---
