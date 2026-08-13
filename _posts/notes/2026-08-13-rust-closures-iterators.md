---
title: "Rust Notes: Closures and Iterators"
categories:
  - Learning Notes
  - Rust
tags: [rust, closures, iterators, ownership, traits]
description: "A concise reference to Rust closures and iterators."

toc: true
---

# Rust Closures and Iterators

Closures and iterators are two important features of idiomatic Rust.

A **closure** is an anonymous function that can capture variables from its surrounding environment.

An **iterator** represents a sequence of values and produces those values one at a time.

They are commonly used together:

```rust
let result: Vec<_> = numbers
    .iter()
    .filter(|x| **x > 10)
    .map(|x| x * 2)
    .collect();
```

The closure provides the behavior, the iterator provides the sequence, and the consumer drives the computation.

## 1. Closures

A closure is an anonymous function.

### Basic syntax

```rust
|parameters| expression
```

Examples:

```rust
let add = |x, y| x + y;

let square = |x: i32| x * x;

let add = |x: i32, y: i32| -> i32 {
    x + y
};

let hello = || {
    println!("Hello");
};
```

A closure can contain multiple statements:

```rust
let calculate = |x: i32| {
    let y = x * 2;
    y + 10
};
```

The last expression is the return value.

## 2. Closure vs Function

A normal function:

```rust
fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

A closure:

```rust
let add = |x: i32, y: i32| x + y;
```

The important difference is that closures can capture variables from their surrounding environment.

```rust
let multiplier = 10;

let multiply = |x| x * multiplier;

println!("{}", multiply(5));
```

A normal function cannot automatically access `multiplier`.

## 3. Closure Capture

Closures can capture variables by:

* immutable borrow
* mutable borrow
* ownership

### Immutable borrow

```rust
let name = String::from("Amir");

let print_name = || {
    println!("{name}");
};
```

The closure only reads `name`, so it can capture it by shared reference.

### Mutable borrow

```rust
let mut counter = 0;

let mut increment = || {
    counter += 1;
};

increment();
increment();

println!("{counter}");
```

The closure captures `counter` mutably.

The closure variable itself must be mutable:

```rust
let mut increment = ...
```

### Move / ownership

```rust
let name = String::from("Amir");

let consume = || {
    drop(name);
};

consume();
```

The closure moves `name` into its environment and then consumes it when called.

## 4. The `move` Keyword

The `move` keyword forces a closure to capture variables by value.

```rust
let name = String::from("Amir");

let print_name = move || {
    println!("{name}");
};
```

Without `move`, Rust chooses the capture mode based on how the variable is used.

With `move`, ownership of the captured value is transferred to the closure.

This is especially important for threads:

```rust
use std::thread;

let data = String::from("hello");

thread::spawn(move || {
    println!("{data}");
});
```

The spawned thread owns `data`.

## 5. `move` Does Not Mean `FnOnce`

This is an important distinction.

`move` describes **how a closure captures variables**.

`Fn`, `FnMut`, and `FnOnce` describe **how the closure can be called**.

For example:

```rust
let name = String::from("Amir");

let print_name = move || {
    println!("{name}");
};

print_name();
print_name();
```

The closure owns `name`, but it only reads it.

Therefore it can still implement `Fn`.

The distinction is:

```text
move
    capture mode

Fn / FnMut / FnOnce
    calling behavior
```

## 6. `Fn`, `FnMut`, and `FnOnce`

Rust has three closure-call traits:

```rust
Fn
FnMut
FnOnce
```

Their relationship is:

```text
Fn
 |
 v
FnMut
 |
 v
FnOnce
```

Therefore:

```text
Fn      can be used where FnMut is required
FnMut   can be used where FnOnce is required
```

Every closure implements `FnOnce`.

Some closures also implement `FnMut`.

Some closures also implement `Fn`.

## 7. `Fn`

A closure implements `Fn` when it can be called repeatedly without mutating or consuming captured values.

```rust
let name = String::from("Amir");

let print_name = || {
    println!("{name}");
};

print_name();
print_name();
```

A function can require an `Fn` closure:

```rust
fn execute<F>(f: F)
where
    F: Fn(),
{
    f();
}
```

Or:

```rust
fn execute(f: impl Fn()) {
    f();
}
```

## 8. `FnMut`

`FnMut` is used when the closure can be called repeatedly and may mutate captured state.

```rust
let mut counter = 0;

let mut increment = || {
    counter += 1;
};

increment();
increment();
increment();

println!("{counter}");
```

A function accepting `FnMut`:

```rust
fn execute<F>(mut f: F)
where
    F: FnMut(),
{
    f();
    f();
}
```

Or:

```rust
fn execute(mut f: impl FnMut()) {
    f();
    f();
}
```

## 9. `FnOnce`

`FnOnce` is used when calling the closure may consume captured values.

```rust
let name = String::from("Amir");

let consume = || {
    drop(name);
};

consume();
```

The closure consumes `name`, so it cannot normally be called again.

A function accepting `FnOnce`:

```rust
fn execute<F>(f: F)
where
    F: FnOnce(),
{
    f();
}
```

Or:

```rust
fn execute(f: impl FnOnce()) {
    f();
}
```

## 10. Choosing the Closure Trait

Use `Fn` when:

```text
the closure can be called repeatedly
and does not mutate or consume captured state
```

Use `FnMut` when:

```text
the closure can be called repeatedly
and may mutate captured state
```

Use `FnOnce` when:

```text
the closure may consume captured state
or only needs to be called once
```

Examples:

```rust
fn run_once(f: impl FnOnce()) {
    f();
}
```

```rust
fn run_many(mut f: impl FnMut()) {
    f();
    f();
}
```

```rust
fn run_read_only(f: impl Fn()) {
    f();
    f();
}
```

## 11. Closure Trait Syntax

Closure traits can be used with generic bounds:

```rust
F: Fn() -> i32
```

```rust
F: Fn(i32) -> i32
```

```rust
F: Fn(i32, i32) -> i32
```

```rust
F: Fn(&str) -> bool
```

The same syntax applies to:

```rust
FnMut(...)
```

and:

```rust
FnOnce(...)
```

Example:

```rust
fn transform<F>(value: i32, f: F) -> i32
where
    F: Fn(i32) -> i32,
{
    f(value)
}
```

## 12. `impl Fn` Syntax

Instead of:

```rust
fn execute<F>(f: F)
where
    F: Fn(),
{
    f();
}
```

you can write:

```rust
fn execute(f: impl Fn()) {
    f();
}
```

Similarly:

```rust
fn execute(f: impl FnMut()) {
    // ...
}
```

```rust
fn execute(f: impl FnOnce()) {
    // ...
}
```

With parameters and return values:

```rust
fn transform(
    value: i32,
    f: impl Fn(i32) -> i32,
) -> i32 {
    f(value)
}
```

## 13. Returning a Closure

Closure types are anonymous, so `impl Fn` is commonly used when returning closures.

```rust
fn create_adder() -> impl Fn(i32) -> i32 {
    |x| x + 10
}
```

Usage:

```rust
let add_ten = create_adder();

println!("{}", add_ten(5));
```

A returned closure can capture values:

```rust
fn create_adder(value: i32) -> impl Fn(i32) -> i32 {
    move |x| x + value
}
```

The `move` makes the returned closure own `value`.

## 14. Closure Trait Objects

Closures can also be stored behind trait objects.

```rust
let f: Box<dyn Fn(i32) -> i32> =
    Box::new(|x| x + 1);
```

Other forms:

```rust
&dyn Fn(i32) -> i32
```

```rust
Box<dyn Fn(i32) -> i32>
```

```rust
Arc<dyn Fn(i32) -> i32 + Send + Sync>
```

This uses dynamic dispatch.

By contrast:

```rust
impl Fn(i32) -> i32
```

normally uses static dispatch.

## 15. Function Pointers

A function pointer has the syntax:

```rust
fn(i32) -> i32
```

Example:

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

let f: fn(i32) -> i32 = add_one;
```

A non-capturing closure can also become a function pointer:

```rust
let f: fn(i32) -> i32 = |x| x + 1;
```

A capturing closure generally cannot become a plain function pointer because a function pointer has no environment in which to store captured variables.

# Iterators

## 16. The `Iterator` Trait

The core iterator abstraction is the `Iterator` trait:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

The most important method is:

```rust
next()
```

It returns:

```rust
Some(value)
```

when another value exists, or:

```rust
None
```

when iteration is finished.

Example:

```rust
let numbers = vec![1, 2, 3];

let mut iter = numbers.iter();

println!("{:?}", iter.next());
println!("{:?}", iter.next());
println!("{:?}", iter.next());
println!("{:?}", iter.next());
```

Output:

```text
Some(1)
Some(2)
Some(3)
None
```

## 17. `for` Loops and Iterators

A `for` loop uses the iterator protocol internally.

```rust
for value in collection {
    println!("{value}");
}
```

Conceptually, this is similar to:

```rust
let mut iterator = collection.into_iter();

loop {
    match iterator.next() {
        Some(value) => {
            println!("{value}");
        }
        None => break,
    }
}
```

Therefore, understanding `Iterator::next()` explains how `for` loops work.

## 18. `iter()`

`iter()` creates an iterator that yields immutable references.

```rust
let numbers = vec![1, 2, 3];

for number in numbers.iter() {
    println!("{number}");
}

println!("{numbers:?}");
```

The vector still owns its elements.

Conceptually:

```text
Vec<T>
 |
 +---- &T
 +---- &T
 +---- &T
```

## 19. `iter_mut()`

`iter_mut()` yields mutable references.

```rust
let mut numbers = vec![1, 2, 3];

for number in numbers.iter_mut() {
    *number *= 2;
}

println!("{numbers:?}");
```

Conceptually:

```text
Vec<T>
 |
 +---- &mut T
 +---- &mut T
 +---- &mut T
```

## 20. `into_iter()`

`into_iter()` consumes the collection and yields owned values.

```rust
let numbers = vec![1, 2, 3];

for number in numbers.into_iter() {
    println!("{number}");
}
```

After this, `numbers` can no longer be used.

Conceptually:

```text
Vec<T>
 |
 +---- T
 +---- T
 +---- T

Vec<T> is consumed
```

The basic rule is:

```text
iter()
    immutable borrow

iter_mut()
    mutable borrow

into_iter()
    ownership
```

## 21. Iterator Item Type

The `Iterator` trait has an associated type:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

For:

```rust
let numbers = vec![1, 2, 3];

let iter = numbers.iter();
```

the `Item` type is:

```rust
&i32
```

For:

```rust
let iter = numbers.into_iter();
```

the `Item` type is:

```rust
i32
```

This is one reason `iter()`, `iter_mut()`, and `into_iter()` behave differently.

## 22. Lazy Iterators

Iterator adapters are lazy.

Consider:

```rust
let numbers = vec![1, 2, 3];

let result = numbers
    .iter()
    .map(|x| {
        println!("processing {x}");
        x * 2
    });
```

The `map()` operation has not processed all values yet.

It has created another iterator.

The computation happens when something consumes the iterator:

```rust
let result: Vec<_> = numbers
    .iter()
    .map(|x| {
        println!("processing {x}");
        x * 2
    })
    .collect();
```

A useful mental model:

```text
iterator adapters
        |
        | describe computation
        v
consumer
        |
        | drives iterator
        v
actual computation
```

## 23. Iterator Adapters

Iterator adapters usually return another iterator.

Common adapters include:

```text
map
filter
filter_map
enumerate
zip
take
skip
chain
rev
peekable
flat_map
flatten
inspect
```

Example:

```rust
let iter = numbers
    .iter()
    .filter(|x| **x > 10)
    .map(|x| x * 2);
```

The result is still an iterator.

## 24. `map`

`map` transforms every item.

```rust
let numbers = vec![1, 2, 3];

let result: Vec<_> = numbers
    .iter()
    .map(|x| x * 2)
    .collect();
```

Result:

```text
[2, 4, 6]
```

The closure:

```rust
|x| x * 2
```

defines the transformation.

## 25. `filter`

`filter` keeps items for which the closure returns `true`.

```rust
let numbers = vec![1, 2, 3, 4, 5];

let result: Vec<_> = numbers
    .iter()
    .filter(|x| **x % 2 == 0)
    .collect();
```

Result:

```text
[2, 4]
```

## 26. `filter_map`

`filter_map` combines filtering and mapping.

```rust
let values = vec!["1", "hello", "2"];

let numbers: Vec<i32> = values
    .iter()
    .filter_map(|value| value.parse().ok())
    .collect();
```

Result:

```text
[1, 2]
```

The closure returns:

```rust
Some(value)
```

to keep a transformed value, or:

```rust
None
```

to discard it.

## 27. `enumerate`

`enumerate()` adds an index.

```rust
let names = vec!["Alice", "Bob", "Charlie"];

for (index, name) in names.iter().enumerate() {
    println!("{index}: {name}");
}
```

Output:

```text
0: Alice
1: Bob
2: Charlie
```

## 28. `zip`

`zip` combines two iterators.

```rust
let names = vec!["Alice", "Bob"];
let ages = vec![20, 30];

let result: Vec<_> = names
    .iter()
    .zip(ages.iter())
    .collect();
```

Result:

```text
[
    ("Alice", 20),
    ("Bob", 30),
]
```

Iteration stops when either iterator ends.

## 29. `take`

`take` limits the number of elements.

```rust
let numbers = 1..100;

let result: Vec<_> = numbers
    .take(5)
    .collect();
```

Result:

```text
[1, 2, 3, 4, 5]
```

## 30. `skip`

`skip` ignores the first elements.

```rust
let numbers = 1..6;

let result: Vec<_> = numbers
    .skip(2)
    .collect();
```

Result:

```text
[3, 4, 5]
```

## 31. `chain`

`chain` joins two iterators.

```rust
let a = vec![1, 2];
let b = vec![3, 4];

let result: Vec<_> = a
    .into_iter()
    .chain(b.into_iter())
    .collect();
```

Result:

```text
[1, 2, 3, 4]
```

## 32. `rev`

`rev()` reverses a double-ended iterator.

```rust
let numbers = vec![1, 2, 3];

let result: Vec<_> = numbers
    .into_iter()
    .rev()
    .collect();
```

Result:

```text
[3, 2, 1]
```

## 33. `flatten`

`flatten` removes one level of nesting.

```rust
let values = vec![
    vec![1, 2],
    vec![3, 4],
];

let result: Vec<_> = values
    .into_iter()
    .flatten()
    .collect();
```

Result:

```text
[1, 2, 3, 4]
```

## 34. `flat_map`

`flat_map` maps and then flattens.

```rust
let words = vec!["hello", "world"];

let chars: Vec<_> = words
    .into_iter()
    .flat_map(|word| word.chars())
    .collect();
```

Result:

```text
['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']
```

## 35. `inspect`

`inspect` is useful for debugging an iterator chain.

```rust
let result: Vec<_> = numbers
    .iter()
    .inspect(|x| println!("before: {x}"))
    .map(|x| x * 2)
    .inspect(|x| println!("after: {x}"))
    .collect();
```

It does not change the values.

## 36. Consumers

Consumers drive an iterator and produce a final result or side effect.

Common consumers include:

```text
collect
sum
product
count
fold
reduce
find
position
any
all
for_each
nth
last
```

## 37. `collect`

`collect()` converts an iterator into a collection.

```rust
let numbers = 1..5;

let result: Vec<i32> = numbers.collect();
```

Type inference can also determine the target type:

```rust
let result: Vec<_> = (1..5).collect();
```

You can explicitly specify the collection:

```rust
let result = (1..5).collect::<Vec<_>>();
```

## 38. `sum`

```rust
let total: i32 = (1..5).sum();
```

Result:

```text
10
```

## 39. `count`

```rust
let count = (1..10).count();
```

Result:

```text
9
```

## 40. `fold`

`fold` accumulates values.

```rust
let sum = (1..5).fold(0, |acc, value| {
    acc + value
});
```

Result:

```text
10
```

The first argument is the initial accumulator:

```rust
0
```

The closure receives:

```text
accumulator
current item
```

and returns the next accumulator.

A shorter version:

```rust
let sum = (1..5).fold(0, |acc, x| acc + x);
```

## 41. `find`

`find` returns the first element matching a condition.

```rust
let result = (1..10)
    .find(|x| *x > 5);
```

Result:

```text
Some(6)
```

If nothing matches:

```text
None
```

## 42. `any`

`any` returns `true` if at least one item matches.

```rust
let result = (1..10)
    .any(|x| x > 8);
```

Result:

```text
true
```

## 43. `all`

`all` returns `true` if every item matches.

```rust
let result = (1..10)
    .all(|x| *x > 0);
```

Result:

```text
true
```

## 44. `for_each`

`for_each` executes a closure for every item.

```rust
(1..5).for_each(|x| {
    println!("{x}");
});
```

It is similar to:

```rust
for x in 1..5 {
    println!("{x}");
}
```

## 45. `nth`

`nth(n)` returns the element at index `n`.

```rust
let result = (10..20).nth(3);
```

Result:

```text
Some(13)
```

Note that `nth()` consumes all preceding iterator elements.

## 46. `last`

`last()` returns the final element.

```rust
let result = (1..5).last();
```

Result:

```text
Some(4)
```

The iterator must be consumed to determine the final element.

# Iterator Chains

## 47. Combining Adapters and Consumers

Iterator chains are one of the most common patterns in Rust.

```rust
let numbers = vec![1, 2, 3, 4, 5, 6];

let result: Vec<_> = numbers
    .iter()
    .filter(|x| **x % 2 == 0)
    .map(|x| x * 10)
    .collect();
```

The flow is:

```text
numbers
   |
   v
iter()
   |
   v
filter()
   |
   v
map()
   |
   v
collect()
   |
   v
Vec
```

The closures define what happens to each element.

The iterator controls when each element is requested.

## 48. Why Iterators Are Lazy

Consider:

```rust
let result = (1..100)
    .filter(|x| x % 2 == 0)
    .map(|x| x * 10);
```

Nothing has necessarily been computed yet.

`result` is an iterator.

When we call:

```rust
result.next()
```

the iterator starts doing the work necessary to produce one value.

For example:

```text
next()
  |
  v
1 -> rejected
2 -> accepted
  |
  v
20
```

The iterator does not need to process all 100 values first.

This allows iterator chains to process values incrementally.

---

# Iterator Ownership

## 49. `iter`, `iter_mut`, and `into_iter`

These three methods are fundamental.

| Method        | Yields   | Collection       |
| ------------- | -------- | ---------------- |
| `iter()`      | `&T`     | borrowed         |
| `iter_mut()`  | `&mut T` | mutably borrowed |
| `into_iter()` | `T`      | consumed         |

Example:

```rust
let values = vec![String::from("a"), String::from("b")];

for value in values.iter() {
    println!("{value}");
}

println!("{values:?}");
```

The vector remains available.

With `into_iter()`:

```rust
let values = vec![String::from("a"), String::from("b")];

for value in values.into_iter() {
    println!("{value}");
}
```

The vector is consumed.

---

# Implementing Your Own Iterator

## 50. Custom Iterator

You can implement `Iterator` for your own type.

```rust
struct Counter {
    current: u32,
    max: u32,
}
```

Implement `Iterator`:

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.current < self.max {
            let value = self.current;
            self.current += 1;
            Some(value)
        } else {
            None
        }
    }
}
```

Usage:

```rust
let counter = Counter {
    current: 0,
    max: 5,
};

for value in counter {
    println!("{value}");
}
```

Output:

```text
0
1
2
3
4
```

The key idea is that the iterator controls its state through `&mut self`.

---

# Iterators and Closures Together

## 51. The Common Rust Pattern

A very common Rust pattern is:

```rust
collection
    .iter()
    .filter(...)
    .map(...)
    .collect()
```

For example:

```rust
let users = vec![
    ("Alice", 20),
    ("Bob", 15),
    ("Charlie", 30),
];

let names: Vec<_> = users
    .iter()
    .filter(|(_, age)| *age >= 18)
    .map(|(name, _)| *name)
    .collect();
```

Here:

```text
iter()
    creates the iterator

filter()
    uses a closure to decide what stays

map()
    uses a closure to transform values

collect()
    consumes the iterator
```

---

# Iterator vs Collection

A collection stores values:

```rust
Vec<T>
```

An iterator describes how to obtain values:

```rust
Iterator<Item = T>
```

Conceptually:

```text
Collection
    |
    | owns/stores values
    v
Vec<T>

Iterator
    |
    | produces values
    v
Item
```

An iterator does not necessarily store all the values.

For example:

```rust
let numbers = 1..1_000_000_000;
```

The range iterator does not need to store one billion integers.

It can produce them one at a time.

---

# Iterator Adaptors vs Consumers

A useful distinction:

## Adapters

Adapters return another iterator.

Examples:

```rust
map()
filter()
take()
skip()
zip()
enumerate()
chain()
rev()
flatten()
flat_map()
```

Example:

```rust
let iter = numbers
    .iter()
    .filter(...)
    .map(...);
```

Still an iterator.

## Consumers

Consumers finish the iteration.

Examples:

```rust
collect()
sum()
count()
fold()
find()
any()
all()
for_each()
```

Example:

```rust
let result = numbers
    .iter()
    .filter(...)
    .map(...)
    .collect();
```

`collect()` consumes the iterator.

---

# Static Dispatch with Closures and Iterators

Generic closures and iterators commonly use static dispatch.

For example:

```rust
fn transform<I, F>(iter: I, f: F)
where
    I: Iterator<Item = i32>,
    F: Fn(i32) -> i32,
{
    for value in iter {
        println!("{}", f(value));
    }
}
```

The compiler knows the concrete types of:

```text
I
F
```

and can generate specialized code.

This can allow inlining and optimization.

---

# Dynamic Dispatch with Closure Trait Objects

You can also use dynamic dispatch:

```rust
let functions: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(|x| x + 1),
    Box::new(|x| x * 2),
];
```

Now all closures are represented through:

```rust
dyn Fn(i32) -> i32
```

This is useful when different closure types need to be stored together.

Remember that every closure expression has its own anonymous type:

```rust
let a = |x| x + 1;
let b = |x| x * 2;
```

Even though both have similar signatures, their concrete types are different.

Therefore:

```rust
Vec<_>
```

cannot directly contain both closure types.

Trait objects solve this:

```rust
Vec<Box<dyn Fn(i32) -> i32>>
```

---

# Closures, Iterators, and Ownership

Closures inside iterator chains follow normal Rust ownership rules.

For example:

```rust
let names = vec![
    String::from("Alice"),
    String::from("Bob"),
];

let lengths: Vec<_> = names
    .iter()
    .map(|name| name.len())
    .collect();
```

`iter()` yields:

```rust
&String
```

so the closure receives references.

If we consume the vector:

```rust
let names = vec![
    String::from("Alice"),
    String::from("Bob"),
];

let lengths: Vec<_> = names
    .into_iter()
    .map(|name| name.len())
    .collect();
```

the closure receives owned `String` values.

---

# Closures and `move` in Iterator Chains

A `move` closure can take ownership of external variables.

```rust
let multiplier = 10;

let result: Vec<_> = (1..5)
    .map(move |x| x * multiplier)
    .collect();
```

The closure owns `multiplier`.

This is particularly useful when a closure needs to outlive its current scope or when it is passed to another thread/task.

---

# Closures in Multithreaded Code

Closures are heavily used with threads:

```rust
use std::thread;

let data = vec![1, 2, 3];

let handle = thread::spawn(move || {
    for value in data {
        println!("{value}");
    }
});

handle.join().unwrap();
```

The `move` transfers ownership of `data` into the closure.

The closure must satisfy the requirements imposed by `thread::spawn`, including appropriate lifetime and thread-safety bounds.

---

# Closures in Async Rust

Closures and async blocks are different language constructs, but they can work together.

For example:

```rust
let task = async move {
    println!("{data}");
};
```

An `async move` block captures variables by value and produces a `Future`.

This is conceptually similar to:

```rust
move || {
    // closure body
}
```

but an async block produces a future rather than a callable closure.

---

# Important Mental Model

A useful way to think about closures is:

```text
Closure
    =
function
+
captured environment
```

For example:

```rust
let multiplier = 10;

let multiply = |x| x * multiplier;
```

Conceptually:

```text
multiply
+-------------------+
| multiplier: 10    |
|                   |
| call(x):          |
|   x * multiplier  |
+-------------------+
```

A useful way to think about iterators is:

```text
Iterator
    =
state
+
next() operation
```

Conceptually:

```text
Iterator
+-------------------+
| current state     |
|                   |
| next()            |
|   -> Some(value)  |
|   -> None         |
+-------------------+
```

---

# Common Syntax Cheat Sheet

## Closures

```rust
|| {}
```

```rust
|x| x + 1
```

```rust
|x: i32| x + 1
```

```rust
|x: i32| -> i32 {
    x + 1
}
```

```rust
move |x| x + value
```

## Closure bounds

```rust
F: Fn()
```

```rust
F: Fn(i32) -> i32
```

```rust
F: FnMut()
```

```rust
F: FnOnce()
```

```rust
impl Fn()
```

```rust
impl FnMut()
```

```rust
impl FnOnce()
```

## Closure trait objects

```rust
dyn Fn()
```

```rust
Box<dyn Fn()>
```

```rust
Arc<dyn Fn() + Send + Sync>
```

## Function pointers

```rust
fn(i32) -> i32
```

## Iterators

```rust
collection.iter()
```

```rust
collection.iter_mut()
```

```rust
collection.into_iter()
```

```rust
iterator.next()
```

## Adapters

```rust
.map(...)
```

```rust
.filter(...)
```

```rust
.filter_map(...)
```

```rust
.enumerate()
```

```rust
.zip(...)
```

```rust
.take(...)
```

```rust
.skip(...)
```

```rust
.chain(...)
```

```rust
.rev()
```

```rust
.flatten()
```

```rust
.flat_map(...)
```

```rust
.inspect(...)
```

## Consumers

```rust
.collect()
```

```rust
.sum()
```

```rust
.count()
```

```rust
.fold(initial, closure)
```

```rust
.find(closure)
```

```rust
.any(closure)
```

```rust
.all(closure)
```

```rust
.for_each(closure)
```

```rust
.nth(index)
```

```rust
.last()
```

---

# Key Takeaways

* A closure is an anonymous function that can capture its environment.
* Closures can capture by immutable reference, mutable reference, or ownership.
* `move` forces a closure to capture values by ownership.
* `move` and `FnOnce` are different concepts.
* `Fn` means the closure can be called without mutating or consuming its environment.
* `FnMut` allows mutation of captured state.
* `FnOnce` allows captured values to be consumed.
* Every closure implements `FnOnce`.
* `iter()` yields `&T`.
* `iter_mut()` yields `&mut T`.
* `into_iter()` yields owned `T` values and consumes the collection.
* `Iterator::next()` is the core iterator operation.
* Iterator adapters such as `map` and `filter` are lazy.
* Consumers such as `collect`, `sum`, and `fold` drive the iterator.
* Iterator chains are a combination of iterator state and closures.
* Generic closures and iterators normally use static dispatch.
* `dyn Fn`, `Box<dyn Fn>`, and similar types provide dynamic dispatch.
* Closures and iterators work closely with Rust's ownership and borrowing system.
* Understanding `Fn/FnMut/FnOnce` and `iter/iter_mut/into_iter` is essential for idiomatic Rust.
