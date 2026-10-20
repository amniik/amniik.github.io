---
title: "Linux Signal Handlers: Reentrancy and Async-Signal-Safety"
date: 2026-10-20
categories: [Linux, Systems Programming]
tags: [linux, tlpi, signals, c, reentrancy, async-signal-safety, concurrency]
---

# Linux Signal Handlers: Reentrancy and Async-Signal-Safety

This note covers the main concepts in Chapter 21 of *The Linux Programming Interface (TLPI)*, including safe signal-handler design, reentrant functions, `sig_atomic_t`, signal information, alternate signal stacks, and interrupted system calls.

## 1. Designing Signal Handlers

A signal handler executes asynchronously relative to the program's normal flow. A signal can interrupt normal execution while a library function or data structure is in an intermediate state.

For example:

```c
void handler(int sig) {
    printf("Signal received\n");
}

int main(void) {
    signal(SIGINT, handler);

    while (1) {
        printf("Working...\n");
    }
}
```

If `SIGINT` interrupts `printf()` and the handler calls `printf()` again, the second invocation may deadlock or interfere with internal stdio state.

**Design principle:** Keep signal handlers minimal. Prefer setting a flag, writing to a suitable file descriptor, or using another documented signal-safe mechanism. Perform complex work in normal execution.

### Standard signals are not queued

Multiple occurrences of the same standard signal may coalesce into one pending signal. A signal handler should not assume that it will run once for every occurrence.

For reliable event counting or data transfer, consider a mechanism designed for that purpose rather than using ordinary signals as a queue.

## 2. Reentrancy, Thread Safety, and Async-Signal-Safety

These concepts are related but have different meanings.

### Reentrant functions

A function is reentrant when it can be entered again before a previous invocation finishes without corrupting shared state or producing incorrect results.

For example:

```c
int add(int a, int b) {
    return a + b;
}
```

This function uses only its arguments and local computation. It does not modify shared state, so overlapping invocations do not interfere with one another.

A function may be non-reentrant if it relies on:
- Shared mutable global or static state.
- A static buffer overwritten by later calls.
- Internal state that can be observed in an inconsistent intermediate condition.
- Other functions that are unsafe when re-entered.

### Thread-safe functions

A function is thread-safe when concurrent calls from multiple threads behave correctly.

Thread safety can be implemented using synchronization, such as mutexes or atomic operations.

However, a mutex creates a problem if a signal interrupts code that already holds the mutex and the handler attempts to acquire the same mutex.

Therefore:

**Thread-safe does not necessarily mean reentrant or async-signal-safe.**

### Async-signal-safe functions

An async-signal-safe function can safely be called from a signal handler under the applicable POSIX guarantees, including when the signal interrupts normal execution.

POSIX specifies which functions are async-signal-safe. Examples include:

- `write()`
- `read()`
- `close()`
- `kill()`
- `_exit()`

Common functions that are not async-signal-safe include:

- `printf()` and other stdio functions
- `malloc()` and `free()`
- `crypt()`
- `exit()`

Always consult `signal-safety(7)` or the applicable POSIX documentation rather than assuming that a function is safe.

### Relationship between the three properties

| Property | Main concern |
|---|---|
| Reentrancy | Safe re-entry before an earlier invocation finishes |
| Thread safety | Safe concurrent use by multiple threads |
| Async-signal-safety | Safe invocation from a signal handler |

A function can be thread-safe because it uses locks, yet still be unsafe in a signal handler. Reentrancy alone also does not establish that a function meets the POSIX async-signal-safe requirements.

A function may be async-signal-safe because it is reentrant or because its operation is guaranteed to be safe against signal interruption. The POSIX classification is the practical rule to follow.

## 3. Why `printf()` Is Thread-Safe but Not Async-Signal-Safe

Standard C library implementations commonly protect stdio streams with locks, allowing concurrent threads to use `printf()` safely.

But consider this sequence:

1. Normal execution calls `printf()` and acquires the stdout lock.
2. A signal interrupts the call.
3. The handler calls `printf()` again.
4. The handler attempts to acquire the lock held by the interrupted invocation.
5. The interrupted invocation cannot release the lock until the handler returns.

This can deadlock.

The lock that protects `printf()` against concurrent threads can therefore make it unsafe to call from an asynchronous signal handler.

For simple output in a handler, `write()` is the appropriate POSIX async-signal-safe alternative:

```c
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    const char msg[] = "Signal received\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}
```

`write()` may write fewer bytes than requested or fail, so robust code should account for its return value when necessary.

## 4. Why `crypt()` Is Non-Reentrant

Some traditional implementations of `crypt()` return a pointer to a shared static buffer.

```c
char *hash1 = crypt("password1", salt1);
char *hash2 = crypt("password2", salt2);
```

If both results refer to the same internal buffer, the second call overwrites the first result.

The first pointer still exists, but its contents may have changed. If a signal handler calls `crypt()` while normal execution is using the previous result, the two invocations can interfere.

Some systems provide `crypt_r()`, which accepts caller-provided state. This avoids that particular shared-buffer problem, but does not automatically make the function suitable for signal handlers.

## 5. Communicating with a Handler: `volatile sig_atomic_t`

A common pattern is to let the handler set a flag and have normal execution perform the actual work.

```c
#include <signal.h>

static volatile sig_atomic_t got_signal = 0;

static void handler(int sig) {
    got_signal = 1;
}

int main(void) {
    struct sigaction sa = {0};

    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    if (sigaction(SIGINT, &sa, NULL) == -1)
        return 1;

    while (!got_signal) {
        /* Normal program work */
    }

    /* Handle the signal in normal execution. */
    return 0;
}
```

### What does `sig_atomic_t` guarantee?

`sig_atomic_t` is an integer type suitable for atomic access in the signal-handler context specified by the C standard.

For a simple signal flag, an assignment such as:

```c
got_signal = 1;
```

is the intended pattern.

`volatile` ensures that accesses to the object are treated as observable accesses rather than simply optimized away as ordinary unchanging memory accesses.

However:

- `volatile` alone does not make an ordinary object atomic.
- `volatile sig_atomic_t` is not a general thread-synchronization primitive.
- An operation such as `got_signal++` is not guaranteed to be an atomic read-modify-write operation.
- It does not make complex structures safe to share with a signal handler.

For multithreaded communication, use appropriate synchronization and signal-handling designs.

### Why might an ordinary integer be insufficient?

A normal integer assignment may happen to compile into one machine instruction on a particular architecture. That is not the same as having the required language-level guarantee for signal-handler communication.

Use `volatile sig_atomic_t` for the basic single-threaded flag pattern instead of relying on assumptions about machine instructions.

## 6. Interrupted System Calls: `EINTR` and `SA_RESTART`

A signal can interrupt a blocking system call.

Depending on the operation and circumstances, the call may:
- Complete normally.
- Return `-1` with `errno == EINTR`.
- Be automatically restarted under applicable conditions.

For example:

```c
ssize_t n = read(fd, buffer, sizeof(buffer));

if (n == -1) {
    if (errno == EINTR) {
        /* Decide whether to retry or stop. */
    } else {
        /* Handle another error. */
    }
}
```

`SA_RESTART` is a flag that can be used with `sigaction()` to request automatic restarting of certain interrupted system calls.

It does not restart every operation in every situation. Check the documentation for the particular system call.

When writing loops that retry after `EINTR`, consider whether the application should continue waiting or respond to the signal.

## 7. Extended Signal Handling

### `sigaction()` and `SA_SIGINFO`

Setting `SA_SIGINFO` allows a handler to receive additional information about the signal.

```c
void handler(int sig, siginfo_t *info, void *ucontext);
```

The `siginfo_t` structure can contain information such as:
- `si_signo`: signal number.
- `si_pid`: sender's process ID for applicable signals.
- `si_uid`: sender's user ID for applicable signals.
- `si_code`: information about the origin or reason for the signal.

The availability and meaning of fields depend on the signal and how it was generated.

### Alternate signal stack: `sigaltstack()`

A process can configure an alternate stack for signal handling. With the appropriate `sigaction()` flags, a handler can run on that stack instead of the regular thread stack.

This can be useful for handling certain signals, such as stack-overflow-related `SIGSEGV`, when the normal stack is exhausted.

An alternate stack does not make unsafe functions safe to call from the handler.

### Nonlocal jumps and abnormal termination

Chapter 21 also discusses:
- `sigsetjmp()` and `siglongjmp()` for nonlocal control flow from a signal handler.
- `abort()` for abnormal process termination.

These are specialized techniques. Nonlocal jumps from handlers require careful attention to signal masks and the state of interrupted functions. They should not be used as a general substitute for simple handler design.

## 8. Signals in Multithreaded Processes

A multithreaded process has shared signal dispositions but a separate signal mask for each thread.

- **Process-directed signal:** For example, `kill(pid, SIGTERM)`. The kernel can deliver it to an eligible thread that has not blocked the signal.
- **Thread-directed signal:** For example, `pthread_kill(thread, SIGUSR1)`. The signal targets the specified thread.
- **Signal disposition:** The process-wide action for a signal, such as its default action, a handler, or ignoring it.
- **Signal mask:** The set of signals blocked by a particular thread.

A common design is to block selected signals in all threads and dedicate one thread to receive them synchronously with `sigwait()`. This lets the program handle signals in normal thread execution rather than doing complex work inside an asynchronous handler.

`volatile sig_atomic_t` should not be treated as a replacement for synchronization between threads.

## 9. Practical Rules

1. Keep signal handlers small and predictable.
2. Use only documented async-signal-safe functions in asynchronous handlers.
3. Use `volatile sig_atomic_t` for a basic flag shared between a handler and normal execution.
4. Do not use `printf()`, `malloc()`, or ordinary mutex operations in an asynchronous signal handler.
5. Remember that standard signals may coalesce rather than queue every occurrence.
6. Handle `EINTR` where relevant and understand which calls can be restarted.
7. In multithreaded programs, distinguish process-directed signals from thread-directed signals.
8. For complex signal handling, consider synchronous signal waiting or a file-descriptor-based design.

## Key Takeaways

- Reentrancy, thread safety, and async-signal-safety are related but distinct.
- A function protected by a mutex may be thread-safe but unsafe when called from a signal handler.
- `volatile sig_atomic_t` is appropriate for a simple signal flag, not general concurrent state.
- `SA_RESTART` affects some interrupted system calls, not all of them.
- Signal handlers should do minimal work and defer complex operations to normal execution.
