---
title: "Linux Child Process Monitoring: wait(), waitpid(), SIGCHLD, and exec()"
categories:
  - Learning Notes
  - System Programming
tags: [linux, processes, waitpid, sigchld, signals, exec, fork, zombies, job-control]
---

# Linux Child Process Monitoring

Monitoring child processes is a fundamental responsibility of Unix and Linux programs that create other processes.

A parent process often needs to know:

- When a child terminates.
- Whether it exited normally or was terminated by a signal.
- Whether it stopped or continued.
- What exit status it returned.
- How to prevent terminated children from accumulating as zombies.

These concepts are particularly important when implementing a shell, process supervisor, service manager, or runtime.

## 1. The Lifecycle of a Child Process

A typical process lifecycle looks like this:

```text
             Parent calls fork()
                     |
                     v
                Child exists
                     |
           +---------+----------+
           |                    |
           v                    v
      Child runs            Child stops
           |                    |
           v                    v
      Child exits           Child continues
           |
           v
      Zombie state
           |
           | Parent calls wait() / waitpid()
           v
      Child is reaped
```

A child can also terminate because of a signal rather than exiting normally.

The important distinction is that **termination and reaping are separate events**.

When a child terminates, the kernel retains enough information about it for its parent to retrieve the termination status. Until the child is reaped, it can remain a zombie.

## 2. The `wait()` System Call

`wait()` waits for a state change involving one of the calling process's children, with its traditional interface primarily used to collect terminated children.

```c
#include <sys/wait.h>

int status;
pid_t pid = wait(&status);
```

If a waitable child exists, `wait()` returns its PID and stores status information in `status`.

If no child is currently waitable but children still exist, the call normally blocks until a suitable state change occurs.

If there are no eligible children, it fails with `-1` and sets `errno` to `ECHILD`.

The status is encoded information, not simply the child's exit code.

For example, this is incorrect:

```c
if (status == 42) {
    /* Child exited with status 42 */
}
```

Instead, use the status macros described below.

## 3. The `waitpid()` System Call

`waitpid()` provides more control over which child to monitor and whether to block.

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

The `pid` argument selects the target:

| `pid` value | Meaning |
|---|---|
| `pid > 0` | Wait for the specified child PID |
| `pid == -1` | Wait for any eligible child |
| `pid == 0` | Wait for an eligible child in the caller's process group |
| `pid < -1` | Wait for an eligible child in process group `-pid` |

For example:

```c
int status;
pid_t result = waitpid(child_pid, &status, 0);
```

This blocks until the specified child has a waitable state change.

### Nonblocking waits with `WNOHANG`

A shell often needs to check background jobs without blocking its main loop.

```c
int status;
pid_t result = waitpid(-1, &status, WNOHANG);

if (result > 0) {
    /* A child has a reportable state change. */
} else if (result == 0) {
    /* Matching children exist, but none is waitable now. */
} else {
    /* Error: inspect errno. */
}
```

The exact interpretation of a positive result depends on the options used. Without options requesting stopped or continued states, the main purpose is to collect terminated children.

A return value of `0` is not an error. It means no matching child currently has a reportable state change.

## 4. Understanding Child Status

The integer returned through `status` must be interpreted using the appropriate macros.

### Normal termination

```c
if (WIFEXITED(status)) {
    int exit_code = WEXITSTATUS(status);
}
```

`WEXITSTATUS()` is meaningful only if `WIFEXITED()` is true.

For example, a child that calls:

```c
exit(42);
```

or:

```c
_exit(42);
```

normally produces a termination status from which the parent can retrieve the value `42`.

### Termination by signal

```c
if (WIFSIGNALED(status)) {
    int sig = WTERMSIG(status);
}
```

This tells the parent that the child terminated because of a signal.

For example, a child might be terminated by `SIGKILL` or by the default action for `SIGSEGV`.

### Stopped and continued children

```c
if (WIFSTOPPED(status)) {
    int sig = WSTOPSIG(status);
}

if (WIFCONTINUED(status)) {
    /* The child resumed execution. */
}
```

To report stopped or continued states, `waitpid()` needs the relevant options:

```c
waitpid(child_pid, &status, WUNTRACED | WCONTINUED);
```

- `WUNTRACED` requests reports for stopped children that have not been traced.
- `WCONTINUED` requests reports for children that have resumed after being stopped.

These options are useful when implementing shell job control.

## 5. Zombies and Reaping

A zombie is a process that has terminated but has not yet been reaped by its parent.

The kernel retains limited information, including the termination status and accounting information, so the parent can collect it.

```text
Child calls _exit()
        |
        v
Kernel records termination status
        |
        v
Child becomes a zombie
        |
        v
Parent calls waitpid()
        |
        v
Kernel releases the remaining process entry
```

A zombie does not execute instructions and does not consume memory like a running process. However, it occupies a process-table entry and retains some kernel bookkeeping.

If a parent creates many children but never reaps them, zombies can accumulate.

### Reaping is not the same as killing

- `kill()` sends a signal to a process or process group.
- `waitpid()` retrieves reportable child state and reaps a terminated child.

Calling `kill()` does not itself reap a child. Even if a child is killed immediately, its parent normally still needs to reap it.

If a parent terminates before its child, the child is adopted by an appropriate subreaper or, ordinarily, by the system's init process. The adopting process can then reap it.

## 6. The `SIGCHLD` Signal

The kernel can generate `SIGCHLD` when a child changes state, including termination and, under the relevant conditions, stopping or continuing.

A crucial distinction is:

**`SIGCHLD` is a notification, not the child's exit status.**

The status is obtained through `wait()`, `waitpid()`, or `waitid()`.

### Default disposition versus explicit `SIG_IGN`

The default disposition of `SIGCHLD` is described as ignore. However, on Linux, this does not have the same semantics as explicitly setting its disposition to `SIG_IGN`.

```c
signal(SIGCHLD, SIG_DFL);
```

With the default disposition, a terminated child can still become a zombie.

By contrast:

```c
signal(SIGCHLD, SIG_IGN);
```

Explicitly ignoring `SIGCHLD` on Linux normally prevents terminated children from becoming zombies. Their termination statuses are then unavailable for collection, and `waitpid()` generally fails with `ECHILD` when no other eligible children remain.

The `SA_NOCLDWAIT` flag can also request automatic handling that avoids zombies.

Therefore, do not use explicit `SIG_IGN` if your program needs to collect child exit statuses.

### Why one signal does not equal one child

Standard signals such as `SIGCHLD` can coalesce.

For example:

```text
Child A exits ──┐
                ├── SIGCHLD may be pending only once
Child B exits ──┘
```

The parent must not assume that receiving one `SIGCHLD` means exactly one child has terminated.

Instead, it should check all currently waitable children.

## 7. Reaping Multiple Children

A common pattern is to use `waitpid()` in a loop with `WNOHANG`:

```c
#include <sys/wait.h>
#include <errno.h>

void reap_children(void)
{
    int status;
    pid_t pid;

    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            /* Record normal exit. */
        } else if (WIFSIGNALED(status)) {
            /* Record signal termination. */
        }
    }

    if (pid == -1 && errno != ECHILD) {
        /* Handle or report the error. */
    }
}
```

This example illustrates the loop, but production code should handle errors carefully and ensure that other code does not concurrently reap the same children.

If stopped and continued states also need to be processed, use the corresponding `waitpid()` options and handle each reported state appropriately.

### Where should this function run?

It can run in the normal execution path of the program, such as the main event loop.

It should not automatically be placed inside a signal handler without considering asynchronous-signal-safety, `errno`, and interactions with other code that waits for children.



## 8. Signal Handlers and Race Conditions

Consider this sequence:

```text
Parent creates child
        |
        v
Child terminates immediately
        |
        v
SIGCHLD is generated
        |
        v
Parent begins waiting for notification
```

The child may terminate before the parent begins waiting. Depending on the design, the notification may already have been delivered or may be pending.

A robust design must not depend on a signal arriving only after the parent starts waiting.

### Signals are notifications, not event counters

A useful pattern is:

1. Record that child state may have changed.
2. Return from the signal handler.
3. Reap all currently waitable children using `waitpid()` in a loop.
4. Repeat the check whenever another notification arrives.

However, a simple flag by itself does not solve every lost-wakeup race. If a program uses a flag and then sleeps, it must coordinate checking the flag and waiting for a signal correctly. `sigsuspend()` or a suitable event-loop design can provide the required coordination.

### Keep signal handlers simple

Functions such as `printf()` are not async-signal-safe. Calling them from a handler can corrupt shared library state or cause deadlocks if the signal interrupted code that was already using the same library.

A handler can instead set a suitable `volatile sig_atomic_t` flag:

```c
#include <signal.h>

static volatile sig_atomic_t child_changed = 0;

static void sigchld_handler(int sig)
{
    (void) sig;
    child_changed = 1;
}
```

Normal program execution can inspect the flag and perform the actual bookkeeping.

This flag is appropriate for simple communication in a single-threaded signal-handling design. It is not a general-purpose synchronization primitive for multithreaded programs.

Another approach is to use a self-pipe or an event-loop integration mechanism to notify the main loop.

## 9. What Happens to Signal Dispositions After `exec()`?

The `exec()` family replaces the current process image with a new program. It does not create a new process.

For example:

```c
fork();

execve(path, argv, envp);
```

After a successful `execve()`, the process retains its PID, but its program code, data, stack, and other parts of the process image are replaced.

Signal behavior across `exec()` is important when building shells and process launchers.

### Caught signal dispositions are reset

Suppose a process installs a handler:

```c
struct sigaction sa = {0};
sa.sa_handler = my_handler;

sigaction(SIGINT, &sa, NULL);
```

If this process successfully executes another program:

```c
execve(path, argv, envp);
```

the caught handler for `SIGINT` is reset to the default disposition.

The new program does not inherit the old program's custom handler.

This applies to signal dispositions that were set to caught handlers.

### Ignored signal dispositions are generally preserved

If a signal was explicitly set to `SIG_IGN`, that ignored disposition generally remains ignored across a successful `exec()`.

For example:

```c
signal(SIGPIPE, SIG_IGN);
execve(path, argv, envp);
```

After a successful execution, `SIGPIPE` remains ignored unless the new program changes its disposition.

This is an important difference:

| Before `exec()` | After successful `exec()` |
|---|---|
| Custom caught handler | Reset to `SIG_DFL` |
| `SIG_DFL` | Remains `SIG_DFL` |
| `SIG_IGN` | Generally remains `SIG_IGN` |

There are specialized cases and platform details, but this is the essential POSIX/Linux model.

### Why are caught handlers reset?

A signal handler is a function belonging to the old program's address space.

After `exec()`, the old code and data are gone. The kernel cannot simply keep calling the old handler in the replaced program image.

Resetting caught dispositions avoids leaving a new program with handler addresses that belonged to the previous program.

An ignored disposition does not refer to a function address, so it can be preserved.

### Signal masks are different from dispositions

The signal mask determines which signals are blocked for the thread.

Unlike caught signal dispositions, the calling thread's signal mask is preserved across a successful `exec()`.

For example, if `SIGINT` was blocked before `execve()`, it remains blocked in the new program until the mask is changed.

This means a process launcher should be deliberate about which signals it blocks before executing a new program.

### Pending signals

Pending signals are also distinct from signal dispositions. A successful `exec()` does not generally clear the calling process's pending signals. A pending signal can therefore be delivered to the new program, subject to its disposition and signal mask.

For a multithreaded process, `exec()` also terminates the other threads, leaving the new program with a single thread.

## 10. How `fork()` and `exec()` Interact with Signals

The typical shell launch sequence looks like this:

```text
Shell
  |
  | fork()
  |
  +----------------------+
  |                      |
  v                      v
Parent shell          Child process
                         |
                         | Set up redirections,
                         | process group, and
                         | signal state as needed
                         |
                         | execve()
                         v
                     New program
```

### What does the child inherit from `fork()`?

The child inherits copies of the parent's signal dispositions and its signal mask.

For example, if the parent has a custom `SIGINT` handler at the moment of `fork()`, the child initially inherits that disposition.

### What changes after `exec()`?

When the child successfully calls `execve()`:

- Caught signal handlers are reset to default dispositions.
- Ignored dispositions generally remain ignored.
- The signal mask is preserved.
- Pending signals are not generally cleared simply because `exec()` occurred.

This is especially important when a shell launches a command.

### Example: the shell ignores a signal

Suppose a shell ignores `SIGINT` while it waits for a foreground command:

```c
signal(SIGINT, SIG_IGN);
```

If the shell forks a child, the child initially inherits `SIG_IGN`.

If the child then executes another program without changing that disposition, the new program generally continues to ignore `SIGINT`, because ignored dispositions survive `exec()`.

That may be undesirable for a foreground command: pressing Ctrl+C should normally interrupt the foreground job rather than be ignored by it.

Therefore, a shell often resets selected signal dispositions in the child before `execve()`:

```c
struct sigaction sa = {0};
sa.sa_handler = SIG_DFL;
sigemptyset(&sa.sa_mask);

sigaction(SIGINT, &sa, NULL);
sigaction(SIGQUIT, &sa, NULL);
sigaction(SIGTSTP, &sa, NULL);
```

The exact signals reset depend on the shell's design, whether the command is foreground or background, and the shell's own job-control behavior.

The child may also need to restore an appropriate signal mask before executing the program.

**Important:** A shell should not blindly reset every signal. Some dispositions may intentionally be inherited, and background jobs may require different treatment. A real shell follows a carefully defined launch procedure.

## 11. The `system()` Function

The `system()` library function executes a command through a command interpreter, usually a shell.

Conceptually:

```c
system("ls -l");
```

is similar to launching a shell to interpret the command, but `system()` also has defined signal-handling and waiting behavior.

On POSIX systems, `system()` blocks `SIGCHLD` in the calling thread and ignores `SIGINT` and `SIGQUIT` in the calling process while the command runs, with restoration afterward. The child environment and command execution are arranged so the command can have the expected signal behavior.

This has practical consequences:

- Do not assume that another part of the program can independently reap the child used by `system()`.
- Be careful when combining `system()` with custom `SIGCHLD` handling.
- If you need explicit control over process groups, redirection, signal masks, or child status, use `fork()` and an appropriate `exec()` function instead.

The exact implementation of `system()` is platform-dependent, but its interface behavior is specified by the relevant standards.

## 12. The `waitid()` System Call

`waitid()` provides another interface for monitoring child state changes.

```c
#include <sys/wait.h>

siginfo_t info;

if (waitid(P_PID, child_pid, &info, WEXITED) == -1) {
    /* Handle error. */
}
```

It uses an identifier type to select the child or group and reports details through `siginfo_t`.

Important options include:

- `WEXITED` — wait for termination.
- `WSTOPPED` — wait for stopped children.
- `WCONTINUED` — wait for continued children.
- `WNOHANG` — return without blocking when no matching state change is available.
- `WNOWAIT` — inspect the state without reaping a terminated child.

`waitpid()` is often sufficient for a shell, but understanding `waitid()` helps when reading system-level code that needs more detailed process-state information.

## 13. Applying These Concepts to a Shell

A shell must coordinate child creation, execution, job tracking, and terminal control.

A simplified design is:

```text
Read command
     |
     v
Create child with fork()
     |
     v
Child:
  - Set process group
  - Configure redirections
  - Restore intended signal state
  - Execute program
     |
     v
Parent:
  - Record PID and job information
  - Wait for foreground job, or
    return to prompt for background job
     |
     v
Handle state changes
     |
     v
Reap terminated children
```

For a foreground job, the shell generally waits until the job terminates or stops.

For a background job, the shell should not block waiting for its completion. It can use a signal-driven notification mechanism or periodically check child states through a nonblocking wait.

A real shell must also handle pipelines, where multiple child processes may belong to one job and process group.

### Important implementation rules

1. Do not confuse `SIGCHLD` delivery with child reaping.
2. Do not assume one `SIGCHLD` means one child changed state.
3. Use `waitpid()` to retrieve status and reap terminated children.
4. Handle stopped and continued states if implementing job control.
5. Restore the intended signal dispositions in the child before `exec()`.
6. Remember that ignored dispositions and blocked signals can survive `exec()`.
7. Avoid complex work and non-async-signal-safe functions inside signal handlers.
8. Coordinate waiting and signal handling to avoid race conditions.

## 14. The Essential Mental Model

Keep these four concepts separate:

| Concept | Meaning |
|---|---|
| Signal disposition | What the process does when a signal is delivered |
| Signal mask | Which signals are currently blocked |
| Pending signal | A generated signal that has not yet been delivered |
| Child status | Information the parent retrieves through a wait interface |

For child monitoring:

```text
Child changes state
       |
       v
SIGCHLD may notify parent
       |
       v
Parent calls waitpid()
       |
       v
Parent learns the state
       |
       v
Terminated child is reaped
```

For execution:

```text
fork()
  |
  v
Child inherits signal dispositions and mask
  |
  v
Child prepares execution environment
  |
  v
exec()
  |
  +--> Caught handlers reset to SIG_DFL
  |
  +--> SIG_IGN generally remains SIG_IGN
  |
  +--> Signal mask is preserved
  |
  +--> Pending signals are not generally cleared
```

Understanding these rules is essential for building reliable process supervisors, command interpreters, and other Linux system software.

## 15. Orphan Processes and Orphaned Process Groups

An orphan process is a process whose parent has terminated or otherwise ceased to be its parent. On Linux, when a process's parent terminates, the child is reparented to an appropriate subreaper if one exists, or otherwise to an init process.

An orphan process is not necessarily a problem. Many legitimate processes continue running after their original parent exits.

15.1 Creating an orphan process

Consider this example:

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        sleep(2);
        printf("Child PID: %ld\n", (long)getpid());
        printf("New parent PID: %ld\n", (long)getppid());
        _exit(EXIT_SUCCESS);
    }

    printf("Parent exiting\n");
    exit(EXIT_SUCCESS);
}

The parent exits before the child finishes.

Conceptually:

Initially:

       Parent
          |
          v
        Child

Parent terminates:

       init / subreaper
              |
              v
            Child

The child continues executing. Its parent changes, and getppid() reports the PID of its new parent.

On Linux, the exact adopting process depends on the process hierarchy and whether a child subreaper is configured.

## 15.2 Orphan processes versus zombie processes

These concepts describe different things.

Concept

Meaning

Orphan process

Its original parent has terminated; it is adopted by another process

Zombie process

It has terminated, but its termination status has not yet been collected

Running child

It is still executing and its parent remains alive

An orphan can continue running normally. A zombie has already terminated.

For example:

Parent exits while child is running
             |
             v
      Child becomes orphan
             |
             v
      Child continues running
             |
             v
      Child eventually exits
             |
             v
      New parent can reap it

The new parent is responsible for reaping the adopted child if it terminates.

## 15.3 What happens to SIGCHLD?

When an orphaned child is adopted, the new parent can receive SIGCHLD when that child later changes state, including when it terminates.

The original parent cannot collect the child's status after it has exited and the child has been reparented.

The important rule is that the parent responsible for the child at termination time is the one that can normally wait for it.