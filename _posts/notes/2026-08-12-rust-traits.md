---
title: "Rust Notes: Traits"
categories:
  - Learning Notes
  - Rust
tags: [rust, traits, generics, polymorphism, static-dispatch, dynamic-dispatch]
description: "A review of traits in Rust."

toc: true
---

# Rust Traits — A Practical Guide

Traits are one of the most important features in Rust.

They are used to define **shared behavior**, provide **polymorphism**, constrain **generic types**, enable **static and dynamic dispatch**, and define interfaces between parts of a program.

If you come from C++ or Java, traits can feel similar to a combination of:

* interfaces
* abstract classes
* generic constraints
* type classes

But Rust's model is different because it does not use traditional class inheritance.

A useful mental model is:

> **Structs define what data a type has. Traits define what a type can do.**

---

# 1. Defining a Trait

The simplest trait looks like this:

```rust
trait Animal {
    fn speak(&self);
}
```

This says:

> Any type that implements `Animal` must provide a `speak` method.

The trait itself does not define a concrete `Animal` object.

It defines behavior that other types can implement.

```text
             Animal
             trait
               │
       ┌───────┴───────┐
       ▼               ▼
      Dog             Cat
       │               │
   speak()          speak()
```

---

# 2. Implementing a Trait

Suppose we have two structs:

```rust
struct Dog;

struct Cat;
```

We can implement the trait for each:

```rust
impl Animal for Dog {
    fn speak(&self) {
        println!("Woof!");
    }
}

impl Animal for Cat {
    fn speak(&self) {
        println!("Meow!");
    }
}
```

Now both types implement `Animal`.

```rust
let dog = Dog;
let cat = Cat;

dog.speak();
cat.speak();
```

The syntax is:

```rust
impl TraitName for TypeName {
    // trait implementation
}
```

---

# 3. Traits Are Not Inheritance

Rust does not have traditional class inheritance.

You cannot write:

```rust
struct Dog : Animal {
}
```

Instead, Rust uses traits:

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;

impl Animal for Dog {
    fn speak(&self) {
        println!("Woof!");
    }
}
```

The relationship is:

```text
Animal
  ↑
  │ implements
  │
 Dog
```

not:

```text
Animal
  ↑
  │ inherits
  │
 Dog
```

This distinction is important.

Rust generally favors:

> **Composition + traits instead of inheritance.**

---

# 4. Trait Methods

A trait can contain methods.

```rust
trait Animal {
    fn speak(&self);

    fn name(&self) -> &str;
}
```

A type implementing the trait must implement both:

```rust
struct Dog {
    name: String,
}

impl Animal for Dog {
    fn speak(&self) {
        println!("Woof!");
    }

    fn name(&self) -> &str {
        &self.name
    }
}
```

---

# 5. `&self`, `&mut self`, and `self`

Trait methods can use the same receiver forms as normal methods.

### Immutable reference

```rust
trait Animal {
    fn speak(&self);
}
```

The method can read the object but cannot mutate it.

### Mutable reference

```rust
trait Animal {
    fn rename(&mut self, name: String);
}
```

The method can mutate the object.

### Taking ownership

```rust
trait Animal {
    fn destroy(self);
}
```

Calling the method consumes the object.

Example:

```rust
trait Animal {
    fn consume(self);
}

struct Dog;

impl Animal for Dog {
    fn consume(self) {
        println!("Dog consumed");
    }
}

let dog = Dog;
dog.consume();

// dog can no longer be used
```

---

# 6. Associated Functions

A trait can define a function without `self`.

```rust
trait Animal {
    fn create() -> Self;
}
```

An implementation can provide it:

```rust
struct Dog;

impl Animal for Dog {
    fn create() -> Self {
        Dog
    }
}
```

Call it using the implementing type:

```rust
let dog = Dog::create();
```

This is similar to an associated function such as:

```rust
String::new()
```

---

# 7. Default Trait Methods

Traits can provide default implementations.

```rust
trait Animal {
    fn speak(&self) {
        println!("Some animal sound");
    }
}
```

Now a type can implement the trait without implementing `speak`:

```rust
struct Dog;

impl Animal for Dog {}
```

The default implementation is used:

```rust
let dog = Dog;
dog.speak();
```

A type can also override the default:

```rust
impl Animal for Dog {
    fn speak(&self) {
        println!("Woof!");
    }
}
```

This is useful when many types share the same behavior.

---

# 8. Traits Can Have Multiple Methods

For example:

```rust
trait Animal {
    fn name(&self) -> &str;

    fn speak(&self);

    fn age(&self) -> u32;
}
```

An implementation:

```rust
struct Dog {
    name: String,
    age: u32,
}

impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }

    fn speak(&self) {
        println!("Woof!");
    }

    fn age(&self) -> u32 {
        self.age
    }
}
```

---

# 9. Trait Bounds

One of the most important uses of traits is constraining generic types.

Suppose:

```rust
fn print_animal<T>(animal: &T) {
    animal.speak();
}
```

This does not compile because Rust does not know that `T` has a `speak` method.

We need a trait bound:

```rust
fn print_animal<T: Animal>(animal: &T) {
    animal.speak();
}
```

Now Rust knows:

> `T` must implement `Animal`.

The syntax:

```rust
T: Trait
```

means:

> `T` implements `Trait`.

---

# 10. Generic Trait Bounds with `where`

The same function can be written as:

```rust
fn print_animal<T>(animal: &T)
where
    T: Animal,
{
    animal.speak();
}
```

Both forms mean the same thing:

```rust
fn print_animal<T: Animal>(animal: &T)
```

and:

```rust
fn print_animal<T>(animal: &T)
where
    T: Animal,
```

The `where` syntax becomes more useful when constraints become complicated.

For example:

```rust
fn process<T, U>(value: &T, other: &U)
where
    T: Animal + Clone,
    U: Display + Debug,
{
    // ...
}
```

---

# 11. Multiple Trait Bounds

A type can be required to implement multiple traits.

```rust
fn process<T: Animal + Clone>(animal: &T) {
    // ...
}
```

This means:

```text
T must implement Animal
AND
T must implement Clone
```

Equivalent `where` syntax:

```rust
fn process<T>(animal: &T)
where
    T: Animal + Clone,
{
}
```

---

# 12. `impl Trait` in Function Parameters

Instead of explicitly naming a generic type:

```rust
fn speak<T: Animal>(animal: &T) {
    animal.speak();
}
```

you can write:

```rust
fn speak(animal: &impl Animal) {
    animal.speak();
}
```

This means approximately:

> Accept some type that implements `Animal`.

For simple cases, these are very similar:

```rust
fn speak<T: Animal>(animal: &T)
```

and:

```rust
fn speak(animal: &impl Animal)
```

The generic form becomes more useful when the same type parameter appears multiple times.

---

# 13. Multiple `impl Trait` Parameters

Consider:

```rust
fn compare(a: &impl Animal, b: &impl Animal) {
}
```

This does **not necessarily mean** `a` and `b` have the same concrete type.

They could be:

```text
a → Dog
b → Cat
```

If you want them to have the same type:

```rust
fn compare<T: Animal>(a: &T, b: &T) {
}
```

Now:

```text
T = Dog

a → Dog
b → Dog
```

This distinction is important.

---

# 14. `impl Trait` as a Return Type

Rust can return a type implementing a trait without exposing its concrete type.

```rust
fn create_animal() -> impl Animal {
    Dog
}
```

The caller knows:

> The returned value implements `Animal`.

But the caller does not need to know the exact concrete type.

This is especially useful for iterators and complex types.

For example:

```rust
fn numbers() -> impl Iterator<Item = i32> {
    0..10
}
```

The concrete iterator type can be complicated, so `impl Iterator` hides it.

---

# 15. `impl Trait` Does Not Mean Any Type

This:

```rust
fn create_animal() -> impl Animal {
    Dog
}
```

does not mean the function can sometimes return a `Dog` and sometimes a `Cat`.

This would fail:

```rust
fn create_animal(condition: bool) -> impl Animal {
    if condition {
        Dog
    } else {
        Cat
    }
}
```

because `impl Trait` represents one concrete hidden type for that function.

If you need different concrete types at runtime, use a trait object:

```rust
fn create_animal(condition: bool) -> Box<dyn Animal> {
    if condition {
        Box::new(Dog)
    } else {
        Box::new(Cat)
    }
}
```

---

# 16. Static Dispatch

Generic trait bounds normally use **static dispatch**.

```rust
fn speak<T: Animal>(animal: &T) {
    animal.speak();
}
```

If we call:

```rust
let dog = Dog;
let cat = Cat;

speak(&dog);
speak(&cat);
```

the compiler knows the concrete type.

Conceptually:

```text
speak::<Dog>()
      │
      ▼
Dog::speak()

speak::<Cat>()
      │
      ▼
Cat::speak()
```

The decision is made at compile time.

Advantages include:

* no dynamic dispatch
* compiler can optimize aggressively
* possible inlining
* usually no vtable lookup

---

# 17. Dynamic Dispatch

Rust supports runtime polymorphism through trait objects.

```rust
fn speak(animal: &dyn Animal) {
    animal.speak();
}
```

Now:

```rust
let dog = Dog;
let cat = Cat;

speak(&dog);
speak(&cat);
```

The function receives:

```rust
&dyn Animal
```

It does not know the concrete type directly.

Conceptually:

```text
&dyn Animal
     │
     ├── Dog → Dog::speak()
     │
     └── Cat → Cat::speak()
```

The correct method is selected at runtime.

---

# 18. Trait Objects

The most common forms are:

```rust
&dyn Trait
Box<dyn Trait>
Arc<dyn Trait>
Rc<dyn Trait>
```

For example:

```rust
let animal: Box<dyn Animal> = Box::new(Dog);
```

This allows the concrete type to be hidden behind the trait interface.

---

# 19. Different Types in One Collection

This is one of the main benefits of trait objects.

You cannot do:

```rust
let animals = vec![Dog, Cat];
```

because a vector normally contains one concrete type.

Instead:

```rust
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog),
    Box::new(Cat),
];
```

Now:

```text
Vec<Box<dyn Animal>>
       │
       ├── Dog
       └── Cat
```

We can iterate:

```rust
for animal in animals {
    animal.speak();
}
```

This is dynamic polymorphism.

---

# 20. Trait Object vs Enum

You previously saw an enum solution:

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}
```

An enum explicitly lists all possible variants.

```text
SpreadsheetCell
    │
    ├── Int
    ├── Float
    └── Text
```

A trait object does not require listing every possible implementation:

```rust
Box<dyn Animal>
```

Any type implementing `Animal` can potentially be stored there.

```text
dyn Animal
    │
    ├── Dog
    ├── Cat
    ├── Horse
    └── ...
```

A useful rule:

> **Use an enum when the set of possibilities is known and part of your data model.**

> **Use a trait when you want different types to provide common behavior.**

---

# 21. Associated Types

Traits can define associated types.

```rust
trait Container {
    type Item;

    fn get(&self) -> Self::Item;
}
```

Implementation:

```rust
struct NumberBox;

impl Container for NumberBox {
    type Item = i32;

    fn get(&self) -> Self::Item {
        42
    }
}
```

Another implementation could use another type:

```rust
struct TextBox;

impl Container for TextBox {
    type Item = String;

    fn get(&self) -> Self::Item {
        String::from("hello")
    }
}
```

The associated type is chosen by the implementation.

---

# 22. Associated Type vs Generic Trait

Compare:

```rust
trait Container {
    type Item;
}
```

with:

```rust
trait Container<T> {
}
```

An associated type generally means:

> For a particular implementation of this trait, there is one associated `Item` type.

For example:

```rust
impl Container for NumberBox {
    type Item = i32;
}
```

A generic trait can allow multiple implementations with different type parameters:

```rust
trait Container<T> {
}
```

This distinction is important in APIs such as `Iterator`:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

---

# 23. Associated Constants

Traits can also define constants.

```rust
trait Config {
    const MAX_SIZE: usize;

    fn validate(&self) -> bool;
}
```

Implementation:

```rust
struct NetworkConfig;

impl Config for NetworkConfig {
    const MAX_SIZE: usize = 1500;

    fn validate(&self) -> bool {
        true
    }
}
```

Access:

```rust
println!("{}", NetworkConfig::MAX_SIZE);
```

---

# 24. Supertraits

A trait can require another trait.

```rust
trait Animal: Display {
    fn speak(&self);
}
```

This means:

> Every `Animal` must also implement `Display`.

Then:

```rust
struct Dog;

impl Display for Dog {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "Dog")
    }
}

impl Animal for Dog {
    fn speak(&self) {
        println!("Woof!");
    }
}
```

Conceptually:

```text
Display
   ↑
   │ required
Animal
   ↑
   │
 Dog
```

This is sometimes called a **trait inheritance relationship**, but it is not class inheritance.

---

# 25. Multiple Supertraits

You can require multiple traits:

```rust
trait Animal: Display + Debug + Send {
    fn speak(&self);
}
```

An implementation must satisfy all of them.

```text
Animal
 ├── Display
 ├── Debug
 └── Send
```

---

# 26. Blanket Implementations

Rust allows implementing a trait for every type satisfying some condition.

Example:

```rust
trait Summary {
    fn summary(&self) -> String;
}
```

Suppose every type implementing `Display` should automatically implement `Summary`:

```rust
impl<T: Display> Summary for T {
    fn summary(&self) -> String {
        format!("{self}")
    }
}
```

Now any `Display` type automatically has `Summary`.

This is called a **blanket implementation**.

Conceptually:

```text
Any T
 │
 └── if T: Display
          │
          ▼
       Summary
```

Blanket implementations are heavily used in Rust libraries.

---

# 27. Trait Implementations for References

A trait can be implemented for references too.

```rust
trait Speak {
    fn speak(&self);
}

impl Speak for Dog {
    fn speak(&self) {
        println!("Woof");
    }
}
```

You can also implement a trait for:

```rust
impl Speak for &Dog {
    fn speak(&self) {
        println!("Woof");
    }
}
```

This is useful when behavior differs depending on the exact type being implemented.

However, many standard traits are designed so that implementations for references can often be achieved through borrowing and method resolution without explicitly writing these implementations.

---

# 28. Trait Implementations for Generic Types

You can implement a trait for a generic struct:

```rust
struct Container<T> {
    value: T,
}

trait Describe {
    fn describe(&self);
}

impl<T: Display> Describe for Container<T> {
    fn describe(&self) {
        println!("{}", self.value);
    }
}
```

Now:

```rust
let container = Container {
    value: 42,
};

container.describe();
```

The implementation only exists when `T: Display`.

---

# 29. Conditional Trait Implementations

You can conditionally implement a trait:

```rust
impl<T: Clone> MyTrait for Container<T> {
    // ...
}
```

This means:

> `Container<T>` implements `MyTrait` only when `T` implements `Clone`.

This is a very powerful generic programming technique.

---

# 30. Trait Bounds on `impl` Blocks

You can put the bound directly on the implementation:

```rust
impl<T> Container<T>
where
    T: Display,
{
    fn print(&self) {
        println!("{}", self.value);
    }
}
```

This method is available only when `T: Display`.

---

# 31. Trait Bounds on Methods

You can also restrict individual methods:

```rust
impl<T> Container<T> {
    fn value(&self) -> &T {
        &self.value
    }

    fn print(&self)
    where
        T: Display,
    {
        println!("{}", self.value);
    }
}
```

Now `value()` works for every `T`, while `print()` only works when `T: Display`.

---

# 32. Generic Traits

Traits themselves can have generic parameters:

```rust
trait Converter<T> {
    fn convert(&self, value: T) -> String;
}
```

Implementation:

```rust
struct Formatter;

impl Converter<i32> for Formatter {
    fn convert(&self, value: i32) -> String {
        value.to_string()
    }
}
```

The trait is parameterized by `T`.

---

# 33. Associated Types Are Often Preferred

Instead of:

```rust
trait Converter<T> {
    fn convert(&self, value: T);
}
```

you may use:

```rust
trait Converter {
    type Input;

    fn convert(&self, value: Self::Input);
}
```

Then:

```rust
struct Formatter;

impl Converter for Formatter {
    type Input = i32;

    fn convert(&self, value: i32) {
        println!("{value}");
    }
}
```

Which design is better depends on whether multiple implementations with different parameters should be possible.

---

# 34. Generic Methods Inside Traits

A trait method can itself be generic:

```rust
trait Printer {
    fn print<T: Display>(&self, value: T);
}
```

Implementation:

```rust
struct ConsolePrinter;

impl Printer for ConsolePrinter {
    fn print<T: Display>(&self, value: T) {
        println!("{value}");
    }
}
```

Now:

```rust
let printer = ConsolePrinter;

printer.print(42);
printer.print("hello");
printer.print(3.14);
```

---

# 35. Lifetimes in Traits

Traits can also use lifetimes.

For example:

```rust
trait Parser<'a> {
    fn parse(&self, input: &'a str) -> &'a str;
}
```

The lifetime parameter connects the input and output references.

Traits can also use lifetime bounds:

```rust
trait Trait<'a> {
    fn process(&self, value: &'a str);
}
```

---

# 36. Trait Bounds with Lifetimes

You can combine lifetimes and traits:

```rust
fn process<'a, T>(value: &'a T)
where
    T: Display,
{
    println!("{value}");
}
```

Or:

```rust
fn process<'a, T: Display>(value: &'a T) {
    println!("{value}");
}
```

---

# 37. `Self` in Traits

Inside a trait, `Self` means:

> The concrete type implementing this trait.

Example:

```rust
trait Animal {
    fn create() -> Self;
}
```

For:

```rust
impl Animal for Dog {
    fn create() -> Self {
        Dog
    }
}
```

`Self` means `Dog`.

For:

```rust
impl Animal for Cat {
    fn create() -> Self {
        Cat
    }
}
```

`Self` means `Cat`.

---

# 38. Returning `Self`

A common trait pattern is:

```rust
trait Builder {
    fn new() -> Self;
}
```

This allows each implementation to construct itself.

```rust
struct Server;

impl Builder for Server {
    fn new() -> Self {
        Server
    }
}
```

---

# 39. Traits Returning `Self`

You can also write:

```rust
trait Cloneable {
    fn duplicate(&self) -> Self;
}
```

Each implementation returns its own concrete type.

```rust
impl Cloneable for Dog {
    fn duplicate(&self) -> Self {
        Dog
    }
}
```

---

# 40. Operators Use Traits

Many Rust operators are implemented through traits.

For example:

```rust
a + b
```

is associated with the `Add` trait.

Conceptually:

```rust
trait Add<Rhs = Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

So operator syntax is often just convenient syntax over trait behavior.

Examples include:

```text
+       Add
-       Sub
*       Mul
/       Div
==      PartialEq
<       PartialOrd
[]      Index
```

This is an important Rust idea:

> Much of Rust's language behavior is built around traits.

---

# 41. Common Standard Traits

You will frequently encounter:

```text
Debug
Display
Clone
Copy
PartialEq
Eq
PartialOrd
Ord
Hash
Default
Iterator
IntoIterator
From
Into
AsRef
AsMut
Borrow
Deref
DerefMut
Send
Sync
```

Examples:

```rust
#[derive(Debug, Clone, PartialEq)]
struct User {
    name: String,
}
```

The `derive` attribute automatically generates implementations for supported traits.

---

# 42. `derive`

Instead of manually implementing:

```rust
impl Debug for User {
    // ...
}
```

you can often write:

```rust
#[derive(Debug)]
struct User {
    name: String,
}
```

Rust generates the implementation.

Multiple traits:

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct User {
    name: String,
}
```

---

# 43. Fully Qualified Syntax

Sometimes multiple traits provide methods with the same name.

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("Pilot flying");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Wizard flying");
    }
}
```

Now:

```rust
let human = Human;
```

This is ambiguous:

```rust
human.fly();
```

We can explicitly select the trait:

```rust
Pilot::fly(&human);
Wizard::fly(&human);
```

Or use fully qualified syntax:

```rust
<Human as Pilot>::fly(&human);
<Human as Wizard>::fly(&human);
```

This syntax is especially useful when associated functions are ambiguous.

---

# 44. Fully Qualified Syntax for Associated Functions

Suppose:

```rust
trait Animal {
    fn create() -> Self;
}

struct Dog;

impl Animal for Dog {
    fn create() -> Self {
        Dog
    }
}
```

You can write:

```rust
let dog = Dog::create();
```

But if multiple traits define `create`, you may need:

```rust
let dog = <Dog as Animal>::create();
```

The general syntax is:

```rust
<Type as Trait>::function(...)
```

---

# 45. Calling Trait Methods with UFCS

UFCS means **Universal Function Call Syntax**.

For example:

```rust
<Dog as Animal>::speak(&dog);
```

This explicitly says:

> Call the `Animal` implementation of `speak` for `Dog`.

It is useful when Rust cannot determine which implementation you mean.

---

# 46. Trait Objects and Object Safety

Not every trait can be used as:

```rust
dyn Trait
```

For example, a trait containing:

```rust
trait Example {
    fn create() -> Self;
}
```

cannot generally be used as a trait object because `Self` must represent the concrete type.

Similarly, methods that are generic often prevent a trait from being used as a trait object.

For example:

```rust
trait Example {
    fn process<T>(&self, value: T);
}
```

This cannot be used normally as:

```rust
dyn Example
```

because the runtime vtable cannot contain an implementation for every possible `T`.

A useful simplified rule is:

> Trait objects require methods whose behavior can be represented through a runtime vtable.

Modern Rust documentation often refers to these requirements using the concept of **dyn compatibility**.

---

# 47. Trait Objects Usually Need Indirection

This does not work:

```rust
let animal: dyn Animal = Dog;
```

because `dyn Animal` has an unknown size at compile time.

Instead, use a pointer-like type:

```rust
let animal: Box<dyn Animal> = Box::new(Dog);
```

or:

```rust
let animal: &dyn Animal = &dog;
```

or:

```rust
let animal: Arc<dyn Animal> = Arc::new(Dog);
```

The pointer has a known size even though the concrete object does not.

---

# 48. `dyn Trait` and Dynamic Dispatch

When you write:

```rust
let animal: &dyn Animal = &dog;
```

the type is a trait object.

Conceptually:

```text
&dyn Animal
┌───────────────────┐
│ pointer to Dog    │
├───────────────────┤
│ pointer to vtable │
└───────────────────┘
```

The vtable contains information needed for dynamic dispatch.

Therefore:

```rust
animal.speak();
```

can find the correct implementation at runtime.

---

# 49. Static vs Dynamic Dispatch

The most important comparison:

```rust
fn speak<T: Animal>(animal: &T) {
    animal.speak();
}
```

uses static dispatch.

```rust
fn speak(animal: &dyn Animal) {
    animal.speak();
}
```

uses dynamic dispatch.

Think:

```text
T: Animal
   │
   └── compiler knows concrete type
       └── static dispatch

dyn Animal
   │
   └── concrete type hidden
       └── runtime dispatch
```

Static dispatch generally gives the compiler more opportunities for optimization.

Dynamic dispatch gives more flexibility at runtime.

---

# 50. Choosing Static or Dynamic Dispatch

Prefer static dispatch when:

```rust
fn process<T: Animal>(animal: &T)
```

is sufficient.

Benefits:

* compile-time resolution
* usually no vtable lookup
* easier optimization
* often faster

Use dynamic dispatch when you need runtime polymorphism:

```rust
fn process(animal: &dyn Animal)
```

For example, when you need:

```rust
Vec<Box<dyn Animal>>
```

containing different concrete types.

A useful rule:

> **Generics are usually about compile-time polymorphism.**

> **`dyn Trait` is about runtime polymorphism.**

---

# 51. Traits and Encapsulation

Traits also help with encapsulation.

Suppose:

```rust
pub struct Database {
    connection: String,
}
```

The field is private.

You can expose behavior instead:

```rust
pub trait Repository {
    fn save(&self);
    fn load(&self);
}
```

Then expose only the trait interface.

The caller can depend on:

```rust
&dyn Repository
```

without knowing the concrete implementation.

```text
Application
     │
     ▼
Repository trait
     │
     ├── PostgreSQLRepository
     ├── MemoryRepository
     └── MockRepository
```

This is particularly useful for designing clean boundaries.

---

# 52. Traits and Testing

Traits are useful for dependency injection and testing.

Suppose production code uses:

```rust
trait Storage {
    fn get(&self, key: &str) -> String;
}
```

Production implementation:

```rust
struct DatabaseStorage;

impl Storage for DatabaseStorage {
    fn get(&self, key: &str) -> String {
        // database access
        String::from("real value")
    }
}
```

Test implementation:

```rust
struct MockStorage;

impl Storage for MockStorage {
    fn get(&self, key: &str) -> String {
        String::from("fake value")
    }
}
```

The application can work with the trait instead of depending directly on the database.

---

# 53. Traits and Dependency Injection

For example:

```rust
fn process<S: Storage>(storage: &S) {
    let value = storage.get("key");
    println!("{value}");
}
```

Production:

```rust
process(&DatabaseStorage);
```

Test:

```rust
process(&MockStorage);
```

This is one reason traits are heavily used in Rust systems programming.

---

# 54. Traits and Ownership

Traits do not change Rust's ownership rules.

For example:

```rust
trait Animal {
    fn speak(&self);
}
```

If you have:

```rust
let dog = Dog;
```

you still need to respect ownership when passing it around.

You can borrow:

```rust
speak(&dog);
```

or move:

```rust
fn consume<T: Animal>(animal: T) {
    // owns animal
}
```

Traits describe behavior; ownership determines how the value is accessed and moved.

---

# 55. Traits and `Send` / `Sync`

`Send` and `Sync` are also traits.

For example:

```rust
fn spawn<T: Send + 'static>(value: T) {
    // ...
}
```

This means `T` must satisfy the requirements for being transferred to another thread.

Similarly:

```rust
fn share<T: Sync>(value: &T) {
}
```

requires that references to `T` can safely be shared between threads.

This demonstrates an important property of Rust:

> Traits are not only for application-level interfaces. They are also used by the language and standard library to express important type properties.

---

# 56. Traits Can Express Capabilities

A useful way to think about traits is:

```text
Trait = capability / behavior / contract
```

For example:

```rust
Clone
```

means:

> This type knows how to make a duplicate.

```rust
Iterator
```

means:

> This type can produce a sequence of values.

```rust
Send
```

means:

> This type can be transferred between threads.

```rust
Display
```

means:

> This type knows how to be formatted for users.

```rust
Debug
```

means:

> This type provides a debugging representation.

---

# 57. Trait Bounds as Requirements

When you write:

```rust
fn process<T: Clone + Debug>(value: T) {
}
```

you are effectively saying:

```text
I don't care exactly what T is.

I only require that:

    T implements Clone
    AND
    T implements Debug
```

This is one of the most powerful ideas in Rust.

The function depends on **behavior**, not a concrete type.

---

# 58. Traits vs Concrete Types

Badly coupled code might look like:

```rust
fn process(database: &PostgresDatabase) {
    // ...
}
```

The function is tied to PostgreSQL.

Trait-based code:

```rust
fn process<S: Storage>(storage: &S) {
    // ...
}
```

Now the function only cares that `S` implements `Storage`.

This allows:

```text
PostgresStorage
MemoryStorage
MockStorage
FileStorage
```

to all work with the same function.

---

# 59. The Most Important Trait Syntax

Here is a compact syntax reference.

## Define a trait

```rust
trait TraitName {
    fn method(&self);
}
```

## Implement a trait

```rust
impl TraitName for MyType {
    fn method(&self) {
    }
}
```

## Default method

```rust
trait TraitName {
    fn method(&self) {
    }
}
```

## Associated type

```rust
trait TraitName {
    type Item;
}
```

## Associated constant

```rust
trait TraitName {
    const VALUE: usize;
}
```

## Associated function

```rust
trait TraitName {
    fn create() -> Self;
}
```

## Generic trait

```rust
trait TraitName<T> {
}
```

## Trait bound

```rust
fn process<T: TraitName>(value: T) {
}
```

## Multiple bounds

```rust
fn process<T: TraitA + TraitB>(value: T) {
}
```

## `where`

```rust
fn process<T>(value: T)
where
    T: TraitA + TraitB,
{
}
```

## `impl Trait` parameter

```rust
fn process(value: impl TraitName) {
}
```

## `impl Trait` return type

```rust
fn create() -> impl TraitName {
    MyType
}
```

## Trait object

```rust
fn process(value: &dyn TraitName) {
}
```

## Heap-allocated trait object

```rust
let value: Box<dyn TraitName> = Box::new(MyType);
```

## Reference-counted trait object

```rust
let value: Rc<dyn TraitName> = Rc::new(MyType);
```

## Thread-safe shared trait object

```rust
let value: Arc<dyn TraitName + Send + Sync> = Arc::new(MyType);
```

## Supertrait

```rust
trait Child: Parent {
}
```

## Multiple supertraits

```rust
trait Child: ParentA + ParentB {
}
```

## Blanket implementation

```rust
impl<T: SomeTrait> AnotherTrait for T {
}
```

## Conditional implementation

```rust
impl<T> SomeTrait for Container<T>
where
    T: Clone,
{
}
```

## Fully qualified syntax

```rust
<Type as Trait>::method(...)
```

## Generic method

```rust
trait Printer {
    fn print<T: Display>(&self, value: T);
}
```

## Trait with lifetime

```rust
trait Parser<'a> {
    fn parse(&self, input: &'a str);
}
```

---

# 60. A Complete Example

Putting several concepts together:

```rust
use std::fmt::Display;

trait Animal: Display {
    const SPECIES: &'static str;

    fn speak(&self);

    fn description(&self) -> String {
        format!("{self}")
    }
}

struct Dog {
    name: String,
}

struct Cat {
    name: String,
}

impl Display for Dog {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Dog named {}", self.name)
    }
}

impl Animal for Dog {
    const SPECIES: &'static str = "Canis familiaris";

    fn speak(&self) {
        println!("Woof!");
    }
}

impl Display for Cat {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Cat named {}", self.name)
    }
}

impl Animal for Cat {
    const SPECIES: &'static str = "Felis catus";

    fn speak(&self) {
        println!("Meow!");
    }
}
```

Static dispatch:

```rust
fn static_speak<T: Animal>(animal: &T) {
    animal.speak();
}
```

Dynamic dispatch:

```rust
fn dynamic_speak(animal: &dyn Animal) {
    animal.speak();
}
```

Using them:

```rust
let dog = Dog {
    name: String::from("Rex"),
};

let cat = Cat {
    name: String::from("Milo"),
};

static_speak(&dog);
static_speak(&cat);

dynamic_speak(&dog);
dynamic_speak(&cat);
```

Different types in one collection:

```rust
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(dog),
    Box::new(cat),
];

for animal in animals {
    animal.speak();
}
```

---

# 61. The Mental Model to Remember

Think about Rust's type system like this:

```text
                     Trait
                       │
             defines behavior
                       │
          ┌────────────┴────────────┐
          │                         │
        Dog                        Cat
          │                         │
       impl                        impl
          │                         │
          └────────────┬────────────┘
                       │
                  Animal behavior
```

Then polymorphism can happen in two major ways:

```text
Trait
 │
 ├── Generic T: Trait
 │       │
 │       └── Static dispatch
 │
 └── dyn Trait
         │
         └── Dynamic dispatch
```

And traits can also be used as constraints:

```text
T: Trait
 │
 └── "T must provide this capability"
```

---

# Key Takeaways

* A **trait defines shared behavior**, not shared data.
* A type implements a trait with `impl Trait for Type`.
* Rust does not have traditional class inheritance.
* Rust commonly uses **composition + traits** instead.
* Traits provide one of Rust's main mechanisms for polymorphism.
* `T: Trait` provides a trait bound.
* Generic trait bounds normally use **static dispatch**.
* `dyn Trait` provides **dynamic dispatch**.
* `Box<dyn Trait>` allows different concrete types to be stored together.
* `impl Trait` can be used in function parameters and return types.
* `impl Trait` as a return type represents one hidden concrete type.
* `dyn Trait` allows the concrete type to vary at runtime.
* Traits can contain methods, associated functions, associated types, and associated constants.
* Traits can provide default method implementations.
* A trait can require other traits using **supertraits**.
* Blanket implementations can implement a trait for every type satisfying a condition.
* Traits can be generic.
* Trait methods can be generic.
* Traits can have lifetime parameters.
* `Self` represents the concrete implementing type.
* Fully qualified syntax such as `<Type as Trait>::method()` resolves ambiguity.
* Many Rust operators and standard-library capabilities are implemented through traits.
* `Send` and `Sync` are traits too.
* Traits allow code to depend on **capabilities rather than concrete types**.
* This makes traits useful for abstraction, polymorphism, dependency injection, testing, and API design.

The most important mental model is:

> **Structs describe what a type is made of; traits describe what a type can do.**

And for polymorphism:

> **Generics usually mean static dispatch; `dyn Trait` means dynamic dispatch.**
