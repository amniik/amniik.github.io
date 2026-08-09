---
title: "Rust Notes: Async"
categories:
  - Learning Notes
  - Rust
tags: [rust, async, concurrency, futures, tokio]
description: "A review of asynchronous programming in Rust."

toc: true
---


## 1. Concurrency vs Parallelism

**Concurrency** means multiple tasks make progress independently.

**Parallelism** means multiple pieces of work execute at the same time, usually on different **CPU** cores.

```text
Concurrency:

Task A ──────┐
Task B ──────┼──> **CPU**
Task C ──────┘
one at a time

Parallelism:

Task A ──────> **CPU** 1
Task B ──────> **CPU** 2
Task C ──────> **CPU** 3
```

Async Rust primarily provides **concurrency**, not automatic parallelism.

---

## 2. OS Threads vs Async Tasks

An OS thread is managed by the operating system.

An async task is managed by an **async runtime**.

```text
OS
│
├── OS Thread
│     ├── Task A
│     ├── Task B
│     └── Task C
│
└── OS Thread
    ├── Task D
    └── Task E
```

A runtime can schedule many async tasks onto a smaller number of OS threads.

### 1:1 Threading

One language thread corresponds to one OS thread.

```text
Rust thread ─────> OS thread 
Rust thread ─────> OS thread
Rust thread ─────> OS thread
```

Rust's standard `std::thread` **API** uses OS threads.

### Green Threads / M:N

A runtime manages many language-level tasks over fewer OS threads.

```text
Task A ──┐ Task B ──┤ Task C ──┼──> OS Thread 1 
Task D ──┤ Task E ──┘
```

Async Rust uses this general model through external runtimes such as Tokio.

---

# 3. What Is an Async Runtime?

Rust's `async`/`await` syntax does not itself create OS threads or execute futures.

A **runtime** is responsible for driving futures.

Its responsibilities commonly include:

- polling futures
- scheduling tasks
- managing wake-ups
- integrating with I/O
- managing timers
- managing worker threads

Conceptually:

```text
    Runtime
    │
    ┌────────┼────────┐
    ▼        ▼        ▼
    Task A   Task B   Task C
```

The runtime repeatedly polls tasks that are ready to make progress.

---

# 4. What Is a Future?

A `Future` represents an asynchronous computation that may produce a value later.

Conceptually:

```text
Future
    │
    ├── Pending
    │
    ├── Pending
    │
    └── Ready(value)
```

The important point:

> Creating a future does not necessarily execute it.

A future must be **polled** by something, usually an async runtime.

---

# 5. `async`

An `async` block or function creates a future.

```rust
async fn hello() {
    println!(*hello*);
}
```

Calling:

```rust let future = hello(); ```

does not mean the body has completed.

It creates a future representing the computation.

```text
hello()
    │
    ▼
Future
    │
    ▼
runtime polls it
    │
    ▼
execution
```

The compiler transforms async code into a state machine implementing `Future`.

---

# 6. Async Functions Become State Machines

Consider:

```rust
async fn example() {
    first().await;
    second().await;
}
```

Conceptually, the compiler creates states similar to:

```text
State 1:
    call first()

State 2:
    waiting for first()

State 3:
    call second()

State 4:
    waiting for second()

State 5:
    finished
```

The future remembers where execution should continue.

This is why an async task can stop at an `.await` and later continue from the same point.

---

# 7. What Does `poll()` Do?

A runtime drives a future by calling:

```rust poll() ```

The future returns either:

```rust Poll::Pending ```

or:

```rust Poll::Ready(value) ```

Conceptually:

```text
runtime
    │
    ▼
poll(future)
    │
    ├── Pending
    │
    └── Ready(value)
```

`Pending` means:

> "I cannot make more progress right now. Try me again later."

---

# 8. What Does `.await` Do?

Consider:

```rust let response = request().await; println!(*done*); ```

If `request()` is not ready:

```text
request()
    │
    ▼
Pending
    │
    ▼
task yields
```

The task does **not** continue to:

```rust println!(*done*); ```

Instead, the runtime can execute another task.

When the request becomes ready:

```text
event
    │
    ▼
waker
    │
    ▼
runtime
    │
    ▼
poll task
    │
    ▼
Ready(response)
    │
    ▼
println!(*done*)
```

### Important

`.await` does **not** mean:

> *Go to the next line immediately.*

It means:

> "If this future cannot currently make progress, suspend this task and let the runtime run something else."

---

# 9. Does `.await` Block an OS Thread?

Normally, no.

This:

```rust some_future().await; ```

suspends the **async task** when the future is pending.

It does not necessarily block the OS thread. This is one of the main benefits of async.

---

# 10. What Is a Waker?

A future that returns `Pending` must arrange for the runtime to know when it should be polled again.

This is what the `Waker` is for.

```text
Future
    │
    ├── Pending
    │
    └── registers Waker
    │
    ▼
    event happens
    │
    ▼
    Waker
    │
    ▼
    Runtime
    │
    ▼
    poll future
```

Examples of events:

- network data arrives
- socket becomes writable
- timer expires
- channel receives a message

The waker does not execute the future itself.

It tells the runtime:

> *This future may be ready to make progress again.*

---

# 11. Runtime + Future + Waker

The complete relationship:

```text
    Runtime
    │
    │ poll
    ▼
    Future
    │
    ┌──────┴──────┐
    │             │
    Ready          Pending
    │
    ▼
    Waker
    │
    │ event
    ▼
    Runtime
    │
    │ poll again
    ▼
    Future
```

This is the core mechanism behind async Rust.

---

# 12. `block_on`

`block_on` is commonly used to start an async computation from synchronous code.

```rust
trpl::block_on(async {
    some_future().await;
});
```

`block_on` blocks the **current OS thread** until the future completes.

Conceptually:

```text
OS thread
    │
    ▼
block_on()
    │
    ▼
runtime drives future
    │
    ├── Pending
    │      │
    │      └── runtime continues driving async work
    │
    └── Ready
    │
    ▼
    block_on returns
```

So remember:

```text
.await      → suspends task block_on    → blocks OS thread
```

---

# 13. Async Does Not Automatically Create Threads

This:

```rust
async fn work() {
    // ...
}
```

does not create an OS thread.

Neither does:

```rust some_future().await; ```

The runtime decides how tasks are scheduled onto OS threads.

A runtime may use:

```text
1 OS thread
    ├── Task A
    ├── Task B
    └── Task C
```

or:

```text
4 OS threads
    ├── Task A
    ├── Task B
    ├── Task C
    ├── Task D
    └── ...
```

---

# 14. Async vs CPU-Bound Work

Async is excellent when tasks spend time **waiting**.

For example:

```text
**HTTP** request
    │
    ▼
waiting...
```

While waiting, the runtime can run other tasks.

But consider:

```rust
async fn calculate() {
    expensive_cpu_work();
}
```

If `expensive_cpu_work()` takes 10 seconds and never reaches an `.await`, the task can occupy the runtime's OS thread for those 10 seconds.

Therefore:

> Async is mainly useful for I/O-bound/concurrent workloads, not for making **CPU**-bound work faster.

**CPU**-bound work benefits from parallelism:

```text
**CPU** work
    │
    ├──> **CPU** 1
    ├──> **CPU** 2
    ├──> **CPU** 3
    └──> **CPU** 4
```

---

# 15. Async and Parallelism Can Be Combined

A real application can use both.

```text
Application
    │
    ├───────────────┐
    │               │
 Async I/O        **CPU** work
     │               │
 async tasks      thread pool
    │               │
    ▼               ▼
Concurrency      Parallelism
```

For example:

```text
Async task
    │
    ├── **HTTP** request
    │       │
    │       └── await
    │
    └── **CPU**-heavy operation
    │
    └── worker thread
```

---

# 16. Why Async Is Useful

Suppose we have:

```text
10,**000** network requests
```

A thread-per-request approach could require thousands of OS threads.

With async:

```text
10,**000** async tasks
    │
    ▼
    async runtime
    │
    ▼
    few OS threads
```

Most tasks spend their time waiting for I/O.

When one task waits:

```text
Task A → network → Pending
```

the runtime can run:

```tex
Task B → database Task C → socket Task D → timer
```

This allows many concurrent operations without requiring one OS thread per operation.

---

# 19. `Future` vs `Stream`

A `Future` produces one result:

```text
Future
    │
    ▼
value
```

A `Stream` produces multiple values over time:

```text
Stream
    │
    ├── value 1
    ├── value 2
    ├── value 3
    └── ...
```

For example:

```rust
while let Some(value) = stream.next().await {
    println!(*{value}*);
}
```

Conceptually:

```text
next().await
    │
    ▼
 value 1

next().await
    │
    ▼
 value 2

next().await
    │
    ▼
 value 3
```

A stream is therefore like an asynchronous iterator.

---

# 21. Multiple Futures

Every `async` block can produce a different anonymous future type.

For example:

```rust
let a = async {
    println!(*A*);
};

let b = async {
    println!(*B*);
};
```

Even though both have:

```text
Output = ()
```

their concrete future types are different.

Conceptually:

```text
async block A → FutureTypeA async block B → FutureTypeB
```

Therefore, you cannot simply put them into:

```rust Vec<Future> ```

because `Future` is a trait, not one concrete type.

---

# 22. Trait Objects for Different Futures

You can use a trait object:

```rust Vec<Box<dyn Future<Output = ()>>> ```

This means:

> A collection containing different concrete types, as long as each implements `Future<Output = ()>`.

Conceptually:

```text
Vec │ ├── Box<FutureTypeA> ├── Box<FutureTypeB> └── Box<FutureTypeC>
```

The `Box` gives each value a stable representation on the heap, while the trait object provides dynamic dispatch.

---

# 23. Why Pinning Appears with Futures

You may encounter:

```text
dyn Future<Output = ()> cannot be unpinned
```

This happens because some futures are not `Unpin`.

A future may contain state that must remain at a stable memory location while it is being polled.

`Pin<T>` guarantees that the pinned value will not be moved in memory.

```text
Without Pin:

Future
    │
    ├── memory location A
    │
    └── can move to B

With Pin:

Future
    │
    └── fixed memory location
```

This matters because an async future can contain references to its own internal state.

---

# 25. `Unpin`

`Unpin` means that a value can be safely moved even when it is behind a `Pin`.

Most ordinary Rust types are `Unpin`.

Some futures are not.

```text
Future
    │
    ├── Unpin
    │      └── can be moved
    │
    └── !Unpin
    └── must be pinned before certain operations
```

Remember:

> `Pin` is about preventing movement.

> `Unpin` means movement is safe.

---

# 26. `join_all`

When we have multiple futures and want to wait for all of them:

```rust
trpl::join_all(futures).await;
```

conceptually:

```text
Future A ──┐ Future B ──┼──> join_all ──> await Future C ──┘
```

`join_all` creates a new future that completes when all the futures in the collection complete.

This is useful when multiple independent operations can progress concurrently.

---

# 27. Sequential vs Concurrent Async

This:

```rust
let a = request_a().await; let b = request_b().await;
```

is sequential:

```text
request A
    │
    ▼
wait
    │
    ▼
finish
    │
    ▼
request B
    │
    ▼
wait
```

The second request does not start until the first has completed.

If the operations are independent, we can create both futures first and drive them together.

Conceptually:

```text
request A ────────┐
                  ├──> join
request B ────────┘
```

This allows them to make progress concurrently.

---

# 28. Creating a Future Is Not the Same as Running It

This distinction is important:

```rust
let future = request();
```

creates a future.

It does not mean:

```text
request is already running independently
```

The future needs to be polled.

```text
create future
    │
    ▼
Future exists
    │
    ▼
runtime polls it
    │
    ▼
future makes progress
```

This is different from spawning a task.

---

# 29. Future vs Task

A `Future` is a value representing asynchronous computation.

A **task** is a future that has been scheduled by a runtime for execution.

Conceptually:

```text
Future
    │
    │ spawn
    ▼
Task
    │
    ▼
Runtime schedules it
```

This distinction explains why simply creating several futures does not necessarily make them run concurrently.

---

# 30. One Task vs Multiple Tasks

With one task:

```text
Runtime
    │
    └── Task A
```

If Task A waits:

```text
Task A
    │
    ▼
Pending
```

there may be nothing else for the runtime to run.

With multiple tasks:

```text
Runtime
    │
    ├── Task A → Pending
    ├── Task B → Running
    └── Task C → Ready
```

the runtime can make progress elsewhere.

This is why async concurrency becomes useful when there are multiple independent operations.

---

# 31. Async Is Cooperative

Async runtimes generally rely on tasks yielding at appropriate points.

For example:

```rust
async {
    request().await;
}
```

can yield at `.await`.

But:

```rust
async {
    expensive_cpu_loop();
}
```

does not automatically yield just because it is inside an `async` block.

Therefore:

> Async tasks must avoid long-running synchronous work on runtime worker threads.

---

# 32. Blocking Inside Async Code

This is dangerous:

```rust
async fn work() {
    std::thread::sleep(Duration::from_secs(10));
}
```

`thread::sleep()` blocks the OS thread.

It does not asynchronously suspend the task.

Prefer an async timer:

```rust
async fn work() {
    tokio::time::sleep(Duration::from_secs(10)).await;
}
```

The async timer can return `Pending`, allowing the runtime to run other tasks.

```text
thread::sleep()
    │
    └── blocks OS thread

async sleep().await
    │
    └── suspends task
```

---

# 33. Async vs Threads

Use threads when:

- work is **CPU**-heavy
- you need parallel execution
- you need blocking operations
- the number of concurrent operations is relatively small

Use async when:

- there are many concurrent operations
- operations spend significant time waiting
- the workload is I/O-heavy
- you want efficient task scheduling

Often, real systems use both.

```text
    Application
    │
    ┌───────────┴───────────┐
    │                       │
    Async I/O              **CPU** work
    │                       │
    Runtime                 Threads
    │                       │
    Tasks                  Parallelism
```

---

# 34. The Complete Mental Model

```text
    Async Program
    │
    ▼
    async function
    │
    ▼
    Future
    │
    ▼
    Runtime polls
    │
    ┌─────────┴─────────┐
    │                   │
    Ready              Pending
    │                   │
    │                   ▼
    │                 Waker
    │                   │
    │             event happens
    │                   │
    │                   ▼
    │                Runtime
    │                   │
    │                   ▼
    │              poll again
    │
    ▼
    Result
```

And when there are multiple tasks:

```text
    Runtime
    │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
    Task A          Task B          Task C
    │              │              │
    await           await           **CPU** work
    │              │
    Pending         Pending
    │              │
    └──────┬───────┘
    │
    Wakers
    │
    ▼
    Runtime
    │
    ▼
    poll when ready
```

# 35. Key Takeaways

- `async fn` creates a **Future**.
- A future is a value representing an asynchronous computation.
- Creating a future does not necessarily execute it.
- A runtime drives futures by repeatedly calling `poll()`.
- `poll()` returns either `Pending` or `Ready(value)`.
- `Pending` means the future cannot currently make progress.
- `.await` suspends the current async task when the future is pending.
- `.await` does not normally block the OS thread.
- A `Waker` tells the runtime when a pending future may be ready again.
- An async runtime schedules tasks onto OS threads.
- Async provides **concurrency**, not automatic parallelism.
- Async is especially useful for I/O-bound workloads.
- **CPU**-bound work usually needs threads/parallelism.
- `block_on` blocks the current OS thread while driving a future.
- `Future` produces one result; `Stream` produces multiple results over time.
- Creating multiple futures does not automatically mean they execute concurrently.
- To execute independent futures concurrently, they must be driven together, for example with `join!`, `join_all`, or spawned tasks.
- `async` code can contain blocking operations; `async` does not magically make them non-blocking.
- `thread::sleep()` blocks an OS thread; an async timer with `.await` suspends a task.
- Futures may need to be pinned because some futures are `!Unpin`.
- `Pin` prevents a value from being moved; `Unpin` means moving it is safe.
- Async and threads are complementary: async handles many waiting operations efficiently, while threads provide parallelism for **CPU**-bound work.
