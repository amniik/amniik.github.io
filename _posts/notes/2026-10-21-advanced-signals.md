---
title: "Linux Signals: Advanced Features"
categories:
  - Learning Notes
  - System Programming
tags: [linux, tlpi, signals, signal-mask, realtime-signals, sigsuspend, signalfd, concurrency]
description: "A guide to advanced signal features in Unix."

toc: true
---

# Linux Signals: Advanced Features

This note covers Chapter 22 of *The Linux Programming Interface (TLPI)* and connects its concepts to signal handling, reentrancy, pending signals, and multithreaded processes.

## 1. Core Dump Files

A core dump is a snapshot of a process's memory and other diagnostic state at the time of a crash. It can be examined with a debugger such as GDB.

Some signals have a default action that terminates a process and may produce a core dump.

Examples include:
- `SIGSEGV`: invalid memory access.
- `SIGABRT`: abnormal process termination.
- `SIGQUIT`: termination, potentially with a core dump.
- `SIGFPE`: certain arithmetic exceptions.

Example:

```bash
ulimit -c unlimited
./my_program
```

If the program crashes and core dumps are enabled and permitted by the system, a core file may be produced. On systems using systemd, the dump may instead be collected by `systemd-coredump`.

Useful debugging command:

```bash
gdb ./my_program core
```

A core dump helps inspect the program's state at failure, but it is not guaranteed to be created for every fatal signal.

## 2. Signal Disposition, Blocking, and Pending Signals

These are three distinct concepts.

### Signal disposition

The disposition specifies what happens when a signal is delivered:

- `SIG_DFL`: perform the default action.
- `SIG_IGN`: ignore the signal.
- A handler function: execute the handler.

Dispositions are generally shared by all threads in a process.

### Blocking a signal

Blocking prevents a signal from being delivered to a thread while it remains blocked. The signal can remain pending.

Use `sigprocmask()` in a single-threaded program or `pthread_sigmask()` in a multithreaded program.

### Pending signals

A pending signal has been generated but has not yet been delivered or accepted through a synchronous signal-waiting mechanism.

Use `sigpending()` to inspect the pending signal set.

| Concept | Meaning |
|---|---|
| Disposition | What action should happen when the signal is delivered? |
| Blocked | Is delivery currently prevented by the thread's signal mask? |
| Pending | Is the signal awaiting delivery or acceptance? |

A signal can be both blocked and pending. A signal may also remain pending briefly after it is unblocked, until it is delivered.

For ordinary standard signals, multiple occurrences of the same signal may coalesce into one pending indication. Real-time signals have different queuing semantics.

### Ignoring vs. blocking

Ignoring changes the disposition; blocking changes the signal mask.

For a typical ordinary signal, an occurrence generated while its disposition is `SIG_IGN` is discarded rather than kept for later delivery. By contrast, a blocked signal can remain pending until it becomes eligible for delivery.

`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.

## 3. Interruptible and Uninterruptible Sleep States

Linux tasks can sleep while waiting for an event.

- `TASK_INTERRUPTIBLE`: a task can generally be awakened by an eligible signal.
- `TASK_UNINTERRUPTIBLE`: a task generally does not wake immediately just because a signal is pending.

A process shown as `D` in `ps` is typically in uninterruptible sleep, often waiting for I/O or another kernel event.

This does not mean that signals disappear. A signal can remain pending while the task waits for the condition that allows it to resume.

Do not confuse an uninterruptible sleep state with a blocked signal: one describes the task's kernel execution state; the other describes signal delivery policy.

## 4. Hardware-Generated Signals

Some processor exceptions can result in signals being delivered to the process.

Examples:
- `SIGSEGV`: certain invalid memory accesses.
- `SIGFPE`: certain arithmetic exceptions.
- `SIGILL`: an illegal instruction.
- `SIGBUS`: certain invalid memory or bus-access conditions.

The exact signal depends on the fault and architecture.

For example:

```c
int *p = NULL;
*p = 42;
```

This may cause a `SIGSEGV`.

A synchronous hardware-generated signal is associated with the execution of the thread that caused the fault. It is not equivalent to a terminal-generated signal such as `SIGINT`.

Continuing execution after a hardware fault is generally unsafe unless the program has a carefully designed recovery mechanism.

## 5. Synchronous vs. Asynchronous Signal Generation

### Synchronous signals

These are associated with a particular operation or event in the executing thread.

Examples include certain hardware exceptions that produce `SIGSEGV`, `SIGFPE`, or `SIGILL`.

### Asynchronous signals

These arise independently of the current instruction being executed.

Examples include:
- `SIGINT` from terminal input.
- `SIGTERM` sent by another process.
- `SIGCHLD` following a child-process state change.
- A timer signal.

This distinction matters because an asynchronous signal may interrupt execution at almost any point. A handler must not assume that interrupted library code is in a safe state.

This connects directly to reentrancy and async-signal-safety from Chapter 21.

## 6. Timing and Order of Signal Delivery

Signal generation and signal delivery are separate events.

A signal can be generated, become pending, and then be delivered later when it is eligible.

Important rules:

1. A blocked signal generally remains pending.
2. Unblocking a pending signal makes it eligible for delivery.
3. Standard signals do not necessarily preserve every occurrence; multiple instances may coalesce.
4. Real-time signals are queued, subject to system limits.
5. Do not assume a universal delivery order among different pending standard signals.

In multithreaded programs, process-directed signals can be delivered to an eligible thread chosen by the kernel. Thread-directed signals target a particular thread.

## 7. `signal()` and Portability

The historical `signal()` interface has portability and semantic differences across older systems.

For new POSIX code, prefer `sigaction()` because it provides explicit control over:
- The handler or disposition.
- Additional signals blocked while the handler executes.
- Flags such as `SA_RESTART` and `SA_SIGINFO`.

Example:

```c
struct sigaction sa = {0};

sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;

if (sigaction(SIGINT, &sa, NULL) == -1) {
    /* Handle error */
}
```

The behavior of `SA_RESTART` is not universal across every blocking operation. Always check the documentation for the specific operation.

## 8. Real-Time Signals

Standard signals have limited notification semantics: repeated occurrences can coalesce, and they do not form a reliable message queue.

Real-time signals provide stronger queuing semantics and can carry a small amount of associated data.

### Sending with `sigqueue()`

```c
#include <signal.h>

union sigval value;
value.sival_int = 42;

if (sigqueue(target_pid, SIGRTMIN, value) == -1) {
    /* Handle error */
}
```

A handler installed with `SA_SIGINFO` can receive the associated value through `siginfo_t`.

Conceptually:

```c
void handler(int sig, siginfo_t *info, void *context) {
    int value = info->si_value.sival_int;
}
```

The real-time signal range is available through `SIGRTMIN` and `SIGRTMAX`; do not hard-code assumed numeric values.

Real-time signals are useful for queued notifications, but they are not a replacement for a general-purpose message queue. Queuing is subject to resource limits, and sending can fail.

## 9. `sigsuspend()`: Waiting Without a Race Condition

This is one of the chapter's most important topics.

Consider this incorrect pattern:

```c
if (!got_signal)
    pause();
```

A race can occur:

1. The program checks `got_signal`; it is false.
2. A signal arrives and the handler sets `got_signal = 1`.
3. The handler returns.
4. The program calls `pause()`.
5. The program may sleep indefinitely because the signal has already been handled.

The problem is that checking the flag and going to sleep are separate operations.

### How `sigsuspend()` solves it

`sigsuspend()` temporarily installs a specified signal mask and suspends the calling thread until a signal is caught and its handler returns, or another applicable event interrupts the wait.

The mask change and suspension are atomic with respect to signal delivery, which prevents the gap between unblocking a signal and going to sleep.

A typical pattern is:

1. Block the signal before checking the condition.
2. Check the flag while the signal is blocked.
3. Call `sigsuspend()` with a mask that unblocks the signal.
4. After the handler runs, resume and check the condition again.

Conceptual example:

```c
sigset_t block_set, old_set, wait_set;
volatile sig_atomic_t got_signal = 0;

/* Initialize the handler separately. */

sigemptyset(&block_set);
sigaddset(&block_set, SIGUSR1);

/* Block SIGUSR1 and save the old mask. */
sigprocmask(SIG_BLOCK, &block_set, &old_set);

/* Prepare the mask used while waiting. */
wait_set = old_set;
sigdelset(&wait_set, SIGUSR1);

while (!got_signal) {
    sigsuspend(&wait_set);
}

/* Restore the previous mask. */
sigprocmask(SIG_SETMASK, &old_set, NULL);
```

This is a conceptual example. Production code must check system-call errors and ensure the chosen mask and handler setup match the application's signal policy.

`sigsuspend()` normally returns `-1` with `errno == EINTR` after a caught signal handler returns.

**Key idea:** `sigsuspend()` is not merely a sleep operation. Its atomic signal-mask change makes it a synchronization primitive.

## 10. Synchronous Signal Waiting

Instead of executing an asynchronous handler, a program can wait for signals as part of its normal control flow.

### `sigwait()`

Waits for a signal from a specified set and returns its signal number.

### `sigwaitinfo()`

Waits for a signal from a specified set and provides additional information about the signal.

### `sigtimedwait()`

Similar to `sigwaitinfo()`, but with a timeout.

These functions are particularly useful in multithreaded programs.

A common design is to block selected signals in all threads, then dedicate one thread to receive them synchronously. This keeps signal processing out of asynchronous handlers.

Signals intended for synchronous waiting should normally be blocked in the receiving thread before the wait begins.

Remember that `sigwait()` returns an error number directly, while `sigwaitinfo()` and `sigtimedwait()` return a signal number on success and `-1` on failure.

## 11. `signalfd()`: Receiving Signals Through a File Descriptor

Linux provides `signalfd()`, which exposes selected signals through a file descriptor.

This allows a program to receive signals using `read()` and monitor signal readiness with interfaces such as `poll()` and `epoll()`.

This is particularly useful in event-driven applications.

Conceptually:

```text
Signal generated
      |
      v
Signal becomes pending
      |
      v
signalfd becomes readable
      |
      v
Event loop detects readiness
      |
      v
read() obtains signal information
```

Typical setup:

1. Block the signals to be received through the descriptor.
2. Create a signal file descriptor with `signalfd()`.
3. Register the descriptor with the event loop.
4. Read signal records when the descriptor becomes readable.

Blocking the signals is important: it allows the program to receive them through the descriptor instead of having them delivered through their ordinary asynchronous dispositions.

`signalfd()` is Linux-specific, unlike POSIX interfaces such as `sigaction()` and `sigsuspend()`.

## 12. Signals as Interprocess Communication

Signals can notify one process that something has happened in another process.

Examples:
- `SIGTERM` requests termination.
- `SIGCHLD` notifies a parent about child-process state changes.
- Real-time signals can carry an integer or pointer-sized value through `sigqueue()`.

However, signals are limited as an IPC mechanism:
- Standard signals may coalesce.
- Payload capacity is small.
- Signals are asynchronous.
- Queuing is resource-limited.

For substantial data transfer, use pipes, sockets, shared memory, or message queues. Signals are generally better for notification than for carrying large amounts of data.

## 13. Signal Handling in Multithreaded Processes

Three concepts are especially important.

### Shared dispositions, per-thread masks

- Signal dispositions are shared by the process.
- Each thread has its own signal mask.
- A process-directed signal can be delivered to one eligible thread.
- A thread-directed signal targets a particular thread.

For example, `kill(pid, SIGTERM)` sends a process-directed signal, while `pthread_kill(thread, SIGUSR1)` targets a particular thread.

### Pending signals

A thread-directed signal can be pending for its target thread. A process-directed signal can be pending for the process if it has not yet been delivered to an eligible thread.

If one thread blocks a process-directed signal, the kernel may deliver it to another eligible thread. If every thread blocks it, the signal may remain process-pending.

### A common architecture

Block selected signals before creating worker threads. Then create a dedicated signal-handling thread that waits using `sigwaitinfo()` or `sigwait()`.

This is usually easier to reason about than allowing arbitrary asynchronous handlers to perform complex operations.

## 14. Connection to Reentrancy and Async-Signal-Safety

These concepts from Chapter 21 explain why the synchronous approaches in this chapter are valuable.

### Reentrancy

A function is reentrant if it can be entered again before a previous invocation finishes without unsafe interference.

### Thread safety

A thread-safe function behaves correctly when multiple threads use it concurrently. Locks can provide thread safety, but those same locks can cause problems if a signal handler interrupts code that holds one.

### Async-signal-safety

An async-signal-safe function is safe to call from a signal handler under the applicable POSIX requirements.

For example, `printf()` is not async-signal-safe, even though typical implementations protect stdio operations for concurrent threads.

Use the POSIX async-signal-safe function list rather than assuming that thread safety implies signal safety.

### `volatile sig_atomic_t`

For basic communication between a handler and normal execution in a single-threaded program, a `volatile sig_atomic_t` flag is a common pattern.

```c
static volatile sig_atomic_t got_signal = 0;

void handler(int sig) {
    got_signal = 1;
}
```

This supports simple flag assignment and inspection. It does not make compound operations atomic, and it is not a general synchronization mechanism between threads.

The broader lesson is to minimize asynchronous handler work and use synchronous signal reception where it fits the design.

## 15. Practical Rules

1. Distinguish a signal's disposition, its blocked state, and its pending state.
2. Remember that standard signals can coalesce, while real-time signals provide queuing semantics.
3. Prefer `sigaction()` over historical `signal()` for explicit signal handling.
4. Use `sigsuspend()` to avoid the check-then-sleep race when waiting for a signal.
5. Consider `sigwaitinfo()` or `signalfd()` when synchronous signal handling is a better fit.
6. Keep asynchronous handlers minimal and use only documented async-signal-safe functions.
7. Understand which system calls may be interrupted and when `EINTR` or automatic restart is possible.
8. In multithreaded programs, manage signal masks deliberately and distinguish process-directed from thread-directed signals.
9. Use signals mainly for notification, not bulk data transfer.

## 16. Connection to a Unix Shell

These concepts are relevant to implementing a shell such as `rustsh`.

- `SIGINT` and `SIGTSTP` are central to foreground job control.
- The terminal normally sends keyboard-generated signals to the foreground process group, not necessarily to one process.
- The shell needs to wait for child processes and handle interruptions correctly.
- `sigsuspend()` illustrates how to avoid race conditions when synchronizing with signals.
- `signalfd()` demonstrates a Linux-specific way to integrate signals into an event loop.
- `SIGCHLD` is a notification that a child changed state; the shell must still use `waitpid()` to obtain the status and reap terminated children.

For a Rust implementation, use a signal-handling design or library that respects the restrictions of asynchronous handlers rather than running arbitrary Rust code inside a C-style signal handler.

## Summary

Chapter 22 moves beyond basic handlers toward advanced signal delivery, queued notifications, race-free waiting, and integration with event loops.

The most important mental model is:

**Signals are notifications with complex delivery rules.** Signal masks control eligibility for delivery, pending sets record signals awaiting delivery or acceptance, and synchronous interfaces can make signal processing easier to coordinate with normal program execution.
