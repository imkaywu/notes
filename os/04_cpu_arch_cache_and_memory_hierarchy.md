# 4. CPU Architecture, Caches & Memory Hierarchy

## 4.1 CPU Registers

### Definition

CPU registers are small, extremely fast storage locations inside the processor used to hold data, addresses, and execution state.

Registers are the fastest storage available to the CPU and are directly accessed by machine instructions.

### How It Works

On x86-64, important registers include:

| Register | Purpose |
|---|---|
| RAX | General-purpose register, commonly used for return values |
| RBX | General-purpose register |
| RCX | General-purpose register, often used for counters |
| RDX | General-purpose register |
| RSI / RDI | General-purpose registers, commonly used for function arguments |
| RSP | Stack pointer |
| RBP | General-purpose register, conventionally used as frame pointer |
| RIP | Instruction pointer |
| RFLAGS | CPU status and control flags |
| XMM / YMM / ZMM | SIMD/vector registers |

Example:

```asm
mov rax, 10
add rax, 20
```

The CPU performs the addition using register values without needing to fetch the operands from main memory.

During a thread context switch, the OS preserves and restores the necessary architectural register state.

### Key Concepts

| Concept | Explanation |
|---|---|
| General-Purpose Register | Holds integer values, addresses, or intermediate results |
| RIP | Address of the next instruction to execute |
| RSP | Points to the current stack location |
| RFLAGS | Contains arithmetic and processor-control flags |
| SIMD Registers | Support operations on multiple data elements |
| Register Spilling | Moving values from registers into memory when register capacity is insufficient |

### Interview Questions

**Q: Why are registers faster than CPU caches?**

Registers are directly integrated into the processor's execution architecture and can be accessed by instructions without performing a cache lookup.

**Q: What is register spilling?**

When the compiler cannot keep all required values in registers, it temporarily stores some values in memory, commonly on the stack.

This introduces additional load/store instructions.

**Q: Why do registers matter for low-latency C++?**

Efficient register usage reduces memory accesses and instruction overhead.

However, register allocation is primarily controlled by the compiler, so understanding generated assembly is often more useful than manually trying to control individual registers.

---

## 4.2 CPU Cache Hierarchy

### Definition

CPU caches are small, fast memory structures that store recently or frequently accessed data and instructions.

They reduce the latency of accessing data stored in main memory.

### How It Works

A typical CPU contains several cache levels:

```text
             CPU Core
                |
          +-----------+
          | Registers |
          +-----------+
                |
          +-----------+
          | L1 Cache  |
          +-----------+
                |
          +-----------+
          | L2 Cache  |
          +-----------+
                |
          +-----------+
          | L3 Cache  |
          +-----------+
                |
          +-----------+
          | Main RAM  |
          +-----------+
```

Typical characteristics:

| Level | Relative Size | Relative Latency | Common Organization |
|---|---|---|---|
| Registers | Smallest | Lowest | Per logical execution context |
| L1 | Small | Very low | Usually private to a physical core |
| L2 | Medium | Low | Usually private to a physical core |
| L3 | Large | Higher | Often shared across multiple cores |
| RAM | Very large | Highest | External main memory |

Actual cache sizes and latencies depend on CPU architecture.

Most modern CPUs have separate L1 instruction and data caches.

### Key Concepts

| Concept | Explanation |
|---|---|
| L1I | Level-1 instruction cache |
| L1D | Level-1 data cache |
| L2 | Second-level cache |
| LLC | Last-Level Cache, commonly L3 |
| Temporal Locality | Recently accessed data is likely to be accessed again |
| Spatial Locality | Nearby memory locations are likely to be accessed |

### Interview Questions

**Q: What happens when data is not found in L1 cache?**

The CPU searches the next level of the memory hierarchy.

If the data is absent from all relevant cache levels, it must eventually be fetched from main memory.

**Q: Why is cache locality important for an order book?**

Order processing repeatedly accesses price levels, order metadata, and related structures.

Keeping frequently accessed data close together improves the probability that the required data is already cached.

**Q: Does a larger cache always guarantee better performance?**

No. Performance also depends on access patterns, cache latency, associativity, contention, and the application's working-set size.

---

## 4.3 Cache Lines

### Definition

A cache line is the basic unit of data transfer and storage within the CPU cache hierarchy.

On most modern x86-64 processors, a cache line is 64 bytes.

### How It Works

Suppose an application accesses one integer:

```cpp
int value = array[0];
```

The CPU generally fetches the entire cache line containing that integer, rather than fetching only its four bytes.

Example:

```text
Memory:

Address
0x1000  +---------------------+
        | int array[0]        |
0x1004  | int array[1]        |
0x1008  | int array[2]        |
        | ...                 |
0x103C  | int array[15]       |
0x1040  +---------------------+

        One 64-byte cache line
```

A cache line aligned at `0x1000` covers addresses:

`0x1000` through `0x103F`.

Accessing another integer in the same resident line can reuse the data already fetched.

### Key Concepts

| Concept | Explanation |
|---|---|
| Cache Line | Basic unit of cache storage and transfer |
| Cache-Line Size | Commonly 64 bytes on x86-64 |
| Spatial Locality | Accessing nearby memory locations |
| Cache-Line Boundary | Address separating adjacent cache lines |
| Cache-Line Fill | Bringing a missing cache line into the cache hierarchy |

### Interview Questions

**Q: Why does the CPU fetch an entire cache line instead of individual bytes?**

Transferring data in blocks exploits spatial locality and amortizes memory-access overhead.

**Q: If two variables occupy the same cache line, does accessing one automatically make the other available?**

The entire line is brought into the relevant cache level, so the other variable's bytes are present in that line.

However, additional instruction execution and data dependencies still apply.

**Q: Why is cache-line size important when designing data structures?**

It influences memory layout, locality, alignment, and false sharing.

For example, packing frequently accessed fields into the same cache line may improve locality, while placing independently modified variables on the same line may create contention.

---

## 4.4 Cache Associativity

### Definition

Cache associativity determines how many possible locations within a cache can store a particular memory block.

Most modern CPU caches are **set-associative**.

### How It Works

A memory address is conceptually divided into:

```text
+----------------+-------------+-------------+
|      Tag       |  Set Index  | Block Offset|
+----------------+-------------+-------------+
```

For a 64-byte cache line, the block offset requires six bits.

Consider a hypothetical cache:

- Capacity: 32 KiB.
- Cache-line size: 64 bytes.
- Associativity: 8-way.

Number of cache sets:

`32 KiB / (64 × 8) = 64 sets`

Each memory block maps to one of these 64 sets.

Within its selected set, it may occupy any of the eight ways.

```text
Set 0: [Way 0][Way 1]...[Way 7]
Set 1: [Way 0][Way 1]...[Way 7]
...
Set 63:[Way 0][Way 1]...[Way 7]
```

If more than eight actively used memory blocks compete for the same set, cache lines may repeatedly evict one another.

### Key Concepts

| Concept | Explanation |
|---|---|
| Direct-Mapped | Each block maps to exactly one cache location |
| Fully Associative | A block can occupy any cache entry |
| Set-Associative | A block maps to one set but can occupy multiple ways |
| Cache Set | Group of possible storage locations |
| Cache Tag | Identifies which memory block occupies a cache entry |
| Conflict Miss | Miss caused by multiple blocks competing for the same set |

### Interview Questions

**Q: How does the CPU determine whether an address is present in the cache?**

Conceptually:

1. Extract the set index from the address.
2. Select the corresponding cache set.
3. Compare the requested tag against the valid entries in that set.
4. If a matching tag exists, the lookup is a cache hit.
5. Use the block offset to select the requested bytes.

Actual cache indexing can be more complex, especially for physically indexed caches.

**Q: What is an 8-way set-associative cache?**

Each cache set contains eight possible locations for storing cache lines.

A memory block maps to one set and can occupy any of its eight ways.

**Q: Can a program suffer cache misses even when its working set is smaller than the cache?**

Yes.

If too many frequently accessed addresses map to the same cache set, conflict misses can occur despite unused capacity elsewhere in the cache.

---

## 4.5 Cache Hits & Misses

### Definition

A cache hit occurs when requested data is already present in the relevant cache.

A cache miss occurs when the requested data must be obtained from another level of the memory hierarchy.

### How It Works

```text
CPU requests data
       |
       v
    L1 Lookup
     /    \
   Hit    Miss
    |       |
    |       v
    |    L2 Lookup
    |     /    \
    |   Hit    Miss
    |    |       |
    |    |       v
    |    |    L3 Lookup
    |    |     /    \
    |    |   Hit    Miss
    |    |    |       |
    |    |    |       v
    |    |    |      RAM
    |    |    |       |
    +----+----+-------+
         |
         v
      CPU uses data
```

Cache misses are commonly classified into three categories:

| Type | Explanation |
|---|---|
| Compulsory Miss | First access to a block that has not been cached |
| Capacity Miss | Working set exceeds effective cache capacity |
| Conflict Miss | Multiple blocks compete for the same cache set |

### Key Concepts

| Concept | Explanation |
|---|---|
| Cache Hit Rate | Percentage of accesses served by a cache |
| Cache Miss Rate | Percentage of accesses missing in a cache |
| Cache Pollution | Useful data is displaced by less useful data |
| Working Set | Data actively accessed during a period |
| Memory-Level Parallelism | Ability to overlap multiple outstanding memory accesses |

### Interview Questions

**Q: What is the difference between a capacity miss and a conflict miss?**

A capacity miss occurs because the working set exceeds cache capacity.

A conflict miss occurs because multiple addresses compete for the same cache set, even when total cache capacity may be sufficient.

**Q: Why are linked lists often slower than arrays?**

Linked-list nodes may be scattered throughout memory, causing poor spatial locality and dependent pointer-chasing accesses.

Arrays store elements contiguously, improving cache utilization and hardware prefetching opportunities.

**Q: How would you investigate cache misses in a C++ application?**

Use hardware performance counters through tools such as Linux `perf`.

For example:

```bash
perf stat -e cycles,instructions,cache-references,cache-misses ./app
```

Then investigate data layout, working-set size, allocation patterns, and access locality.

Generic cache counters should be interpreted carefully because their precise meanings depend on the processor.

---

## 4.6 Cache Coherence

### Definition

Cache coherence ensures that multiple CPU cores maintain a consistent view of shared memory locations when they cache the same data.

Without coherence, different cores could continue using stale cached values after another core modifies the underlying memory.

### How It Works

Consider two CPU cores accessing the same variable:

```text
Core 0                    Core 1
+-----------+             +-----------+
| Cache     |             | Cache     |
| X = 10    |             | X = 10    |
+-----------+             +-----------+
       \                     /
        \                   /
         +-----------------+
         | Shared Memory   |
         | X = 10          |
         +-----------------+
```

When Core 0 wants to modify X, the coherence protocol coordinates ownership of the corresponding cache line.

Other cached copies may be invalidated.

A common conceptual protocol is MESI:

| State | Meaning |
|---|---|
| Modified (M) | Line is dirty and exclusively owned by this cache |
| Exclusive (E) | Line is clean and exclusively held |
| Shared (S) | Line is clean and may exist in multiple caches |
| Invalid (I) | Cached copy is not valid |

Actual processors may use more sophisticated coherence protocols.

### Key Concepts

| Concept | Explanation |
|---|---|
| MESI | Common conceptual cache-coherence protocol |
| Cache Invalidation | Marks another cached copy as invalid |
| Cache-Line Ownership | Permission to modify a cache line |
| Coherence Traffic | Communication required to maintain coherent caches |
| Write Invalidation | Mechanism invalidating other copies before modification |

### Interview Questions

**Q: What happens when two cores repeatedly write to the same cache line?**

The cache line's write ownership repeatedly moves between cores.

This generates coherence traffic and can significantly reduce performance.

**Q: Does cache coherence eliminate the need for atomics or mutexes?**

No.

Cache coherence maintains coherent memory contents, but it does not automatically provide application-level synchronization, atomicity for arbitrary operations, or correct memory ordering.

C++ programs must still avoid data races.

**Q: Why can sharing a frequently modified atomic variable be expensive?**

Although atomic operations ensure correctness, writes can require exclusive ownership of the cache line.

Multiple cores repeatedly modifying the same atomic variable may create substantial coherence contention.

---

## 4.7 False Sharing

### Definition

False sharing occurs when multiple threads modify independent variables that happen to occupy the same cache line.

Although the threads do not logically share data, the CPU coherence protocol operates at cache-line granularity.

### How It Works

Consider:

```cpp
struct Counters {
  std::atomic<int> a;
  std::atomic<int> b;
};
```

Thread A modifies `a`, while Thread B modifies `b`.

Both variables may occupy the same cache line:

```text
64-byte cache line
+----------------+----------------+-------------+
| a              | b              | Unused      |
+----------------+----------------+-------------+
    Thread A         Thread B
```

Each thread's writes can invalidate the other core's cached copy, creating unnecessary coherence traffic.

One possible solution:

```cpp
struct Counters {
  alignas(64) std::atomic<int> a;
  alignas(64) std::atomic<int> b;
};
```

Assuming a 64-byte destructive-interference size, the two variables are placed on separate cache lines.

For portable C++, `std::hardware_destructive_interference_size` may also be considered, subject to compiler and platform support.

### Key Concepts

| Concept | Explanation |
|---|---|
| False Sharing | Independent variables contend because they share a cache line |
| True Sharing | Threads actually access the same shared data |
| Cache-Line Padding | Separating variables to avoid false sharing |
| Destructive Interference | Performance degradation caused by conflicting cache-line accesses |

### Interview Questions

**Q: Why can two independent atomic variables cause contention?**

If they occupy the same cache line, writes to either variable can trigger coherence invalidations affecting the other core.

The hardware tracks coherence at cache-line granularity rather than individual-variable granularity.

**Q: How do you fix false sharing?**

Place independently modified variables on separate cache lines using alignment, padding, or a different data layout.

**Q: Should every variable be cache-line aligned?**

No.

Excessive padding increases memory consumption and may reduce cache locality.

Alignment should be applied selectively to data that is frequently modified by different threads.

---

## 4.8 Hardware Prefetching

### Definition

Hardware prefetching is a CPU optimization that attempts to load data into caches before the program explicitly requests it.

The CPU predicts future memory accesses based on observed access patterns.

### How It Works

Consider sequential array access:

```cpp
for (int i = 0; i < n; ++i) {
  sum += array[i];
}
```

The CPU may detect the sequential access pattern and fetch upcoming cache lines in advance.

```text
Current access
      |
      v
[Line 0][Line 1][Line 2][Line 3][Line 4]
   CPU      Prefetched lines ---->
```

Common patterns hardware prefetchers may recognize include:

- Sequential access.
- Regular strides.
- Certain recurring access patterns.

Unpredictable pointer chasing is generally harder to prefetch effectively.

### Key Concepts

| Concept | Explanation |
|---|---|
| Hardware Prefetcher | CPU mechanism predicting future memory accesses |
| Spatial Locality | Accessing nearby addresses |
| Stride Access | Accessing memory at regular intervals |
| Prefetch Distance | How far ahead data is fetched |
| Cache Pollution | Prefetched data displaces more useful cached data |
| Software Prefetch | Explicit prefetch hint issued by application code |

### Interview Questions

**Q: Why is sequential array traversal generally faster than random access?**

Sequential traversal benefits from spatial locality and hardware prefetching.

Random access is harder to predict and may produce more cache misses.

**Q: Why is pointer chasing often expensive?**

The address of the next node depends on loading the current node.

This creates dependent memory accesses that are difficult to overlap or prefetch.

**Q: Should you manually insert prefetch instructions into C++ code?**

Not by default.

Modern CPUs already implement sophisticated hardware prefetching.

Software prefetching should only be introduced when profiling demonstrates a benefit, since unnecessary prefetches can increase memory traffic and cache pollution.

---

## 4.9 Memory Alignment

### Definition

Memory alignment means placing data at addresses that satisfy specific boundary requirements.

For example, a 64-byte-aligned object begins at an address divisible by 64.

### How It Works

Consider a 64-byte cache line:

```text
Cache Line 0               Cache Line 1
0x1000 -------- 0x103F     0x1040 -------- 0x107F
```

An eight-byte object beginning at `0x1008` fits entirely within the first cache line.

An eight-byte object beginning at `0x103C` crosses the cache-line boundary:

```text
Cache Line 0               Cache Line 1
+----------------------+   +----------------------+
|                XXXX  |   | XXXX                 |
+----------------------+   +----------------------+
                  One 8-byte object
```

Crossing a cache-line boundary may require accessing two cache lines.

C++ provides `alignas` to specify alignment:

```cpp
struct alignas(64) CacheAligned {
  std::atomic<int> value;
};
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Natural Alignment | Alignment normally required by a data type |
| `alignas` | C++ alignment specifier |
| Padding | Extra bytes inserted to satisfy layout/alignment requirements |
| Cache-Line Crossing | Object spans multiple cache lines |
| Split Access | Memory access crossing relevant hardware boundaries |
| Unaligned Access | Access not aligned to the operand's natural boundary |

### Interview Questions

**Q: Is an unaligned memory access always slower?**

No.

On modern x86 processors, many unaligned accesses are efficient when they remain within one cache line.

Crossing cache-line or page boundaries can introduce additional overhead.

**Q: Why can alignment matter for atomic operations?**

Atomic operations have architecture-specific alignment requirements and performance characteristics.

Misaligned atomic operations may be unsupported, lose lock-free guarantees, or incur substantial overhead.

**Q: What is the difference between alignment and padding?**

Alignment specifies the address boundary an object must satisfy.

Padding refers to additional bytes inserted into a layout to meet alignment or spacing requirements.

---

## 4.10 NUMA Architecture

### Definition

Non-Uniform Memory Access (NUMA) is a system architecture in which memory-access latency depends on the physical location of the memory relative to the CPU accessing it.

Each NUMA node typically contains a group of CPU cores and associated local memory.

### How It Works

Consider a dual-socket server:

```text
+-----------------------+       +-----------------------+
| NUMA Node 0           |       | NUMA Node 1           |
|                       |       |                       |
| CPU Cores             |       | CPU Cores             |
|     |                 |       |     |                 |
| Local Memory          |       | Local Memory          |
+-----------------------+       +-----------------------+
             \                         /
              +----- Interconnect ----+
```

A CPU can access both local and remote memory, but remote memory access generally incurs additional latency and may have different bandwidth characteristics.

Linux commonly uses first-touch placement for anonymous memory under its default NUMA policy.

The node of the CPU that first physically populates a page often determines where that page is allocated.

### Key Concepts

| Concept | Explanation |
|---|---|
| NUMA Node | Group of CPUs and associated memory resources |
| Local Memory | Memory associated with the accessing CPU's node |
| Remote Memory | Memory associated with another NUMA node |
| First-Touch Policy | Allocating physical pages based on the CPU that first populates them |
| NUMA Affinity | Controlling CPU and memory placement |
| `numactl` | Linux utility for configuring NUMA policies |

### Interview Questions

**Q: Why is remote NUMA memory access generally slower?**

Remote memory access requires communication across the processor interconnect rather than directly accessing the local node's memory controller.

This introduces additional latency and can increase interconnect traffic.

**Q: Why does first-touch allocation matter?**

Allocating memory on one NUMA node and subsequently accessing it from another can introduce unnecessary remote-memory traffic.

Initializing memory from the CPU that will primarily use it helps improve locality.

**Q: How would you optimize NUMA placement for a trading application?**

1. Identify the machine's NUMA topology.
2. Pin critical threads to appropriate CPU cores.
3. Allocate and initialize their working memory on the corresponding NUMA node.
4. Keep frequently communicating threads and their shared data close together where practical.
5. Benchmark local versus remote memory access.

Useful Linux commands:

```bash
numactl --hardware
numactl --cpunodebind=0 --membind=0 ./trading_app
```

**Q: Can NUMA optimization conflict with CPU affinity?**

Yes.

Pinning a thread to one NUMA node while its working memory resides on another may increase remote-memory accesses.

CPU placement and memory placement should be considered together.

---

## Section 4 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] What are the important x86-64 CPU registers, and what do they do?
- [ ] What is register spilling?
- [ ] Explain the CPU memory hierarchy from registers to RAM.
- [ ] What is a cache line, and why is it important?
- [ ] How does the CPU determine whether an address is present in the cache?
- [ ] Explain direct-mapped, fully associative, and set-associative caches.
- [ ] What are compulsory, capacity, and conflict misses?
- [ ] Why are contiguous arrays generally more cache-friendly than linked lists?
- [ ] Explain cache coherence and the MESI protocol.
- [ ] What is false sharing, and how can it be avoided?
- [ ] How does hardware prefetching work?
- [ ] Why is unpredictable pointer chasing expensive?
- [ ] What is memory alignment, and when does misalignment affect performance?
- [ ] Why can two independent atomic variables still contend with each other?
- [ ] Explain local vs remote NUMA memory access.
- [ ] How does first-touch memory allocation work?
- [ ] How would you optimize CPU affinity, memory layout, and NUMA placement for a low-latency order book?
