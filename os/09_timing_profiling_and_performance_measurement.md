# 9. Timing, Profiling & Performance Measurement

## 9.1 Linux Clocks

### Definition

Linux provides several clock sources through `clock_gettime()`, each with different semantics.

Choosing the correct clock is important when measuring execution time, scheduling delays, or real-world timestamps.

### How It Works

Common Linux clocks:

| Clock | Description |
|---|---|
| `CLOCK_REALTIME` | Wall-clock time; can be adjusted |
| `CLOCK_MONOTONIC` | Monotonic time; affected by gradual clock adjustments |
| `CLOCK_MONOTONIC_RAW` | Raw hardware-based monotonic time without NTP frequency adjustments |
| `CLOCK_BOOTTIME` | Monotonic time that includes system suspension |
| `CLOCK_PROCESS_CPUTIME_ID` | CPU time consumed by a process |
| `CLOCK_THREAD_CPUTIME_ID` | CPU time consumed by a thread |

Example:

```cpp
timespec start{};
timespec end{};

clock_gettime(CLOCK_MONOTONIC, &start);

// Operation being measured.

clock_gettime(CLOCK_MONOTONIC, &end);
```

Elapsed time:

```text
elapsed_ns =
    (end.tv_sec - start.tv_sec) * 1,000,000,000
    + (end.tv_nsec - start.tv_nsec)
```

`clock_gettime()` may be implemented through the vDSO for supported clocks, avoiding a full system call.

### Key Concepts

| Concept | Explanation |
|---|---|
| Wall Clock | Represents calendar time |
| Monotonic Clock | Does not jump backward due to wall-clock changes |
| Clock Resolution | Smallest representable or reported clock increment |
| Clock Accuracy | Closeness to the intended time reference |
| vDSO | Kernel-provided userspace interface for certain operations |
| NTP | Network Time Protocol |

### Interview Questions

**Q: Which clock should you use to measure elapsed execution time?**

`CLOCK_MONOTONIC` is a common default.

`CLOCK_MONOTONIC_RAW` is useful when measurements should exclude NTP frequency adjustments.

**Q: Why should you avoid `CLOCK_REALTIME` for latency measurement?**

Wall-clock time can be adjusted, potentially introducing discontinuities into elapsed-time calculations.

**Q: Does clock resolution equal measurement accuracy?**

No.

A clock can expose nanosecond-resolution timestamps without providing nanosecond-level accuracy.

---

## 9.2 Time Stamp Counter (TSC)

### Definition

The Time Stamp Counter (TSC) is an x86 processor counter that can be read to obtain a high-resolution timing value.

Modern processors commonly provide an invariant TSC that advances at a constant rate independent of ordinary CPU frequency scaling.

### How It Works

The `RDTSC` instruction reads the counter.

Conceptually:

```text
Start:
    Read TSC -> 1,000,000

Execute operation

End:
    Read TSC -> 1,000,500

Difference = 500 TSC ticks
```

To convert ticks into time:

\[
\text{Elapsed Time} =
\frac{\text{TSC Difference}}{\text{TSC Frequency}}
\]

For example, assuming a TSC frequency of 2.5 GHz:

\[
500 / 2.5 \times 10^9 = 200\text{ ns}
\]

The TSC frequency must be determined correctly; it is not necessarily the current CPU core frequency.

### Key Concepts

| Concept | Explanation |
|---|---|
| `RDTSC` | Reads the TSC |
| `RDTSCP` | Reads the TSC with additional ordering semantics and auxiliary information |
| Invariant TSC | Advances at a constant rate across normal frequency changes |
| Serialization | Controls instruction execution around timestamp reads |
| TSC Frequency | Rate at which the counter advances |
| CPU Migration | Thread moves between logical CPUs |

### Interview Questions

**Q: Why use TSC instead of `clock_gettime()`?**

TSC reads can have very low overhead, making them attractive for measuring extremely short code paths.

However, they require careful ordering, calibration, and validation of the platform's clock properties.

**Q: Is `RDTSC` a serializing instruction?**

No.

Instructions may execute around it unless suitable ordering mechanisms are used.

A carefully designed benchmark may use `LFENCE`, `RDTSCP`, or other architecture-appropriate serialization techniques.

**Q: Does one TSC tick always equal one CPU cycle?**

No.

On systems with invariant TSC, the counter typically advances at a constant reference rate, which may differ from the processor's current execution frequency.

**Q: Why might CPU affinity matter when measuring with TSC?**

Pinning the benchmark thread reduces scheduling variability and avoids migration between CPUs.

Modern systems often provide synchronized TSCs, but cross-CPU consistency should not be assumed without validation.

---

## 9.3 Clock Synchronization

### Definition

Clock synchronization ensures that clocks on different machines or devices maintain a sufficiently consistent understanding of time.

It is especially important when measuring latency across distributed trading infrastructure.

### How It Works

Consider a message traveling between two machines:

```text
Machine A                         Machine B

Send timestamp: T1
       |
       | Network transmission
       v
                                  Receive timestamp: T2
```

One-way latency is often estimated as:

\[
L = T_2 - T_1
\]

However, this calculation is meaningful only when the two clocks are sufficiently synchronized and timestamp points are understood.

Common synchronization technologies:

| Technology | Description |
|---|---|
| NTP | Network-based clock synchronization |
| PTP | Precision Time Protocol |
| Hardware Timestamping | NIC or hardware-generated packet timestamps |
| PHC | PTP Hardware Clock, commonly associated with a NIC |
| GPS/GNSS | External time reference |

### Key Concepts

| Concept | Explanation |
|---|---|
| Clock Offset | Difference between two clocks |
| Clock Drift | Change in clock offset over time |
| NTP | Network Time Protocol |
| PTP | Precision Time Protocol |
| Hardware Timestamp | Timestamp generated close to the physical packet event |
| One-Way Latency | Time required for data to travel from sender to receiver |

### Interview Questions

**Q: Why is clock synchronization necessary for one-way latency measurement?**

If the sender and receiver clocks have an unknown offset, subtracting their timestamps produces a biased result.

**Q: Why is PTP often preferred over NTP in trading environments?**

PTP can provide higher synchronization precision, particularly with hardware timestamping and suitable network infrastructure.

**Q: Can round-trip latency be measured without synchronizing two machines?**

Yes.

If the start and end timestamps are taken using the same suitable clock, round-trip duration does not require synchronization with the remote machine.

**Q: Why are hardware timestamps useful?**

They reduce uncertainty introduced by software scheduling and networking-stack delays between the physical packet event and timestamp collection.

---

## 9.4 Latency Measurement

### Definition

Latency measurement quantifies the time required for an operation to complete.

The most important requirement is defining exactly where measurement begins and ends.

### How It Works

Consider an order-processing pipeline:

```text
Order Submission
       |
       v
Queue Insertion
       |
       v
Consumer Receives Order
       |
       v
Matching Engine
       |
       v
Processing Complete
```

Different measurement boundaries answer different questions:

| Metric | Measurement |
|---|---|
| Submission Latency | Start of submission → queue insertion complete |
| Queueing Latency | Queue insertion → consumer receives order |
| Processing Latency | Consumer starts processing → processing completes |
| End-to-End Latency | Submission start → processing complete |

Example:

```cpp
auto start = std::chrono::steady_clock::now();

ProcessOrder(order);

auto end = std::chrono::steady_clock::now();

auto latency = std::chrono::duration_cast<
    std::chrono::nanoseconds>(end - start).count();
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Start/End Boundary | Defines what the measurement includes |
| Measurement Overhead | Additional execution cost introduced by instrumentation |
| Warm-Up | Initial execution used to stabilize relevant system state |
| End-to-End Latency | Complete operation duration |
| Service Time | Time spent actively processing a request |
| Queueing Delay | Time waiting before processing begins |

### Interview Questions

**Q: What is the difference between throughput and latency?**

Throughput measures completed operations per unit of time.

Latency measures the time required for an individual operation.

High throughput does not necessarily imply low latency.

**Q: Why can instrumentation distort latency measurements?**

Timestamp collection, logging, memory allocation, and recording samples introduce additional work.

For very short operations, this overhead can become significant relative to the measured operation.

**Q: How do you reduce measurement overhead?**

Preallocate sample storage, avoid logging inside the measured region, use an appropriate clock, and analyze collected samples after the benchmark.

**Q: Why should a matching engine measure both submission and end-to-end latency?**

A producer may submit orders quickly while the consumer falls behind.

Submission throughput alone can conceal queue buildup and increasing end-to-end latency.

---

## 9.5 Latency Distributions & Tail Latency

### Definition

A latency distribution describes how individual operation times vary across a workload.

Tail latency refers to the slowest portion of that distribution.

For low-latency systems, tail behavior is often more important than average performance.

### How It Works

Consider these measurements:

```text
100 ns
105 ns
110 ns
115 ns
120 ns
125 ns
130 ns
140 ns
150 ns
10,000 ns
```

Most operations complete quickly, but one operation experiences a large delay.

The average can conceal important aspects of this behavior.

Common metrics:

| Metric | Meaning |
|---|---|
| Mean | Arithmetic average |
| Median / p50 | 50th percentile |
| p90 | 90th percentile |
| p99 | 99th percentile |
| p99.9 | 99.9th percentile |
| Maximum | Largest observed latency |
| Standard Deviation | Dispersion around the mean |

A p99 latency of 500 ns means approximately 99% of measured operations completed within 500 ns, subject to the percentile calculation method.

### Key Concepts

| Concept | Explanation |
|---|---|
| Percentile | Value below which a specified fraction of observations falls |
| Tail Latency | Latency of unusually slow operations |
| Outlier | Observation substantially different from typical measurements |
| Histogram | Groups observations into latency ranges |
| Coordinated Omission | Measurement bias caused by failing to record delays while the system is stalled |
| Jitter | Variation in timing |

### Interview Questions

**Q: Why is average latency insufficient for an HFT system?**

Rare delays can cause missed trading opportunities even when average performance is excellent.

Percentiles and maximum observed latency reveal behavior that averages may hide.

**Q: What is coordinated omission?**

It occurs when a measurement process stops generating or recording requests during system delays, thereby failing to capture some of the latency experienced under the intended workload.

**Q: Why should benchmark results include sample count and workload conditions?**

Extreme percentiles require sufficient observations to be meaningful.

For example, p99.9 is difficult to interpret from only a few hundred samples.

**Q: How would you investigate a sudden p99 latency regression?**

Correlate slow samples with page faults, context switches, CPU migration, cache misses, lock contention, queue depth, and background system activity.

---

## 9.6 Hardware Performance Counters

### Definition

Hardware performance counters are processor facilities that count selected microarchitectural events during execution.

They help explain why code is slow rather than merely reporting how long it takes.

### How It Works

A CPU may expose counters for events such as:

- CPU cycles.
- Retired instructions.
- Cache misses.
- Branch misses.
- TLB misses.
- Stalled execution cycles.

Linux exposes performance-monitoring facilities through `perf_event_open()` and tools such as `perf`.

Example:

```bash
perf stat -e cycles,instructions,branches,branch-misses ./app
```

A useful derived metric is Instructions Per Cycle:

\[
IPC = \frac{\text{Instructions Retired}}{\text{CPU Cycles}}
\]

The interpretation depends on the CPU architecture and workload.

### Key Concepts

| Concept | Explanation |
|---|---|
| PMU | Performance Monitoring Unit |
| Hardware Counter | Counts selected processor events |
| IPC | Instructions Per Cycle |
| CPI | Cycles Per Instruction |
| Branch Misprediction | Incorrect branch prediction requiring recovery |
| Counter Multiplexing | Sharing limited hardware counters among multiple requested events |

### Interview Questions

**Q: What can hardware counters tell you that wall-clock timing cannot?**

They can reveal potential causes of poor performance, such as cache misses, branch mispredictions, or excessive instruction counts.

**Q: Does a high cache-miss count automatically mean cache misses are the main bottleneck?**

No.

You must consider event definitions, miss costs, memory-level parallelism, and the overall workload.

**Q: What does low IPC indicate?**

It may indicate execution stalls, memory latency, branch mispredictions, instruction dependencies, or other bottlenecks.

Low IPC alone does not identify the cause.

**Q: Are hardware performance counters perfectly accurate?**

Not always.

Some events are sampled or multiplexed, and event semantics vary across CPU models.

Measurements should account for the platform and counter configuration.

---

## 9.7 Linux `perf`

### Definition

`perf` is a Linux performance-analysis tool that uses kernel performance-monitoring infrastructure to collect hardware and software events.

It supports counting, sampling, and profiling.

### How It Works

Three common workflows:

**1. Count events**

```bash
perf stat ./app
```

Reports aggregate statistics such as cycles, instructions, and context switches.

**2. Record samples**

```bash
perf record -g ./app
```

Collects profiling samples with call-stack information where available.

**3. Analyze the recording**

```bash
perf report
```

Displays functions and call paths associated with recorded samples.

Useful event selection:

```bash
perf stat -e cycles,instructions,cache-misses,page-faults,context-switches ./app
```

### Key Concepts

| Concept | Explanation |
|---|---|
| `perf stat` | Counts performance events |
| `perf record` | Records sampled profiling data |
| `perf report` | Analyzes recorded samples |
| Sampling | Periodically records execution state |
| Call Graph | Represents function-call relationships |
| Symbolization | Maps instruction addresses to function names |

### Interview Questions

**Q: What is the difference between `perf stat` and `perf record`?**

`perf stat` reports aggregate event counts.

`perf record` collects samples that can be analyzed to identify where execution time or selected events occur.

**Q: How would you investigate a slow order-processing function?**

Start with a reproducible benchmark, use `perf stat` to examine high-level events, then use `perf record` and `perf report` to locate expensive code paths.

**Q: Why should profiling use optimized builds with debug symbols?**

Optimized builds represent realistic execution behavior.

Debug symbols improve the ability to attribute samples to functions and source locations.

**Q: Can profiling change application performance?**

Yes.

Sampling, stack collection, and instrumentation introduce overhead. Profiling settings should balance visibility against measurement distortion.

---

## 9.8 Tracing

### Definition

Tracing records events and their timing to reconstruct what happened during program or kernel execution.

Unlike aggregate profiling, tracing is particularly useful for investigating individual latency spikes and scheduling behavior.

### How It Works

A trace might contain:

```text
Timestamp       Event
--------------------------------------
100.000000      Thread becomes runnable
100.000020      Scheduler selects thread
100.000025      Thread starts processing
100.000180      Thread blocks on mutex
100.000300      Mutex becomes available
100.000320      Thread resumes
```

Linux tracing mechanisms include:

| Mechanism | Purpose |
|---|---|
| ftrace | Kernel function and event tracing |
| Tracepoints | Predefined kernel instrumentation points |
| kprobes | Dynamic kernel instrumentation |
| uprobes | Dynamic userspace instrumentation |
| eBPF | Programmable instrumentation and event processing |
| `perf sched` | Scheduler-event recording and analysis |

### Key Concepts

| Concept | Explanation |
|---|---|
| Trace Event | Timestamped occurrence |
| Tracepoint | Predefined instrumentation location |
| Scheduling Trace | Records thread scheduling activity |
| Wake-Up Latency | Delay between becoming runnable and executing |
| Off-CPU Time | Time spent not executing on a CPU |
| eBPF | Technology for running verified programs in the kernel |

### Interview Questions

**Q: What is the difference between profiling and tracing?**

Profiling generally identifies where execution resources are consumed.

Tracing reconstructs event sequences and timing, making it useful for diagnosing specific stalls.

**Q: How would you determine whether a latency spike was caused by scheduler delay?**

Record scheduling events and compare the time when the thread became runnable with the time it actually started running.

**Q: Why might tracing be more useful than a flame graph for rare latency spikes?**

A flame graph aggregates sampled execution stacks.

Tracing can preserve the timeline of an individual event, including blocking, wake-ups, and scheduling delays.

**Q: Does tracing introduce overhead?**

Yes.

Detailed tracing can generate substantial data and affect execution. Capture only the events needed for the investigation where possible.

---

## 9.9 Benchmark Methodology

### Definition

Benchmark methodology is the process of designing repeatable experiments that accurately measure system performance.

A reliable benchmark must control workload characteristics, measurement boundaries, and environmental variability.

### How It Works

A practical benchmark workflow:

```text
Define metric
     |
     v
Define workload
     |
     v
Prepare environment
     |
     v
Warm up
     |
     v
Collect measurements
     |
     v
Analyze distributions
     |
     v
Repeat and compare
```

Important considerations:

| Factor | Why It Matters |
|---|---|
| CPU Affinity | Reduces migration variability |
| CPU Frequency | Changes execution timing |
| Warm-Up | Reduces initialization effects |
| Page Faults | Can create latency spikes |
| Allocation | May introduce unpredictable slow paths |
| Background Load | Competes for CPU and memory resources |
| Sample Count | Determines statistical usefulness |
| Compiler Optimization | Changes generated machine code |

### Key Concepts

| Concept | Explanation |
|---|---|
| Microbenchmark | Measures a narrowly defined operation |
| End-to-End Benchmark | Measures a complete processing pipeline |
| Warm-Up | Stabilizes initial execution conditions |
| Reproducibility | Ability to obtain comparable results |
| Measurement Bias | Systematic distortion of collected measurements |
| Baseline | Reference result used for comparison |

### Interview Questions

**Q: Why can a microbenchmark be misleading?**

It may exclude queueing, synchronization, cache contention, allocation, and other costs present in the real application.

**Q: How would you benchmark a mutex-based queue against an SPSC queue?**

Use the same workload, message representation, producer/consumer placement, and measurement boundaries.

Measure submission throughput, end-to-end throughput, latency distributions, and CPU usage.

**Q: Why should benchmarks run multiple times?**

Results vary due to scheduling, thermal conditions, frequency scaling, background activity, and other environmental factors.

Repeated runs help distinguish consistent improvements from noise.

**Q: What is an important benchmarking mistake in producer-consumer systems?**

Measuring only how quickly the producer submits work.

The producer may enqueue items much faster than the consumer processes them, hiding queue buildup and poor end-to-end latency.

---

## 9.10 Flame Graphs

### Definition

A flame graph is a visualization of aggregated sampled call stacks.

It helps identify which execution paths consume the largest share of sampled resources.

### How It Works

Each rectangle represents a function appearing in collected stack samples.

```text
+-------------------------------------------+
|                 main                      |
+----------------------+--------------------+
| ProcessOrders        | NetworkReceive     |
+----------------------+--------------------+
| MatchOrder           | recv               |
+----------+-----------+--------------------+
| FindLevel| UpdateBook|
+----------+-----------+
```

In a typical CPU flame graph:

- Horizontal width represents the proportion of collected samples containing a function.
- Vertical position represents call-stack depth.
- Wider regions indicate more frequently sampled execution paths.

Horizontal position does not represent chronological execution order.

### Key Concepts

| Concept | Explanation |
|---|---|
| Stack Sample | Captured function-call stack |
| Inclusive Cost | Samples attributed to a function and its descendants |
| Exclusive Cost | Samples attributed directly to a function |
| Hot Path | Frequently executed or expensive code path |
| On-CPU Flame Graph | Visualizes sampled execution while running |
| Off-CPU Flame Graph | Visualizes time spent blocked or waiting |

### Interview Questions

**Q: How do you interpret a wide rectangle in a CPU flame graph?**

The corresponding function appears in a large proportion of collected samples.

It may represent a significant optimization opportunity, depending on the workload and sampling event.

**Q: Does the horizontal axis represent time?**

No.

A conventional flame graph aggregates stack samples; horizontal position is not a chronological timeline.

**Q: Why might an on-CPU flame graph miss mutex contention?**

A thread blocked on a mutex may spend substantial time off-CPU.

An on-CPU profile primarily captures time spent executing, so off-CPU analysis or tracing may be needed.

**Q: How would you use flame graphs to optimize a matching engine?**

Identify expensive call paths, investigate their underlying causes, make targeted changes, and compare benchmark results before and after optimization.

---

## Section 9 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Compare `CLOCK_REALTIME`, `CLOCK_MONOTONIC`, and `CLOCK_MONOTONIC_RAW`.
- [ ] Why should wall-clock time generally not be used for elapsed-time measurement?
- [ ] What is the TSC, and how do you convert TSC ticks into time?
- [ ] Why does `RDTSC` require careful instruction ordering?
- [ ] What is the difference between clock resolution and accuracy?
- [ ] Why is clock synchronization necessary for one-way network latency measurement?
- [ ] Compare NTP, PTP, and hardware timestamping.
- [ ] Define submission, queueing, processing, and end-to-end latency.
- [ ] Why is average latency insufficient for a trading system?
- [ ] Explain p50, p99, p99.9, maximum latency, and jitter.
- [ ] What is coordinated omission?
- [ ] What are hardware performance counters?
- [ ] What can IPC, cache misses, and branch misses tell you?
- [ ] Compare `perf stat`, `perf record`, and `perf report`.
- [ ] What is the difference between profiling and tracing?
- [ ] How would you investigate a scheduler-related latency spike?
- [ ] What factors should be controlled during a benchmark?
- [ ] Why should producer-consumer benchmarks measure end-to-end latency?
- [ ] How do you interpret a CPU flame graph?
- [ ] Design a benchmarking methodology for comparing two implementations of a low-latency C++ matching engine.
