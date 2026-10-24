# 6. Inter-Process Communication (IPC)

## 6.1 IPC Fundamentals

### Definition

Inter-Process Communication (IPC) refers to mechanisms that allow separate processes to exchange data and coordinate their execution.

Processes normally have isolated virtual address spaces, so one process cannot directly access another process's private memory.

The OS provides controlled mechanisms for communication.

### How It Works

IPC mechanisms generally fall into two categories:

**1. Message passing**

Data is exchanged through a communication channel, usually managed by the kernel.

Examples:
- Pipes.
- Unix domain sockets.
- Message queues.

**2. Shared memory**

Multiple processes map the same underlying physical memory into their virtual address spaces.

```text
Message Passing:

Process A         Kernel          Process B
+---------+     +---------+     +---------+
| Data    | --> | Channel | --> | Data    |
+---------+     +---------+     +---------+


Shared Memory:

Process A                       Process B
+---------+                     +---------+
| Mapping | ----+         +---- | Mapping |
+---------+     |         |     +---------+
                v         v
             +---------------+
             | Shared Memory |
             +---------------+
```

### Key Concepts

| Concept | Explanation |
|---|---|
| IPC | Communication between separate processes |
| Message Passing | Exchanges discrete messages or byte streams |
| Shared Memory | Multiple processes access the same physical memory |
| Synchronization | Coordinates access to shared resources |
| Blocking IPC | May suspend a process until progress is possible |
| Zero-Copy | Avoids one or more unnecessary data-copying operations |

### Interview Questions

**Q: Why is IPC necessary?**

Processes normally have isolated address spaces. IPC provides controlled ways to exchange information without removing that isolation entirely.

**Q: What is the difference between message passing and shared memory?**

Message passing transfers data through a communication abstraction.

Shared memory allows processes to access the same underlying memory directly, but they must coordinate access themselves.

**Q: Which IPC mechanism is best for a low-latency trading application?**

It depends on the architecture.

Shared memory is attractive for high-frequency, large-volume local communication because it can avoid repeated payload copying and system calls.

However, it requires careful synchronization, memory layout, and failure handling.

---

## 6.2 Pipes

### Definition

A pipe is a kernel-managed communication channel that transfers a stream of bytes between processes.

A traditional Unix pipe is unidirectional: one endpoint writes data, and the other reads it.

### How It Works

A pipe is created using `pipe()` or `pipe2()`.

```cpp
int fd[2];
pipe(fd);

// fd[0]: Read endpoint.
// fd[1]: Write endpoint.
```

Conceptually:

```text
Process A              Kernel              Process B

write(fd[1])      +---------------+       read(fd[0])
    |             | Pipe Buffer   |            ^
    +-----------> | A B C D E     | -----------+
                  +---------------+
```

The kernel maintains an internal pipe buffer.

When a process writes, bytes enter the pipe. When another process reads, bytes are removed.

This is also how shell pipelines work:

```bash
cat file.txt | grep ERROR | sort
```

The shell creates pipes and connects the standard output of one command to the standard input of the next.

### Key Concepts

| Concept | Explanation |
|---|---|
| `pipe()` | Creates a unidirectional pipe |
| Read Endpoint | File descriptor used to receive bytes |
| Write Endpoint | File descriptor used to send bytes |
| Kernel Buffer | Temporary storage for pipe contents |
| EOF | Reader observes end-of-file after all write endpoints close and buffered data is consumed |
| `PIPE_BUF` | Size threshold for guaranteed atomic writes under specified conditions |

### Interview Questions

**Q: How does the shell transfer data between commands using `|`?**

The shell creates a pipe, starts the commands, and redirects their standard input/output file descriptors.

For example:

```bash
command_a | command_b
```

`command_a` writes bytes into the pipe through standard output.

`command_b` reads those bytes from the pipe through standard input.

The commands can execute concurrently.

**Q: Does a pipe preserve message boundaries?**

No. A pipe is a byte stream.

A reader may receive fewer or more bytes than one logical message, depending on how the application performs its reads.

**Q: What happens when a pipe becomes full?**

A blocking write waits until space becomes available.

A non-blocking write may fail with `EAGAIN` or `EWOULDBLOCK`.

**Q: Are pipes suitable for extremely low-latency communication?**

They are simple and efficient, but data passes through a kernel-managed channel and may involve system calls, copying, and scheduling.

Shared memory may be preferable when minimizing these costs is essential.

---

## 6.3 Named Pipes (FIFO)

### Definition

A named pipe, also called a FIFO, is a pipe exposed through a filesystem pathname.

Unlike an anonymous pipe, independently started processes can open the same FIFO without inheriting file descriptors from a common parent.

### How It Works

Create a FIFO:

```bash
mkfifo /tmp/my_fifo
```

Process A writes:

```bash
echo "Hello" > /tmp/my_fifo
```

Process B reads:

```bash
cat /tmp/my_fifo
```

Conceptually:

```text
Process A
    |
    | open("/tmp/my_fifo")
    | write()
    v
+-------------------+
| Kernel FIFO       |
| Byte Stream       |
+-------------------+
    |
    | read()
    v
Process B
```

The pathname identifies the communication channel; the transmitted data is not stored in that filesystem entry like ordinary file contents.

### Key Concepts

| Concept | Explanation |
|---|---|
| FIFO | First-In, First-Out byte-stream channel |
| `mkfifo()` | Creates a named pipe |
| Filesystem Path | Allows unrelated processes to locate the channel |
| Blocking Open | Opening a FIFO can wait for a corresponding reader or writer |
| Non-blocking Mode | Changes how operations behave when peers or capacity are unavailable |

### Interview Questions

**Q: What is the difference between a pipe and a named pipe?**

An anonymous pipe is typically created using `pipe()` and shared through inherited or transferred file descriptors.

A named pipe has a filesystem pathname that unrelated processes can open.

**Q: Does a FIFO store data permanently?**

No. It is a communication channel, not a persistent data file.

**Q: When would you use a FIFO instead of a Unix domain socket?**

A FIFO is useful for simple one-way byte-stream communication.

Unix domain sockets are generally more flexible when bidirectional communication, multiple clients, or richer messaging features are required.

---

## 6.4 Shared Memory

### Definition

Shared memory allows multiple processes to map the same underlying physical memory into their separate virtual address spaces.

After setup, processes can exchange data through ordinary memory loads and stores.

### How It Works

One common Linux approach uses POSIX shared memory:

1. Create or open a shared-memory object using `shm_open()`.
2. Set its size using `ftruncate()`.
3. Map it using `mmap()` with `MAP_SHARED`.
4. Other processes open and map the same object.
5. Processes access the shared region through their respective virtual addresses.

```text
Process A                         Process B

Virtual Address                   Virtual Address
0x70000000                        0x50000000
    |                                 |
    v                                 v
+-----------+                     +-----------+
| Mapping A |                     | Mapping B |
+-----------+                     +-----------+
       \                             /
        \                           /
         v                         v
         +-------------------------+
         | Shared Physical Pages   |
         +-------------------------+
```

The virtual addresses do not need to be identical.

### Key Concepts

| Concept | Explanation |
|---|---|
| `shm_open()` | Creates or opens a POSIX shared-memory object |
| `ftruncate()` | Sets the size of the underlying object |
| `mmap()` | Maps shared memory into a process |
| `MAP_SHARED` | Enables shared mapping semantics |
| `shm_unlink()` | Removes the shared-memory object's name |
| `memfd_create()` | Creates an anonymous RAM-backed file descriptor suitable for mapping and sharing |

### Interview Questions

**Q: Why is shared memory generally faster than pipes for frequent large data exchanges?**

After the mapping is established, processes can access shared data directly without repeatedly copying the payload through a kernel-managed pipe buffer.

**Q: Does shared memory eliminate system calls completely?**

No.

Creating and managing the mapping requires kernel operations. Synchronization or notification may also involve system calls.

However, ordinary reads and writes to an established mapping do not require a system call per access.

**Q: Can two processes map shared memory at different virtual addresses?**

Yes.

They can map the same physical pages at different virtual addresses.

This is why shared-memory data structures commonly use offsets rather than raw process-local pointers.

**Q: What are the main disadvantages of shared memory?**

- Synchronization complexity.
- Potential data races.
- Cache-coherence contention.
- Difficult failure recovery.
- Memory layout and lifetime management.

---

## 6.5 Shared Memory Synchronization

### Definition

Shared-memory synchronization ensures that processes accessing the same physical memory observe valid, correctly ordered data without corrupting shared state.

Sharing physical memory alone does not make concurrent access safe.

### How It Works

Consider a producer writing data into a shared-memory region and a consumer reading it.

```text
Producer                          Consumer
--------                          --------

Write message payload
       |
       v
Publish ready flag  ---------->   Observe ready flag
                                      |
                                      v
                                  Read payload
```

The producer must finish writing the payload before publishing that it is ready.

The consumer must observe the publication before reading the payload.

Possible synchronization mechanisms include:

- Process-shared mutexes.
- Process-shared semaphores.
- Suitable lock-free atomic protocols.
- Futex-based synchronization.

For example, POSIX mutexes can be configured for inter-process use with:

```cpp
pthread_mutexattr_setpshared(
    &attr,
    PTHREAD_PROCESS_SHARED
);
```

The mutex itself must reside in memory shared between the participating processes.

### Key Concepts

| Concept | Explanation |
|---|---|
| Process-Shared Mutex | Mutex usable by threads in separate processes |
| Acquire/Release | Ordering semantics for publishing and consuming shared data |
| Memory Visibility | Ensuring another participant observes completed writes |
| Cache Coherence | Hardware mechanism maintaining coherent cached memory |
| Robust Mutex | Mutex capable of reporting that its previous owner terminated |
| Lock-Free IPC | Shared-memory communication using atomic coordination rather than blocking locks |

### Interview Questions

**Q: Is `volatile` sufficient for shared-memory synchronization?**

No.

`volatile` does not provide the required atomicity or inter-thread synchronization guarantees.

Use synchronization primitives designed for concurrent access.

**Q: Can `std::mutex` automatically be placed in shared memory and used between processes?**

No portable C++ guarantee provides this behavior.

On Linux, explicitly configured process-shared POSIX mutexes are a suitable option.

**Q: Why might a robust mutex be useful for shared-memory IPC?**

If a process dies while holding a robust mutex, another process can detect the owner-death condition and attempt to recover the protected shared state.

**Q: What synchronization design is attractive for a low-latency producer-consumer pipeline?**

An SPSC ring buffer in shared memory can provide efficient communication with one writer per index.

The producer and consumer coordinate through properly implemented atomic publication and consumption.

For inter-process use, the atomic implementation and shared-memory ABI must be suitable for the target platform.

---

## 6.6 Unix Domain Sockets

### Definition

Unix domain sockets provide communication between processes on the same machine using the socket API.

Unlike TCP/IP sockets, they do not require communication through an external network.

### How It Works

Unix domain sockets use the `AF_UNIX` address family.

They support multiple socket types:

| Type | Description |
|---|---|
| `SOCK_STREAM` | Reliable, ordered byte stream |
| `SOCK_DGRAM` | Datagram-oriented communication with message boundaries |
| `SOCK_SEQPACKET` | Reliable, ordered, connection-oriented communication preserving message boundaries |

Conceptually:

```text
Process A                         Process B
+-----------+                     +-----------+
| Socket FD |                     | Socket FD |
+-----------+                     +-----------+
      |                                 ^
      v                                 |
   send()                           recv()
      |                                 ^
      +---------- Kernel ---------------+
```

Unix domain sockets can also transfer file descriptors between processes using ancillary data and `SCM_RIGHTS`.

### Key Concepts

| Concept | Explanation |
|---|---|
| `AF_UNIX` | Unix domain socket address family |
| `SOCK_STREAM` | Ordered byte-stream communication |
| `SOCK_DGRAM` | Datagram communication |
| `SOCK_SEQPACKET` | Ordered, message-preserving communication |
| `SCM_RIGHTS` | Mechanism for passing file descriptors |
| Peer Credentials | Kernel-provided information about a local socket peer |

### Interview Questions

**Q: What is the difference between Unix domain sockets and TCP sockets?**

Unix domain sockets communicate locally on the same machine.

TCP sockets communicate through the TCP/IP networking stack and can connect processes on different machines.

Unix domain sockets generally avoid some networking protocol overhead.

**Q: Do Unix domain stream sockets preserve message boundaries?**

No.

Like TCP, `SOCK_STREAM` provides a byte stream.

Use an explicit framing protocol or a message-preserving socket type when message boundaries matter.

**Q: Can one process pass an open file descriptor to another process?**

Yes.

Unix domain sockets support file-descriptor passing through `sendmsg()` and `SCM_RIGHTS`.

**Q: When would you choose Unix domain sockets over shared memory?**

When simpler communication semantics, kernel-managed buffering, peer identification, or file-descriptor passing are more important than minimizing every data copy.

---

## 6.7 Signals

### Definition

A signal is an asynchronous notification delivered to a process or thread to indicate that an event has occurred.

Signals can originate from the kernel or another process.

### How It Works

Examples:

| Signal | Typical Meaning |
|---|---|
| `SIGINT` | Interrupt request, commonly Ctrl+C |
| `SIGTERM` | Termination request |
| `SIGKILL` | Forced termination; cannot be caught or ignored |
| `SIGSEGV` | Invalid memory access |
| `SIGCHLD` | Child process changed state |
| `SIGUSR1` / `SIGUSR2` | Application-defined notifications |

Signal delivery may interrupt normal application execution and invoke an installed signal handler.

```text
Process executing normally
           |
           v
       Signal arrives
           |
           v
  Kernel arranges delivery
           |
           v
  Execute signal handler
           |
           v
  Resume interrupted execution
```

The exact behavior depends on signal disposition, masks, and whether the signal is synchronous or asynchronous.

### Key Concepts

| Concept | Explanation |
|---|---|
| Signal Handler | Function invoked when a handled signal is delivered |
| Signal Mask | Controls which signals are temporarily blocked |
| `sigaction()` | Configures signal handling |
| `kill()` | Sends a signal to a process or process group |
| `pthread_kill()` | Sends a signal to a specific thread |
| Async-Signal-Safe | Operation permitted inside an asynchronous signal handler |

### Interview Questions

**Q: Can `SIGKILL` be caught or ignored?**

No.

`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.

**Q: Is it safe to call arbitrary C++ functions inside a signal handler?**

No.

Many functions are not async-signal-safe. Calling them from an asynchronous signal handler can cause deadlocks or undefined behavior.

Signal handlers should perform minimal work.

**Q: Are ordinary signals suitable for transferring high-frequency market data?**

No.

Signals carry limited information, standard signals may coalesce, and asynchronous delivery can introduce execution unpredictability.

They are better suited to control notifications than high-throughput payload transfer.

**Q: How can a Linux application handle signals through its event loop?**

One approach is `signalfd()`, which allows selected blocked signals to be received through a file descriptor and integrated with mechanisms such as `epoll`.

---

## 6.8 Event Notification

### Definition

Event notification allows one thread or process to inform another that a particular condition or event has occurred.

Unlike payload transfer, notification primarily communicates that work is available or state has changed.

### How It Works

Linux provides `eventfd()`, which creates a kernel-maintained 64-bit counter exposed through a file descriptor.

A producer can write to the descriptor to increment the counter.

A consumer can read it to receive and consume the accumulated count, according to the configured mode.

```text
Producer                           Consumer
--------                           --------

Write data to queue
        |
        v
Write to eventfd
        |
        v
Kernel counter  ---------------->  poll / epoll wakes
                                       |
                                       v
                                  Read eventfd
                                       |
                                       v
                                  Process queue
```

`eventfd` can be monitored using `poll()`, `select()`, or `epoll()`.

### Key Concepts

| Concept | Explanation |
|---|---|
| `eventfd()` | Kernel counter exposed as a file descriptor |
| Notification | Indicates that an event or state change occurred |
| `epoll` | Linux readiness-notification mechanism |
| Blocking Read | Waits when no event is available |
| Non-blocking Read | Returns immediately if no event is available |
| Event Coalescing | Multiple notifications may be represented by accumulated state |

### Interview Questions

**Q: What is the difference between an event notification and a message queue?**

An event notification indicates that something happened.

A message queue transfers the actual messages or payloads.

**Q: Why is `eventfd` useful with shared memory?**

Shared memory can carry the payload while `eventfd` notifies the consumer that new data is available.

This separates data transfer from wake-up coordination.

**Q: Does one `eventfd` write always correspond to exactly one wake-up?**

No.

The counter can accumulate multiple writes, and notification behavior depends on the current counter state and consumer activity.

Applications should process the actual shared state rather than assuming a strict one-write-to-one-wakeup relationship.

**Q: Would you always use `eventfd` in a low-latency queue?**

No.

A dedicated busy-polling consumer may avoid notification and wake-up overhead.

`eventfd` is useful when reducing idle CPU consumption is more important than achieving the lowest possible wake-up latency.

---

## 6.9 Futex

### Definition

A futex (fast userspace mutex) is a Linux mechanism that enables efficient synchronization using userspace atomic operations combined with kernel-assisted waiting when necessary.

It is commonly used as a building block for mutexes, condition variables, and other synchronization primitives.

### How It Works

The central idea is:

**Stay in user space when synchronization is uncontended; enter the kernel when waiting or wake-up assistance is required.**

A simplified mutex example:

```text
Thread A                         Thread B
--------                         --------

Atomic acquire succeeds
                                 Atomic acquire fails
                                         |
                                         v
                                   futex wait
                                         |
                                         v
                                   Thread sleeps

Release mutex
      |
      v
futex wake  --------------------> Thread becomes runnable
```

A futex operates on a suitably aligned 32-bit userspace word.

The kernel checks the word's value as part of the wait operation, helping prevent a race between checking the condition and going to sleep.

### Key Concepts

| Concept | Explanation |
|---|---|
| Futex | Fast userspace synchronization mechanism |
| `FUTEX_WAIT` | Waits if the futex word still matches an expected value |
| `FUTEX_WAKE` | Wakes eligible waiters |
| Fast Path | Synchronization handled without entering the kernel |
| Slow Path | Kernel-assisted waiting or wake-up |
| PI Futex | Futex operations supporting priority inheritance |

### Interview Questions

**Q: Why are futexes called fast userspace mutexes?**

Because uncontended synchronization can often be completed using userspace atomic operations without making a system call.

The kernel becomes involved when threads need to wait or be awakened.

**Q: Does every mutex lock operation require a futex system call?**

No.

A typical uncontended mutex acquisition can complete entirely in user space.

**Q: What is the difference between a futex and a spinlock?**

A spinlock continuously retries while waiting.

A futex-based synchronization primitive can put a waiting thread to sleep, reducing wasted CPU consumption at the cost of possible scheduling overhead.

**Q: Why is the expected-value check important in `FUTEX_WAIT`?**

It prevents a thread from sleeping based on a condition that has already changed.

If the futex word no longer matches the expected value, the wait does not proceed normally.

---

## 6.10 Zero-Copy IPC

### Definition

Zero-copy IPC refers to communication techniques that eliminate one or more unnecessary payload-copying operations between processes or memory regions.

The term does not necessarily mean that absolutely no copying occurs anywhere in the system.

### How It Works

Consider transferring a large message through a conventional pipe:

```text
Process A Memory
       |
       | Copy
       v
Kernel Pipe Buffer
       |
       | Copy
       v
Process B Memory
```

With shared memory:

```text
Process A                       Process B
    |                               |
    v                               v
+---------------------------------------+
|       Same Shared Physical Memory     |
+---------------------------------------+
```

The producer writes data into a shared region, and the consumer reads it directly.

This avoids a separate kernel-buffer payload copy for each transfer.

Other Linux mechanisms that can reduce copying in suitable scenarios include:

- `splice()`.
- `vmsplice()`.
- `sendfile()` for supported file-to-socket transfers.

Their applicability and actual copy behavior depend on the source, destination, kernel, and underlying implementation.

### Key Concepts

| Concept | Explanation |
|---|---|
| Zero-Copy | Avoids unnecessary payload copying |
| Shared Memory | Enables direct access to common physical pages |
| `splice()` | Moves data between file descriptors using a pipe without conventional userspace payload copying |
| `vmsplice()` | Connects userspace memory buffers with a pipe |
| `sendfile()` | Transfers data between supported file descriptors without a conventional userspace read/write loop |
| DMA | Allows devices to transfer data to/from memory without CPU-driven byte-by-byte copying |

### Interview Questions

**Q: Why is zero-copy useful for low-latency systems?**

It can reduce:

- CPU time spent copying payloads.
- Memory bandwidth consumption.
- Cache pollution.
- Additional buffering overhead.

However, synchronization and cache-coherence costs may still remain.

**Q: Does shared memory guarantee zero-copy communication?**

It enables direct sharing of physical memory, but the application may still copy data internally.

A genuinely copy-avoiding design must account for the complete data path.

**Q: What is the difference between zero-copy and lock-free?**

They address different problems:

- Zero-copy minimizes data movement.
- Lock-free provides a concurrency progress guarantee without relying on blocking locks.

A shared-memory queue can be zero-copy but use a mutex, or lock-free while still copying messages into queue slots.

**Q: How would you design low-latency IPC between a market-data process and a strategy process?**

One possible design:

1. Allocate a fixed-size shared-memory region.
2. Place an SPSC ring buffer inside it.
3. Use fixed-size messages or offsets instead of process-local pointers.
4. Give the producer ownership of the write index.
5. Give the consumer ownership of the read index.
6. Publish and consume messages using suitable acquire/release synchronization.
7. Separate frequently modified indices onto different cache lines.
8. Define explicit behavior for queue-full conditions and process failures.

A dedicated consumer can busy-poll the queue for minimum wake-up latency. Alternatively, `eventfd` can be added when idle CPU efficiency is important.

---

## Section 6 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] What is IPC, and why is it necessary?
- [ ] Compare message passing and shared memory.
- [ ] Explain how a Unix pipe works internally.
- [ ] How does the shell implement `command_a | command_b`?
- [ ] Do pipes preserve message boundaries?
- [ ] What happens when a pipe becomes full?
- [ ] What is the difference between anonymous pipes and named pipes?
- [ ] Explain how two processes can map the same physical memory.
- [ ] Why should shared-memory data structures avoid ordinary process-local pointers?
- [ ] How do you synchronize access to shared memory?
- [ ] Compare Unix domain sockets with TCP sockets.
- [ ] What is `SCM_RIGHTS`, and when is it useful?
- [ ] What are signals, and what are their limitations?
- [ ] What is `eventfd`, and how does it work?
- [ ] Explain how a futex avoids kernel involvement on the uncontended path.
- [ ] Compare futex-based waiting with busy-waiting spinlocks.
- [ ] What does zero-copy actually mean?
- [ ] Compare shared memory, pipes, and Unix domain sockets for low-latency IPC.
- [ ] Design an SPSC shared-memory queue between two processes.
- [ ] How would you handle synchronization, backpressure, and process failure in that design?
