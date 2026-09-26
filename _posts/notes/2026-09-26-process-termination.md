---
title: "Linux Process Termination"
categories:
  - Learning Notes
  - System Programming
tags: [linux, process\]
description: "A practical guide to Linux Process Termination."

toc: true
---

# Linux Process Termination: `exit()`, `_exit()`, and Exit Handlers

A process can terminate normally or abnormally. This chapter focuses on normal termination and the differences between `_exit()` and `exit()`, exit handlers, exit status, and the interaction between `fork()` and stdio buffers.

## 1. Process Termination

A process can terminate normally by calling:

```c
_exit(status);
```

or:

```c
exit(status);
```

There is also abnormal termination, for example when a signal terminates the process.

The important difference is:

```text
_exit()
    ↓
terminate process

exit()
    ↓
run exit handlers
    ↓
flush stdio streams
    ↓
terminate process
```

`exit()` is a C library function, while `_exit()` is the low-level POSIX/Linux interface used to terminate the process.

---

# 2. `_exit()`

```c
#include <unistd.h>

void _exit(int status);
```

`_exit()` terminates the calling process without performing the stdio cleanup performed by `exit()`.

In particular, `_exit()`:

* does not call functions registered with `atexit()`
* does not call functions registered with `on_exit()`
* does not flush stdio streams

The process's file descriptors are still closed as part of process termination.

For example:

```c
printf("Hello\n");

_exit(EXIT_SUCCESS);
```

The `"Hello\n"` may never reach the output because it can still be sitting in the stdio buffer.

---

# 3. `exit()`

```c
#include <stdlib.h>

void exit(int status);
```

`exit()` performs normal C-library termination processing.

Conceptually:

```text
exit(status)
    │
    ├── call atexit()/on_exit() handlers
    │
    ├── flush stdio streams
    │
    └── terminate process
```

The exit handlers are called in reverse order of registration, and stdio streams are flushed before termination.

For example:

```c
printf("Hello\n");

exit(EXIT_SUCCESS);
```

The stdio buffer is flushed before the process terminates.

---

# 4. `exit()` vs `_exit()`

|                       | `exit()`                   | `_exit()`                                        |
| --------------------- | -------------------------- | ------------------------------------------------ |
| Type                  | C library function         | POSIX/Linux interface                            |
| `atexit()` handlers   | Yes                        | No                                               |
| `on_exit()` handlers  | Yes                        | No                                               |
| Flushes stdio streams | Yes                        | No                                               |
| Terminates process    | Yes                        | Yes                                              |
| Common use            | Normal program termination | Low-level termination, especially after `fork()` |

The key mental model is:

```text
exit()
  │
  ├── cleanup registered by the C library/application
  ├── flush stdio
  └── terminate

_exit()
  │
  └── terminate
```

---

# 5. Exit Status

Both functions receive an integer:

```c
exit(status);
_exit(status);
```

The status communicates information about how the process terminated to its parent.

By convention:

```text
0       → success
nonzero → failure
```

The standard constants are:

```c
EXIT_SUCCESS
EXIT_FAILURE
```

For the traditional `wait()`/`waitpid()` status, only the low 8 bits of the exit status are made available to the parent.

For example:

```c
exit(5);
```

The parent can later obtain the value `5` from the child's termination status.

```c
exit(-1);
```

The parent can later obtain the value `255` from the child's termination status.
-1 = 0xFFFFFFFF (two's complement) -> 255 = 0xFF

# 6. `return` from `main()`

Returning from `main()` is equivalent to calling `exit()` with the returned value.

For example:

```c
int main(void)
{
    return 0;
}
```

is effectively equivalent to:

```c
int main(void)
{
    exit(0);
}
```

Similarly:

```c
int main(void)
{
    return 5;
}
```

is equivalent to:

```c
int main(void)
{
    exit(5);
}
```

This is important because returning from `main()` performs the normal `exit()` processing, including stdio flushing and exit handlers.

---

# 7. Exit Handlers: `atexit()`

The C library provides:

```c
#include <stdlib.h>

int atexit(void (*function)(void));
```

It allows a program to register a function that should be called when the program terminates normally through `exit()`.

Example:

```c
void cleanup(void)
{
    printf("Cleaning up\n");
}

int main(void)
{
    atexit(cleanup);

    printf("Program running\n");

    exit(EXIT_SUCCESS);
}
```

Output:

```text
Program running
Cleaning up
```

The important idea is:

```text
atexit(cleanup)
       │
       ▼
register function
       │
       ...
       │
     exit()
       │
       ▼
cleanup()
```

---

# 8. Multiple `atexit()` Handlers

You can register multiple functions:

```c
atexit(func1);
atexit(func2);
atexit(func3);
```

They are called in **reverse order of registration**:

```text
func3()
func2()
func1()
```

This is similar to a stack:

```text
Registration:

func1
func2
func3
 ↓
exit()

Execution:

func3
func2
func1
```

This behavior is useful because resources are often released in the reverse order in which they were acquired.

---

# 9. `on_exit()`

Linux/glibc also provides:

```c
int on_exit(
    void (*function)(int, void *),
    void *arg
);
```

Unlike `atexit()`, the handler receives:

1. the exit status
2. an arbitrary argument supplied when registering the handler

Conceptually:

```text
on_exit(handler, argument)
             │
             ▼
          exit(42)
             │
             ▼
handler(42, argument)
```

`on_exit()` is a GNU/Linux-specific extension and is not as portable as `atexit()`.

---

# 10. Exit Handlers and `_exit()`

Exit handlers are executed by `exit()`, not `_exit()`.

For example:

```c
atexit(cleanup);

_exit(EXIT_SUCCESS);
```

`cleanup()` will **not** run.

But:

```c
atexit(cleanup);

exit(EXIT_SUCCESS);
```

will call:

```text
cleanup()
```

before termination.

Therefore:

```text
             process termination
                    │
          ┌─────────┴─────────┐
          │                   │
        exit()             _exit()
          │                   │
          ▼                   │
    exit handlers             │
          │                   │
          ▼                   │
    flush stdio               │
          │                   │
          └─────────┬─────────┘
                    ▼
               terminate
```

---

# 11. `fork()` and Stdio Buffers

This is one of the most important connections between Chapters 24 and 25.

Consider:

```c
printf("Hello\n");

fork();

exit(EXIT_SUCCESS);
```

Suppose stdout is redirected to a file:

```bash
./program > output.txt
```

stdout is normally block-buffered when connected to a file.

Therefore, before `fork()`:

```text
Parent

stdio buffer:
┌─────────────────┐
│ "Hello\n"       │
└─────────────────┘
```

The buffer is part of the process's user-space memory.

`fork()` creates a child with a copy of that memory:

```text
Parent                    Child

stdio buffer              stdio buffer
┌──────────────┐          ┌──────────────┐
│ "Hello\n"    │          │ "Hello\n"    │
└──────────────┘          └──────────────┘
```

Now both processes have their own copy of the buffered data.

If both call:

```c
exit(EXIT_SUCCESS);
```

both flush their buffers:

```text
Parent → "Hello\n"
Child  → "Hello\n"
```

So the file contains:

```text
Hello
Hello
```

The important lesson is:

> `fork()` duplicates user-space stdio buffers, not just your program's variables.

---

# 12. Why `write()` Behaves Differently

Compare:

```c
printf("Hello\n");
```

with:

```c
write(STDOUT_FILENO, "Hello\n", 6);
```

`printf()` uses the stdio layer:

```text
printf()
   ↓
stdio buffer
   ↓
write()
   ↓
kernel
```

Whereas `write()` directly enters the kernel:

```text
write()
   ↓
kernel
```

Therefore, if `write()` has already transferred its data before `fork()`, there isn't a copy of that data sitting in the user-space stdio buffer for the child to inherit.

This explains why:

```c
printf("Hello\n");
write(STDOUT_FILENO, "Ciao\n", 5);
fork();
exit(EXIT_SUCCESS);
```

can produce:

```text
Hello
Ciao
Hello
```

when stdout is redirected to a file.

---

# 13. Why `_exit()` Is Important After `fork()`

Consider the common shell pattern:

```text
shell
  │
  ├── fork()
  │
  ├── child
  │    │
  │    ├── dup2(...)
  │    ├── exec(...)
  │    │
  │    └── if exec fails
  │          _exit(...)
  │
  └── parent
```

The child inherits the parent's stdio buffers.

If `exec()` fails and the child does:

```c
exit(EXIT_FAILURE);
```

it can flush the buffers that were inherited from the parent.

That can result in duplicated output.

Therefore, after `fork()`, when the child needs to terminate without successfully executing the new program, `_exit()` is commonly used.

```c
if (execve(...) == -1)
    _exit(EXIT_FAILURE);
```

The reason is not simply " `_exit()` is faster."

The important reason is:

> **The child must not perform the parent's pending stdio cleanup.**

---

# 14. `fork()` + `exit()` Mental Model

The whole relationship can be summarized as:

```text
             Parent
               │
          stdio buffer
               │
             fork()
          ┌────┴────┐
          │         │
       Parent      Child
          │         │
       buffer      buffer
       copy A      copy B
          │         │
        exit()    exit()
          │         │
       flush A    flush B
          │         │
          └────┬────┘
               ▼
             output
```

But with `_exit()`:

```text
             Parent
               │
          stdio buffer
               │
             fork()
          ┌────┴────┐
          │         │
       Parent      Child
          │         │
       exit()     _exit()
          │         │
       flush       no flush
          │
          ▼
        output
```

---

# 15. What Actually Gets Cleaned Up?

When a process terminates, the kernel performs process-level cleanup, including closing the process's open file descriptors and releasing the process's memory and other kernel-managed resources.

`exit()` adds C-library-level cleanup such as:

```text
exit()
 ├── atexit()/on_exit() handlers
 ├── flush stdio streams
 └── terminate
```

It is useful to distinguish:

```text
C library cleanup
        │
        ▼
     exit()
        │
        ▼
kernel/process cleanup
        │
        ▼
   process gone
```

You therefore should not think of `exit()` as simply "the syscall that kills the process." It is a library-level termination procedure that eventually causes process termination.

---

# 16. Important Mental Model

The main concepts from this chapter fit together like this:

```text
                    Process
                       │
             ┌─────────┴─────────┐
             │                   │
           exit()             _exit()
             │                   │
             ▼                   │
      exit handlers              │
             │                   │
             ▼                   │
       flush stdio               │
             │                   │
             └─────────┬─────────┘
                       ▼
                 process terminates
```

And with `fork()`:

```text
                 fork()
                   │
          ┌────────┴────────┐
          │                 │
       Parent             Child
          │                 │
    stdio buffer      copied stdio buffer
          │                 │
       exit()             _exit()
          │                 │
       flush              no flush
          │                 │
          └───────┬─────────┘
                  ▼
               output
```

## Key Takeaways

* `exit()` is a **C library termination function**.
* `_exit()` is the low-level termination interface.
* `exit()` calls registered exit handlers and flushes stdio streams.
* `_exit()` does not perform those stdio/exit-handler actions.
* `return` from `main()` is equivalent to calling `exit()` with the returned value.
* `atexit()` registers cleanup functions.
* `atexit()` handlers execute in reverse registration order.
* `on_exit()` is a Linux/glibc-specific mechanism that also provides the exit status and an argument.
* The exit status is used to communicate termination information to the parent.
* `fork()` duplicates user-space stdio buffers.
* Calling `exit()` in both parent and child can therefore flush the same inherited buffered output twice.
* After `fork()`, `_exit()` is commonly used in a child when termination should happen without flushing inherited stdio buffers.
* `write()` is different from `printf()` because `write()` does not use the C stdio buffering layer.
