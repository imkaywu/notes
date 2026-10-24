# Operating Systems Notes

## Table of Contents

### 1. OS Fundamentals & System Calls (1–10)
- 1.1 What Is an Operating System?
- 1.2 Kernel Mode vs User Mode
- 1.3 System Calls
- 1.4 Interrupts
- 1.5 Exceptions & Traps
- 1.6 Context Switching
- 1.7 Monolithic vs Microkernel
- 1.8 Linux Kernel Architecture
- 1.9 System Call Overhead
- 1.10 OS Design for Low-Latency Systems

### 2. Processes, Threads & CPU Scheduling (11–20)
- 2.1 Processes
- 2.2 Threads
- 2.3 Process vs Thread
- 2.4 Process Control Block (PCB)
- 2.5 Thread Context Switching
- 2.6 CPU Scheduling
- 2.7 Scheduling Algorithms
- 2.8 CPU Affinity
- 2.9 User-Level vs Kernel-Level Threads
- 2.10 Thread Pools

### 3. Virtual Memory & Address Translation (21–30)
- 3.1 Virtual vs Physical Memory
- 3.2 Pages & Page Frames
- 3.3 Page Tables
- 3.4 Multi-Level Page Tables
- 3.5 Translation Lookaside Buffer (TLB)
- 3.6 Page Faults
- 3.7 Demand Paging
- 3.8 Huge Pages
- 3.9 Memory Mapping (`mmap`)
- 3.10 Memory Locking (`mlock` / `mlockall`)

### 4. CPU Architecture, Caches & Memory Hierarchy (31–40)
- 4.1 CPU Registers
- 4.2 Cache Hierarchy
- 4.3 Cache Lines
- 4.4 Cache Associativity
- 4.5 Cache Hits & Misses
- 4.6 Cache Coherence
- 4.7 False Sharing
- 4.8 Hardware Prefetching
- 4.9 Memory Alignment
- 4.10 NUMA Fundamentals

### 5. Concurrency & Synchronization (41–50)
- 5.1 Race Conditions & Data Races
- 5.2 Mutexes
- 5.3 Spinlocks
- 5.4 Condition Variables
- 5.5 Semaphores
- 5.6 Atomics
- 5.7 Memory Ordering
- 5.8 Memory Barriers
- 5.9 Lock-Free Algorithms
- 5.10 Concurrent Queues

### 6. Inter-Process Communication (51–60)
- 6.1 IPC Fundamentals
- 6.2 Pipes
- 6.3 Named Pipes (FIFO)
- 6.4 Shared Memory
- 6.5 Shared Memory Synchronization
- 6.6 Unix Domain Sockets
- 6.7 Signals
- 6.8 Event Notification
- 6.9 Futex
- 6.10 Zero-Copy IPC

### 7. Networking & Linux Network Stack (61–70)
- 7.1 TCP/IP Fundamentals
- 7.2 TCP vs UDP
- 7.3 Sockets
- 7.4 Blocking vs Non-Blocking I/O
- 7.5 I/O Multiplexing
- 7.6 Linux Network Receive Path
- 7.7 Interrupt Coalescing & NAPI
- 7.8 Socket Buffers
- 7.9 Multicast
- 7.10 Kernel Bypass Networking

### 8. Memory Allocation & Resource Management (71–80)
- 8.1 Process Memory Layout
- 8.2 Stack vs Heap
- 8.3 Dynamic Memory Allocation
- 8.4 Allocator Internals
- 8.5 Memory Pools & Object Pools
- 8.6 Custom Allocators
- 8.7 Copy-on-Write (COW)
- 8.8 Memory Fragmentation
- 8.9 Memory Reclamation
- 8.10 Swapping & Memory Pressure

### 9. Timing, Profiling & Performance Measurement (81–90)
- 9.1 Linux Clocks
- 9.2 Time Stamp Counter (TSC)
- 9.3 Clock Synchronization
- 9.4 Latency Measurement
- 9.5 Latency Distributions & Tail Latency
- 9.6 Hardware Performance Counters
- 9.7 Linux `perf`
- 9.8 Tracing
- 9.9 Benchmark Methodology
- 9.10 Flame Graphs

### 10. Advanced Linux Tuning & Deterministic Design (91–100)
- 10.1 CPU Power Management
- 10.2 Simultaneous Multithreading (SMT)
- 10.3 IRQ Affinity
- 10.4 Tickless Kernel & CPU Isolation
- 10.5 Kernel Preemption & Scheduling
- 10.6 Advanced NUMA Tuning
- 10.7 Kernel Bypass & Busy Polling
- 10.8 DMA & NIC Queues
- 10.9 Resource Limits & System Configuration
- 10.10 Deterministic Low-Latency System Design
