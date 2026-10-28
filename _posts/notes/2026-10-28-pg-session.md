---
title: "Process Groups, Sessions, and Job Control"
categories:
  - Learning Notes
  - System Programming
tags: [linux, tlpi, processes, process-groups, sessions, job-control, signals, terminals, rust]
---

# Process Groups, Sessions, and Job Control

## 1. Overview

Linux provides mechanisms for organizing processes into **process groups** and **sessions**. These mechanisms allow a shell to manage related processes as jobs, control which job interacts with a terminal, and respond correctly to terminal-generated signals.

The shell implements the user-facing concepts of foreground jobs, background jobs, and job management commands. The kernel provides the underlying process-group, session, signal, and terminal-control mechanisms.

The main concepts are:

- Process groups
- Sessions
- Controlling terminals
- Foreground and background process groups
- Job-control signals
- Implementing shell job control
- Orphaned process groups and `SIGHUP`

## 2. Process groups

A **process group** is a collection of processes identified by a process group ID (PGID). Process groups allow signals to be sent to multiple related processes at once.

Every process belongs to a process group. A newly created child normally inherits its parent's process group.

### 2.1 Process group ID

The PGID identifies a process group. By convention, the PGID of a process group leader equals the PID of that leader.

A process group can contain multiple processes, and its members don't have to be direct children of the group leader.

Useful functions:

```c
#include <unistd.h>

pid_t getpgrp(void);
pid_t getpgid(pid_t pid);
int setpgid(pid_t pid, pid_t pgid);
```

- `getpgrp()` returns the calling process's PGID.
- `getpgid(pid)` returns the PGID of a specified process.
- `setpgid(pid, pgid)` moves a process into a process group or creates a group, subject to system restrictions.

For `setpgid()`:

- If `pgid` matches the PID of an eligible process, that process group can be joined.
- If `pgid` equals `pid`, a new process group can be created with that process as its leader.
- Passing `pid == 0` refers to the calling process.
- Passing `pgid == 0` means to use the target process's PID as the PGID.

Example:

```c
if (setpgid(0, 0) == -1) {
    perror("setpgid");
}
```

This attempts to make the calling process the leader of a new process group.

### 2.2 Sending signals to a process group

A negative PID passed to `kill()` targets a process group, with the exception of special values such as `-1`.

```c
kill(-pgid, SIGTERM);
```

The equivalent convenience function is:

```c
killpg(pgid, SIGTERM);
```

These operations send a signal to eligible members of the target process group, subject to permissions and signal semantics.

This is important for pipelines. If several processes belong to the same job, the shell can signal the entire job instead of sending a signal to each PID individually.

## 3. Sessions

A **session** is a collection of process groups. A session is identified by its session ID (SID).

Sessions organize process groups into a larger unit, especially in relation to controlling terminals.

Useful functions:

```c
pid_t getsid(pid_t pid);
pid_t setsid(void);
```

- `getsid(pid)` obtains the session ID of a process.
- `setsid()` creates a new session, if the calling process is permitted to do so.

A successful `setsid()` call:

1. Creates a new session.
2. Makes the caller the session leader.
3. Creates a new process group with the caller as its leader.
4. Initially leaves the new session without a controlling terminal.

A process group leader cannot successfully call `setsid()`. A common technique is to fork and let the child call `setsid()` when the child is not a process group leader.

Sessions are useful for understanding terminal ownership and daemonization, although `setsid()` alone is not a complete daemonization procedure.

## 4. Controlling terminals

A **controlling terminal** is a terminal associated with a session. An interactive shell typically has a controlling terminal, and the shell's session can contain several process groups.

On a modern Linux desktop, a terminal emulator typically uses a pseudoterminal (PTY). The shell and its child processes interact with the PTY slave, while the terminal emulator communicates through the PTY master.

The terminal device is not the graphical screen itself. The terminal emulator renders received output on the screen.

The following concepts are distinct:

- **Standard input/output:** file descriptors such as `STDIN_FILENO` and `STDOUT_FILENO`.
- **Controlling terminal:** the terminal associated with the process's session.
- **Foreground process group:** the process group currently designated by the terminal as its foreground group.

A process can have standard output redirected to a file while still having a controlling terminal. Similarly, checking `isatty(STDOUT_FILENO)` does not establish whether the descriptor refers to the process's controlling terminal.

## 5. Foreground and background process groups

A terminal has one **foreground process group** at a time. Other process groups in the session may be background groups.

The foreground process group is kernel-managed terminal state. The shell uses this state to implement foreground and background jobs.

### 5.1 Querying and changing the foreground group

```c
#include <unistd.h>

pid_t tcgetpgrp(int fd);
int tcsetpgrp(int fd, pid_t pgrp);
```

- `tcgetpgrp(fd)` returns the PGID of the terminal's foreground process group.
- `tcsetpgrp(fd, pgrp)` requests that the specified process group become the terminal's foreground group.

The descriptor must refer to the appropriate terminal. If it doesn't refer to a terminal, these functions fail with `ENOTTY` in the relevant cases.

A shell typically uses `STDIN_FILENO` if its standard input refers to its controlling terminal.

### 5.2 Foreground command

For:

```bash
sleep 10
```

A typical interactive shell:

1. Creates the child process.
2. Places the child in a new process group.
3. Gives that group foreground ownership of the terminal using `tcsetpgrp()`.
4. Waits for the job to finish or stop.
5. Reclaims foreground ownership for the shell's process group.

The shell must handle synchronization carefully because the child and parent may both participate in setting up the child's process group.

### 5.3 Background command

For:

```bash
sleep 100 &
```

The shell typically creates a separate process group for the job but does not assign it foreground terminal ownership. The shell returns a prompt without waiting for the job to finish.

Background processes can still write to the terminal under normal terminal settings. Attempts to read from the controlling terminal normally trigger `SIGTTIN`.

The shell's job table tracks the background job and its status.

### 5.4 A job is not necessarily one process

Consider:

```bash
cat file.txt | grep error | sort
```

This pipeline normally represents one job containing three processes. All three processes belong to the same process group.

That shared process group lets the shell manage the pipeline as a unit, including foreground terminal ownership and job-control signals.

## 6. Job-control signals

Job control relies on several signals.

| Signal | Default action | Purpose |
|---|---|---|
| `SIGINT` | Terminate | Commonly generated by Ctrl+C |
| `SIGTSTP` | Stop | Commonly generated by Ctrl+Z |
| `SIGTTIN` | Stop | Background process attempts to read from its controlling terminal |
| `SIGTTOU` | Stop | Background process performs certain terminal operations |
| `SIGCONT` | Continue | Resume a stopped process |
| `SIGHUP` | Terminate | Often associated with terminal/session hangup |

### 6.1 Ctrl+C

When the terminal's interrupt character is enabled, pressing Ctrl+C normally causes the terminal driver to send `SIGINT` to the foreground process group.

It is not the shell's standard output descriptor that determines which processes receive the signal. The terminal's foreground process-group state determines the recipients.

### 6.2 Ctrl+Z

Pressing Ctrl+Z normally causes the terminal driver to send `SIGTSTP` to the foreground process group.

`SIGTSTP` has a default action of stopping the process, but it can be caught, ignored, or blocked.

The shell detects stopped child processes using `waitpid()` and options such as `WUNTRACED`, then updates its job table and reclaims terminal ownership.

If a process catches or ignores `SIGTSTP`, pressing Ctrl+Z does not necessarily stop it.

### 6.3 Background terminal reads

If a background process attempts to read from its controlling terminal, the terminal driver normally sends `SIGTTIN` to the process group.

There are exceptions: depending on the signal disposition and process-group circumstances, the operation may instead fail with `EIO`.

### 6.4 Background terminal writes

Background writes to a terminal are normally permitted. If the terminal's `TOSTOP` flag is enabled, a background write can instead trigger `SIGTTOU`.

Certain terminal-control operations can also trigger `SIGTTOU` for background process groups.

### 6.5 `SIGCONT`

`SIGCONT` resumes stopped processes, even if the signal is caught or ignored. If caught, the process can execute its handler when the signal is delivered.

A shell typically sends `SIGCONT` when resuming a stopped job through commands such as `fg` or `bg`.

## 7. Implementing shell job control

A shell needs to coordinate process creation, process groups, terminal ownership, and child status.

For a foreground pipeline, a typical high-level sequence is:

1. Parse the pipeline and create the required pipes.
2. Fork the pipeline's processes.
3. Place all pipeline processes in a shared process group using `setpgid()`.
4. Connect each process's standard input and output to the correct pipe endpoints.
5. Set the pipeline's process group as the terminal's foreground group using `tcsetpgrp()`.
6. Wait for children to exit or stop.
7. If the job exits or stops, reclaim the terminal for the shell.
8. Update the job table.

For a background pipeline, the shell typically performs the same process-group setup but leaves the shell's process group in the terminal foreground and returns the prompt without waiting for job completion.

### 7.1 Waiting for stopped children

The shell can use:

```c
int status;
pid_t pid = waitpid(-1, &status, WUNTRACED);

if (pid > 0 && WIFSTOPPED(status)) {
    int sig = WSTOPSIG(status);
    /* Record the stopped process in the job table. */
}
```

`WUNTRACED` allows `waitpid()` to report stopped children that aren't being traced.

For a pipeline, the shell must track all relevant children to determine whether the job has stopped, completed, or still has running processes.

For more complete job control, the shell must also account for continued children, commonly using `WCONTINUED`, and for child exit notifications.

### 7.2 Avoiding races

Process-group setup can involve races between parent and child. Both may call `setpgid()` as part of establishing the intended process group.

A shell must ensure that process-group setup and terminal handoff occur in a safe order. Otherwise, a child may execute or interact with the terminal before the shell has completed job setup.

Job-control implementations also need to account for the shell being in the background. A shell that manipulates terminal foreground ownership while in a background process group can be subject to `SIGTTOU`.

These details matter when implementing a robust interactive shell.

## 8. Orphaned process groups

A process group is considered **orphaned** when its members have no parent process in the same session that is outside the process group, subject to the POSIX definition.

Orphaned process groups matter because job-control signals behave differently in certain circumstances.

For example, when a process group becomes orphaned while it contains stopped processes, the system can send `SIGHUP` followed by `SIGCONT` to members of that group.

This behavior helps avoid stopped jobs being left permanently suspended after the shell or another relevant parent process disappears.

An orphaned process group is not necessarily a group of abandoned processes, nor is it the same thing as a zombie process.

## 9. Important distinctions

### Process group vs. session

- A process group groups processes for job control and group-directed signals.
- A session contains process groups and can be associated with a controlling terminal.

### Shell job vs. process group

- A shell job is a shell-level abstraction tracked in the job table.
- A process group is a kernel-managed grouping mechanism.
- A shell commonly uses one process group per job, but the concepts are not identical.

### Foreground job vs. foreground process group

- The shell considers a job to be in the foreground when it manages that job as the active terminal job.
- The kernel maintains the terminal's foreground process group.
- The shell coordinates the two using `tcsetpgrp()`, `waitpid()`, and process-group management.

### `SIGTSTP` vs. `SIGSTOP`

Both stop a process by default, but `SIGTSTP` can be caught, blocked, or ignored. `SIGSTOP` cannot.

## Summary

Linux process groups and sessions provide the foundation for job control. The shell builds foreground/background job management on top of these mechanisms, while the kernel enforces terminal foreground ownership and delivers terminal-generated signals to the appropriate process groups.

For a systems engineer, this chapter connects process creation, signals, terminal I/O, and process lifecycle management into one coherent model.