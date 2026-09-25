---
title: "Linux Process Creation: fork(), vfork(), and Copy-on-Write"
categories:
  - Learning Notes
  - System Programming
tags: [linux, process, fork, vfork, copy-on-write, file-descriptors]
description: "A practical guide to Linux Process Creation."

toc: true
---

# Linux Process Creation

Linux provides mechanisms for creating new processes. The most important traditional mechanism is `fork()`, which creates a new process from an existing process.

## 1. `fork()`

`fork()` creates a new process called the **child** from the calling process, which becomes the **parent**.

```text
                fork()
                  |
          ┌───────┴───────┐
          │               │
       Parent           Child
       PID = P          PID = C
```

`fork()` returns:

* **Child PID** to the parent.
* **0** to the child.
* **-1** to the parent if creation fails.

After `fork()`, both processes continue execution from the instruction following the `fork()` call.

```text
Parent:
    fork()
      |
      └── continues

Child:
    fork()
      |
      └── continues
```

The child initially has a logical copy of the parent's process state.

## 2. Memory After `fork()`

The child receives its own virtual address space containing the same memory contents as the parent at the time of `fork()`.

```text
Before fork:

Parent
  |
  └── Virtual address space
          |
          v
       Physical memory
```

After `fork()`:

```text
Parent ──→ Virtual address space ──┐
                                   ├──→ Physical pages
Child  ──→ Virtual address space ──┘
```

The parent and child have **separate virtual address spaces**.

Therefore, if the child changes one of its variables, the parent's corresponding variable does not normally change.

## 3. Copy-on-Write

Immediately copying all of the parent's physical memory would be expensive.

Linux therefore uses **copy-on-write (COW)**.

Initially, parent and child can map the same physical pages:

```text
Parent ──┐
         ├──→ Physical page
Child  ──┘
```

The kernel marks the shared pages so that a write causes a page fault.

If the child modifies a page:

```text
Before:

Parent ──┐
         ├──→ Page A
Child  ──┘


After child writes:

Parent ─────→ Page A

Child  ─────→ Page B
```

The kernel creates a private copy for the child.

This means `fork()` can be relatively inexpensive even when the parent has a large address space.

### Why COW is particularly useful

A common pattern is:

```text
fork()
  |
  └── child
        |
        └── exec()
```

If the child immediately executes another program, copying all of the parent's memory would be unnecessary. COW avoids most of that work.

## 4. File Descriptors After `fork()`

The child inherits copies of the parent's file descriptor table.

However, the parent and child descriptors refer to the **same open file descriptions (OFDs)**.

```text
Parent FD 3 ──┐
              ├──→ Open File Description ──→ File
Child  FD 3 ──┘
```

Therefore, they share properties stored in the OFD, such as:

* Current file offset
* File status flags

For example, if the parent reads from the shared OFD:

```text
Parent
  │
  └── read()
        │
        └── OFD offset changes
```

the child sees the updated offset when it uses its inherited descriptor.

This is the same OFD-sharing relationship created by `dup()`.

```text
dup():

FD 3 ──┐
        ├──→ OFD
FD 4 ──┘


fork():

Parent FD 3 ──┐
              ├──→ OFD
Child  FD 3 ──┘
```

The difference is:

* `dup()` creates another FD in the **same process**.
* `fork()` creates another **process**, whose FD table contains references to the same OFDs.

## 5. File Descriptors and Process Independence

Although parent and child have separate FD tables:

```text
Parent FD table        Child FD table
┌───────────┐          ┌───────────┐
│ FD 0      │          │ FD 0      │
│ FD 1      │          │ FD 1      │
│ FD 2      │          │ FD 2      │
│ FD 3 ─────┼──────┐   │ FD 3 ─────┼──────┐
└───────────┘      │   └───────────┘      │
                   ▼                      ▼
                   └────── OFD ───────────┘
```

Closing an FD in one process does **not** close the corresponding FD in the other process.

For example:

```text
Parent: close(3)
```

removes FD 3 only from the parent's FD table.

The child still has its own FD 3.

The OFD remains alive while at least one reference to it remains.

## 6. Process Attributes After `fork()`

The child inherits many attributes from the parent, but some attributes are different.

For example:

```text
Parent:
    PID  = 1000
    PPID = 500

Child:
    PID  = 1001
    PPID = 1000
```

The child receives a new PID, and its parent becomes the process that called `fork()`.

The child generally inherits things such as:

* Virtual memory contents
* Environment
* Open file descriptors
* Current working directory
* File creation mask
* Resource limits

Some attributes are reset or have different values in the child.

The important idea is that `fork()` creates a new process with a largely inherited state, but the child is still a distinct process.

## 7. Parent and Child Execution

After `fork()`, there are two independent flows of execution:

```text
                fork()
                  |
          ┌───────┴───────┐
          │               │
       Parent           Child
          │               │
          ▼               ▼
     continues         continues
     execution         execution
```

The kernel scheduler decides when each process runs.

Therefore, you should not assume that the parent always executes first or that the child always executes first.

If the processes need a particular ordering, they need some synchronization mechanism.

## 8. `vfork()`

`vfork()` is a specialized process-creation mechanism designed primarily for the case where the child will quickly call `exec()` or `_exit()`.

Unlike `fork()`, the parent may be suspended while the child temporarily shares the parent's address space.

Conceptually:

```text
Parent
  │
vfork()
  │
  ▼
Child
  │
  ├── exec()
  │
  └── _exit()
  │
  ▼
Parent continues
```

This makes `vfork()` fundamentally different from normal `fork()`.

### Why is `vfork()` dangerous?

Because the child temporarily shares the parent's address space, the child must not arbitrarily modify memory or perform operations that assume it has an independent process state.

The child should normally do very little before:

```text
exec()
```

or:

```text
_exit()
```

For ordinary process creation, `fork()` is much easier to reason about.

## 9. `fork()` vs. `vfork()`

|                                | `fork()`                 | `vfork()`                    |
| ------------------------------ | ------------------------ | ---------------------------- |
| Creates child                  | Yes                      | Yes                          |
| Separate virtual address space | Yes                      | Temporarily shared           |
| Uses COW                       | Yes                      | Not in the same way          |
| Parent can run immediately     | Yes                      | Usually suspended            |
| Child can freely modify memory | Yes                      | No                           |
| Typical purpose                | General process creation | Quickly followed by `exec()` |
| Safety/ease of use             | Easier                   | More restrictive             |

## 10. Connection to `exec()`

A very common Unix process-creation pattern is:

```text
Shell
  │
  ├── fork()
  │
  ├── Child
  │     │
  │     ├── configure FDs
  │     ├── redirect stdin/stdout/stderr
  │     └── exec()
  │
  └── Parent
        │
        └── wait for child
```

This is how a shell can create a new process and then make that process run another program.

For example, redirection:

```text
fork()
  │
  └── child
        │
        ├── dup2(file_fd, STDOUT_FILENO)
        ├── close(file_fd)
        └── exec()
```

The important point is that `fork()` gives the child the parent's FDs, and the child can then modify its own FD table before `exec()`.

## Summary

The main concepts are:

```text
fork()
  │
  ├── creates a new process
  │
  ├── child gets a separate virtual address space
  │       └── initially same contents
  │
  ├── memory uses copy-on-write
  │
  ├── child inherits file descriptors
  │       └── parent/child FDs refer to same OFDs
  │
  ├── parent and child execute independently
  │
  └── child has its own PID

vfork()
  │
  ├── specialized process creation
  ├── temporarily shares address space
  ├── parent is suspended
  └── child should quickly exec() or _exit()
```

### Mental model

> **`fork()` creates a new process with an independent virtual address space, initially populated using copy-on-write, while inherited file descriptors can still refer to the same open file descriptions.**
