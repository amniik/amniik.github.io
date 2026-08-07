---
title: "Rust Notes: Concurrency, Threads, and Thread Safety"
categories:
  - Learning Notes
  - Rust
tags: [rust, concurrency, threads, send, sync, ownership]
description: "Learning notes about concurrency, parallelism, Rust threads, Send, Sync, and communication models."

toc: true
---

# Rust Notes: Concurrency, Threads, and Thread Safety

> Mental model:
>
> - Concurrency: multiple tasks make progress independently.
> - Parallelism: multiple tasks execute at the same time.
> - `Send`: ownership can move between threads.
> - `Sync`: references can be safely shared between threads.

---

# Concurrency vs Parallelism

## Concurrency

Concurrency means that different parts of a program can make progress independently.

Example:

```
Task A: ========

Task B:     ========
```

The tasks may be executed:

- interleaved on a single CPU core,
- or actually in parallel on multiple cores.

Examples:

- handling multiple network connections,
- responding to user input while performing background work.

---

## Parallelism

Parallelism means that different parts of a program execute at the same time.

Example:

```
CPU 1: Task A ========

CPU 2: Task B ========
```

Parallelism requires multiple execution units, usually multiple CPU cores.

Examples:

- image processing,
- scientific computation,
- large data processing.

---

# Threads

A thread is an independent execution path inside a process.

A process can contain multiple threads:

```
Process
 |
 +-- Thread 1
 |
 +-- Thread 2
 |
 +-- Thread 3
```

Threads share:

- memory,
- address space,
- resources.

Sharing memory makes communication fast, but it can introduce problems such as data races.

---

# Data Races

A data race happens when:

- multiple threads access the same memory,
- at least one access is a write,
- there is no synchronization.

Example:

```
Initial value:

counter = 0


Thread A              Thread B

read counter          read counter

+1                    +1

write 1               write 1
```

Expected:

```
counter = 2
```

Actual:

```
counter = 1
```

Rust prevents these problems using:

- ownership,
- borrowing rules,
- `Send`,
- `Sync`.

---

# Thread Models

Programming languages use different models for creating threads.

---

# 1:1 Thread Model

Rust standard library uses the **1:1 threading model**.

Meaning:

```
Language Thread
       |
       |
       v
Operating System Thread
```

Each language thread corresponds to one OS thread.

Example:

```rust
use std::thread;

thread::spawn(|| {
    println!("hello");
});
```

Rust calls operating system APIs to create the thread.

## Advantages

- Simple model.
- Uses the operating system scheduler.
- Predictable performance.
- No large runtime required.

## Disadvantages

- OS threads are relatively expensive.
- Creating millions of threads is not practical.

---

# Green Threads

Some languages provide threads managed by the language runtime.

These are called:

- green threads,
- user-level threads,
- lightweight threads.

The model is:

```
Many Language Threads

          |
          v

   Language Runtime

          |
          v

     OS Threads
```

The runtime is responsible for scheduling these threads.

## Advantages

- Very lightweight.
- Can create many more threads.
- Good for highly concurrent applications.

## Disadvantages

- Requires a runtime scheduler.
- Adds complexity to the language runtime.

---

# M:N Thread Model

The M:N model maps many language threads to fewer OS threads.

```
M Language Threads

        |
        v

Runtime Scheduler

        |
        v

N OS Threads
```

Example:

```
10000 application tasks

          |
          v

Runtime Scheduler

          |
          v

8 OS Threads
```

The runtime decides where tasks execute.

Rust's standard library does not provide M:N green threads.

The standard `std::thread` API uses the 1:1 model.

---

# Why Rust Uses 1:1 Threads

Rust focuses on:

- zero-cost abstractions,
- predictable performance,
- minimal runtime.

Instead of putting a thread scheduler into the language, Rust allows libraries to provide higher-level concurrency models.

Examples:

- async runtimes,
- Tokio.

---

# Send Trait

`Send` is a marker trait that means:

> Ownership of a value can safely move to another thread.

Example:

```rust
use std::thread;

let value = String::from("hello");

thread::spawn(move || {
    println!("{}", value);
});
```

The ownership moves:

```
Thread A

String

  |
  | move
  v

Thread B

String
```

The compiler checks:

```
String: Send
```

---

# Sync Trait

`Sync` is a marker trait that means:

> A type can safely be shared between threads through immutable references (`&T`).

Example:

```
Thread A
    |
   &String
    |
    String
    |
   &String
    |
Thread B
```

Multiple threads can read the same value safely.

Therefore:

```
String: Sync
```

The formal definition:

```
T is Sync if &T is Send
```

Meaning:

If a reference to `T` can safely move to another thread, then `T` is safe to share.

---

# Send vs Sync

## Send

Question:

> Can ownership move to another thread?

```
Thread A

    T

    |
    v

Thread B

    T
```

Requires:

```
T: Send
```

---

## Sync

Question:

> Can multiple threads safely have `&T`?

```
Thread A ----\
              \
               &T
              /
Thread B ----/
```

Requires:

```
T: Sync
```

---

# Example: String

`String` is both `Send` and `Sync`.

## Send

Move ownership:

```rust
thread::spawn(move || {
    println!("{}", s);
});
```

The `String` moves to another thread.

---

## Sync

Share references:

```rust
thread::scope(|scope| {
    scope.spawn(|| {
        println!("{}", s);
    });

    scope.spawn(|| {
        println!("{}", s);
    });
});
```

The closures capture references:

```
&String
```

The compiler checks that:

```
String: Sync
```

---

# Example: Rc vs Arc

## Rc<T>

`Rc<T>` uses a normal reference counter.

Multiple threads updating the counter could create a data race.

Therefore:

```
Rc<T>: !Send
Rc<T>: !Sync
```

---

## Arc<T>

`Arc<T>` uses atomic reference counting.

It is designed for multi-threaded shared ownership.

```
Thread A ----\
              \
               Arc
                |
                T
              /
Thread B ----/
```

Therefore:

```
Arc<T>: Send
Arc<T>: Sync
```

if `T` is thread safe.

---

# Shared Memory vs Message Passing

A common concurrency principle:

> "Do not communicate by sharing memory; instead, share memory by communicating."

---

# Shared Memory

Example:

```rust
Arc<Mutex<T>>
```

Model:

```
Thread A ----\
              \
              Mutex
                |
                T
              /
Thread B ----/
```

The mutex ensures only one thread can mutate the data at a time.

Advantages:

- Direct access to data.
- Sometimes better performance.

Disadvantages:

- Locking complexity.
- Possible deadlocks.
- Harder reasoning.

---

# Message Passing

Instead of sharing memory, threads communicate by sending messages.

Example:

```rust
use std::sync::mpsc;

let (sender, receiver) = mpsc::channel();
```

Sender:

```rust
sender.send(value);
```

Receiver:

```rust
receiver.recv();
```

Ownership moves through the channel:

```
Thread A

   value

     |
     v

 Channel

     |
     v

Thread B
```

Advantages:

- Easier reasoning.
- Ownership prevents many races.

Disadvantages:

- Message passing can have overhead.

---

# Common Thread Safety Rules

| Type | Send | Sync | Reason |
|------|------|------|--------|
| `String` | ✅ | ✅ | Safe ownership transfer and shared reading |
| `Vec<T>` | ✅ | ✅ | If `T` is safe |
| `Box<T>` | ✅ | ✅ | If `T` is safe |
| `Rc<T>` | ❌ | ❌ | Non-atomic reference counter |
| `Arc<T>` | ✅ | ✅ | Atomic reference counter |
| `RefCell<T>` | depends | ❌ | Runtime borrowing is not thread safe |
| `Mutex<T>` | ✅ | ✅ | Synchronizes access |

---

# Rust Concurrency Philosophy

Rust does not prevent concurrency.

Rust prevents **unsafe concurrency**.

The compiler checks:

- Can this value move between threads? → `Send`
- Can references to this value be shared between threads? → `Sync`

If the answer is yes, the program compiles.

If not, Rust rejects the program before runtime.

---

# Key Takeaways

- Concurrency means independent progress.
- Parallelism means simultaneous execution.
- Rust standard threads use the 1:1 OS thread model.
- Green threads require a runtime scheduler.
- Rust avoids built-in green threads to keep the language lightweight.
- `Send` allows ownership transfer between threads.
- `Sync` allows safe sharing through immutable references.
- Shared memory requires synchronization.
- Message passing avoids many shared-state problems.
- Rust uses ownership rules to make concurrent programming safer.
