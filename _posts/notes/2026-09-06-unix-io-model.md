---
title: "Unix I/O Model: File Descriptors, Open File Descriptions, and Inodes"
categories:
  - Learning Notes
  - System Programming
tags: [file-io, linux, unix, system-programming, system-calls]
description: "A practical guide to I/O model in unix."

toc: true
---

A useful way to understand Unix I/O is to separate three concepts:

1. **File descriptor (FD)** — a process-local integer
2. **Open file description (OFD)** — kernel state created by `open()`
3. **Inode** — the filesystem object

This model explains how `open()`, `read()`, `write()`, `lseek()`, `dup()`, `fork()`, and `fcntl()` work.


## The Basic Model

```text
Process
┌──────────────────────────┐
│ FD table                 │
│                          │
│ fd 0 ───────────────┐    │
│ fd 1 ───────────┐   │    │
│ fd 3 ─────────┐ │   │    │
└────────────────┼─┼───┼────┘
                 │ │   │
                 ▼ ▼   ▼
              Open File Description
              ┌───────────────────┐
              │ file offset       │
              │ status flags      │
              │ access mode       │
              │ pointer to inode  │
              └─────────┬─────────┘
                        │
                        ▼
                     Inode
              ┌───────────────────┐
              │ type              │
              │ permissions       │
              │ owner             │
              │ size              │
              │ timestamps        │
              │ file data         │
              └───────────────────┘
```

The **file descriptor is only an index in the process's FD table**. The actual open-file state is stored in the open file description. Linux calls the kernel object implementing an open file description a `struct file`.

---

# 1. `open()`

```c
int fd = open("file.txt", O_RDWR);
```

`open()` does two important things:

```text
open("file.txt")
       │
       ├── creates an open file description
       │
       └── creates an FD referring to it
```

For example:

```text
Process
FD table

fd 3 ─────────→ OFD #1 ─────────→ inode
                 │
                 ├── offset = 0
                 └── flags = O_RDWR
```

Every successful `open()` creates a **new open file description**.

---

# 2. `read()`

```c
read(fd, buffer, 100);
```

`read()` uses the offset stored in the open file description.

```text
OFD
┌─────────────────┐
│ offset = 100    │
└────────┬────────┘
         │
         ▼
       inode/data
```

After reading 50 bytes:

```text
offset = 150
```

So:

```text
read()
  │
  ├── reads data
  └── advances OFD offset
```

The offset belongs to the **open file description**, not directly to the FD.

---

# 3. `write()`

```c
write(fd, "hello", 5);
```

`write()` writes at the current offset and normally advances it.

```text
Before:

OFD
offset = 100

write("hello", 5)

After:

OFD
offset = 105
```

If the write extends beyond the previous end of the file, the file size in the filesystem object increases.

---

# 4. `lseek()`

```c
lseek(fd, 1000, SEEK_SET);
```

`lseek()` changes the offset in the **open file description**.

```text
lseek()
   │
   ▼
OFD
offset = 1000
```

It does **not** itself change the file size.

For example:

```c
lseek(fd, 1000, SEEK_SET);
write(fd, "X", 1);
```

creates a file whose logical size is `1001` bytes, with a hole between the beginning and offset `999`.

---

# 5. Two `open()` Calls

Consider:

```c
int fd1 = open("file.txt", O_RDWR);
int fd2 = open("file.txt", O_RDWR);
```

The important point is that these create **two different open file descriptions**:

```text
fd1 ─────→ OFD #1 ─────→ inode
             │
             └── offset = 0


fd2 ─────→ OFD #2 ─────→ inode
             │
             └── offset = 0
```

They refer to the same filesystem object, but have independent offsets.

For example:

```c
read(fd1, buf, 10);
```

results in:

```text
OFD #1 offset = 10
OFD #2 offset = 0
```

The two offsets are independent.

---

# 6. `dup()`

Now consider:

```c
int fd2 = dup(fd1);
```

This is different from calling `open()` again.

```text
fd1 ─────┐
         │
         ▼
       OFD #1 ─────→ inode
         ▲
         │
fd2 ─────┘
```

Both FDs refer to the **same open file description**.

Therefore they share:

* file offset
* file status flags

For example:

```c
read(fd1, buf, 10);
```

changes the offset seen through `fd2` as well.

```text
Before:
fd1 ──┐
      ├──→ OFD offset = 0
fd2 ──┘

read(fd1, 10)

After:
fd1 ──┐
      ├──→ OFD offset = 10
fd2 ──┘
```

This is why `dup()` is commonly used when redirecting standard input/output.

`dup()` creates a new FD, but both descriptors refer to the same OFD.

---

# 7. `fork()`

Consider:

```c
int fd = open("file.txt", O_RDWR);

fork();
```

After `fork()`, the child gets a copy of the parent's FD table.

But the corresponding FDs refer to the **same open file description**:

```text
Parent                 Child

FD 3 ─────┐             FD 3 ─────┐
           │                         │
           └────────→ OFD ←─────────┘
                       │
                       ▼
                      inode
```

Therefore parent and child share:

* file offset
* file status flags

For example, if the child reads 100 bytes, the shared offset changes for the parent too.

Linux explicitly documents this sharing behavior for `fork()`.

---

# 8. File Descriptor Flags vs File Status Flags

This distinction is important.

### File descriptor flags

These belong to the **FD itself**.

Example:

```text
FD 3
 └── FD_CLOEXEC
```

They are manipulated with:

```c
fcntl(fd, F_GETFD);
fcntl(fd, F_SETFD);
```

### File status flags

These belong to the **open file description**.

Examples:

```text
O_APPEND
O_NONBLOCK
O_ASYNC
```

They are manipulated with:

```c
fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags);
```

Therefore:

```text
FD
 └── FD flags

OFD
 ├── file offset
 └── file status flags
```

Duplicated FDs share the OFD and therefore share its status flags, but their FD flags are separate.

# O_APPEND and lseek()

When a file is opened with `O_APPEND`, every `write()` operation writes data at the **end of the file**, regardless of the current file offset.

```c
int fd = open("file.txt", O_WRONLY | O_APPEND);

lseek(fd, 0, SEEK_SET);
write(fd, "HELLO", 5);
```

If the file initially contains:

```text
ABCDEF
```

the result is:

```text
ABCDEFHELLO
```

## Why?

There are two separate concepts:

* `lseek()` changes the **current file offset** stored in the open file description.
* `O_APPEND` changes the behavior of `write()` so that the kernel positions the write at **EOF** before performing it.

```text
lseek()
   │
   ▼
changes current offset
   │
   │
write() + O_APPEND
   │
   ▼
ignore offset for write position
   │
   ▼
write at EOF
```

Without `O_APPEND`:

```text
lseek(fd, 0, SEEK_SET);
write(fd, "HELLO", 5);

ABCDEF
   ↓
HELLOF
```

With `O_APPEND`:

```text
lseek(fd, 0, SEEK_SET);
write(fd, "HELLO", 5);

ABCDEF
      ↓
ABCDEFHELLO
```

### Important rule

> `lseek()` controls the file offset, but `O_APPEND` makes each `write()` occur at the end of the file.

`O_APPEND` is commonly used for **log files**, where multiple writers need to add data without overwriting existing contents.

On Linux, positioning the offset at EOF and performing the append write are performed as one atomic step, which is important when multiple processes write to the same file.

---

# 9. `fcntl()`

`fcntl()` performs various operations on an existing FD.

For example:

```c
int flags = fcntl(fd, F_GETFL);

flags |= O_NONBLOCK;

fcntl(fd, F_SETFL, flags);
```

This modifies the **file status flags of the open file description**.

It can also manipulate the FD itself:

```c
fcntl(fd, F_GETFD);
fcntl(fd, F_SETFD, FD_CLOEXEC);
```

So:

```text
fcntl()
   │
   ├── F_GETFD / F_SETFD
   │       └── FD flags
   │
   └── F_GETFL / F_SETFL
           └── OFD status flags
```

---

# 10. `close()`

```c
close(fd);
```

`close()` removes the FD from the process's FD table.

```text
Before:

FD 3 ─────→ OFD

After:

FD 3       X

             OFD
```

If other FDs still reference the OFD, the OFD remains alive.

For example:

```text
fd1 ─────┐
         ├──→ OFD
fd2 ─────┘
```

After:

```c
close(fd1);
```

we have:

```text
fd2 ─────→ OFD
```

The OFD still exists.

---

# 11. `stat()`

`stat()` retrieves metadata about a filesystem object:

```c
struct stat st;

stat("file.txt", &st);
```

For example:

```text
inode
 ├── file type
 ├── permissions
 ├── owner
 ├── size
 ├── timestamps
 └── inode number
```

This is different from the OFD's state.

For example:

```text
OFD:
    offset = 500
    O_APPEND

inode:
    size = 4096
    permissions = 0644
```

---

# 12. `unlink()`

```c
unlink("file.txt");
```

`unlink()` removes the directory entry referring to the inode.

It does not necessarily immediately remove the underlying data.

For example:

```text
directory
   │
   └── "file.txt" ─────→ inode
                           ▲
                           │
                          OFD
```

After `unlink()`:

```text
directory
   │
   └── "file.txt"   X

                          inode
                            ▲
                            │
                           OFD
```

If a process still has the file open, it can continue using the file through its FD.

---

# 13. `ioctl()`

`ioctl()` performs operations that don't fit the normal:

```text
read()
write()
lseek()
```

model.

For example, getting terminal size:

```c
struct winsize ws;

ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws);
```

Mental model:

```text
read()    → transfer data
write()   → transfer data
lseek()   → change offset
fcntl()   → manipulate FD/OFD properties
ioctl()   → resource/device-specific operation
```

This is why Unix can use the same FD abstraction for many different resources, while still providing specialized operations when necessary.

---

# 14. The Three Most Important Cases

## Case 1: Two `open()` Calls

```c
fd1 = open("file");
fd2 = open("file");
```

```text
fd1 ──→ OFD #1 ──→ inode
                 ↑
fd2 ──→ OFD #2 ──┘
```

**Different OFDs → different offsets.**

---

## Case 2: `dup()`

```c
fd2 = dup(fd1);
```

```text
fd1 ──┐
      ├──→ OFD ──→ inode
fd2 ──┘
```

**Same OFD → same offset.**

---

## Case 3: `fork()`

```c
fd = open("file");
fork();
```

```text
Parent FD ──┐
            │
            ├──→ OFD ──→ inode
            │
Child FD ───┘
```

**Same OFD → shared offset.**

---

# Final Mental Model

Keep this picture in your head:

```text
                 PROCESS
        ┌─────────────────────┐
        │ File Descriptor     │
        │ Table               │
        │                     │
        │  0 ──────┐         │
        │  1 ───┐  │         │
        │  3 ─┐ │  │         │
        └─────┼─┼──┼─────────┘
              │ │  │
              ▼ ▼  ▼
        Open File Descriptions
        ┌─────────────────────┐
        │ offset              │
        │ status flags        │
        │ access mode         │
        │                     │
        │ pointer to inode ───┼─────┐
        └─────────────────────┘     │
                                    ▼
                                  INODE
                           ┌────────────────┐
                           │ type           │
                           │ permissions    │
                           │ owner          │
                           │ size           │
                           │ timestamps     │
                           │ data           │
                           └────────────────┘
```

The key rule is:

```text
FD
  = process-local reference

OFD
  = state of one open() instance

inode
  = filesystem object
```

And the most important sharing rule is:

```text
open()  → new OFD
dup()   → same OFD
fork()  → inherited FD referring to same OFD
```

This model explains most of the behavior of Unix file I/O.

# Redirection

Shell redirection is a direct application of the file descriptor model.

The key idea is:

> **The shell changes the process's file descriptors before executing the program.**

For example:

```bash
ls > output.txt
```

means that the shell makes `ls`'s standard output (FD 1) refer to `output.txt`.

---

## Output Redirection

Normally:

```text
Process
┌─────────────────────┐
│ FD table            │
│                     │
│ 0 ──→ terminal      │
│ 1 ──→ terminal      │
│ 2 ──→ terminal      │
└─────────────────────┘
```

For:

```bash
ls > output.txt
```

the shell approximately does:

```c
int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

dup2(fd, STDOUT_FILENO);

close(fd);

execve(...);
```

If `open()` returns FD 3:

```text
Before dup2():

FD 1 ──→ terminal
FD 3 ──→ OFD ──→ output.txt
```

After:

```c
dup2(3, 1);
```

we have:

```text
FD 1 ──┐
       ├──→ OFD ──→ output.txt
FD 3 ──┘
```

After `close(3)`:

```text
FD 1 ──→ OFD ──→ output.txt
```

The shell then executes `ls`.

`ls` simply writes to FD 1:

```c
write(STDOUT_FILENO, buffer, size);
```

It does not need to know that FD 1 now refers to a file.

---

## Input Redirection

For:

```bash
sort < input.txt
```

the shell approximately does:

```c
int fd = open("input.txt", O_RDONLY);

dup2(fd, STDIN_FILENO);

close(fd);

execve(...);
```

The result is:

```text
FD 0 ──→ OFD ──→ input.txt
FD 1 ──→ terminal
FD 2 ──→ terminal
```

`sort` simply reads from standard input:

```c
read(STDIN_FILENO, buffer, size);
```

It doesn't need to know whether the input comes from a terminal or a file.

---

## Standard File Descriptors

Unix processes normally start with three standard file descriptors:

```text
0 → stdin  → standard input
1 → stdout → standard output
2 → stderr → standard error
```

Redirection changes where these descriptors point.

For example:

```bash
command > output.txt
```

results in:

```text
FD 0 ──→ terminal
FD 1 ──→ output.txt
FD 2 ──→ terminal
```

While:

```bash
command 2> errors.txt
```

results in:

```text
FD 0 ──→ terminal
FD 1 ──→ terminal
FD 2 ──→ errors.txt
```

---

## `2>&1`

Consider:

```bash
command > output.txt 2>&1
```

Redirections are processed from left to right.

First:

```bash
> output.txt
```

gives:

```text
FD 1 ──→ output.txt
FD 2 ──→ terminal
```

Then:

```bash
2>&1
```

makes FD 2 refer to the same open file description as FD 1:

```text
FD 1 ──┐
       ├──→ OFD ──→ output.txt
FD 2 ──┘
```

Therefore both stdout and stderr go to `output.txt`.

The order matters:

```bash
command > output.txt 2>&1
```

is different from:

```bash
command 2>&1 > output.txt
```

because the shell processes the redirections in order.

---

# Pipes

Pipes use the same FD model.

For:

```bash
ls | grep txt
```

the shell creates a pipe:

```c
int pipefd[2];
pipe(pipefd);
```

Conceptually:

```text
pipefd[0] → read end
pipefd[1] → write end
```

The shell then configures the two processes.

For `ls`:

```c
dup2(pipefd[1], STDOUT_FILENO);
```

For `grep`:

```c
dup2(pipefd[0], STDIN_FILENO);
```

The resulting setup is:

```text
             PIPE
          ┌──────────┐
ls        │          │       grep
FD 1 ─────→ write    │
          │          │
          │ read ────→ FD 0
          │          │
          └──────────┘
```

So:

```text
ls
 │
 │ write(1, ...)
 ▼
pipe
 │
 │ read(0, ...)
 ▼
grep
```

Neither program needs special knowledge of the other program.

`ls` thinks it is writing to stdout.

`grep` thinks it is reading from stdin.

The shell and kernel connect them.

---

# Redirection and `exec()`

A typical shell performs these steps:

```text
Shell
 │
 ├── open files / create pipes
 │
 ├── configure FDs using dup2()
 │
 ├── fork()
 │
 └── child: execve()
             │
             ▼
          Program
```

The important property of `execve()` is that it replaces the program's code and memory, but **open file descriptors normally remain open**.

Therefore, the child starts with the FD configuration prepared by the shell.

---

# Why Redirection Works

Programs don't need to know where their input and output come from.

A program generally uses:

```c
read(0, ...);      // stdin
write(1, ...);     // stdout
write(2, ...);     // stderr
```

Those descriptors can refer to different kinds of resources:

```text
                    File Descriptor
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           file          pipe       terminal
             │            │            │
             ▼            ▼            ▼
           disk         kernel      terminal
```

This is another consequence of the Unix **"everything through file descriptors"** design.

The shell determines the connections **before** the program runs.

The program only needs to use the standard file descriptors.

---

# The Complete Mental Model

```text
                       SHELL
                         │
             ┌───────────┴───────────┐
             │                       │
          open()                   pipe()
             │                       │
             └───────────┬───────────┘
                         ▼
                       fork()
                         │
                         ▼
                      dup2()
                         │
                         ▼
                      execve()
                         │
                         ▼
                      PROGRAM
                    ┌────┼────┐
                    │    │    │
                   FD0  FD1  FD2
                    │    │    │
                    ▼    ▼    ▼
                  input output errors
```

The important sequence to remember is:

```text
open() / pipe()
       ↓
     dup2()
       ↓
    close()
       ↓
     fork()
       ↓
    execve()
```

This sequence is the foundation of how Unix shells implement **redirection and pipelines**.
