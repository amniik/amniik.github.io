---
title: "Program Execution in Linux"
categories:
  - Learning Notes
  - System Programming
tags: [linux, tlpi, exec, process, system-programming]
description: "A practical guide to Linux Program Execution."

toc: true
---

## Program Execution

Linux provides the `exec()` family of functions to replace the current process image with a new program. Unlike `fork()`, `exec()` does not create a new process.

### The exec() Family

Important functions include:

* `execve()` — explicit pathname, argument array, and environment.
* `execl()` / `execv()` — arguments as a list or array.
* `execlp()` / `execvp()` — search for the executable using `PATH`.
* `execle()` — allows an explicit environment.

A successful `exec()` **never returns** because the current program has been replaced. It returns only when execution fails.

### What Happens During exec()?

When a process successfully calls `execve()`:

```text
Before exec:

Process
├── Old code
├── Old data
├── Old heap
├── Old stack
└── File descriptors

             execve()
                 ↓

After exec:

Process
├── New code
├── New data
├── New heap
├── New stack
└── Existing file descriptors
```

The old virtual memory image is replaced, but the process itself continues to exist.

Therefore:

* **PID remains the same**
* Code, data, heap, and stack are replaced
* The process's file descriptors are preserved by default
* Current working directory is preserved
* The new program receives new `argv` and `envp`

### File Descriptors and exec()

The file descriptor table survives `exec()` by default.

```text
Before exec:

fd 0 ──→ stdin
fd 1 ──→ terminal
fd 3 ──→ output.txt

             exec()

After exec:

fd 0 ──→ stdin
fd 1 ──→ terminal
fd 3 ──→ output.txt
```

This is fundamental to shell redirection. A shell can open a file and use `dup2()` to configure `stdout` before calling `exec()`.

#### FD_CLOEXEC

An FD can be marked with `FD_CLOEXEC`. Such a descriptor is automatically closed when `exec()` succeeds.

This provides a way to control which file descriptors are inherited by the new program.

### fork() + exec()

A shell normally uses `fork()` and `exec()` together:

```text
Shell
  |
  +-- fork()
        |
        +-- Parent
        |     └── waitpid()
        |
        +-- Child
              ├── dup2()          ← configure redirection
              ├── close()
              └── execve()
                    ↓
                 New program
```

`fork()` creates a new process, while `exec()` transforms that child into the requested program.

This separation gives the shell an opportunity to configure the child before execution.

### Shell exec Builtin

`exec` is also normally a **shell builtin**.

```bash
exec ls
```

Here the shell itself calls `execve()` instead of creating a child first:

```text
Shell
  |
  └── execve("ls")
          ↓
         ls
```

The shell's process image becomes `ls`. When `ls` exits, the original shell is gone.

The builtin is also useful for changing the shell's file descriptors permanently:

```bash
exec > output.log 2>&1
```

After this, subsequent output from the shell or script is redirected to `output.log`.

### PATH Lookup

When a command such as:

```bash
ls
```

is executed without a pathname, the shell must locate the executable.

`execvp()` and `execlp()` perform `PATH` lookup by searching directories such as:

```text
PATH=/usr/local/bin:/usr/bin:/bin
```

The shell can therefore execute:

```text
ls
```

instead of requiring:

```text
/bin/ls
```

### Environment

Programs receive environment variables through `envp`.

For example:

```text
PATH=/usr/bin:/bin
HOME=/home/amir
LANG=en_US.UTF-8
```

The shell builds the environment and passes it to the new program during `execve()`.

`export` in a shell determines which shell variables become part of this environment for executed programs.

### Interpreter Scripts

Linux supports scripts beginning with a **shebang**:

```text
#!/bin/bash
```

The shebang tells the system which interpreter should execute the script.

For example:

```text
./script.sh
      ↓
/bin/bash
      ↓
script.sh
```

This allows scripts to behave like executable programs.

### system()

The C library function `system()` provides a higher-level way to execute a shell command:

```c
system("ls -l");
```

Conceptually:

```text
C program
   |
   └── system()
         |
         ├── create child
         |
         ├── execute /bin/sh -c "ls -l"
         |
         └── wait for shell
```

The important distinction is:

```text
execve()
    → directly executes a program
    → replaces the current process image
    → does not create a new process

system()
    → asks a shell to execute a command
    → normally creates a child
    → waits for the command
    → supports shell syntax
```

For example, shell syntax works with `system()`:

```c
system("ls | grep txt");
```

because the shell interprets the pipe.

`system()` should not be used with untrusted input because shell interpretation can result in command injection.

## Core Mental Model

The three operations have different responsibilities:

```text
fork()
  │
  └── "Create another process"

exec()
  │
  └── "Turn this process into another program"

waitpid()
  │
  └── "Wait for a child process"
```

Together they form the foundation of Unix process execution:

```text
             fork()
Shell ─────────────────→ Child
                           │
                           ├── configure FDs
                           ├── configure environment
                           │
                           └── exec()
                                │
                                ↓
                           New program
```

This model is particularly important when implementing a Unix shell: **the shell creates the process with `fork()`, prepares its execution environment, and then replaces the child with the requested program using `exec()`**.
