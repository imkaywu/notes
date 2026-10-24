# 1. OS Fundamentals & System Calls

## 1.1 OS Architecture

### Definition

An Operating System (OS) manages hardware resources and provides abstractions and services to applications.

Its two primary responsibilities are:
- **Resource management:** CPU scheduling, memory management, device management.
- **Abstraction and protection:** Processes, virtual memory, files, and access control.

### How It Works

A simplified OS architecture:

```text
+----------------------------------+
|        User Applications         |
+----------------------------------+
|       System Call Interface      |
+----------------------------------+
|              Kernel              |
|                                  |
|  - Process / Thread Scheduler    |
|  - Virtual Memory Management     |
|  - File Systems                  |
|  - Networking Stack              |
|  - Device Drivers                |
+----------------------------------+
|             Hardware             |
|       CPU / RAM / NIC / SSD      |
+----------------------------------+
```

Applications interact with the kernel through system calls to request protected OS services.

Linux uses a **monolithic kernel architecture**, where major OS subsystems execute within a shared kernel address space. It also supports dynamically loadable kernel modules.

### Key Concepts

| Concept | Explanation |
|---|---|
| Kernel | Privileged core of the operating system |
| User Space | Execution environment for ordinary applications |
| System Call | Controlled entry point into the kernel |
| Kernel Module | Code dynamically loaded into the running kernel |
| Device Driver | Software responsible for managing hardware devices |

### Interview Questions

**Q: What is the difference between user space and kernel space?**

User space runs applications with restricted privileges. Kernel space executes privileged OS code responsible for managing hardware and system resources.

**Q: Is Linux a monolithic or microkernel OS?**

Linux uses a monolithic kernel architecture. Its core subsystems, including scheduling, memory management, and networking, execute in kernel space.

**Q: Why do low-latency applications try to minimize kernel involvement?**

Kernel interactions can introduce additional overhead through privilege transitions, synchronization, scheduling, and interrupt handling.

Reducing unnecessary kernel involvement helps improve latency predictability.

---

## 1.2 User Mode vs Kernel Mode

### Definition

Modern CPUs provide different privilege levels to protect the operating system from unauthorized application behavior.

On x86-64, Linux primarily uses:

- **Ring 3:** User mode (restricted privileges).
- **Ring 0:** Kernel mode (privileged execution).

### How It Works

A user application cannot directly execute privileged instructions or access protected kernel memory.

To request a privileged operation, it must enter the kernel through a controlled mechanism, such as a system call.

```text
User Mode
    |
    | SYSCALL instruction
    v
Kernel Mode
    |
    | Validate arguments
    | Execute requested operation
    |
    | SYSRET / IRET (as appropriate)
    v
User Mode
```

The CPU enforces privilege separation through hardware mechanisms.

### Key Concepts

| User Mode | Kernel Mode |
|---|---|
| Restricted privileges | Privileged execution |
| Runs application code | Runs kernel code |
| Cannot access protected kernel memory | Can access kernel memory |
| Cannot execute privileged instructions | Can execute privileged instructions |

### Interview Questions

**Q: Does every system call cause a context switch?**

No. A system call causes a privilege-mode transition, but not necessarily a thread context switch.

The same thread may enter the kernel, execute the requested operation, and return to user mode without being scheduled out.

**Q: Can a user-space application directly access hardware?**

Generally, privileged hardware operations require kernel mediation.

However, mechanisms such as memory-mapped device registers and kernel-bypass frameworks can provide controlled hardware access from user space.

**Q: Why are user/kernel transitions relevant to low-latency applications?**

They introduce execution overhead and may involve additional kernel processing. Minimizing unnecessary transitions can improve performance and reduce latency variability.

---

## 1.3 System Calls

### Definition

A system call is a controlled interface through which a user-space application requests a service from the OS kernel.

Common Linux system calls include:

- `read()` / `write()` — Read or write data.
- `mmap()` — Create a memory mapping.
- `clone()` — Create a new task.
- `futex()` — Kernel-assisted synchronization.
- `epoll_wait()` — Wait for I/O events.

### How It Works

Consider calling `read()`:

```text
C++ Application
      |
      v
read(fd, buffer, size)
      |
      v
C Library Wrapper
      |
      v
SYSCALL instruction
      |
      v
Kernel System Call Handler
      |
      v
Validate arguments
      |
      v
Perform requested operation
      |
      v
Return result to user space
```

On Linux x86-64:

1. The system-call number is placed in `RAX`.
2. Arguments are passed through designated registers.
3. The `SYSCALL` instruction transfers execution to the kernel.
4. The kernel dispatches the corresponding system-call handler.
5. The result is returned through `RAX`.
6. Execution returns to user mode.

### Key Concepts

| Concept | Explanation |
|---|---|
| System-call ABI | Convention defining how arguments and results are passed |
| libc Wrapper | Library function exposing a system call |
| Blocking Syscall | May suspend the calling thread |
| Non-blocking I/O | Returns without waiting for I/O readiness |
| vDSO | Kernel-provided userspace code for certain operations without a conventional syscall transition |

### Interview Questions

**Q: Is calling a C library function equivalent to making a system call?**

No. Many library functions execute entirely in user space, while others wrap one or more kernel system calls.

For example, `strlen()` normally executes in user space, whereas `read()` generally invokes a system call.

**Q: Why can system calls be expensive?**

They require entering privileged kernel execution and may involve argument validation, synchronization, scheduling, or hardware interaction.

The actual cost depends heavily on the requested operation.

**Q: How can low-latency applications reduce system-call overhead?**

- Reduce unnecessary system calls.
- Batch operations where appropriate.
- Preallocate resources.
- Avoid blocking calls on critical execution paths.
- Use shared memory or kernel-bypass mechanisms when justified.

---

## 1.4 Interrupts

### Definition

An interrupt is an event that causes the CPU to temporarily redirect execution to an interrupt handler.

Hardware interrupts are generally **asynchronous**, meaning they are not directly caused by the currently executing instruction.

### How It Works

Example: A Network Interface Card (NIC) receives a packet.

```text
NIC receives packet
        |
        v
NIC performs DMA
        |
        v
NIC raises interrupt
        |
        v
CPU enters interrupt handler
        |
        v
Kernel acknowledges interrupt
and schedules packet processing
        |
        v
CPU resumes interrupted execution
```

On Linux, network packet processing commonly uses NAPI, which combines interrupt notification with polling.

The initial interrupt handler performs limited work, while additional packet processing can be deferred.

### Key Concepts

| Concept | Explanation |
|---|---|
| IRQ | Interrupt Request |
| ISR | Interrupt Service Routine |
| Interrupt Controller | Routes interrupts to CPUs |
| Interrupt Masking | Temporarily prevents selected interrupt delivery |
| Interrupt Coalescing | Groups events to reduce interrupt frequency |
| NAPI | Linux networking mechanism combining interrupts and polling |

### Interview Questions

**Q: What happens when an interrupt arrives?**

The CPU preserves the necessary execution state and transfers control to the appropriate interrupt handler.

After handling the event, execution resumes, subject to any scheduling decisions.

**Q: What is the difference between polling and interrupts?**

- **Interrupts:** Hardware notifies the CPU when an event occurs.
- **Polling:** Software repeatedly checks whether an event has occurred.

**Q: Why might polling outperform interrupts in low-latency trading systems?**

Polling avoids interrupt delivery and thread wake-up overhead, potentially reducing latency jitter.

The tradeoff is continuous CPU consumption, even when no events are available.

**Q: What is IRQ affinity?**

IRQ affinity controls which CPUs are allowed to handle particular hardware interrupts.

Low-latency systems can route unrelated interrupts away from CPUs dedicated to critical trading threads.

---

## 1.5 Exceptions & Traps

### Definition

An exception is a synchronous event generated by the CPU while executing an instruction.

Unlike external hardware interrupts, exceptions are directly associated with the current instruction stream.

A trap is a particular category of exception. The term is also historically used to describe deliberate entry into the kernel.

### How It Works

On x86, exceptions are traditionally categorized into three types:

| Type | Description | Example |
|---|---|---|
| Fault | Usually reported before the instruction completes; may be recoverable | Page fault |
| Trap | Reported after the triggering instruction executes | Breakpoint |
| Abort | Severe condition without reliable restart semantics | Double fault |

Example: Page-fault handling.

```text
Application accesses memory
          |
          v
CPU cannot complete translation
          |
          v
Page-fault exception (#PF)
          |
          v
Kernel page-fault handler
          |
          v
Is the access valid?
          |
     Yes /   \ No
        v     v
    Resolve  Deliver signal
      fault  (e.g. SIGSEGV)
        |
        v
Retry instruction
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Interrupt | Generally asynchronous hardware-generated event |
| Exception | Synchronous CPU-generated event |
| Fault | Exception that may allow the instruction to be retried |
| Trap | Exception reported after the triggering instruction |
| Page Fault | Exception caused by an address-translation or memory-access problem |
| System Call | Controlled request for a kernel service |

Modern x86-64 Linux typically uses the dedicated `SYSCALL` instruction for system calls rather than a traditional software interrupt.

### Interview Questions

**Q: What is the difference between an interrupt and an exception?**

An external hardware interrupt is generally asynchronous.

A CPU exception is synchronous and caused by the execution of a particular instruction.

**Q: What does "trap into the kernel" mean?**

It means transferring execution from user mode into the kernel through a controlled CPU mechanism.

Historically, software traps were commonly used to implement system calls.

**Q: Is every page fault a program error?**

No. Page faults are also used to implement demand paging, lazy allocation, and copy-on-write.

**Q: Why are page faults undesirable in low-latency applications?**

Handling them can involve page allocation, zero-filling, page-table updates, and potentially disk I/O, introducing unpredictable latency.

---

## 1.6 Process vs Thread

### Definition

A **process** is an OS abstraction representing a running program and its associated resources.

A **thread** is an independently schedulable execution context within a process.

A process can contain multiple threads.

### How It Works

```text
Process A
+--------------------------------+
| Shared Virtual Address Space   |
|                                |
| Heap / Global Data / Code      |
|                                |
| Thread 1       Thread 2        |
| - Registers    - Registers     |
| - Stack        - Stack         |
| - PC           - PC            |
+--------------------------------+

Process B
+--------------------------------+
| Separate Virtual Address Space |
|                                |
| Thread 1                       |
+--------------------------------+
```

Threads belonging to the same process generally share:

- Virtual address space.
- Heap and global variables.
- Code.
- File descriptor table.

Each thread maintains its own:

- CPU registers.
- Program counter.
- Stack.
- Thread-local storage.
- Scheduling state.

### Key Concepts

| Property | Process | Thread |
|---|---|---|
| Address Space | Normally separate | Shared within process |
| Registers | Per execution thread | Per thread |
| Stack | Contains thread stack(s) | Each thread has its own |
| Scheduling | Threads are scheduled | Independently scheduled |
| Communication | Typically requires IPC | Can communicate through shared memory |
| Isolation | Stronger isolation | Weaker isolation |

### Interview Questions

**Q: Why is switching between threads sometimes cheaper than switching between processes?**

Threads within the same process share an address space, so switching between them generally avoids changing the active address-space mapping and associated translation overhead.

**Q: Can two threads execute simultaneously?**

Yes, provided they are scheduled on different logical CPUs.

**Q: Why do trading systems often use dedicated threads?**

Dedicated threads can separate responsibilities such as market-data ingestion, strategy execution, and order transmission.

This can reduce interference between components and improve execution predictability.

However, shared-memory communication introduces synchronization and cache-coherence costs.

---

## 1.7 Process Control Block (PCB)

### Definition

A Process Control Block (PCB) is a conceptual kernel data structure containing the information required to manage a process.

Linux represents processes and threads primarily through `task_struct`, together with related structures such as `mm_struct`.

### How It Works

The kernel maintains process-related information throughout the process lifecycle.

When a process is created, scheduled, blocked, resumed, or terminated, the kernel consults or updates the corresponding structures.

Conceptually:

```text
Process Control Block
+--------------------------+
| Process ID               |
| Process State            |
| CPU Context              |
| Scheduling Information   |
| Memory Information       |
| File Descriptors         |
| Security Credentials     |
+--------------------------+
```

### Key Concepts

| Information | Purpose |
|---|---|
| Process ID | Identifies the process |
| Process State | Running, sleeping, stopped, etc. |
| CPU Context | Saved execution state |
| Scheduling Information | Priority, scheduling policy, metadata |
| Memory Information | Address-space management |
| File Descriptors | References to open resources |
| Credentials | Permissions and security context |

### Interview Questions

**Q: Where is the PCB stored?**

In kernel-managed memory, not in ordinary application-accessible user memory.

**Q: What is the purpose of a PCB?**

It allows the operating system to track, schedule, suspend, resume, and manage a process throughout its lifecycle.

**Q: Does Linux have a separate PCB and TCB?**

Not in the simplistic textbook sense.

Linux uses `task_struct` for individual schedulable tasks, including threads, with appropriate sharing of process-level resources.

---

## 1.8 Thread Control Block (TCB)

### Definition

A Thread Control Block (TCB) is a conceptual structure containing the information required to manage and resume an individual thread.

Each thread maintains its own execution state, even when sharing resources with other threads.

### How It Works

When a thread stops executing, the kernel preserves the necessary CPU context.

When the thread resumes, its saved execution context is restored.

Conceptually:

```text
Thread Control Block
+--------------------------+
| Thread ID                |
| Thread State             |
| Program Counter          |
| Stack Pointer            |
| CPU Registers            |
| Scheduling Information   |
| Thread-Specific Metadata |
+--------------------------+
```

Linux maintains per-task kernel structures. User-space threading libraries also maintain their own thread bookkeeping.

### Key Concepts

| Information | Purpose |
|---|---|
| Thread ID | Identifies the thread |
| Program Counter | Address of the next instruction |
| Stack Pointer | Current stack location |
| CPU Registers | Execution state |
| Thread State | Running, runnable, blocked, etc. |
| Scheduling Metadata | Priority, affinity, scheduling policy |
| Thread-Local Storage | Per-thread data |

### Interview Questions

**Q: What information must be preserved when switching threads?**

The CPU execution state required to resume the thread, including relevant registers, instruction pointer, stack pointer, and architecture-specific state.

**Q: Do threads share a stack?**

No. Each thread normally has its own stack, although all thread stacks are mapped into the shared process address space.

**Q: Why do low-latency applications often create threads during initialization?**

Thread creation and destruction involve OS and runtime overhead.

Creating long-lived worker threads during initialization avoids repeatedly paying these costs on critical execution paths.

---

## 1.9 Context Switching

### Definition

A context switch occurs when the CPU stops executing one thread and begins executing another, preserving and restoring their execution contexts.

### How It Works

```text
Thread A running
      |
      v
Timer interrupt / blocking / preemption
      |
      v
Kernel scheduler
      |
      v
Save Thread A's context
      |
      v
Select Thread B
      |
      v
Switch address space if necessary
      |
      v
Restore Thread B's context
      |
      v
Thread B resumes
```

A context switch generally involves:

1. Entering the kernel through an interrupt, exception, or system call.
2. Saving the outgoing thread's required CPU state.
3. Selecting the next runnable thread.
4. Switching address spaces if necessary.
5. Restoring the incoming thread's CPU state.
6. Resuming its execution.

### Key Concepts

| Type | Description |
|---|---|
| Voluntary Switch | Thread blocks, sleeps, or yields |
| Involuntary Switch | Scheduler preempts a running thread |
| Thread Switch | Switch between execution threads |
| Process Switch | Switch between threads belonging to different processes |

A context switch may disrupt CPU cache locality, branch-predictor state, and TLB locality.

### Interview Questions

**Q: Is a context switch the same as switching from user mode to kernel mode?**

No.

A system call can enter kernel mode and return to the same thread without a context switch.

A context switch specifically involves changing the currently executing thread.

**Q: Why are context switches expensive?**

Besides saving and restoring CPU state, switching execution can disrupt cache locality, address-translation state, and execution predictability.

**Q: What causes a context switch?**

Common causes include:

- A thread blocks on I/O.
- A thread waits on a synchronization primitive.
- A higher-priority thread becomes runnable.
- The scheduler preempts the current thread.
- A thread voluntarily yields the CPU.

**Q: How can low-latency applications reduce context-switch overhead?**

- Pin critical threads to dedicated CPUs.
- Avoid CPU oversubscription.
- Minimize blocking operations.
- Use busy polling where appropriate.
- Reduce synchronization-induced contention.

---

## 1.10 System Call Overhead

### Definition

System-call overhead is the additional cost incurred when requesting a kernel service instead of executing entirely in user space.

It includes both the privilege transition overhead and the actual work performed by the kernel.

### How It Works

The total cost can be conceptually divided into:

\[
T_{\text{syscall}} =
T_{\text{entry}} +
T_{\text{kernel work}} +
T_{\text{exit}} +
T_{\text{additional delays}}
\]

Where:

- **Entry:** Transition from user mode to kernel mode.
- **Kernel work:** Validation and execution of the requested operation.
- **Exit:** Return from kernel mode to user mode.
- **Additional delays:** Potential blocking, scheduling, lock contention, or hardware interaction.

Not every system call incurs all these costs.

### Key Concepts

| Concept | Explanation |
|---|---|
| Mode-Switch Overhead | Cost of entering/exiting privileged execution |
| Blocking | Thread cannot immediately continue |
| Batching | Amortizing overhead across multiple operations |
| vDSO | Certain kernel-supported operations execute through userspace mappings |
| Kernel Bypass | Avoiding portions of the traditional kernel I/O path |

### Interview Questions

**Q: How can you reduce system-call overhead?**

- Reduce unnecessary system calls.
- Batch operations.
- Preallocate resources.
- Avoid blocking operations on critical paths.
- Use shared memory or kernel-bypass mechanisms where appropriate.

**Q: What is vDSO, and why is it useful?**

The Virtual Dynamic Shared Object (vDSO) is a small kernel-provided library mapped into a process's address space.

It allows certain operations, such as supported clock queries, to execute without a conventional system-call transition.

**Q: How does batching improve performance?**

Consider transmitting 1,000 messages.

A naive implementation might issue 1,000 individual system calls.

Batching multiple messages into fewer calls amortizes the transition and kernel-processing overhead across more work.

However, waiting to accumulate a batch can increase individual message latency.

**Q: Is avoiding all system calls always optimal?**

No.

For example, busy polling avoids blocking and wake-up overhead but continuously consumes CPU resources.

The correct design depends on latency targets, throughput requirements, and available hardware resources.

---

## Section 1 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the architecture of a modern operating system.
- [ ] What is the difference between user mode and kernel mode?
- [ ] Walk through what happens during a Linux system call.
- [ ] What is the difference between interrupts, exceptions, and traps?
- [ ] Explain the hardware interrupt handling process.
- [ ] What is the difference between a process and a thread?
- [ ] What information does a PCB contain?
- [ ] What information must be maintained for each thread?
- [ ] Explain a context switch step by step.
- [ ] Why can system calls introduce latency, and how can we reduce their overhead?
