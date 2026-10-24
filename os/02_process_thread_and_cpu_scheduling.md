# 2. Processes, Threads & CPU Scheduling

## 2.1 Process Lifecycle

### Definition

A process is a running instance of a program, managed by the operating system.

During its lifetime, a process transitions between different execution states, from creation to termination.

### How It Works

A simplified process lifecycle:

```text
              Created
                 |
                 v
               Ready <----------------+
                 |                    |
             Scheduled                |
                 |                    |
                 v                    |
              Running                 |
              /  |   \                |
             /   |    \               |
     Blocking  Preempted  Exit         |
          |      |         |          |
          v      +---------+----------+
       Waiting             |
          |                v
     Event occurs      Terminated
          |
          +----------> Ready
```

Note: The diagram represents conceptual states. Actual Linux task states differ from this simplified model.

Common Linux process-related operations:

1. `fork()` — Creates a child process.
2. `execve()` — Replaces the current process image with a new program.
3. `waitpid()` — Waits for and collects a child process's termination status.
4. `exit()` — Terminates a process through the C runtime.

A typical shell launches a program using `fork()` followed by `execve()`.

### Key Concepts

| Concept | Explanation |
|---|---|
| PID | Unique process identifier within its PID namespace |
| Parent Process | Process that creates another process |
| Child Process | Process created by its parent |
| Zombie | Terminated process whose exit status has not been collected |
| Orphan | Process whose parent has terminated |
| Copy-on-Write | Allows parent and child to initially share physical pages after `fork()` |

### Interview Questions

**Q: What is the difference between `fork()` and `exec()`?**

- `fork()` creates a new child process.
- `exec()` replaces the calling process's program image without creating a new process.

They are commonly used together to launch programs.

**Q: What is a zombie process?**

A process that has terminated but still retains minimal kernel information because its parent has not collected its exit status using `wait()` or a related function.

**Q: Why can `fork()` be problematic in a large, multithreaded trading application?**

It can introduce unpredictable overhead, including copying page-table structures and creating copy-on-write mappings.

Additionally, after `fork()` in a multithreaded process, only the calling thread survives in the child, potentially leaving synchronization primitives in problematic states.

---

## 2.2 Thread Lifecycle

### Definition

A thread is the smallest independently schedulable execution unit managed by the operating system.

Multiple threads within the same process generally share its address space and resources while maintaining independent execution contexts.

### How It Works

A simplified thread lifecycle:

```text
       Created
          |
          v
       Runnable <-------------+
          |                   |
       Scheduled              |
          |                   |
          v                   |
        Running               |
       /   |    \             |
      /    |     \            |
 Blocking Yield  Termination  |
    |      |         |        |
    v      +---------+--------+
 Waiting             |
    |                 v
 Event occurs     Terminated
    |
    +----------> Runnable
```

In C++, threads are commonly managed using `std::thread` or `std::jthread`.

On Linux, POSIX threads are implemented using the NPTL library, which uses Linux task-creation mechanisms such as `clone()`.

### Key Concepts

| Concept | Explanation |
|---|---|
| Thread Creation | Allocates thread resources and creates an execution context |
| Runnable | Eligible to execute when scheduled |
| Running | Currently executing on a logical CPU |
| Blocked | Waiting for an event or resource |
| Join | Waits for a thread to terminate |
| Detach | Releases the caller's obligation to join the thread |
| Thread-Local Storage | Data maintained separately for each thread |

### Interview Questions

**Q: What is the difference between `join()` and `detach()`?**

- `join()` waits for a thread to finish.
- `detach()` separates the thread's lifetime from its `std::thread` object.

Detaching does not terminate the thread.

**Q: Does creating a thread immediately cause it to execute?**

No. Creation makes the thread eligible for scheduling, but the OS determines when it actually runs.

**Q: Why should low-latency applications avoid creating threads on the hot path?**

Thread creation can involve memory allocation, stack setup, kernel operations, and scheduler activity.

Long-lived worker threads avoid repeatedly paying these costs during latency-sensitive execution.

---

## 2.3 CPU Scheduling

### Definition

CPU scheduling is the mechanism through which the operating system determines which runnable thread executes on a CPU.

Its purpose is to allocate CPU time while balancing fairness, responsiveness, throughput, and scheduling priorities.

### How It Works

Each logical CPU can execute one software thread at a time.

When multiple threads are runnable, the scheduler determines which one should execute.

```text
           Runnable Threads
          +------------------+
          | A | B | C | D | E|
          +------------------+
                   |
                   v
             CPU Scheduler
                   |
          +--------+--------+
          |                 |
          v                 v
        CPU 0             CPU 1
       Thread A          Thread C
```

Scheduling decisions may occur when:

1. A thread blocks or voluntarily yields.
2. A sleeping thread becomes runnable.
3. A higher-priority thread becomes runnable.
4. A scheduling timer or another event triggers preemption.
5. CPU load balancing moves work between CPUs.

### Key Concepts

| Concept | Explanation |
|---|---|
| Runnable | Thread is eligible to execute |
| Run Queue | Scheduler structure containing runnable tasks |
| Preemption | Suspending a running thread to schedule another |
| Time Slice | Amount of CPU time allocated before reconsidering scheduling |
| Scheduling Latency | Delay between becoming runnable and actually executing |
| CPU Migration | Moving a thread between logical CPUs |

### Interview Questions

**Q: What is preemptive scheduling?**

Preemptive scheduling allows the OS to interrupt a running thread and schedule another without requiring the current thread to voluntarily yield.

**Q: What is the difference between concurrency and parallelism?**

- Concurrency: Multiple tasks make progress over overlapping periods.
- Parallelism: Multiple tasks execute simultaneously on different processing resources.

**Q: Why is CPU scheduling important for low-latency applications?**

A trading thread may become runnable but still have to wait for CPU time.

This scheduling delay introduces latency jitter, even when the application code itself is highly optimized.

---

## 2.4 Linux Scheduler

### Definition

The Linux scheduler manages runnable tasks and determines which task executes on each CPU.

Linux supports multiple scheduling classes, including normal time-sharing and real-time scheduling.

Modern Linux uses the **EEVDF (Earliest Eligible Virtual Deadline First)** algorithm for normal scheduling, replacing the older CFS task-selection approach.

### How It Works

For normal tasks, EEVDF uses virtual runtime, eligibility, and virtual deadlines to determine scheduling order.

Conceptually:

1. Track how much CPU service each task has received.
2. Determine which tasks are eligible to execute.
3. Prefer the eligible task with the earliest virtual deadline.
4. Update scheduling accounting as tasks execute.

The scheduler also performs load balancing across CPUs, subject to affinity and other constraints.

### Key Concepts

| Concept | Explanation |
|---|---|
| CFS | Completely Fair Scheduler, the historical Linux fair-scheduling design |
| EEVDF | Earliest Eligible Virtual Deadline First |
| Virtual Runtime | Weighted accounting of CPU execution time |
| Virtual Deadline | Scheduling deadline used by EEVDF |
| Nice Value | Influences a normal task's relative scheduling weight |
| Scheduling Class | Defines the scheduling rules applied to a task |
| Run Queue | Per-CPU scheduler structure managing runnable tasks |

### Interview Questions

**Q: What is the difference between CFS and EEVDF?**

CFS historically selected tasks primarily based on their virtual runtime.

EEVDF incorporates eligibility and virtual deadlines to improve fairness and responsiveness.

Both aim to distribute CPU resources fairly among normal tasks.

**Q: Does increasing a thread's priority guarantee immediate execution?**

No.

Actual execution depends on its scheduling class, competing tasks, CPU availability, affinity constraints, and kernel execution state.

**Q: Why is the default Linux scheduler not always ideal for extremely latency-sensitive trading threads?**

Normal scheduling prioritizes general-purpose fairness and responsiveness rather than guaranteeing uninterrupted execution for one particular application.

Critical trading threads may therefore benefit from dedicated CPUs, carefully selected scheduling policies, and reduced interference.

---

## 2.5 Scheduling Policies

### Definition

A scheduling policy determines how the OS prioritizes runnable threads and allocates CPU time.

Linux provides different policies for normal workloads, real-time workloads, and deadline-based scheduling.

### How It Works

Important Linux scheduling policies:

| Policy | Description |
|---|---|
| `SCHED_OTHER` | Default normal scheduling policy |
| `SCHED_BATCH` | Optimized for CPU-intensive, non-interactive workloads |
| `SCHED_IDLE` | Very low-priority scheduling |
| `SCHED_FIFO` | Real-time first-in, first-out scheduling |
| `SCHED_RR` | Real-time round-robin scheduling |
| `SCHED_DEADLINE` | Deadline-based scheduling using runtime, deadline, and period parameters |

For `SCHED_FIFO`:

- A higher-priority runnable real-time thread generally takes precedence over lower-priority threads.
- Threads of equal priority follow FIFO ordering.
- A running thread does not have a fixed round-robin time quantum.

For `SCHED_RR`:

- Similar priority rules apply.
- Equal-priority threads receive time slices.

### Key Concepts

| Concept | Explanation |
|---|---|
| Static Priority | Explicit scheduling priority |
| Real-Time Scheduling | Scheduling class with stronger priority guarantees |
| Time Quantum | Maximum continuous execution interval under round-robin scheduling |
| Deadline Scheduling | Scheduling based on specified timing constraints |
| Starvation | A task receives insufficient CPU time because other tasks dominate scheduling |

### Interview Questions

**Q: What is the difference between `SCHED_FIFO` and `SCHED_RR`?**

Both are fixed-priority real-time scheduling policies.

`SCHED_FIFO` does not impose a round-robin time quantum on equal-priority threads, while `SCHED_RR` rotates execution among equal-priority threads using time slices.

**Q: Is `SCHED_FIFO` always better for low-latency applications?**

No.

A continuously running high-priority thread can starve lower-priority threads, including important system services.

Real-time scheduling must be configured carefully, especially when combined with CPU isolation.

**Q: Does real-time scheduling guarantee that a task always meets its deadline?**

No. Real-time scheduling improves scheduling predictability, but actual deadline guarantees depend on the scheduling policy, workload, kernel configuration, and resource availability.

---

## 2.6 CPU Affinity

### Definition

CPU affinity specifies which logical CPUs a thread is allowed to execute on.

By restricting a thread to a particular CPU or set of CPUs, the OS can reduce unwanted CPU migration.

### How It Works

Without explicit affinity:

```text
Time 1: Thread A -> CPU 0
Time 2: Thread A -> CPU 3
Time 3: Thread A -> CPU 1
```

With affinity restricted to CPU 2:

```text
Time 1: Thread A -> CPU 2
Time 2: Thread A -> CPU 2
Time 3: Thread A -> CPU 2
```

On Linux, CPU affinity can be configured through:

- `sched_setaffinity()`
- `pthread_setaffinity_np()`
- `taskset`

Example:

```bash
taskset -c 2 ./trading_app
```

This restricts the launched process's initial thread to logical CPU 2; subsequently created threads normally inherit the affinity mask.

### Key Concepts

| Concept | Explanation |
|---|---|
| Logical CPU | CPU execution context visible to the OS |
| CPU Affinity Mask | Set of CPUs a thread is allowed to execute on |
| CPU Migration | Moving execution between logical CPUs |
| Cache Locality | Reusing data already present in nearby CPU caches |
| Hard Affinity | Explicit restriction on allowed CPUs |

### Interview Questions

**Q: Why can CPU pinning improve low-latency performance?**

CPU pinning can reduce thread migration and improve cache locality.

It also makes execution placement more predictable, simplifying performance analysis and system tuning.

**Q: Does CPU affinity guarantee exclusive ownership of a CPU?**

No.

Other threads, interrupts, and kernel activity may still execute on the same CPU.

Additional CPU isolation and IRQ configuration may be required.

**Q: Can CPU pinning hurt performance?**

Yes.

Pinning multiple busy threads to the same CPU can create contention, while restricting threads too aggressively may prevent the scheduler from balancing workload efficiently.

---

## 2.7 CPU Isolation

### Definition

CPU isolation is the practice of reserving selected CPUs for latency-sensitive workloads by minimizing interference from general-purpose scheduling and system activity.

CPU isolation is stronger than simply setting CPU affinity.

### How It Works

Consider an eight-CPU system:

```text
+-----------------------------------+
| CPU 0-1: Housekeeping             |
|                                   |
| - General OS tasks                |
| - Background services             |
| - Most hardware interrupts        |
+-----------------------------------+
| CPU 2-7: Latency-Sensitive Tasks  |
|                                   |
| - Market Data                     |
| - Order Book                      |
| - Strategy                        |
| - Order Gateway                   |
+-----------------------------------+
```

Common Linux mechanisms include:

| Mechanism | Purpose |
|---|---|
| `isolcpus` | Restricts selected CPUs from normal scheduler load balancing, depending on configuration |
| `nohz_full` | Reduces periodic scheduler ticks on selected CPUs under suitable conditions |
| `rcu_nocbs` | Offloads selected RCU callback processing |
| CPU Affinity | Restricts application execution placement |
| IRQ Affinity | Controls where hardware interrupts are handled |
| Cpuset | Restricts CPU and memory-node placement for groups of tasks |

CPU isolation requires coordinated configuration. A single kernel parameter does not eliminate all OS activity from a CPU.

### Key Concepts

| Concept | Explanation |
|---|---|
| Isolated CPU | CPU reserved primarily for designated workloads |
| Housekeeping CPU | CPU responsible for general system and kernel activities |
| Scheduler Tick | Periodic timer event used for scheduling and accounting |
| IRQ Affinity | Determines which CPUs handle hardware interrupts |
| RCU Callback | Deferred kernel cleanup operation |

### Interview Questions

**Q: What is the difference between CPU affinity and CPU isolation?**

CPU affinity restricts where a particular thread can execute.

CPU isolation attempts to reduce unrelated scheduling and system activity on selected CPUs.

They are complementary techniques.

**Q: Why are housekeeping CPUs necessary?**

The operating system still needs CPU resources for background services, interrupts, kernel workers, timers, and other system activities.

Moving these activities away from critical CPUs can reduce interference.

**Q: Does `isolcpus` completely prevent interrupts and kernel activity?**

No.

Additional mechanisms, including IRQ affinity and tick/RCU configuration, may be required. Some unavoidable kernel activity can still occur.

---

## 2.8 Priority Inversion

### Definition

Priority inversion occurs when a high-priority thread is indirectly delayed by a lower-priority thread holding a resource it needs.

The problem can become worse when medium-priority threads prevent the lower-priority thread from running and releasing that resource.

### How It Works

Consider three threads:

- H: High priority.
- M: Medium priority.
- L: Low priority.

```text
1. L acquires mutex X.

2. H attempts to acquire mutex X.
   H blocks because L owns it.

3. M becomes runnable.

4. M preempts L.

5. L cannot run and release X.

6. H remains blocked.
```

Although H has the highest priority, its progress depends on L.

This is priority inversion.

### Key Concepts

| Concept | Explanation |
|---|---|
| Priority Inversion | High-priority task waits indirectly on lower-priority work |
| Priority Inheritance | Temporarily raises a lock owner's priority |
| Priority Ceiling | Synchronization protocol restricting priority behavior |
| PI Mutex | Mutex supporting priority inheritance |
| Starvation | A thread is repeatedly denied execution or resource access |

### Interview Questions

**Q: How does priority inheritance solve priority inversion?**

When H blocks on a mutex held by L, priority inheritance temporarily raises L's effective priority.

This allows L to execute, release the mutex, and unblock H without being repeatedly preempted by intermediate-priority tasks.

**Q: Does a normal mutex automatically support priority inheritance?**

Not necessarily.

On Linux, POSIX mutexes can explicitly request priority inheritance using `PTHREAD_PRIO_INHERIT`.

**Q: How can priority inversion affect a trading system?**

A high-priority order-processing thread may block on a lock held by a lower-priority logging or maintenance thread.

This can introduce unexpectedly large tail-latency spikes.

A common design approach is to avoid sharing blocking locks between critical and noncritical execution paths.

---

## 2.9 CPU Oversubscription

### Definition

CPU oversubscription occurs when the number of runnable, CPU-demanding threads exceeds the available logical CPUs.

The OS must time-share CPU resources among competing threads.

### How It Works

Consider four CPU-bound threads competing for two logical CPUs:

```text
Runnable Threads:
[A] [B] [C] [D]

Available CPUs:
CPU 0       CPU 1
  A           B

Waiting:
  C           D
```

The scheduler must periodically preempt running threads to give other threads CPU time.

This introduces:

- Additional context switches.
- Scheduling delays.
- Cache disruption.
- Greater latency variability.

Oversubscription depends on the number of simultaneously runnable threads, not simply the total number of threads created.

A process can have hundreds of mostly sleeping threads without continuously oversubscribing the CPUs.

### Key Concepts

| Concept | Explanation |
|---|---|
| Logical CPU | Hardware execution context visible to the OS |
| Physical Core | Physical processor core |
| SMT | Simultaneous multithreading; multiple logical CPUs share a physical core |
| Runnable Thread | Thread eligible to execute |
| Run-Queue Contention | Multiple runnable threads compete for CPU time |
| CPU Utilization | Proportion of time a CPU is executing work |

### Interview Questions

**Q: Is having more threads than CPU cores always bad?**

No.

For I/O-bound workloads, additional threads can improve utilization because many threads spend time blocked.

For CPU-bound, latency-sensitive workloads, excessive runnable threads often increase contention and scheduling jitter.

**Q: If a server has eight physical cores and sixteen logical CPUs, can sixteen CPU-intensive threads execute without contention?**

Not necessarily.

With SMT, sibling logical CPUs share physical execution resources, including portions of the execution pipeline and caches.

Sixteen runnable threads may therefore compete for hardware resources even though the OS exposes sixteen logical CPUs.

**Q: Why do low-latency trading systems often use a small number of dedicated threads?**

Reducing competition for CPU resources helps improve execution predictability.

A small, carefully designed thread topology also simplifies CPU pinning, cache locality, and communication between components.

---

## 2.10 Scheduler Latency & Jitter

### Definition

**Scheduler latency** is the delay between a thread becoming runnable and actually beginning or resuming execution on a CPU.

**Latency jitter** refers to variation in observed latency between otherwise similar operations.

A system may have excellent average latency while still suffering from occasional large delays.

### How It Works

Consider a thread waiting for market data:

```text
T0: Packet arrives.
        |
        v
T1: Kernel makes thread runnable.
        |
        | Scheduler delay
        |
        v
T2: Thread begins executing.
        |
        | Application processing
        |
        v
T3: Processing completes.
```

Scheduler latency:

```text
T_scheduler = T2 - T1
```

End-to-end latency:

```text
T_total = T3 - T0
```

The total includes packet handling, wake-up, scheduling, and application processing.

Common causes of scheduling jitter include:

1. CPU contention.
2. Higher-priority runnable threads.
3. Interrupt handling.
4. CPU migration.
5. Blocking synchronization.
6. Kernel background activity.
7. CPU power-state transitions.

### Key Concepts

| Concept | Explanation |
|---|---|
| Wake-Up Latency | Delay between an event waking a thread and its execution |
| Scheduling Latency | Time spent runnable but not executing |
| Tail Latency | Latency at high percentiles, such as p99 and p99.9 |
| Jitter | Variation in latency |
| Preemption | Running thread is interrupted to execute another |
| Run-Queue Delay | Time spent waiting for CPU scheduling |

### Interview Questions

**Q: What is the difference between execution latency and scheduling latency?**

Execution latency measures the time spent performing the actual work.

Scheduling latency measures how long a runnable thread waits before receiving CPU time.

**Q: Why is average latency insufficient for HFT systems?**

A low average can hide occasional extreme delays.

For example, an application might process most market-data updates in 2 µs but occasionally experience 100 µs spikes.

Such outliers can be more important than average performance in latency-sensitive trading.

**Q: How would you investigate unexpected scheduling latency?**

A practical investigation:

1. Measure latency distributions rather than only averages.
2. Monitor voluntary and involuntary context switches.
3. Inspect CPU utilization and run-queue contention.
4. Check CPU affinity and migration.
5. Inspect interrupt distribution.
6. Use Linux tracing tools such as `perf`, ftrace, or eBPF to identify scheduling delays.

**Q: How can scheduling jitter be reduced?**

- Dedicate CPUs to critical threads.
- Avoid oversubscription.
- Configure IRQ affinity.
- Minimize blocking operations.
- Consider appropriate scheduling policies.
- Reduce unrelated kernel activity on critical CPUs.
- Measure the results under realistic workloads.

No single optimization guarantees deterministic execution.

---

## Section 2 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the lifecycle of a Linux process.
- [ ] What is the difference between `fork()` and `exec()`?
- [ ] Explain the lifecycle of a thread.
- [ ] What events can trigger a scheduling decision?
- [ ] Explain how the modern Linux scheduler works.
- [ ] Compare `SCHED_OTHER`, `SCHED_FIFO`, and `SCHED_RR`.
- [ ] What is CPU affinity, and why is it useful?
- [ ] Explain CPU isolation and housekeeping CPUs.
- [ ] What is priority inversion, and how does priority inheritance address it?
- [ ] What is CPU oversubscription?
- [ ] Why can SMT introduce contention between threads?
- [ ] What is scheduler latency, and how is it different from execution latency?
- [ ] How would you diagnose unexpected scheduling jitter?
- [ ] Design a CPU/thread allocation strategy for a low-latency trading application.
