# 8. Memory Allocation & Resource Management

## 8.1 Process Memory Layout

### Definition

A process's virtual address space is divided into regions serving different purposes, such as executable code, global variables, dynamically allocated memory, stacks, and memory mappings.

These regions are managed through the operating system's virtual-memory subsystem.

### How It Works

A simplified Linux process memory layout:

```text
Higher Virtual Addresses
+---------------------------+
| Kernel-space mappings     |
+---------------------------+
| Thread stacks / mappings  |
+---------------------------+
| Memory-mapped regions     |
| Shared libraries / mmap   |
+---------------------------+
|                           |
| Heap                      |
|                           |
+---------------------------+
| BSS                       |
| Uninitialized globals     |
+---------------------------+
| Data                      |
| Initialized globals       |
+---------------------------+
| Read-only data            |
+---------------------------+
| Text                      |
| Executable instructions   |
+---------------------------+
Lower Virtual Addresses
```

This is conceptual: actual addresses and region placement depend on architecture, executable format, ASLR, and OS configuration.

The kernel maintains virtual memory areas (VMAs) describing mapped regions and their permissions.

### Key Concepts

| Region | Purpose |
|---|---|
| Text | Executable instructions |
| Read-only Data | Constants and other read-only objects |
| Data | Initialized global/static variables |
| BSS | Zero-initialized or uninitialized global/static variables |
| Heap | Dynamic allocation |
| Stack | Function frames and automatic storage |
| mmap Region | Files, shared memory, anonymous mappings, shared libraries |

### Interview Questions

**Q: Where are global, local, and dynamically allocated variables stored?**

Typically:
- Global/static variables: Data, BSS, or read-only regions.
- Ordinary local variables: Stack or registers, depending on optimization.
- Dynamically allocated objects: Memory managed by an allocator, commonly backed by the heap or `mmap`.

**Q: Does every thread have its own heap?**

Threads in the same process share its virtual address space and can access the same dynamically allocated objects.

Each thread has its own stack. Allocators may additionally maintain thread-local allocation caches or arenas.

**Q: What is ASLR?**

Address Space Layout Randomization randomizes the placement of selected virtual-memory regions to make memory-corruption exploits harder.

---

## 8.2 Stack vs Heap

### Definition

The stack and heap are two important memory-management mechanisms.

The stack supports function execution and automatic storage duration.

The heap supports dynamic allocation with lifetimes determined by the application.

### How It Works

Consider:

```cpp
void ProcessOrder() {
  int quantity = 100;

  Order* order = new Order{};
  delete order;
}
```

Conceptually:

```text
Stack                         Heap
+------------------+          +------------------+
| quantity = 100   |          | Order object     |
| order pointer ---+--------> |                  |
+------------------+          +------------------+
```

The pointer variable may reside on the stack, while the dynamically allocated object resides in allocator-managed memory.

The compiler can optimize either representation when observable behavior permits.

### Key Concepts

| Property | Stack | Heap |
|---|---|---|
| Allocation | Usually automatic | Explicit/dynamic |
| Lifetime | Generally scope-based | Application-controlled |
| Allocation Cost | Usually very low | Depends on allocator |
| Capacity | Usually relatively limited | Depends on available virtual memory/resources |
| Fragmentation | Usually limited by stack discipline | Can be significant |
| Thread Ownership | Each thread has a stack | Dynamic objects can be shared |

### Interview Questions

**Q: Why is stack allocation generally faster than heap allocation?**

Stack allocation commonly requires only adjusting the stack pointer, and compilers can often eliminate the operation entirely.

Heap allocation may require allocator metadata operations, synchronization, searching for free blocks, or obtaining more memory from the OS.

**Q: Can stack memory cause page faults?**

Yes.

Stack pages may be populated lazily, and stack growth can trigger page faults.

**Q: Why avoid allocating large arrays on the stack?**

Thread stacks have limited capacity. Large allocations can exhaust the stack and cause a stack overflow.

**Q: Is heap allocation inherently slow?**

No.

Modern allocators optimize common allocation sizes and may use thread-local caches.

The main concern in low-latency systems is often unpredictable worst-case allocation latency rather than average allocation speed.

---

## 8.3 Dynamic Memory Allocation

### Definition

Dynamic memory allocation allows applications to request memory during execution instead of fixing all storage requirements at compile time.

In C++, common allocation interfaces include `new`, `delete`, `malloc()`, and `free()`.

### How It Works

```cpp
Order* order = new Order{};

delete order;
```

A typical allocation involves:

1. Determining the required size and alignment.
2. Requesting storage from the allocator.
3. Obtaining a suitable memory block.
4. Constructing the object, for C++ `new`.
5. Returning a pointer.

Deallocation destroys the object, when applicable, and returns storage to the allocator.

Importantly, returning memory to the allocator does not necessarily return physical memory to the operating system immediately.

### Key Concepts

| Concept | Explanation |
|---|---|
| `malloc()` | Allocates raw storage |
| `free()` | Releases storage allocated by compatible C allocation APIs |
| `new` | Allocates storage and constructs a C++ object |
| `delete` | Destroys an object and releases its storage |
| Allocation Alignment | Required address boundary |
| Allocation Failure | Inability to satisfy an allocation request |
| `std::bad_alloc` | Exception normally thrown by a failing ordinary C++ `new` |

### Interview Questions

**Q: What is the difference between `malloc()` and `new`?**

`malloc()` allocates raw storage without constructing a C++ object.

A `new` expression allocates storage and initializes the requested object.

Similarly, `delete` performs object destruction before deallocation.

**Q: Does every `new` call result in a system call?**

No.

Allocators generally obtain larger memory regions from the OS and satisfy many individual allocations from those regions.

**Q: Why can dynamic allocation introduce latency spikes?**

Occasionally, the allocator may need to acquire locks, expand its memory pool, request additional mappings, populate pages, or perform internal maintenance.

**Q: Should a trading system completely avoid dynamic allocation?**

Not necessarily.

A common strategy is to allocate and initialize required resources during startup, then avoid unexpected allocation on latency-sensitive execution paths.

---

## 8.4 Allocator Internals

### Definition

A memory allocator manages reusable blocks of storage obtained from the operating system.

Its job is to satisfy allocation requests efficiently while controlling fragmentation and memory overhead.

### How It Works

A simplified allocator:

```text
Application
    |
    | malloc(64)
    v
+-----------------------+
| Allocator             |
|                       |
| 64-byte free list     |
| 128-byte free list    |
| 256-byte free list    |
| Large allocations     |
+-----------------------+
    |
    | Request more memory
    v
Operating System
    |
    +--> brk / mmap
```

Many allocators organize small allocations into size classes.

A request for 50 bytes might be satisfied from a larger size class, depending on the allocator.

Modern implementations may also use:

- Thread-local caches.
- Multiple arenas.
- Free lists.
- Slabs.
- Per-size allocation pools.

### Key Concepts

| Concept | Explanation |
|---|---|
| Arena | Region or collection of memory managed by an allocator |
| Size Class | Group of allocations served by similarly sized blocks |
| Free List | Structure tracking available blocks |
| Thread Cache | Per-thread storage of reusable allocation blocks |
| Slab Allocation | Allocates objects from groups of similarly sized slots |
| Allocator Contention | Multiple threads compete for shared allocator metadata |

### Interview Questions

**Q: Why do allocators use size classes?**

They simplify block management and allow previously freed blocks to be reused efficiently.

**Q: Why do allocators maintain thread-local caches?**

They reduce contention on shared allocator structures and allow common allocations to complete without cross-thread synchronization.

**Q: What happens when an allocator runs out of available blocks?**

It may request additional memory from the OS, reuse other available regions, or fail the allocation if resources are insufficient.

**Q: Why is average allocation time insufficient for evaluating an HFT allocator?**

An allocator might be fast for most requests but occasionally perform expensive operations.

Tail latency and maximum observed allocation time are therefore important measurements.

---

## 8.5 Memory Pools & Object Pools

### Definition

A memory pool preallocates storage and manages it for subsequent allocation requests.

An object pool specializes this idea for objects of a particular type or size.

The goal is to reduce repeated general-purpose allocation overhead and improve predictability.

### How It Works

Consider an order-processing system that repeatedly creates and destroys order objects.

Instead of calling `new` for every order:

```text
Startup:

Allocate storage for 100,000 Orders
                |
                v
+------+------+------+------+------+
| Slot | Slot | Slot | Slot | ...  |
+------+------+------+------+------+
                |
                v
           Free List
```

When a new order arrives:

1. Obtain an available slot.
2. Construct or initialize the order.
3. Process the order.
4. Destroy or reset the order when appropriate.
5. Return the slot to the pool.

### Key Concepts

| Concept | Explanation |
|---|---|
| Memory Pool | Preallocated reusable storage |
| Object Pool | Pool managing objects of a specific type |
| Free List | Tracks available slots |
| Fixed Capacity | Maximum number of simultaneously allocated objects |
| Placement New | Constructs an object in existing storage |
| Pool Exhaustion | Occurs when no free slots remain |

### Interview Questions

**Q: Why use an object pool for an order book?**

Order creation and cancellation can occur frequently.

A preallocated pool reduces dependence on general-purpose allocation during the hot path and can improve memory locality and latency predictability.

**Q: What happens when the pool becomes full?**

The application must define an explicit policy, such as rejecting the operation, using a controlled fallback, or expanding the pool.

For deterministic latency, unexpected fallback allocation may be undesirable.

**Q: Does returning an object to the pool automatically destroy it?**

No.

That depends on the pool implementation.

For non-trivial C++ objects, construction, destruction, and storage reuse must be handled correctly.

**Q: What is a disadvantage of object pools?**

They can reserve more memory than necessary, introduce capacity constraints, and complicate lifetime management.

They should be justified by actual allocation patterns and benchmarks.

---

## 8.6 Custom Allocators

### Definition

A custom allocator is a memory-management strategy designed for a particular application's allocation patterns and performance requirements.

Unlike a general-purpose allocator, it can exploit knowledge of object sizes, lifetimes, and access patterns.

### How It Works

Common allocator designs:

| Allocator | Strategy |
|---|---|
| Bump Allocator | Advances a pointer through preallocated storage |
| Arena Allocator | Allocates many objects and releases them together |
| Pool Allocator | Reuses fixed-size blocks |
| Slab Allocator | Manages groups of similarly sized objects |
| Free-List Allocator | Tracks and reuses released blocks |

A bump allocator is conceptually simple:

```text
Preallocated Memory
+-------------------------------------------+
| Used | Used | Used | Free Space            |
+-------------------------------------------+
                     ^
                 Next Allocation
```

Allocation typically involves checking capacity and advancing a pointer.

Individual deallocation is often unsupported; instead, the entire arena is reset.

### Key Concepts

| Concept | Explanation |
|---|---|
| Bump Pointer | Address of the next available allocation position |
| Arena Reset | Releases all allocations associated with an arena |
| Allocation Policy | Rules governing storage selection |
| Object Lifetime | Period during which a constructed object exists |
| Alignment | Boundary requirements for allocated objects |
| `std::pmr` | C++ polymorphic memory-resource framework |

### Interview Questions

**Q: When is a bump allocator useful?**

When many objects have similar lifetimes and can be released together.

For example, temporary data used during one batch-processing cycle.

**Q: When is a pool allocator preferable to an arena allocator?**

When individual objects must be allocated and released independently, especially if they have uniform or predictable sizes.

**Q: Why might a custom allocator outperform `malloc()`?**

It can eliminate unnecessary general-purpose bookkeeping, reduce synchronization, improve locality, and make allocation costs more predictable.

**Q: What should you benchmark before adopting a custom allocator?**

- Average allocation latency.
- p99 and maximum allocation latency.
- Deallocation cost.
- Memory overhead.
- Fragmentation.
- Behavior under realistic object lifetimes.
- Multithreaded contention, where applicable.

---

## 8.7 Copy-on-Write (COW)

### Definition

Copy-on-Write is a memory-management technique that allows multiple processes or mappings to share physical memory until one participant attempts to modify it.

A private physical copy is created only when necessary.

### How It Works

A common example is `fork()`.

Immediately after a fork, the parent and child can share many of the same physical pages.

```text
Before modification:

Parent                    Child
   |                         |
   +-----------+-------------+
               |
               v
        Shared Physical Page
```

When the child modifies a shared private page:

```text
Child writes
     |
     v
Page fault
     |
     v
Kernel allocates new frame
     |
     v
Copies original page
     |
     v
Child mapping updated
```

The parent continues using the original page.

### Key Concepts

| Concept | Explanation |
|---|---|
| COW | Defers copying until modification |
| `fork()` | Creates a child process |
| Private Mapping | Mapping using private copy-on-write semantics |
| Write-Protected Page | Page temporarily protected to detect writes |
| COW Fault | Fault triggered when a shared private page is modified |
| Page Reference Count | Helps track references to underlying physical pages |

### Interview Questions

**Q: Does `fork()` immediately copy all of the parent's physical memory?**

No.

Linux commonly uses copy-on-write so parent and child initially share physical pages where possible.

Some kernel bookkeeping still needs to be established.

**Q: What triggers a copy-on-write page fault?**

A write to a page currently shared under copy-on-write semantics.

The kernel creates or assigns an appropriate private writable page and updates the mapping.

**Q: Why can COW introduce unpredictable latency?**

The first write may trigger a page fault, physical allocation, copying, and page-table updates.

**Q: Is COW relevant to low-latency systems?**

Yes.

Applications should understand whether their memory-management strategy can introduce COW faults during critical execution.

For example, pre-faulting memory before a later `fork()` does not eliminate COW faults caused by subsequent writes.

---

## 8.8 Memory Fragmentation

### Definition

Memory fragmentation occurs when available memory is divided or used inefficiently, reducing allocation efficiency.

Two important categories are internal and external fragmentation.

### How It Works

**Internal fragmentation**

An allocator provides a block larger than the requested allocation.

```text
Requested: 50 bytes

Allocated Block: 64 bytes
+-------------------------+-------+
| 50 bytes used           | 14 B  |
+-------------------------+-------+
                           Wasted
```

**External fragmentation**

Free memory exists, but it is distributed across separate regions.

```text
+------+------+------+------+------+------+
| Used | Free | Used | Free | Used | Free |
+------+------+------+------+------+------+
```

Even if total free memory is sufficient, a particular contiguous allocation request may be difficult to satisfy.

### Key Concepts

| Concept | Explanation |
|---|---|
| Internal Fragmentation | Unused space inside allocated blocks |
| External Fragmentation | Free space scattered across separate regions |
| Coalescing | Combining adjacent free blocks |
| Compaction | Rearranging memory to create larger contiguous regions |
| Size Classes | Simplify allocation but may introduce internal fragmentation |
| Virtual Contiguity | Consecutive virtual addresses need not map to consecutive physical frames |

### Interview Questions

**Q: What is the difference between internal and external fragmentation?**

Internal fragmentation wastes space inside allocated blocks.

External fragmentation leaves free memory distributed across separate regions, making some allocation requests difficult.

**Q: Can virtual memory reduce external fragmentation?**

It allows a virtually contiguous region to map to noncontiguous physical frames.

However, physical fragmentation still matters for operations requiring physically contiguous memory, such as some huge-page allocations.

**Q: How can memory pools reduce fragmentation?**

Fixed-size pools avoid arbitrary allocation sizes and allow slots to be reused predictably.

However, they can introduce internal fragmentation or waste capacity when underutilized.

**Q: Why does fragmentation matter for long-running trading applications?**

Allocation patterns may evolve over hours or days, increasing memory overhead and causing occasional expensive allocator operations.

Long-duration benchmarks can reveal issues that short microbenchmarks miss.

---

## 8.9 Memory Reclamation

### Definition

Memory reclamation is the process of safely recovering storage that is no longer needed.

In concurrent systems, reclamation becomes difficult because another thread may still hold a reference to an object that has logically been removed.

### How It Works

Consider a lock-free linked structure:

```text
Thread A                       Thread B
--------                       --------

Reads pointer to Node X
                               Removes Node X
                               Frees Node X

Attempts to access Node X
          |
          v
     Use-after-free
```

Removing an object from a shared structure does not guarantee that no thread still references it.

Common reclamation techniques:

| Technique | Description |
|---|---|
| Reference Counting | Tracks outstanding references |
| Hazard Pointers | Threads announce which objects they may access |
| Epoch-Based Reclamation | Defers freeing objects until earlier readers have completed |
| RCU | Read-Copy-Update; readers access data while reclamation waits for a grace period |
| Deferred Reclamation | Retires objects and frees them later when safe |

### Key Concepts

| Concept | Explanation |
|---|---|
| Use-After-Free | Accessing an object after its lifetime ends |
| Memory Leak | Allocated memory is never reclaimed |
| Hazard Pointer | Protects an object currently referenced by a thread |
| Epoch | Logical period used to track concurrent readers |
| Grace Period | Interval after which earlier readers are known to have finished |
| ABA Problem | A shared value changes A → B → A, hiding an intermediate change |

### Interview Questions

**Q: Why is memory reclamation difficult in lock-free algorithms?**

A thread may still be reading an object after another thread removes it from the shared data structure.

Freeing it immediately can cause use-after-free.

**Q: What is a hazard pointer?**

A hazard pointer is a published reference indicating that a thread may currently access a particular object.

Reclamation is delayed while the object remains protected.

**Q: What is epoch-based reclamation?**

Threads participate in logical epochs.

Removed objects are retired and reclaimed only after all relevant readers from earlier epochs have completed.

**Q: What would you use for an SPSC queue with fixed storage?**

A fixed-size ring buffer can avoid general-purpose dynamic reclamation for individual queue slots.

However, the producer must not overwrite a slot before the consumer has finished reading it.

Correct index ownership and acquire/release synchronization enforce that lifecycle.

---

## 8.10 Swapping & Memory Pressure

### Definition

Memory pressure occurs when the system has insufficient readily available physical memory for its active workload.

Swapping allows the OS to move eligible memory pages between RAM and swap storage to free physical memory.

### How It Works

A simplified memory-reclamation process:

```text
Physical memory pressure
          |
          v
Kernel attempts reclamation
          |
          +--> Reclaim clean file-backed pages
          |
          +--> Write back dirty file-backed pages
          |
          +--> Swap eligible anonymous pages
          |
          v
More physical memory available
```

If a swapped-out page is accessed again, a major page fault may be required to retrieve it.

Memory pressure can also trigger reclaim work, writeback, compaction, and ultimately out-of-memory handling if sufficient memory cannot be obtained.

### Key Concepts

| Concept | Explanation |
|---|---|
| Memory Pressure | Demand for physical memory approaches or exceeds available capacity |
| Swap | Storage used to hold evicted memory pages |
| Page Reclaim | Kernel process of recovering physical memory |
| Major Page Fault | Fault requiring disk I/O |
| OOM Killer | Linux mechanism that may terminate processes under unrecoverable memory pressure |
| `mlock()` | Prevents selected resident pages from being swapped out |
| Page Cache | RAM used to cache file-backed data |

### Interview Questions

**Q: Why is swapping undesirable for low-latency applications?**

Accessing swapped-out memory can require storage I/O, producing latency far beyond ordinary RAM access.

**Q: Does disabling swap eliminate all memory-related latency spikes?**

No.

Page faults, direct reclaim, compaction, allocation slow paths, and other memory-management operations can still introduce latency.

**Q: What does `mlockall()` help prevent?**

It prevents locked memory pages from being swapped out.

However, it does not guarantee that the application will never experience another page fault.

**Q: How would you reduce memory-management jitter in a trading application?**

1. Preallocate critical memory.
2. Initialize and touch required pages before the hot path begins.
3. Lock critical memory where appropriate.
4. Use fixed-capacity pools for frequently created objects.
5. Avoid unexpected allocation and deallocation.
6. Keep CPU and memory placement NUMA-aware.
7. Monitor page faults, memory pressure, and allocation tail latency.

---

## Section 8 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the major regions of a Linux process's virtual address space.
- [ ] What is the difference between stack and heap allocation?
- [ ] Does every thread have its own heap?
- [ ] Compare `malloc/free` with C++ `new/delete`.
- [ ] Does every dynamic allocation require a system call?
- [ ] How do general-purpose allocators manage free memory?
- [ ] What are arenas, size classes, and thread-local caches?
- [ ] Why can dynamic allocation introduce tail-latency spikes?
- [ ] What is an object pool, and why is it useful for an order book?
- [ ] Compare bump, arena, pool, and free-list allocators.
- [ ] What happens when a fixed-capacity memory pool is exhausted?
- [ ] Explain copy-on-write and its relationship to `fork()`.
- [ ] What triggers a COW page fault?
- [ ] Compare internal and external fragmentation.
- [ ] Why is memory reclamation difficult in lock-free algorithms?
- [ ] Explain hazard pointers and epoch-based reclamation.
- [ ] How does a fixed-size SPSC queue avoid dynamic memory reclamation?
- [ ] What happens when Linux experiences memory pressure?
- [ ] Why can swapping introduce severe latency spikes?
- [ ] Design a memory-management strategy for a low-latency C++ matching engine.
