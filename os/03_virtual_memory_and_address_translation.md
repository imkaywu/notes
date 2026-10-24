# 3. Virtual Memory & Address Translation

## 3.1 Virtual vs Physical Memory

### Definition

**Virtual memory** is an abstraction that gives each process its own virtual address space, independent of the actual physical RAM layout.

The CPU and operating system cooperate to translate virtual addresses into physical addresses.

### How It Works

Applications operate using virtual addresses. The CPU's Memory Management Unit (MMU) translates them into physical addresses using page tables maintained by the OS.

```text
Process A                    Physical Memory
+----------------+           +----------------+
| Virtual 0x1000 | --------> | Frame 5        |
| Virtual 0x2000 | ----+     | Frame 2        |
+----------------+     |     | Frame 9        |
                       |     +----------------+
Process B              |
+----------------+     +---> | Frame 9        |
| Virtual 0x1000 | --------> | Frame 2        |
+----------------+
```

Two processes can use the same virtual address while mapping it to different physical memory.

Virtual memory provides:

- Process isolation.
- Memory protection.
- Flexible memory allocation.
- Shared-memory mappings.
- Demand paging.

### Key Concepts

| Concept | Explanation |
|---|---|
| Virtual Address | Address used by application code |
| Physical Address | Address identifying a location in physical memory |
| MMU | Hardware responsible for address translation |
| Page Table | Data structure mapping virtual pages to physical frames |
| Address Space | Set of virtual addresses available to a process |
| ASLR | Address Space Layout Randomization |

### Interview Questions

**Q: Why do operating systems use virtual memory?**

Virtual memory provides isolation, protection, and flexible memory management.

Applications can use independent address spaces without needing to know the physical layout of RAM.

**Q: Can two processes use the same virtual address?**

Yes. Each process has its own address-space mappings, so the same virtual address can refer to different physical locations.

**Q: Does every virtual address correspond to physical RAM?**

No.

An address may be unmapped, reserved without physical backing, backed by a file, swapped out, or configured for lazy allocation.

Physical memory is generally assigned when needed.

---

## 3.2 Pages & Page Frames

### Definition

Virtual memory is divided into fixed-size blocks called **pages**.

Physical memory is divided into corresponding blocks called **page frames**.

A page table maps virtual pages to physical frames.

On typical x86-64 Linux systems, the base page size is 4 KiB.

### How It Works

For a 4 KiB page:

```text
Virtual Address (48-bit example)

+---------------------------+--------------+
| Virtual Page Number       | Page Offset  |
|        36 bits            |   12 bits    |
+---------------------------+--------------+
```

The page offset identifies the byte within the page.

Example:

```text
Virtual Address: 0x12345
Page Size:       0x1000 (4096 bytes)

Virtual Page Number = 0x12
Page Offset         = 0x345
```

Suppose virtual page `0x12` maps to physical frame `0xAB`.

The resulting physical address is:

```text
Physical Address = 0xAB000 + 0x345
                 = 0xAB345
```

The page offset remains unchanged during translation.

### Key Concepts

| Concept | Explanation |
|---|---|
| Page | Fixed-size block of virtual memory |
| Page Frame | Corresponding block of physical memory |
| Page Size | Size of each page, commonly 4 KiB |
| Page Offset | Position of a byte within its page |
| Page Alignment | Address aligned to a page-size boundary |
| Internal Fragmentation | Unused space within an allocated memory block |

### Interview Questions

**Q: Why does a 4 KiB page require a 12-bit offset?**

Because:

`4096 = 2^12`

Therefore, 12 bits are required to address every byte within the page.

**Q: Must virtual pages map to contiguous physical frames?**

No. Consecutive virtual pages can map to completely different physical frames.

**Q: Why does page size matter for performance?**

Page size affects page-table overhead, TLB coverage, memory fragmentation, and page-fault behavior.

Larger pages can reduce address-translation overhead but may waste memory or reduce allocation flexibility.

---

## 3.3 Page Tables

### Definition

A page table is an OS-managed data structure used by the MMU to translate virtual addresses into physical addresses.

Each process normally has its own address-space mappings.

### How It Works

Conceptually, a page table contains entries mapping virtual page numbers to physical page frames.

```text
Virtual Page       Page Table       Physical Frame
    0                  5                 5
    1                  9                 9
    2                  3                 3
    3                  7                 7
```

Each Page Table Entry (PTE) contains more than just a physical frame number.

Typical information includes:

- Physical frame address.
- Present/valid bit.
- Read/write permissions.
- User/supervisor permissions.
- Accessed bit.
- Dirty bit.
- Execute-disable bit.

When accessing memory:

1. CPU generates a virtual address.
2. MMU looks up the translation.
3. Access permissions are checked.
4. The physical address is produced.
5. If translation or permission checks fail, a page fault may occur.

### Key Concepts

| Concept | Explanation |
|---|---|
| PTE | Page Table Entry |
| Present Bit | Indicates whether the page has a valid present mapping |
| Dirty Bit | Indicates that the page has been written |
| Accessed Bit | Records page access |
| NX Bit | Prevents instruction execution from a page |
| CR3 | x86 register identifying the active page-table root |

### Interview Questions

**Q: Where are page tables stored?**

Page tables are stored in physical memory and managed by the kernel.

The CPU uses architecture-specific registers, such as `CR3` on x86-64, to locate the active page-table hierarchy.

**Q: Does every memory access require reading the page table from RAM?**

No.

The CPU caches recent address translations in the Translation Lookaside Buffer (TLB).

A successful TLB lookup avoids a page-table walk.

**Q: Can two virtual pages map to the same physical frame?**

Yes.

This is used for shared memory, copy-on-write, and other memory-sharing mechanisms.

---

## 3.4 Multi-Level Page Tables

### Definition

Multi-level page tables organize address translations hierarchically instead of maintaining one enormous flat page table.

This allows the OS to allocate page-table structures only for virtual address regions that are actually mapped.

### How It Works

A common x86-64 configuration uses four levels of page tables with 4 KiB pages and 48-bit canonical virtual addresses.

```text
Virtual Address (48-bit)

+--------+--------+--------+--------+------------+
| PML4   | PDPT   | PD     | PT     | Offset     |
| 9 bits | 9 bits | 9 bits | 9 bits | 12 bits    |
+--------+--------+--------+--------+------------+
```

Each level contains 512 entries because:

`2^9 = 512`

Translation proceeds as follows:

```text
CR3
 |
 v
PML4
 |
 v
PDPT
 |
 v
Page Directory
 |
 v
Page Table
 |
 v
Physical Frame + Offset
```

Modern x86-64 systems can also support five-level paging, extending the virtual address range.

### Key Concepts

| Concept | Explanation |
|---|---|
| PML4 | Page Map Level 4 |
| PDPT | Page Directory Pointer Table |
| PD | Page Directory |
| PT | Page Table |
| Page-Table Walk | Traversal of the hierarchy to resolve an address |
| Sparse Address Space | Address space containing large unmapped regions |

### Interview Questions

**Q: Why use multi-level page tables instead of a flat page table?**

A flat page table covering a large virtual address space would consume enormous amounts of memory.

Multi-level tables allocate lower-level structures only where mappings are needed.

**Q: Does a four-level page table always require four RAM accesses for translation?**

No.

A TLB hit avoids the walk entirely.

Even during a page-table walk, CPUs may cache intermediate page-table entries, reducing the number of actual DRAM accesses.

**Q: What is the main performance disadvantage of multi-level page tables?**

A TLB miss can require several dependent memory accesses to resolve a translation, potentially increasing memory-access latency.

---

## 3.5 Translation Lookaside Buffer (TLB)

### Definition

The Translation Lookaside Buffer (TLB) is a small, fast hardware cache that stores recently used virtual-to-physical address translations.

Its purpose is to avoid repeatedly walking page tables.

### How It Works

When the CPU accesses a virtual address:

```text
             Virtual Address
                    |
                    v
                TLB Lookup
                 /      \
               Hit      Miss
                |         |
                v         v
          Translation   Page-Table
            Found         Walk
                |         |
                |         v
                |     Update TLB
                |         |
                +----+----+
                     |
                     v
             Physical Address
                     |
                     v
              Cache / Memory
```

The CPU checks the TLB using the virtual page number and relevant address-space identification information.

A TLB entry typically stores:

- Virtual page identifier.
- Physical frame identifier.
- Page size.
- Access permissions.
- Address-space identification metadata where supported.

### Key Concepts

| Concept | Explanation |
|---|---|
| TLB Hit | Translation found in the TLB |
| TLB Miss | Translation requires a page-table walk |
| ITLB | Instruction translation cache |
| DTLB | Data translation cache |
| TLB Reach | Total virtual-memory coverage of TLB entries |
| TLB Shootdown | Invalidation of cached translations across relevant CPUs |
| PCID | Process-Context Identifier, allowing translations from different address spaces to coexist in the TLB |

### Interview Questions

**Q: What happens during a TLB miss?**

The CPU performs a page-table walk to resolve the virtual address.

If a valid mapping exists, the translation can be inserted into the TLB and execution continues.

If translation fails because the mapping is absent or access is disallowed, a page fault occurs.

**Q: Is a TLB miss the same as a page fault?**

No.

A TLB miss means the translation is not cached.

A page fault means the CPU cannot complete the requested translation or access under the current page-table state.

A TLB miss can be resolved entirely by hardware without kernel involvement.

**Q: Why are huge pages useful for TLB performance?**

A single huge-page TLB entry covers a larger memory region.

For example, a 2 MiB page covers the same memory as 512 ordinary 4 KiB pages.

This increases effective TLB coverage and can reduce TLB misses.

**Q: What is a TLB shootdown?**

When page mappings change, the kernel may need to invalidate stale TLB entries on other CPUs using the same address space.

This can involve inter-processor interrupts and synchronization, introducing additional overhead.

---

## 3.6 Page Faults

### Definition

A page fault is a CPU exception raised when a memory access cannot be completed using the current page-table mappings and permissions.

Page faults are not necessarily programming errors.

They are also an essential mechanism used by virtual memory.

### How It Works

```text
Application accesses virtual address
                |
                v
        Translation / access fails
                |
                v
        CPU raises page fault
                |
                v
        Kernel fault handler
                |
         Is access valid?
          /           \
        Yes            No
         |              |
         v              v
   Resolve fault     Deliver signal
         |           (e.g. SIGSEGV)
         v
   Update page table
         |
         v
   Retry instruction
```

Two important categories:

| Type | Explanation |
|---|---|
| Minor Page Fault | Resolved without reading page contents from disk |
| Major Page Fault | Requires disk I/O to obtain the requested page |

Examples of minor faults include allocating a zero-filled page or establishing a mapping to an already-resident physical page.

### Key Concepts

| Concept | Explanation |
|---|---|
| Minor Fault | Page fault resolved without disk I/O |
| Major Fault | Page fault requiring disk I/O |
| Demand-Zero Page | Newly allocated zero-initialized page |
| Copy-on-Write | Physical copying deferred until a shared page is modified |
| SIGSEGV | Signal commonly generated for invalid memory access |
| SIGBUS | Signal that can arise from certain invalid memory-mapped accesses |

### Interview Questions

**Q: What happens when an application accesses an unmapped address?**

The CPU raises a page fault.

The kernel checks whether the address belongs to a valid virtual-memory region and whether the access is permitted.

If it cannot resolve the access, the process typically receives a signal such as `SIGSEGV`.

**Q: What is the difference between minor and major page faults?**

Minor faults can be resolved without reading page contents from disk.

Major faults require disk I/O and can therefore introduce substantially larger delays.

**Q: Why should low-latency applications avoid page faults on the hot path?**

Even minor faults involve kernel exception handling and potentially memory allocation, zero-filling, and page-table updates.

These operations can introduce unpredictable latency spikes.

---

## 3.7 Demand Paging

### Definition

Demand paging is a memory-management technique in which physical memory is populated only when the application actually accesses the corresponding virtual pages.

This allows the OS to defer expensive memory operations until necessary.

### How It Works

Consider:

```cpp
char* buffer = new char[1024 * 1024 * 1024];
```

The application requests approximately 1 GiB of memory.

However, depending on the allocator and OS configuration, the kernel may initially reserve virtual address space without immediately assigning physical frames to every page.

When the application first writes to a page:

```text
Virtual memory reserved
          |
          v
First access to a page
          |
          v
Page fault
          |
          v
Kernel allocates physical frame
          |
          v
Frame is zero-initialized
          |
          v
Page table updated
          |
          v
Instruction retried
```

This is commonly called first-touch allocation.

### Key Concepts

| Concept | Explanation |
|---|---|
| Lazy Allocation | Deferring physical allocation until needed |
| First Touch | First access that causes a page to be physically populated |
| Demand-Zero Paging | Providing zero-initialized memory on demand |
| Overcommit | Allowing virtual-memory commitments beyond immediately available physical resources |
| RSS | Resident Set Size; physical memory currently resident for a process |

### Interview Questions

**Q: Does allocating 1 GiB automatically consume 1 GiB of physical RAM?**

Not necessarily.

An allocator may reserve virtual address space while physical pages are assigned lazily.

The exact behavior depends on the allocator, allocation size, operating system, and memory configuration.

**Q: Why can the first access to allocated memory be slower?**

The first access may trigger a page fault, requiring the kernel to allocate a physical frame, initialize it, and update page tables.

**Q: How can a trading application avoid first-touch latency during market hours?**

Preallocate and explicitly touch the required memory during initialization.

For example, write to at least one location in every page that will be needed later.

This moves initial page-population costs outside the critical execution path.

---

## 3.8 Huge Pages

### Definition

Huge pages are memory pages larger than the standard base page size.

On typical x86-64 Linux systems, supported page sizes include:

- 4 KiB — Standard page.
- 2 MiB — Huge page.
- 1 GiB — Gigantic page, where supported.

### How It Works

Consider mapping 1 GiB of memory.

| Page Size | Number of Pages |
|---|---:|
| 4 KiB | 262,144 |
| 2 MiB | 512 |
| 1 GiB | 1 |

Using larger pages reduces the number of page-table entries and increases the amount of memory covered by each TLB entry.

Two common Linux mechanisms:

**Transparent Huge Pages (THP)**

The kernel automatically attempts to use larger pages for eligible memory regions.

**Explicit Huge Pages (HugeTLB)**

Applications explicitly request huge-page-backed memory, typically using preconfigured huge-page pools.

### Key Concepts

| Concept | Explanation |
|---|---|
| Huge Page | Page larger than the base page size |
| THP | Transparent Huge Pages |
| HugeTLB | Explicit Linux huge-page mechanism |
| TLB Reach | Amount of memory covered by cached translations |
| Page Compaction | Rearranging physical memory to obtain contiguous free regions |
| Internal Fragmentation | Memory wasted when an allocation does not fully utilize a large page |

### Interview Questions

**Q: Why can huge pages improve performance?**

Huge pages increase TLB coverage and reduce page-table overhead.

This is particularly useful for applications accessing large memory regions.

**Q: What are the disadvantages of huge pages?**

- Potential internal fragmentation.
- Reduced allocation flexibility.
- Possible allocation or compaction overhead.
- More complicated memory management.
- THP-related latency spikes in certain configurations.

**Q: Would you always enable Transparent Huge Pages for an HFT application?**

No.

Although THP can improve TLB performance, automatic page promotion, compaction, and memory-management activity may introduce latency variability.

For predictable workloads, explicitly allocated huge pages may provide more controlled behavior.

The decision should be validated through benchmarking.

---

## 3.9 Memory Mapping (`mmap`)

### Definition

`mmap()` is a Linux system call that creates a mapping between a region of a process's virtual address space and an underlying memory object.

The mapping may be backed by a file or anonymous memory.

### How It Works

Basic example:

```cpp
void* ptr = mmap(
    nullptr,
    size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
);
```

This requests a writable anonymous memory mapping.

The kernel creates a virtual-memory region, but physical pages may still be allocated lazily when accessed.

Common mapping types:

| Mapping | Explanation |
|---|---|
| Anonymous | Not backed by a regular file |
| File-Backed | Maps file contents into virtual memory |
| `MAP_PRIVATE` | Changes use private copy-on-write semantics |
| `MAP_SHARED` | Changes are visible through other shared mappings |
| `MAP_POPULATE` | Requests prefaulting of eligible pages |

### Key Concepts

| Concept | Explanation |
|---|---|
| `mmap()` | Creates a virtual-memory mapping |
| `munmap()` | Removes a mapping |
| `PROT_READ` | Allows reading |
| `PROT_WRITE` | Allows writing |
| `MAP_PRIVATE` | Private copy-on-write mapping |
| `MAP_SHARED` | Shared mapping |
| `MAP_ANONYMOUS` | Mapping without regular file backing |

### Interview Questions

**Q: What is the difference between `mmap()` and `malloc()`?**

`malloc()` is a user-space memory allocation interface.

`mmap()` is a system call for creating virtual-memory mappings.

Memory allocators may internally use `mmap()` or other mechanisms such as `brk()` to obtain memory from the OS.

**Q: What is the difference between `MAP_PRIVATE` and `MAP_SHARED`?**

`MAP_PRIVATE` uses private copy-on-write semantics, so modifications are not propagated to the underlying file or other private mappings.

`MAP_SHARED` allows modifications to be visible through shared mappings and, for writable file-backed mappings, to be propagated to the underlying file.

**Q: Why is `mmap()` useful for high-performance IPC?**

Multiple processes can map the same shared-memory object into their address spaces.

This allows them to exchange data through shared physical memory without repeatedly copying payloads through traditional IPC interfaces.

Synchronization is still required.

---

## 3.10 Memory Locking (`mlock` / `mlockall`)

### Definition

Memory locking prevents selected virtual-memory pages from being swapped out while they remain locked.

Linux provides:

- `mlock()` — Locks a specified memory region.
- `mlockall()` — Locks memory mappings associated with the calling process according to the selected flags.
- `munlock()` / `munlockall()` — Removes memory locks.

### How It Works

Example:

```cpp
mlockall(MCL_CURRENT | MCL_FUTURE);
```

The flags mean:

| Flag | Explanation |
|---|---|
| `MCL_CURRENT` | Lock currently mapped pages |
| `MCL_FUTURE` | Lock future mappings as they are created |

The kernel tracks locked memory and prevents the corresponding pages from being swapped out.

Memory locking is subject to resource limits such as `RLIMIT_MEMLOCK` and applicable privileges.

### Key Concepts

| Concept | Explanation |
|---|---|
| `mlock()` | Locks a specified memory region |
| `mlockall()` | Locks process memory according to flags |
| `MCL_CURRENT` | Applies to current mappings |
| `MCL_FUTURE` | Applies to future mappings |
| `RLIMIT_MEMLOCK` | Resource limit controlling lockable memory |
| Swapping | Moving memory pages between RAM and swap storage |

### Interview Questions

**Q: Why is memory locking useful in low-latency trading systems?**

It prevents locked pages from being swapped out, avoiding potentially expensive swap-related page faults during critical execution.

**Q: Does `mlockall()` guarantee that an application will never experience a page fault?**

No.

Memory locking prevents swapping of locked pages, but it does not eliminate every possible page fault.

For example, copy-on-write operations, new mappings, or stack growth can still introduce faults depending on configuration and execution behavior.

**Q: What is a common memory preparation strategy for low-latency applications?**

1. Allocate the required memory during startup.
2. Initialize and touch the memory that will be used.
3. Lock critical memory where appropriate.
4. Preallocate thread stacks and other hot-path resources.
5. Avoid unexpected allocations during latency-sensitive execution.

This reduces runtime memory-management overhead and improves latency predictability.

---

## Section 3 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the difference between virtual and physical memory.
- [ ] How does the MMU translate a virtual address into a physical address?
- [ ] What is the difference between a page and a page frame?
- [ ] Given a virtual address and page size, calculate its page number and offset.
- [ ] What information is stored in a Page Table Entry?
- [ ] Why do operating systems use multi-level page tables?
- [ ] Walk through a four-level x86-64 page-table translation.
- [ ] What is a TLB, and how does it work?
- [ ] Explain the difference between a TLB miss and a page fault.
- [ ] What is a TLB shootdown, and why can it be expensive?
- [ ] Explain minor vs major page faults.
- [ ] Why might the first access to allocated memory be slower?
- [ ] How do huge pages improve TLB performance?
- [ ] Compare Transparent Huge Pages and explicit HugeTLB pages.
- [ ] What is the difference between `mmap()` and `malloc()`?
- [ ] Explain `MAP_PRIVATE` vs `MAP_SHARED`.
- [ ] What does `mlockall()` do, and what does it not guarantee?
- [ ] How would you prepare memory before starting a latency-sensitive trading workload?
