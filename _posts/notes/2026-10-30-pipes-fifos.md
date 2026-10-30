---
title: "Linux Pipes and FIFOs: Interprocess Communication"
categories:
  - Learning Notes
  - System Programming
tags: [linux, ipc, pipes, fifos, fork, exec, dup2, pipe-buf]
---

# Linux Pipes and FIFOs: Interprocess Communication

Pipes and FIFOs are fundamental mechanisms for **interprocess communication (IPC)** in Unix and Linux.

They allow processes to exchange data through a byte stream using familiar file I/O operations:

```c
read()
write()
close()
```

This is another example of the Unix philosophy: different resources can be accessed through a common file-descriptor interface.

There are two main types:

- **Anonymous pipes:** Usually used for communication between related processes.
- **FIFOs (named pipes):** Can be used for communication between unrelated processes.

---

# 1. What Is a Pipe?

A pipe is a unidirectional communication channel with two ends:

```text
        Write end                  Read end
            │                         │
            ▼                         ▼
      ┌──────────┐               ┌──────────┐
      │ Process A│               │ Process B│
      └────┬─────┘               └────▲─────┘
           │                          │
           │ write()                  │ read()
           ▼                          │
      ┌──────────────────────────────────┐
      │            Pipe buffer           │
      └──────────────────────────────────┘
```

One process writes data into the pipe, and another process reads that data.

Important properties:

- A pipe is a byte stream.
- It has no message boundaries.
- It is normally unidirectional.
- Data is read in the order it was written.
- A pipe has finite capacity.
- Pipes are accessed through file descriptors.

For example, if a process writes:

```c
write(pipefd[1], "Hello", 5);
```

another process can read those bytes:

```c
char buffer[10];

ssize_t n = read(pipefd[0], buffer, sizeof(buffer));
```

The reader receives bytes, not a complete message object.

For example, a single write of 100 bytes might be consumed by several reads. Likewise, one read can consume data from multiple writes.

Therefore, applications that need message boundaries must define their own protocol.

---

# 2. Creating a Pipe with `pipe()`

The `pipe()` system call creates a pipe and returns two file descriptors.

```c
#include <unistd.h>

int pipe(int pipefd[2]);
```

The descriptors have these meanings:

```text
pipefd[0] → Read end
pipefd[1] → Write end
```

Example:

```c
int pipefd[2];

if (pipe(pipefd) == -1) {
    perror("pipe");
    exit(EXIT_FAILURE);
}
```

After successful execution:

```text
pipefd[0] ────────┐
                 │
              Pipe buffer
                 │
pipefd[1] ────────┘
```

We can use these descriptors with ordinary file I/O:

```c
write(pipefd[1], "Linux", 5);

char buffer[10];
ssize_t n = read(pipefd[0], buffer, sizeof(buffer));
```

The important point is that a pipe is not a regular file. It does not support arbitrary file offsets.

For example, calling `lseek()` on a pipe fails with `ESPIPE`.

---

# 3. Pipes and `fork()`

Anonymous pipes are commonly used between related processes.

A typical pattern is:

1. Create a pipe.
2. Call `fork()`.
3. The parent and child inherit the pipe descriptors.
4. Close the descriptors that each process does not need.
5. Communicate through the remaining descriptors.

Example:

```c
int pipefd[2];

pipe(pipefd);

pid_t pid = fork();

if (pid == 0) {
    // Child: read from the pipe.
    close(pipefd[1]);

    char buffer[100];
    ssize_t n = read(pipefd[0], buffer, sizeof(buffer));

    close(pipefd[0]);
    _exit(0);
} else {
    // Parent: write to the pipe.
    close(pipefd[0]);

    write(pipefd[1], "Hello child", 11);

    close(pipefd[1]);
    wait(NULL);
}
```

The resulting structure is:

```text
                 Parent
              ┌──────────┐
              │ write()  │
              └────┬─────┘
                   │
                   ▼
              ┌──────────┐
              │   Pipe   │
              └────┬─────┘
                   │
                   ▼
              ┌──────────┐
              │  read()  │
              └──────────┘
                  Child
```

## Why must unused descriptors be closed?

This is one of the most important details about pipes.

Suppose the child reads from the pipe. It expects EOF after the parent closes its write end.

However, if the child still has its own write descriptor open, the kernel considers that a writer still exists.

Consequently, the child's `read()` may block instead of returning EOF.

This means that closing unused pipe descriptors is not just resource cleanup. It is part of the communication protocol.

---

# 4. Pipe EOF Semantics

A pipe reader receives EOF only when:

1. The pipe buffer is empty.
2. All file descriptors referring to the write end have been closed.

Consider:

```text
Parent ── write end ── Pipe ── read end ── Child
```

When the parent closes its write descriptor:

```c
close(pipefd[1]);
```

the child does not necessarily receive EOF immediately.

If data remains in the pipe buffer, the child can still read it.

The sequence is:

```text
Parent closes write end
          │
          ▼
Pipe still contains data?
          │
      ┌───┴───┐
      │       │
     Yes      No
      │       │
      ▼       ▼
 Read data   read() returns 0
```

The essential rule is:

> EOF is returned only after all writers have closed their descriptors and all buffered data has been consumed.

This is why accidentally retaining a write descriptor can prevent a reader from detecting EOF.

---

# 5. Blocking Behavior

Pipes have finite buffer capacity.

Their behavior depends on whether data is available and whether the pipe is full.

## 5.1 Reading from an empty pipe

If the pipe is empty but at least one write descriptor remains open:

```c
read(pipefd[0], buffer, sizeof(buffer));
```

the call blocks until data becomes available.

If all write descriptors are closed and the pipe is empty, `read()` returns:

```text
0
```

This indicates EOF.

## 5.2 Writing to a full pipe

If the pipe buffer is full, a blocking `write()` waits until space becomes available.

```c
write(pipefd[1], buffer, size);
```

The reader must consume data before the writer can make further progress.

## 5.3 Nonblocking I/O

A pipe can be configured for nonblocking operation using `fcntl()`:

```c
int flags = fcntl(pipefd[0], F_GETFL);

fcntl(pipefd[0], F_SETFL, flags | O_NONBLOCK);
```

With `O_NONBLOCK`:

| Operation | Behavior |
|---|---|
| Read from empty pipe with writers | Returns `-1`, `errno = EAGAIN` |
| Write to full pipe | Returns `-1`, `errno = EAGAIN` |
| Read from empty pipe with no writers | Returns `0` (EOF) |

Nonblocking I/O is useful when building event-driven applications using `poll()` or `epoll()`.

---

# 6. Pipe Capacity and `PIPE_BUF`

Two concepts must be distinguished:

- **Pipe capacity:** The total amount of data that can be buffered in the pipe.
- **`PIPE_BUF`:** The maximum write size for which writes are guaranteed to be atomic under the applicable pipe semantics.

These are not the same thing.

## 6.1 Pipe capacity

The pipe buffer has a finite capacity.

If the buffer becomes full, a blocking writer waits until space is available.

On Linux, the capacity can be queried and adjusted using `fcntl()`:

```c
int capacity = fcntl(pipefd[0], F_GETPIPE_SZ);

fcntl(pipefd[0], F_SETPIPE_SZ, 65536);
```

The requested size may be subject to kernel limits and permissions.

## 6.2 Atomic writes and `PIPE_BUF`

`PIPE_BUF` defines the maximum write size that is guaranteed not to interleave with writes from other processes writing to the same pipe.

On Linux, `PIPE_BUF` is 4096 bytes.

Suppose two processes write to the same pipe:

```text
Process A: write("AAAA", 4)
Process B: write("BBBB", 4)
```

Because these writes are smaller than `PIPE_BUF`, their bytes will not be interleaved.

The reader can observe:

```text
AAAABBBB
```

or:

```text
BBBBAAAA
```

but not:

```text
AABBAABB
```

This guarantee concerns the atomicity of each write operation. It does not guarantee that one `read()` returns one complete write.

## 6.3 Writes larger than `PIPE_BUF`

For writes larger than `PIPE_BUF`, data from multiple writers may interleave.

Therefore, if multiple processes write to the same pipe and need atomic records, they should keep each record within `PIPE_BUF` or use an explicit synchronization mechanism.

---

# 7. Broken Pipes: `SIGPIPE` and `EPIPE`

What happens if a process writes to a pipe after all read descriptors have been closed?

The kernel generates `SIGPIPE` for the writer.

By default, `SIGPIPE` terminates the process.

If the signal is ignored or handled, `write()` fails with:

```text
-1
errno = EPIPE
```

Example:

```c
ssize_t n = write(pipefd[1], buffer, size);

if (n == -1 && errno == EPIPE) {
    // There are no readers.
}
```

The distinction is important:

- `SIGPIPE` is the signal-level notification.
- `EPIPE` is the error returned by `write()` when the process is allowed to continue.

This behavior is common in shell pipelines.

For example:

```bash
yes | head -n 5
```

`head` exits after reading five lines. Once its read end closes, `yes` may receive `SIGPIPE` when it tries to write again.

---

# 8. Pipes and Standard I/O

A pipe provides file descriptors, while the C standard I/O library provides `FILE *` streams.

These are different interfaces.

You can convert a file descriptor into a standard I/O stream using:

```c
FILE *stream = fdopen(pipefd[0], "r");
```

Then use:

```c
char buffer[100];

fgets(buffer, sizeof(buffer), stream);
```

Remember:

- `read()` and `write()` operate directly on file descriptors.
- `fread()`, `fwrite()`, and `fgets()` operate on buffered `FILE *` streams.
- Mixing standard I/O and raw file-descriptor operations on the same underlying open file description requires care because of buffering and file-offset synchronization.

When using `fdopen()`, the stream takes responsibility for closing the descriptor when `fclose()` is called.

---

# 9. How Shells Implement Pipelines

One of the most important applications of pipes is shell pipelines.

Consider:

```bash
ls -l | grep ".txt"
```

The shell must connect:

```text
ls stdout → grep stdin
```

It accomplishes this using:

- `pipe()`
- `fork()`
- `dup2()`
- `execve()`
- `close()`

## 9.1 Creating the pipe

```c
int pipefd[2];

pipe(pipefd);
```

The descriptors are:

```text
pipefd[0] → Read end
pipefd[1] → Write end
```

## 9.2 Creating the first process

The shell forks the process that executes `ls`.

In the child:

```c
dup2(pipefd[1], STDOUT_FILENO);

close(pipefd[0]);
close(pipefd[1]);

execvp("ls", argv);
```

The important operation is:

```c
dup2(pipefd[1], STDOUT_FILENO);
```

It makes standard output refer to the pipe's write end.

Therefore, when `ls` writes to file descriptor 1, its output enters the pipe.

## 9.3 Creating the second process

The shell forks the process that executes `grep`.

In this child:

```c
dup2(pipefd[0], STDIN_FILENO);

close(pipefd[0]);
close(pipefd[1]);

execvp("grep", argv);
```

Now standard input refers to the pipe's read end.

When `grep` reads from file descriptor 0, it receives the output produced by `ls`.

## 9.4 Closing descriptors in the parent

After creating both children, the shell must close its own copies:

```c
close(pipefd[0]);
close(pipefd[1]);
```

Otherwise, the shell itself might keep the write end open and prevent `grep` from receiving EOF.

The complete architecture is:

```text
                  Shell
                    │
                  pipe()
                    │
          ┌─────────┴─────────┐
          │                   │
        fork()              fork()
          │                   │
          ▼                   ▼
      ┌────────┐          ┌────────┐
      │   ls   │          │  grep  │
      └───┬────┘          └───▲────┘
          │                   │
       stdout               stdin
          │                   │
          ▼                   │
      ┌─────────────────────────┐
      │          Pipe           │
      └─────────────────────────┘
```

The key idea is:

> `dup2()` changes where a process's standard input or output points. The executed program does not need to know that it is communicating through a pipe.

This is exactly the abstraction that makes Unix pipelines powerful.

---

# 10. Multiple Processes and Pipelines

A pipeline can contain more than two processes:

```bash
cat file.txt | grep error | sort | uniq
```

Conceptually:

```text
cat
 │
 ▼
Pipe 1
 │
 ▼
grep
 │
 ▼
Pipe 2
 │
 ▼
sort
 │
 ▼
Pipe 3
 │
 ▼
uniq
```

For each connection, the shell creates a pipe and configures the corresponding standard input and output descriptors.

All processes in a foreground pipeline normally belong to the same process group. This allows the shell to manage the pipeline as one job.

For a Rust shell, this means pipeline execution needs more than simply creating a pipe:

- Create all required pipes.
- Fork the pipeline processes.
- Set up each child's standard input and output.
- Close unused descriptors in every child.
- Close the shell's copies of the pipe descriptors.
- Execute each program.
- Wait for the pipeline according to the shell's job-control rules.

---

# 11. What Is a FIFO?

A FIFO is a named pipe.

Unlike an anonymous pipe, a FIFO has a filesystem name and can be opened by unrelated processes.

Create one using:

```c
#include <sys/stat.h>

mkfifo("/tmp/myfifo", 0666);
```

The FIFO appears in the filesystem:

```bash
ls -l /tmp/myfifo
```

Its file type is represented by `p`:

```text
prw-r--r-- 1 user user 0 ... /tmp/myfifo
```

The filesystem entry provides a way for independent processes to locate the same communication channel.

However, the FIFO's data is not stored as an ordinary file on disk. It is transferred through a kernel-managed pipe buffer.

A FIFO is opened and accessed using normal file operations:

```c
int fd = open("/tmp/myfifo", O_WRONLY);

write(fd, "Hello", 5);

close(fd);
```

Another process can open it for reading:

```c
int fd = open("/tmp/myfifo", O_RDONLY);

char buffer[100];
ssize_t n = read(fd, buffer, sizeof(buffer));

close(fd);
```

---

# 12. Anonymous Pipes vs. FIFOs

| Property | Anonymous pipe | FIFO |
|---|---|---|
| Creation | `pipe()` | `mkfifo()` |
| Filesystem name | No | Yes |
| Communication | Usually related processes | Related or unrelated processes |
| Data transfer | Byte stream | Byte stream |
| File I/O interface | `read()` / `write()` | `read()` / `write()` |
| Supports `lseek()` | No | No |
| Kernel buffer | Yes | Yes |

The most important difference is how processes obtain access to the communication channel.

With an anonymous pipe, processes usually inherit descriptors through `fork()`.

With a FIFO, independent processes can open the same filesystem path.

---

# 13. Opening a FIFO

Opening a FIFO has special synchronization behavior.

## 13.1 Opening for reading

```c
open(path, O_RDONLY);
```

Normally blocks until another process opens the FIFO for writing.

## 13.2 Opening for writing

```c
open(path, O_WRONLY);
```

Normally blocks until another process opens the FIFO for reading.

This means opening the FIFO can itself synchronize two processes.

```text
Reader                         Writer
  │                              │
  │ open(O_RDONLY)               │
  │ blocks                       │
  │                              │
  │                         open(O_WRONLY)
  │                              │
  └──────── Both opens complete ─┘
```

## 13.3 Nonblocking FIFO opens

With `O_NONBLOCK`, the behavior changes:

| Operation | Behavior |
|---|---|
| `open(O_RDONLY \| O_NONBLOCK)` | Succeeds even if no writer exists |
| `open(O_WRONLY \| O_NONBLOCK)` | Fails with `ENXIO` if no reader exists |

This is useful for applications that cannot afford to block during FIFO initialization.

---

# 14. FIFO EOF and Broken Pipes

FIFO read and write behavior is similar to anonymous pipes.

## EOF

When all write descriptors are closed and the FIFO buffer becomes empty:

```c
read(fd, buffer, sizeof(buffer));
```

returns:

```text
0
```

The reader sees EOF.

If a new writer later opens the FIFO, the reader can receive new data.

## Broken pipe

If all read descriptors are closed, a writer attempting to write receives `SIGPIPE`.

If `SIGPIPE` is ignored or handled, `write()` fails with `EPIPE`.

These behaviors make descriptor lifetime important when designing FIFO-based protocols.

---

# 15. FIFO Client-Server Communication

FIFOs can be used to build simple client-server applications.

A common design is:

- The server creates a well-known FIFO.
- Clients write requests to that FIFO.
- Each client creates its own FIFO for responses.
- The server reads requests and writes responses to the corresponding client FIFO.

Architecture:

```text
                 Client 1
              ┌────────────┐
              │ Request    │
              └─────┬──────┘
                    │
                    ▼
             Server request FIFO
                    │
                    ▼
              ┌────────────┐
              │   Server   │
              └─────┬──────┘
                    │
                    ▼
             Client 1 response FIFO
                    │
                    ▼
                 Client 1
```

A request might contain:

```c
struct request {
    pid_t client_pid;
    char response_fifo[256];
    int requested_operation;
};
```

The server uses the client's information to locate the response FIFO.

In a real application, the protocol should define:

- How requests are framed.
- How client identity is represented.
- How response FIFO paths are validated.
- What happens when a client exits unexpectedly.
- How concurrent clients are handled.
- How the server handles malformed or incomplete requests.

## Why use a separate FIFO for each client?

If all clients read from one shared response FIFO, they could consume each other's responses.

A dedicated response FIFO gives each client its own communication channel.

---

# 16. The Dummy Writer Technique

A FIFO reader normally receives EOF when all writers have closed the FIFO and the buffer is empty.

Consider a server:

```c
int serverFd = open(serverFifo, O_RDONLY);
```

When the last client closes its write descriptor, the server eventually receives EOF.

One possible approach is:

1. Detect EOF.
2. Close the read descriptor.
3. Open the FIFO again for reading.

But this approach has a race condition.

## The race condition

Suppose:

1. The server's `read()` returns 0 because there are currently no writers.
2. Before the server closes its read descriptor, a new client opens the FIFO for writing.
3. The client writes a request and closes its descriptor.
4. The server closes its read descriptor after detecting EOF.
5. The FIFO's unread buffered data is discarded when the last read descriptor is closed.

The client's request is lost.

## The dummy writer solution

Instead of closing and reopening the read descriptor, the server opens a second descriptor for writing:

```c
int serverFd = open(serverFifo, O_RDONLY);
int dummyFd = open(serverFifo, O_WRONLY);
```

The second descriptor is not used to send data.

Its purpose is to keep a writer open.

```text
                 Server
          ┌──────────────────┐
          │                  │
          │ serverFd         │
          │ O_RDONLY         │
          │                  │
          │ dummyFd          │
          │ O_WRONLY         │
          └───────┬──────────┘
                  │
                  ▼
             Server FIFO
                  ▲
                  │
             Real clients
```

Because the dummy write descriptor remains open:

- The server does not receive EOF merely because all real clients have disconnected.
- When the FIFO buffer is empty, the server's `read()` blocks waiting for data.
- The server avoids repeatedly closing and reopening the FIFO.
- It avoids the race condition that could discard a client's request.

The dummy descriptor should remain open for the lifetime of the server's FIFO-reading loop and be closed during shutdown.

---

# 17. Important Distinction: Pipe Data Is a Byte Stream

Neither anonymous pipes nor FIFOs preserve message boundaries.

Suppose a client writes:

```c
write(fd, "REQUEST_A", 9);
write(fd, "REQUEST_B", 9);
```

The server might read:

```text
REQUEST_AREQUEST_B
```

in one read, or receive the data in smaller chunks.

Therefore, this is not a safe assumption:

```text
one write() = one read()
```

Instead, applications should define a framing protocol.

Common approaches include:

- Fixed-size messages.
- Length-prefixed messages.
- Delimiter-terminated messages.
- Structured binary protocols.

For example, a length-prefixed protocol could look like:

```text
┌──────────────┬────────────────────────┐
│ Length: 100  │ 100 bytes of payload   │
└──────────────┴────────────────────────┘
```

The receiver first reads the length and then reads exactly that many payload bytes.

When multiple processes write to the same FIFO, keeping each complete request within `PIPE_BUF` can also help ensure that concurrent writes do not interleave.

---

# 18. Common Mistakes

### Mistake 1: Forgetting to close unused descriptors

This can prevent EOF from being delivered.

### Mistake 2: Assuming `read()` returns one complete message

Pipes and FIFOs are byte streams, not message-oriented channels.

### Mistake 3: Confusing `PIPE_BUF` with pipe capacity

`PIPE_BUF` defines atomic-write guarantees, not the total buffer size.

### Mistake 4: Ignoring `SIGPIPE`

A writer may be terminated when the reader disappears.

### Mistake 5: Assuming FIFOs store data on disk

The filesystem entry names the FIFO, but the data passes through a kernel-managed buffer.

### Mistake 6: Reopening a FIFO after EOF without considering races

A client may write between the server's EOF detection and its closing of the read descriptor. Closing the last read descriptor can discard buffered data.

### Mistake 7: Forgetting to close pipe descriptors in the shell

The shell may accidentally keep a writer open and prevent the next pipeline process from receiving EOF.

---

# 19. Pipes and FIFOs Compared with Other IPC Mechanisms

Pipes and FIFOs are useful, but they are not appropriate for every communication requirement.

| Mechanism | Main characteristics |
|---|---|
| Anonymous pipe | Simple byte-stream IPC, usually between related processes |
| FIFO | Named byte-stream IPC between independent processes |
| Unix domain socket | Bidirectional communication, message-oriented options, and FD passing |
| TCP socket | Network communication between processes |
| Shared memory | Multiple processes access the same memory region |
| Message queue | Kernel-managed messages with message boundaries |

Choose pipes and FIFOs when a simple byte stream is sufficient.

Choose Unix domain sockets when you need more advanced local IPC features, such as bidirectional communication or passing file descriptors.

Choose shared memory when high-throughput data exchange justifies the additional synchronization complexity.

---

# 20. The Important Mental Model

Keep these concepts separate:

**Pipe/FIFO descriptor**

Identifies one end of a communication channel:

```text
read end  → consumes data
write end → produces data
```

**Pipe buffer**

Temporarily holds bytes between writers and readers:

```text
writer → kernel buffer → reader
```

**Descriptor lifetime**

Determines important behavior:

```text
All writers closed + buffer empty → EOF

All readers closed + writer writes → SIGPIPE / EPIPE
```

**Atomicity**

`PIPE_BUF` determines whether concurrent writes can interleave.

**Message framing**

The application protocol determines where one logical message ends and another begins.

---

# 21. Final Takeaways

The central ideas of Chapter 44 are:

1. Pipes provide unidirectional byte-stream communication.
2. `pipe()` creates a pair of file descriptors.
3. `fork()` allows parent and child processes to share pipe descriptors.
4. Closing unused descriptors is essential for correct EOF behavior.
5. Pipes have finite capacity, so blocking I/O is part of their synchronization behavior.
6. `PIPE_BUF` provides atomic-write guarantees; it is not the pipe's capacity.
7. Writing without readers can generate `SIGPIPE` or return `EPIPE`.
8. Shells use `pipe()`, `fork()`, and `dup2()` to implement pipelines.
9. FIFOs provide named communication channels for unrelated processes.
10. FIFO `open()` behavior can synchronize readers and writers.
11. Pipes and FIFOs are byte streams, so applications must define message framing.
12. Keeping a dummy write descriptor open is a useful FIFO-server technique for avoiding EOF/reopen races.

The most important connection to shell development is how `pipe()`, `fork()`, `dup2()`, `exec()`, and `close()` work together to implement pipelines. Understanding descriptor inheritance and EOF semantics is essential for implementing this correctly.
