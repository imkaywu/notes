# 5. Concurrency & Synchronization

## 5.1 Race Conditions

### Definition

A race condition occurs when a program's behavior depends on the relative timing or ordering of concurrent operations.

A **data race** is a specific C++ concurrency violation: two potentially concurrent operations access the same memory location, at least one modifies it, and there is no sufficient synchronization or atomic protection.

A data race results in undefined behavior in C++.

### How It Works

Consider two threads incrementing a shared counter:

```cpp
int counter = 0;

// Thread A and Thread B:
++counter;
```

An increment is conceptually a read-modify-write operation:

```text
Thread A                  Thread B
--------                  --------
Read counter (0)
                          Read counter (0)
Increment (1)
                          Increment (1)
Write counter (1)
                          Write counter (1)

Expected: 2
Possible outcome: 1
```

In actual C++, this example contains a data race, so the language does not guarantee any particular outcome.

It can be fixed using a mutex or an atomic variable.

### Key Concepts

| Concept | Explanation |
|---|---|
| Race Condition | Result depends on concurrent execution ordering |
| Data Race | Conflicting unsynchronized memory accesses |
| Critical Section | Code accessing shared resources that requires synchronization |
| Mutual Exclusion | Ensures only one thread enters a protected section at a time |
| Happens-Before | C++ ordering relationship used to establish visibility and avoid data races |
| Thread Safety | Correct behavior under concurrent execution |

### Interview Questions

**Q: What is the difference between a race condition and a data race?**

A race condition is a broader logical problem caused by timing-dependent execution.

A data race is a specific violation of the C++ memory model involving conflicting, insufficiently synchronized memory accesses.

A program can have a race condition without having a data race.

**Q: Does making a variable `volatile` prevent data races?**

No.

`volatile` does not provide atomicity or inter-thread synchronization in C++.

Use `std::atomic` or appropriate synchronization primitives instead.

**Q: How would you detect data races in C++?**

Use ThreadSanitizer (TSan), code review, and targeted concurrency tests.

For example:

```bash
clang++ -std=c++20 -g -fsanitize=thread main.cpp -o main
```

ThreadSanitizer is useful, but it cannot guarantee detection of every concurrency bug.

---

## 5.2 Mutexes

### Definition

A mutex (mutual exclusion lock) ensures that only one thread at a time can execute a protected critical section.

It prevents concurrent access to shared mutable state.

### How It Works

A thread must acquire the mutex before entering the critical section and release it afterward.

```cpp
std::mutex mutex;
int counter = 0;

void Increment() {
  std::lock_guard<std::mutex> lock(mutex);
  ++counter;
}
```

Conceptually:

```text
Thread A                    Thread B
--------                    --------
Acquire mutex
                            Attempt acquire
                            -> Must wait

Modify shared data

Release mutex
                            Acquire mutex
                            Modify shared data
                            Release mutex
```

A typical mutex implementation has:

1. **Uncontended path:** Attempts to acquire the lock using userspace atomic operations.
2. **Contended path:** May use OS-assisted waiting, such as Linux futexes, instead of spinning indefinitely.

The precise implementation depends on the mutex type and runtime library.

### Key Concepts

| Concept | Explanation |
|---|---|
| Lock | Acquires exclusive access |
| Unlock | Releases exclusive access |
| Critical Section | Code protected by synchronization |
| Contention | Multiple threads compete for the same lock |
| Deadlock | Threads cannot progress because of circular resource dependencies |
| RAII | Associates resource acquisition and release with object lifetime |
| `std::lock_guard` | Scoped mutex ownership |
| `std::unique_lock` | Flexible mutex ownership supporting explicit lock/unlock |

### Interview Questions

**Q: What happens when a thread attempts to acquire an already-locked mutex?**

It cannot enter the protected critical section until the mutex becomes available.

Depending on the implementation, it may briefly spin before entering an OS-assisted waiting state.

**Q: Why is a mutex potentially expensive for low-latency applications?**

Under contention, acquiring a mutex may involve blocking, kernel operations, and scheduler wake-up delays.

It can also introduce cache-coherence traffic and unpredictable waiting time.

**Q: What is deadlock?**

Deadlock occurs when threads are unable to make progress because each is waiting for resources held by another.

A common prevention technique is enforcing a consistent lock acquisition order.

**Q: Is a mutex always slower than a lock-free algorithm?**

No.

An uncontended mutex can be very efficient, while lock-free algorithms may suffer from retries, contention, and cache-coherence overhead.

Performance must be measured under realistic workloads.

---

## 5.3 Spinlocks

### Definition

A spinlock is a synchronization primitive that repeatedly checks whether a lock is available instead of putting the waiting thread to sleep.

It uses busy waiting.

### How It Works

A simplified spinlock using `std::atomic_flag`:

```cpp
class Spinlock {
 public:
  void Lock() {
    while (flag_.test_and_set(std::memory_order_acquire)) {
      // Busy wait.
    }
  }

  void Unlock() {
    flag_.clear(std::memory_order_release);
  }

 private:
  std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
};
```

Execution:

```text
Thread A                    Thread B
--------                    --------
Acquire lock
                            Try acquire
                            Failed
                            Spin...
                            Spin...

Release lock
                            Acquire lock
                            Continue
```

Unlike a blocking mutex, the waiting thread continues consuming CPU time.

### Key Concepts

| Concept | Explanation |
|---|---|
| Busy Waiting | Repeatedly checking a condition without sleeping |
| Spinlock | Lock implemented using busy waiting |
| Atomic Flag | Atomic Boolean-like synchronization primitive |
| Acquire | Ensures subsequent operations observe properly published data |
| Release | Publishes preceding operations to acquiring threads |
| Backoff | Delaying retries to reduce contention |

### Interview Questions

**Q: When is a spinlock preferable to a mutex?**

When the critical section is extremely short, contention is limited, and the lock owner is expected to release it quickly.

Avoiding a scheduler sleep/wakeup cycle can be beneficial.

**Q: When can spinlocks perform badly?**

- Long critical sections.
- Heavy contention.
- CPU oversubscription.
- Lock owner being preempted.
- Multiple threads competing on the same physical core.

**Q: Why can a spinlock be dangerous with a high-priority thread?**

If a high-priority thread spins while waiting for a lock held by a lower-priority thread, it may prevent the lock owner from getting CPU time to release the lock.

This can create severe priority inversion.

**Q: What does the x86 `PAUSE` instruction do?**

It provides a processor hint for spin-wait loops, potentially reducing resource contention and power consumption.

It does not put the thread to sleep.

---

## 5.4 Condition Variables

### Definition

A condition variable allows threads to wait efficiently until a shared condition becomes true.

It is typically used together with a mutex to coordinate threads without continuously polling.

### How It Works

Example: A consumer waits until a queue contains data.

```cpp
std::mutex mutex;
std::condition_variable cv;
std::queue<int> queue;

void Consumer() {
  std::unique_lock<std::mutex> lock(mutex);

  cv.wait(lock, [] {
    return !queue.empty();
  });

  int value = queue.front();
  queue.pop();
}

void Producer(int value) {
  {
    std::lock_guard<std::mutex> lock(mutex);
    queue.push(value);
  }

  cv.notify_one();
}
```

When the consumer calls `wait()`:

1. It checks the condition while holding the mutex.
2. If the condition is false, it atomically releases the mutex and enters the wait operation.
3. Another thread modifies the shared state and notifies it.
4. The consumer wakes and reacquires the mutex.
5. It checks the condition again before proceeding.

### Key Concepts

| Concept | Explanation |
|---|---|
| `wait()` | Releases the mutex and waits for notification |
| `notify_one()` | Notifies one waiting thread |
| `notify_all()` | Notifies all waiting threads |
| Predicate | Shared condition determining whether waiting should continue |
| Spurious Wakeup | Thread wakes without the desired condition becoming true |
| Lost Wakeup | Notification is missed because the waiting protocol is incorrect |

### Interview Questions

**Q: Why must condition-variable waits use a loop or predicate?**

Because wakeups do not guarantee that the desired condition is true.

Spurious wakeups can occur, and another thread may consume the resource before the awakened thread acquires the mutex.

**Q: Does `notify_one()` immediately transfer execution to the waiting thread?**

No.

It makes a waiting thread eligible to resume, but the thread must still reacquire the mutex and be scheduled.

**Q: Why might condition variables be unsuitable for an ultra-low-latency hot path?**

Waiting and notification can involve OS-assisted blocking and scheduling.

Busy polling may offer lower wake-up latency when dedicated CPU resources are available.

---

## 5.5 Semaphores

### Definition

A semaphore is a synchronization primitive that maintains a counter representing available resources or permits.

Threads can acquire permits and release them.

### How It Works

A counting semaphore initialized to three permits:

```text
Initial permits: 3

Thread A acquires -> 2
Thread B acquires -> 1
Thread C acquires -> 0

Thread D attempts acquire
        |
        v
      Blocks

Thread A releases -> Thread D can proceed
```

C++20 provides `std::counting_semaphore` and `std::binary_semaphore`.

Example:

```cpp
std::counting_semaphore<3> semaphore(3);

void Worker() {
  semaphore.acquire();

  // Access limited resource.

  semaphore.release();
}
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Counting Semaphore | Tracks multiple available permits |
| Binary Semaphore | Semaphore with a maximum count of one |
| Acquire | Obtains a permit, waiting if none is available |
| Release | Returns a permit |
| Mutex Ownership | Mutexes have ownership semantics; semaphores generally do not |

### Interview Questions

**Q: What is the difference between a mutex and a semaphore?**

A mutex provides mutual exclusion and has ownership semantics.

A semaphore manages a count of permits and can be used for resource limits or event notification.

**Q: Can a semaphore be used for producer-consumer synchronization?**

Yes.

A producer can release a permit when an item becomes available, and a consumer can acquire a permit before consuming an item.

**Q: Are semaphores inherently lock-free?**

No.

Semaphore implementations may use atomic operations for fast paths but can rely on OS-assisted blocking when permits are unavailable.

---

## 5.6 Atomics

### Definition

Atomic operations are operations on shared variables that execute indivisibly with respect to other atomic accesses to the same object.

C++ provides atomic operations through `std::atomic`.

Atomics are fundamental building blocks for lock-free synchronization.

### How It Works

Consider:

```cpp
std::atomic<int> counter{0};

void Increment() {
  counter.fetch_add(1, std::memory_order_relaxed);
}
```

Unlike a normal unsynchronized increment, `fetch_add()` performs an atomic read-modify-write operation.

Common atomic operations:

| Operation | Purpose |
|---|---|
| `load()` | Atomically reads a value |
| `store()` | Atomically writes a value |
| `exchange()` | Replaces a value and returns its previous value |
| `fetch_add()` | Atomically adds and returns the previous value |
| `fetch_sub()` | Atomically subtracts and returns the previous value |
| `compare_exchange_weak()` | Conditional update, may fail spuriously |
| `compare_exchange_strong()` | Conditional update without spurious failure |

Example: Compare-and-swap (CAS).

```cpp
std::atomic<int> value{10};

int expected = 10;

bool success = value.compare_exchange_strong(
    expected,
    20
);
```

If `value == expected`, the operation writes `20` and returns true.

Otherwise, it returns false and updates `expected` with the observed value.

### Key Concepts

| Concept | Explanation |
|---|---|
| Atomicity | Operation cannot be observed partially |
| RMW | Read-Modify-Write operation |
| CAS | Compare-And-Swap |
| Lock-Free | System-wide progress is guaranteed despite contention |
| Wait-Free | Every operation completes within a bounded number of its own steps |
| Memory Ordering | Controls ordering and visibility constraints between threads |

### Interview Questions

**Q: Is `std::atomic` always lock-free?**

No.

Whether an atomic type is lock-free depends on its size, alignment, implementation, and target architecture.

Use `is_lock_free()` or `is_always_lock_free` to inspect the guarantee.

**Q: What is the difference between `compare_exchange_weak()` and `compare_exchange_strong()`?**

Weak CAS may fail spuriously even when the expected value matches.

Strong CAS does not fail spuriously.

Weak CAS is commonly used inside retry loops.

**Q: Does using atomics automatically make an algorithm thread-safe?**

No.

Atomics protect individual atomic operations, but the overall algorithm must still establish correct ordering, synchronization, ownership, and state transitions.

---

## 5.7 Memory Ordering

### Definition

Memory ordering defines the ordering and visibility guarantees associated with atomic operations between threads.

Modern CPUs and compilers may reorder operations when doing so preserves the behavior required by the language memory model.

C++ provides explicit memory-order options to control inter-thread synchronization.

### How It Works

Consider a producer publishing data to a consumer:

```cpp
int data = 0;
std::atomic<bool> ready{false};

// Producer
data = 42;
ready.store(true, std::memory_order_release);

// Consumer
while (!ready.load(std::memory_order_acquire)) {
}

int result = data;
```

The release store and acquire load establish synchronization when the consumer observes the producer's published value.

This ensures the consumer can safely observe the preceding write to `data`.

### Key Concepts

| Memory Order | Meaning |
|---|---|
| `relaxed` | Atomicity without additional synchronization ordering |
| `acquire` | Prevents subsequent operations from moving before the acquire in the relevant memory-model sense |
| `release` | Prevents preceding operations from moving after the release in the relevant memory-model sense |
| `acq_rel` | Combines acquire and release semantics for read-modify-write operations |
| `seq_cst` | Adds a single global order across sequentially consistent operations |

Note: These are C++ memory-model guarantees, not simply literal instructions preventing every kind of CPU movement.

### Interview Questions

**Q: When is `memory_order_relaxed` appropriate?**

When atomicity is required but the operation does not need to publish or synchronize other memory accesses.

For example, an independent statistics counter.

**Q: What is the difference between acquire and release?**

Release publishes preceding memory operations.

Acquire allows a thread to observe those published operations when it reads from the corresponding release sequence.

**Q: Why not always use `memory_order_seq_cst`?**

Sequential consistency is easier to reason about and is often a good default.

However, it may impose stronger ordering constraints than necessary, potentially reducing optimization opportunities or introducing additional hardware costs.

**Q: What memory ordering is commonly used in an SPSC ring buffer?**

The producer typically publishes its updated write index using release semantics.

The consumer observes the write index using acquire semantics.

The reverse applies to the read index when communicating available capacity back to the producer.

---

## 5.8 Memory Barriers

### Definition

A memory barrier (or memory fence) imposes ordering constraints on memory operations.

Barriers are used to prevent certain compiler and hardware reorderings that could violate synchronization requirements.

### How It Works

There are two important sources of reordering:

1. **Compiler reordering:** The compiler rearranges instructions during optimization.
2. **CPU reordering:** The processor executes or makes memory operations visible in an order different from program order, subject to architectural rules.

Consider:

```cpp
data = 42;
ready = true;
```

Without appropriate synchronization, another thread cannot safely assume that observing `ready == true` means it can also observe `data == 42`.

Atomic release/acquire synchronization provides the required ordering.

C++ also supports explicit fences:

```cpp
std::atomic_thread_fence(std::memory_order_release);
```

However, fences require carefully constructed synchronization relationships and should not be used as arbitrary replacements for atomic operations.

### Key Concepts

| Concept | Explanation |
|---|---|
| Compiler Barrier | Restricts compiler instruction reordering |
| Hardware Memory Barrier | Enforces ordering constraints at the CPU level |
| Acquire Fence | Participates in ordering subsequent memory operations |
| Release Fence | Participates in ordering preceding memory operations |
| Full Barrier | Provides stronger bidirectional ordering constraints |
| Happens-Before | Formal C++ relationship establishing operation ordering |

### Interview Questions

**Q: What is the difference between a compiler barrier and a CPU memory barrier?**

A compiler barrier restricts compiler transformations.

A hardware memory barrier constrains how the processor orders memory operations.

C++ memory-ordering primitives provide language-level guarantees that the compiler maps to suitable architecture-specific instructions.

**Q: Are memory barriers always implemented as explicit CPU fence instructions?**

No.

Some ordering guarantees are already provided by the underlying architecture or by the instructions used for atomic operations.

For example, acquire/release loads and stores on x86-64 often require no additional hardware fence instructions.

**Q: Why are memory barriers important for lock-free algorithms?**

Lock-free algorithms often publish shared data using atomic variables.

Correct ordering ensures that another thread cannot observe a published state before the associated data is safely visible.

---

## 5.9 Lock-Free Algorithms

### Definition

A lock-free algorithm guarantees system-wide progress: even if some threads are delayed, the system as a whole continues completing operations.

Lock-free does not mean every thread is guaranteed to make progress independently.

### How It Works

Lock-free algorithms commonly coordinate shared state using atomic operations such as compare-and-swap.

Example: A CAS retry loop.

```cpp
std::atomic<int> counter{0};

void Increment() {
  int expected = counter.load(std::memory_order_relaxed);

  while (!counter.compare_exchange_weak(
      expected,
      expected + 1,
      std::memory_order_relaxed)) {
    // Retry using the updated expected value.
  }
}
```

If another thread modifies the counter between the load and CAS, the CAS may fail and retry.

### Key Concepts

| Concept | Explanation |
|---|---|
| Blocking | Progress may depend on another thread releasing a resource |
| Lock-Free | Guarantees system-wide progress |
| Wait-Free | Guarantees bounded completion for every operation |
| Obstruction-Free | Guarantees progress when a thread executes without interference |
| CAS Loop | Repeatedly attempts an atomic conditional update |
| ABA Problem | A value changes A → B → A, making CAS unable to detect the intermediate modification |
| Memory Reclamation | Safely recovering memory that concurrent readers may still access |

### Interview Questions

**Q: What is the difference between lock-free and wait-free?**

Lock-free guarantees that at least one thread continues making progress.

Wait-free guarantees that every operation completes within a bounded number of its own steps.

Wait-free provides a stronger progress guarantee.

**Q: What is the ABA problem?**

Suppose a thread reads pointer A.

Another thread changes the shared pointer from A to B and eventually back to A.

The first thread's CAS may succeed because the pointer still equals A, even though the underlying structure changed.

Common solutions include tagged pointers, hazard pointers, and appropriate memory-reclamation schemes.

**Q: Is lock-free always faster than using a mutex?**

No.

Under contention, repeated CAS failures and cache-line ownership transfers can be expensive.

Lock-free algorithms are often chosen for their progress guarantees and predictable avoidance of lock-owner blocking, not simply because they always provide higher throughput.

**Q: Why is memory reclamation difficult in lock-free data structures?**

A node removed by one thread may still be referenced by another.

Freeing it too early can cause use-after-free bugs.

Techniques such as hazard pointers and epoch-based reclamation defer reclamation until it is safe.

---

## 5.10 Concurrent Queues

### Definition

A concurrent queue allows multiple execution contexts to exchange data safely.

Concurrent queues are commonly used to decouple producers and consumers in multithreaded systems.

### How It Works

Queues are often classified by their producer/consumer topology:

| Queue | Producers | Consumers |
|---|---:|---:|
| SPSC | Single | Single |
| MPSC | Multiple | Single |
| SPMC | Single | Multiple |
| MPMC | Multiple | Multiple |

An SPSC ring buffer is particularly useful for low-latency pipelines because each index has a single writer.

Conceptual layout:

```text
Fixed-size Ring Buffer

+-----+-----+-----+-----+-----+-----+-----+-----+
|  A  |  B  |  C  |     |     |     |     |     |
+-----+-----+-----+-----+-----+-----+-----+-----+
                  ^     ^
                 Read  Write
```

Typical ownership rules:

- Producer owns and updates the write index.
- Consumer owns and updates the read index.
- Producer reads the consumer's read index to detect available capacity.
- Consumer reads the producer's write index to detect available data.

For a ring buffer that reserves one slot to distinguish full from empty:

```cpp
bool Empty = (write_index == read_index);

bool Full =
    ((write_index + 1) % Capacity == read_index);
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Ring Buffer | Fixed-size circular storage |
| SPSC | Single-Producer Single-Consumer |
| MPSC | Multi-Producer Single-Consumer |
| MPMC | Multi-Producer Multi-Consumer |
| Head / Tail | Indices tracking queue state |
| Backpressure | Mechanism for handling production faster than consumption |
| False Sharing | Independent queue metadata contending on the same cache line |

### Interview Questions

**Q: Why is an SPSC queue simpler than an MPMC queue?**

In SPSC, each index has exactly one writer.

This avoids the need for multiple producers or consumers to compete for ownership of the same index through CAS or other coordination mechanisms.

**Q: Why are ring buffers commonly used in low-latency trading systems?**

They provide:

- Fixed-capacity storage.
- Predictable memory usage.
- No per-element dynamic allocation.
- Good cache locality.
- Efficient producer-consumer communication.

**Q: Why does a ring buffer commonly reserve one unused slot?**

If only read and write indices are maintained, reserving one slot provides a simple distinction:

- Empty: `read == write`.
- Full: advancing `write` would equal `read`.

Other designs can use all slots by maintaining additional state or monotonic sequence counters.

**Q: How should an SPSC queue use acquire/release ordering?**

The producer:

1. Writes the element into the buffer.
2. Publishes the updated write index with release semantics.

The consumer:

1. Reads the write index with acquire semantics.
2. Reads the published element.
3. Publishes the updated read index with release semantics.

The producer uses an acquire load of the read index when it needs to observe released capacity.

**Q: What happens when the producer is faster than the consumer?**

The queue eventually fills.

The application must define a backpressure policy, such as:

- Rejecting new submissions.
- Blocking or waiting.
- Spinning until capacity becomes available.
- Dropping data, but only when the application protocol permits it.

For order submission, silently dropping messages is generally unacceptable.

---

## Section 5 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the difference between a race condition and a data race.
- [ ] Why does `volatile` not provide thread synchronization?
- [ ] How does a mutex work, and what happens under contention?
- [ ] Compare mutexes and spinlocks.
- [ ] Why can spinlocks perform poorly under CPU oversubscription?
- [ ] Explain how condition variables work.
- [ ] Why must condition-variable waits always recheck a predicate?
- [ ] What is the difference between a mutex and a semaphore?
- [ ] Explain atomic load, store, fetch-add, and compare-and-swap.
- [ ] What is the difference between weak and strong CAS?
- [ ] Explain relaxed, acquire, release, acq_rel, and sequential consistency.
- [ ] What is a happens-before relationship?
- [ ] What is the difference between compiler and hardware memory barriers?
- [ ] Explain lock-free vs wait-free progress guarantees.
- [ ] What is the ABA problem?
- [ ] Why is memory reclamation challenging in lock-free algorithms?
- [ ] Compare SPSC, MPSC, SPMC, and MPMC queues.
- [ ] Implement an SPSC ring buffer using C++ atomics.
- [ ] Explain why an SPSC queue needs acquire/release synchronization.
- [ ] How would you handle backpressure when a producer outpaces its consumer?
