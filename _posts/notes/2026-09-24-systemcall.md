---
title: "System Call mechanism"
categories:
  - Learning Notes
  - System Programming
tags: [linux, system-call]
description: "A brief guide to System Call mechanism."

toc: true
---

# What Happens When We Call a System Call?

A **system call** is the mechanism a user-space program uses to request a service from the kernel.

For example:

```c
ssize_t n = read(fd, buf, 100);
```

The program is running in **user mode**, but reading from a file requires the kernel.

The system-call mechanism transfers execution from **user mode to kernel mode**, lets the kernel perform the operation, and then returns to user mode.

---

## User Mode and Kernel Mode

Modern CPUs provide different privilege levels.

```text
User mode
──────────────
Applications
C library
Shell
    │
    │ system call
    ▼
Kernel mode
──────────────
Kernel
File systems
Device drivers
Process management
Memory management
    │
    │ return
    ▼
User mode
```

User-space programs cannot directly perform privileged operations such as accessing kernel data structures or controlling hardware.

---

# The System Call Path

Consider:

```c
read(fd, buf, 100);
```

The simplified flow is:

```text
Application
     │
     │ read()
     ▼
C library wrapper
     │
     │ system-call instruction
     ▼
CPU
     │
     │ switch to kernel mode
     ▼
Kernel system-call entry
     │
     │ identify syscall
     ▼
sys_read()
     │
     │ perform operation
     ▼
Kernel
     │
     │ return value
     ▼
system-call return
     │
     │ switch to user mode
     ▼
C library
     │
     ▼
Application
```

---

# 1. Application Calls a Library Function

Usually the application doesn't directly contain the low-level CPU instructions for entering the kernel.

It calls a C library function:

```c
read(fd, buf, 100);
```

The C library provides a **system-call wrapper**.

For example, on Linux, the wrapper eventually executes the architecture-specific system-call instruction.

---

# 2. Arguments Are Prepared

Before entering the kernel, the arguments must be placed where the kernel expects them.

Conceptually:

```text
fd      → register
buf     → register
count   → register
syscall number → register
```

The exact registers depend on the CPU architecture.

The kernel needs a **system-call number** to determine which service was requested.

For example, conceptually:

```text
syscall number = read
arguments      = fd, buf, count
```

---

# 3. Trap / System-Call Instruction

The CPU provides a special mechanism for entering the kernel.

Historically this was often called a **trap** or software interrupt.

Modern x86-64 Linux normally uses:

```asm
syscall
```

For example, conceptually:

```text
user mode
   │
   │ syscall instruction
   ▼
CPU
   │
   ├── switches privilege level
   ├── saves necessary user context
   └── transfers control to kernel entry point
   ▼
kernel mode
```

A system call is therefore a controlled transition from user mode to kernel mode.

---

# 4. Kernel System-Call Entry

The CPU transfers execution to a kernel entry point.

The kernel then:

1. saves/restores the required CPU state
2. determines the system-call number
3. obtains the arguments
4. dispatches to the appropriate system-call implementation

Conceptually:

```text
             Kernel
               │
        syscall number
               │
               ▼
       system-call table
               │
      ┌────────┼─────────┐
      ▼        ▼         ▼
    read()   write()   open()
      │
      ▼
 filesystem / driver
```

The **system-call table** maps system-call numbers to kernel implementations.

---

# 5. Kernel Performs the Operation

For:

```c
read(fd, buf, 100);
```

the kernel may need to:

```text
FD
 │
 ▼
process FD table
 │
 ▼
open file description
 │
 ├── offset
 │
 ▼
filesystem / device
 │
 ▼
data
```

The kernel validates the FD, checks permissions and state, accesses the relevant object, and transfers data to the user-space buffer.

---

# 6. Return From the System Call

The kernel produces a return value.

For example:

```c
ssize_t n = read(fd, buf, 100);
```

If 100 bytes were read:

```text
return value = 100
```

If an error occurs:

```text
return value = -1
errno = appropriate error
```

The CPU then executes the architecture-specific return-from-system-call mechanism.

Conceptually:

```text
Kernel mode
     │
     │ return from syscall
     ▼
CPU
     │
     │ restore user context
     │ switch to user mode
     ▼
User mode
```

The application continues execution after the original call.

---

# Trap vs System Call

These terms are related but not exactly identical.

A **trap** is a synchronous transfer of control from user mode to privileged code caused by the currently executing instruction.

A **system call** is a specific, intentional use of such a mechanism to request a kernel service.

Historically:

```text
system call
     │
     ▼
software interrupt / trap
     │
     ▼
kernel
```

Modern architectures often provide a dedicated system-call instruction:

```text
syscall
```

So it is useful to think:

> **System call = the service/request**  
> **Trap/system-call instruction = the mechanism used to enter the kernel**

---

# Complete Picture

```text
                  USER SPACE
┌───────────────────────────────────────────┐
│                                           │
│  Application                              │
│      │                                    │
│      │ read(fd, buf, 100)                 │
│      ▼                                    │
│  C library wrapper                        │
│      │                                    │
│      │ syscall instruction                │
└──────┼────────────────────────────────────┘
       │
       │ CPU privilege transition
       ▼
                  KERNEL SPACE
┌───────────────────────────────────────────┐
│                                           │
│  System-call entry                        │
│      │                                    │
│      ▼                                    │
│  System-call dispatcher                   │
│      │                                    │
│      ▼                                    │
│  read() implementation                    │
│      │                                    │
│      ├── FD table                         │
│      ├── open file description            │
│      ├── filesystem                       │
│      └── device driver                    │
│      │                                    │
│      ▼                                    │
│  return value                             │
│                                           │
└──────┬────────────────────────────────────┘
       │
       │ return from syscall
       ▼
                  USER SPACE
┌───────────────────────────────────────────┐
│                                           │
│  C library                                │
│      │                                    │
│      ▼                                    │
│  Application                              │
│                                           │
└───────────────────────────────────────────┘
```

## Important Points

```text
Application
    │
    │ library call
    ▼
C library
    │
    │ syscall instruction
    ▼
CPU
    │
    │ user → kernel
    ▼
Kernel
    │
    │ perform operation
    ▼
CPU
    │
    │ kernel → user
    ▼
Application
```

The important distinction is that **a library function is not necessarily a system call**.

For example:

```c
printf("hello");
```

is a C library function. It may eventually use:

```text
printf()
   ↓
write()
   ↓
syscall
   ↓
kernel
```

while some library functions can be implemented entirely in user space.

Also, `read()`, `write()`, and `open()` are commonly exposed as C library functions that invoke the corresponding Linux system calls.

---

# Relation to File I/O

This connects directly to the FD/OFD model:

```text
read(fd, buf, 100)
        │
        ▼
   system call
        │
        ▼
   Kernel
        │
        ▼
 Process FD table
        │
        ▼
 Open File Description
        │
        ▼
 Filesystem / Device
```

So the **file descriptor is the user-visible handle**, while the kernel uses it to find the corresponding kernel objects and perform the requested operation.

The system-call boundary is the point where the process asks the kernel to perform an operation that user space cannot perform directly.
