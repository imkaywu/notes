# 7. Networking & Linux Network Stack

## 7.1 TCP/IP Fundamentals

### Definition

TCP/IP is a family of networking protocols that enables communication between applications across interconnected networks.

It organizes communication into layers, each responsible for a different part of data transmission.

### How It Works

A simplified TCP/IP model:

```text
+----------------------------------+
| Application                      |
| Trading Protocol / HTTP / DNS    |
+----------------------------------+
| Transport                        |
| TCP / UDP                        |
+----------------------------------+
| Internet                         |
| IP                               |
+----------------------------------+
| Link                             |
| Ethernet                         |
+----------------------------------+
| Physical                         |
| NIC / Cable / Fiber              |
+----------------------------------+
```

When an application sends data, each layer adds the information needed for transmission.

This is called **encapsulation**.

```text
Application Data
       |
       v
[TCP Header | Application Data]
       |
       v
[IP Header | TCP Header | Application Data]
       |
       v
[Ethernet Header | IP Packet | Ethernet Trailer]
```

The receiver processes the headers in reverse order and delivers the application payload to the appropriate socket.

### Key Concepts

| Concept | Explanation |
|---|---|
| Ethernet | Common link-layer networking technology |
| MAC Address | Link-layer address |
| IP Address | Identifies a network interface at the Internet layer |
| Port | Identifies a transport-layer communication endpoint |
| TCP | Reliable, ordered byte-stream transport |
| UDP | Datagram-oriented transport |
| MTU | Maximum packet size supported by a link-layer transmission unit |

### Interview Questions

**Q: What is the difference between an IP address and a port?**

An IP address identifies a network endpoint, while a port identifies a transport-layer endpoint associated with an application or service.

**Q: What is encapsulation?**

Each networking layer adds its own protocol information around the data received from the layer above.

For example, TCP adds a TCP header, IP adds an IP header, and Ethernet adds a frame header.

**Q: Why does networking introduce latency even on a fast local network?**

Latency includes application processing, protocol processing, queueing, NIC operations, transmission, propagation, and receiving-side processing.

A fast physical link does not eliminate software and queueing overhead.

---

## 7.2 TCP vs UDP

### Definition

TCP and UDP are transport-layer protocols with different delivery guarantees and communication semantics.

**TCP** provides a reliable, ordered byte stream.

**UDP** provides independent datagrams without built-in reliability or ordering guarantees.

### How It Works

TCP establishes a connection using a three-way handshake:

```text
Client                         Server
  |                              |
  | -------- SYN --------------> |
  | <----- SYN + ACK ----------- |
  | -------- ACK --------------> |
  |                              |
  |      Connection Ready        |
```

TCP maintains sequence numbers, acknowledgments, retransmission mechanisms, flow control, and congestion control.

UDP does not establish a transport-layer connection before sending datagrams.

### Key Concepts

| Property | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented | Connectionless |
| Delivery | Reliable while connection remains viable | Best effort |
| Ordering | Ordered byte stream | No ordering guarantee |
| Message Boundaries | Not preserved | Preserved per datagram |
| Retransmission | Built in | Application responsibility |
| Flow Control | Built in | Not built in |
| Congestion Control | Built in | Application/protocol responsibility |

### Interview Questions

**Q: Why is TCP considered reliable?**

TCP uses sequence numbers, acknowledgments, retransmissions, and other mechanisms to provide reliable, ordered byte-stream delivery while the connection remains operational.

A successful `send()` does not, by itself, guarantee that the remote application has processed the data.

**Q: Does TCP preserve application message boundaries?**

No.

If an application sends two messages of 100 bytes each, the receiver might read 50, 150, or 200 bytes depending on timing and buffer availability.

Applications must implement message framing.

**Q: Why is UDP commonly used for market-data distribution?**

UDP supports datagram delivery and IP multicast, allowing exchanges to distribute market data efficiently to many subscribers.

Trading protocols typically add sequence numbers and recovery mechanisms to detect and handle packet loss.

**Q: Is UDP always faster than TCP?**

No. UDP has simpler transport semantics, but actual latency depends on the complete network and application design.

TCP may be entirely appropriate for order-entry protocols requiring reliable communication.

---

## 7.3 Sockets

### Definition

A socket is an operating-system abstraction representing a communication endpoint.

Applications use sockets to send and receive data through networking protocols.

### How It Works

Typical TCP server:

```text
socket()
   |
   v
bind()
   |
   v
listen()
   |
   v
accept()
   |
   v
send() / recv()
   |
   v
close()
```

Typical TCP client:

```text
socket()
   |
   v
connect()
   |
   v
send() / recv()
   |
   v
close()
```

A socket is represented by a file descriptor on Linux.

Important socket types:

| Type | Description |
|---|---|
| `SOCK_STREAM` | Byte-stream communication, commonly TCP |
| `SOCK_DGRAM` | Datagram communication, commonly UDP |
| `SOCK_RAW` | Access to lower-level protocol data, subject to privileges |

### Key Concepts

| Concept | Explanation |
|---|---|
| File Descriptor | Process-local handle identifying an open resource |
| `bind()` | Associates a socket with a local address |
| `listen()` | Marks a stream socket as accepting connections |
| `accept()` | Retrieves a completed incoming connection |
| `connect()` | Initiates a connection or sets a peer association |
| `send()` / `recv()` | Socket data-transfer operations |
| Socket Options | Configuration through `setsockopt()` |

### Interview Questions

**Q: What is the difference between a listening socket and a connected socket?**

A listening socket accepts incoming connection requests.

`accept()` returns a new socket representing an established connection, leaving the original listening socket available for additional connections.

**Q: What happens when `send()` returns successfully?**

It generally means the kernel accepted the specified amount of data for transmission.

It does not guarantee that the remote application has received or processed it.

**Q: Why should production network code handle partial sends and receives?**

Stream socket operations may transfer fewer bytes than requested.

Applications must track progress and continue until the complete logical message has been transferred or an error occurs.

---

## 7.4 Blocking vs Non-Blocking I/O

### Definition

Blocking I/O allows a system call to wait until an operation can make progress.

Non-blocking I/O returns immediately when the requested operation cannot currently proceed.

### How It Works

Consider receiving data from a socket.

**Blocking mode:**

```text
recv()
   |
   v
No data available
   |
   v
Thread sleeps
   |
   v
Data arrives
   |
   v
Thread wakes and returns
```

**Non-blocking mode:**

```text
recv()
   |
   v
No data available
   |
   v
Return -1
errno = EAGAIN / EWOULDBLOCK
```

Non-blocking mode can be enabled using `fcntl()`:

```cpp
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

### Key Concepts

| Concept | Explanation |
|---|---|
| Blocking I/O | Operation may suspend the calling thread |
| Non-Blocking I/O | Operation returns if progress is not currently possible |
| `O_NONBLOCK` | File status flag enabling non-blocking behavior |
| `EAGAIN` | Operation cannot proceed immediately |
| `EWOULDBLOCK` | Equivalent to `EAGAIN` on Linux |
| Busy Polling | Repeatedly checking for data without sleeping |

### Interview Questions

**Q: Does non-blocking I/O mean the operation happens asynchronously?**

No.

Non-blocking means the call does not wait for readiness.

Asynchronous I/O refers to a model where an operation can be submitted and completed independently, with completion reported later.

**Q: Why is non-blocking I/O useful for an event loop?**

One thread can manage many connections without becoming stuck waiting on a single socket.

**Q: Is repeatedly calling non-blocking `recv()` efficient?**

It depends.

Continuous polling consumes CPU but can avoid wake-up latency. For general-purpose servers, readiness notification through `epoll` is usually more CPU-efficient.

---

## 7.5 I/O Multiplexing

### Definition

I/O multiplexing allows a thread to monitor multiple file descriptors and determine which ones are ready for I/O.

Linux provides:

- `select()`
- `poll()`
- `epoll`

### How It Works

Instead of dedicating one thread to each socket, an event loop monitors many sockets:

```text
Socket A ----+
Socket B ----+
Socket C ----+----> epoll_wait()
Socket D ----+           |
                         v
                  Ready descriptors
                         |
                         v
                  Process available I/O
```

A typical `epoll` workflow:

1. Create an epoll instance using `epoll_create1()`.
2. Register descriptors using `epoll_ctl()`.
3. Wait for readiness using `epoll_wait()`.
4. Process ready descriptors.
5. Repeat.

Two important notification modes:

| Mode | Behavior |
|---|---|
| Level-Triggered | Reports readiness while the readiness condition remains true |
| Edge-Triggered | Reports changes in readiness; applications must handle the available I/O carefully |

### Key Concepts

| Concept | Explanation |
|---|---|
| `select()` | Monitors sets of descriptors |
| `poll()` | Monitors an array of descriptor/event entries |
| `epoll` | Linux readiness-notification facility |
| Level-Triggered | Readiness remains reportable while the condition holds |
| Edge-Triggered | Notification occurs when readiness changes |
| `EPOLLIN` | Descriptor is ready for reading |
| `EPOLLOUT` | Descriptor is ready for writing |

### Interview Questions

**Q: Why is `epoll` generally more scalable than `select()` for many connections?**

`epoll` maintains a kernel-side interest set and returns ready events without requiring the application to resubmit and scan the entire descriptor set on every wait.

**Q: What is the difference between level-triggered and edge-triggered `epoll`?**

Level-triggered mode continues reporting a descriptor while it remains ready.

Edge-triggered mode reports readiness transitions, requiring the application to consume available I/O correctly.

**Q: Why should edge-triggered sockets generally be non-blocking?**

The application should repeatedly read or write until the operation returns `EAGAIN`.

A blocking socket could otherwise stall the event loop while draining I/O.

**Q: Is `epoll` always the lowest-latency option?**

No.

It is efficient for managing many connections, but waiting for notifications introduces wake-up and scheduling overhead.

Dedicated busy-polling approaches may provide lower latency when CPU resources are available.

---

## 7.6 Linux Network Receive Path

### Definition

The Linux network receive path describes how incoming packets travel from the Network Interface Card (NIC), through the kernel networking stack, and eventually to an application's socket.

### How It Works

A simplified conventional receive path:

```text
Network Packet
      |
      v
NIC receives packet
      |
      v
DMA into receive buffer
      |
      v
NIC raises interrupt
      |
      v
Driver / NAPI processing
      |
      v
Kernel network stack
      |
      v
IP / TCP / UDP processing
      |
      v
Socket receive queue
      |
      v
recv() / recvfrom()
      |
      v
Application buffer
```

Important steps:

1. The NIC receives a frame.
2. DMA transfers packet data into host memory.
3. The driver processes receive descriptors.
4. NAPI coordinates packet processing.
5. The kernel processes network and transport protocols.
6. Data becomes available through the appropriate socket.
7. The application receives the data.

The exact path depends on hardware, drivers, protocol, offloads, and configuration.

### Key Concepts

| Concept | Explanation |
|---|---|
| NIC | Network Interface Card |
| DMA | Direct Memory Access |
| RX Ring | Receive descriptor ring used by the NIC and driver |
| NAPI | Linux interrupt/polling-based packet processing |
| `sk_buff` | Linux kernel structure representing network packets |
| Socket Receive Queue | Kernel-managed queue holding received socket data |
| RSS | Receive Side Scaling; distributes packet processing across receive queues |

### Interview Questions

**Q: Does the CPU copy every packet from the NIC into RAM?**

No.

NICs typically use DMA to transfer packet data directly into host memory.

The CPU still performs protocol processing and may participate in subsequent copying or buffer management.

**Q: What is an RX ring?**

An RX ring is a circular structure containing descriptors that coordinate packet-buffer ownership and completion between the NIC and driver.

**Q: Where can latency occur in the conventional Linux receive path?**

Potential sources include:

- Interrupt delivery.
- NAPI scheduling.
- Packet processing.
- Socket buffering.
- Data copying.
- Application wake-up.
- CPU contention and scheduling.

**Q: Why do low-latency applications care about NIC queue placement?**

Matching NIC receive queues, IRQ affinity, and application CPU placement can reduce cross-core traffic and improve cache locality.

---

## 7.7 Interrupt Coalescing & NAPI

### Definition

Interrupt coalescing reduces interrupt frequency by allowing a NIC to group packet events before notifying the CPU.

NAPI is a Linux networking mechanism that combines interrupt-driven notification with polling-based packet processing.

### How It Works

Without coalescing, a NIC might generate an interrupt for every received packet.

```text
Packet 1 -> Interrupt
Packet 2 -> Interrupt
Packet 3 -> Interrupt
Packet 4 -> Interrupt
```

With coalescing:

```text
Packet 1 --+
Packet 2 --+
Packet 3 --+--> One interrupt
Packet 4 --+
```

NAPI commonly operates as follows:

1. A packet arrives and triggers an interrupt.
2. The driver schedules NAPI processing.
3. Receive interrupts for the queue are typically suppressed.
4. The kernel polls and processes packets up to a budget.
5. When processing completes and the queue is drained, interrupt notification is re-enabled.

### Key Concepts

| Concept | Explanation |
|---|---|
| Interrupt Coalescing | Groups packet notifications |
| NAPI | Combines interrupt notification and polling |
| Poll Budget | Limits work performed during a polling cycle |
| Softirq | Linux deferred kernel processing mechanism |
| IRQ Affinity | Controls interrupt CPU placement |
| Adaptive Coalescing | Adjusts coalescing behavior based on traffic patterns |

### Interview Questions

**Q: Why does interrupt coalescing improve throughput?**

It reduces the number of interrupts and associated CPU overhead, allowing packet processing to be amortized across multiple packets.

**Q: Why can interrupt coalescing hurt latency?**

The NIC may intentionally delay notification while waiting for additional packets or a coalescing timer.

This can increase the latency of individual packets.

**Q: Why does Linux use NAPI?**

Pure interrupt-driven processing can become expensive under heavy packet rates.

NAPI switches to polling-based processing during bursts, reducing interrupt overhead and improving throughput.

**Q: Would you disable interrupt coalescing for a trading system?**

Possibly, if minimizing per-packet latency is more important than CPU efficiency.

However, disabling coalescing can increase interrupt load and degrade performance under bursts. The correct setting should be benchmarked under realistic traffic.

---

## 7.8 Socket Buffers

### Definition

Socket buffers are kernel-managed memory used to hold incoming and outgoing network data.

They decouple application execution from packet arrival and transmission timing.

### How It Works

A simplified TCP socket:

```text
Application
    |
    | send()
    v
+------------------+
| Socket Send      |
| Buffer           |
+------------------+
    |
    v
Kernel / NIC / Network


Network / NIC / Kernel
    |
    v
+------------------+
| Socket Receive   |
| Buffer           |
+------------------+
    |
    | recv()
    v
Application
```

Linux exposes socket-buffer configuration through options such as:

- `SO_RCVBUF`
- `SO_SNDBUF`

Actual buffer behavior also depends on kernel limits, protocol-specific accounting, and automatic tuning.

### Key Concepts

| Concept | Explanation |
|---|---|
| Receive Buffer | Holds incoming socket data |
| Send Buffer | Holds outgoing data awaiting transmission or acknowledgment, as applicable |
| `SO_RCVBUF` | Configures receive-buffer capacity |
| `SO_SNDBUF` | Configures send-buffer capacity |
| Buffer Overflow | Incoming data exceeds available buffering capacity |
| Backpressure | Mechanism that slows or restricts producers when consumers cannot keep up |

### Interview Questions

**Q: What happens if a UDP socket's receive buffer fills?**

Additional incoming datagrams may be dropped.

UDP does not provide automatic retransmission to recover those losses.

**Q: What happens if a TCP application's receive buffer fills?**

TCP flow control can reduce the advertised receive window, limiting how much additional data the sender may transmit.

**Q: Does increasing socket-buffer size always improve latency?**

No.

Larger buffers may reduce drops during bursts but can also allow more data to queue, potentially increasing latency.

**Q: Why should market-data applications monitor packet loss and receive-buffer pressure?**

Dropped market-data packets can create sequence gaps and require recovery.

Insufficient buffering, delayed application processing, or CPU contention can contribute to packet loss.

---

## 7.9 Multicast

### Definition

IP multicast allows one sender to distribute packets to multiple receivers that subscribe to the same multicast group.

It is widely used for distributing market data to multiple participants.

### How It Works

Instead of sending a separate copy to every receiver, the sender transmits packets to a multicast group address.

```text
               Exchange
                  |
                  v
           Multicast Group
                  |
          +-------+-------+
          |       |       |
          v       v       v
       Client A Client B Client C
```

Receivers join a multicast group using socket options such as `IP_ADD_MEMBERSHIP` for IPv4.

Network infrastructure determines how multicast traffic is distributed to interested receivers.

### Key Concepts

| Concept | Explanation |
|---|---|
| Multicast | One-to-many packet distribution |
| Multicast Group | Logical group identified by a multicast IP address |
| IGMP | Internet Group Management Protocol for IPv4 membership management |
| UDP Multicast | Common transport combination for market-data distribution |
| Sequence Number | Application-level identifier used to detect missing messages |
| Gap Recovery | Mechanism for recovering missed market-data messages |

### Interview Questions

**Q: Why do exchanges commonly use UDP multicast for market data?**

It allows efficient one-to-many distribution without requiring a separate TCP connection and transmission stream for every subscriber.

**Q: Does multicast guarantee that every subscriber receives every packet?**

No.

UDP multicast does not provide built-in reliable delivery.

Applications must detect missing packets and implement appropriate recovery procedures.

**Q: How does a trading system detect missing market-data messages?**

Market-data protocols commonly include sequence numbers.

For example:

```text
Received: 1001, 1002, 1003, 1005
                         ^
                    Missing 1004
```

The feed handler detects the gap and follows the exchange-specific recovery protocol.

**Q: What is the difference between multicast and broadcast?**

Broadcast targets all devices within a broadcast domain.

Multicast targets receivers interested in a particular multicast group, subject to network configuration.

---

## 7.10 Kernel Bypass Networking

### Definition

Kernel bypass networking refers to techniques that allow applications to exchange network packets while avoiding substantial portions of the conventional kernel networking path.

The goal is to reduce packet-processing overhead, data copying, and scheduling-related latency.

### How It Works

Conventional networking:

```text
NIC
 |
 v
Kernel Driver
 |
 v
Kernel Network Stack
 |
 v
Socket Buffer
 |
 v
Application
```

A kernel-bypass approach:

```text
NIC
 |
 v
DMA / Userspace-Accessible Buffers
 |
 v
Userspace Packet Processing
 |
 v
Application
```

Common technologies include:

| Technology | Description |
|---|---|
| DPDK | Userspace packet-processing framework |
| AF_XDP | Linux socket family providing high-performance access to packets through XDP |
| RDMA | Remote Direct Memory Access; allows supported network operations to access registered memory with reduced CPU involvement |
| Vendor-Specific Stacks | Optimized networking implementations, sometimes accelerating or bypassing parts of the kernel stack |

These technologies differ significantly in their programming models and guarantees. They are not interchangeable.

### Key Concepts

| Concept | Explanation |
|---|---|
| Kernel Bypass | Avoids substantial portions of the conventional kernel networking path |
| Userspace Driver | Device-management or packet-processing functionality executed in user space |
| Polling | Application repeatedly checks for available packets |
| DMA | Device transfers data directly to/from memory |
| DPDK | Data Plane Development Kit |
| AF_XDP | High-performance packet I/O through Linux XDP |
| RDMA | Remote Direct Memory Access |

### Interview Questions

**Q: Why can kernel bypass reduce networking latency?**

It can eliminate or reduce:

- Conventional socket-layer processing.
- Repeated kernel/user transitions.
- Certain data copies.
- Interrupt-driven wake-up overhead.
- Scheduler involvement in packet delivery.

**Q: Does kernel bypass eliminate the operating system entirely?**

No.

The OS is still involved in process management, memory allocation, device setup, permissions, and other system operations.

Kernel bypass primarily changes the packet-processing data path.

**Q: What are the disadvantages of kernel bypass?**

- Greater implementation complexity.
- More responsibility for protocol handling.
- Dedicated CPU consumption from polling.
- Hardware and driver dependencies.
- Additional operational and debugging complexity.

**Q: When would you choose conventional sockets instead of kernel bypass?**

When portability, maintainability, standard protocol support, and operational simplicity are more important than extracting the lowest possible packet-processing latency.

Kernel bypass should be justified through actual latency and throughput measurements.

---

## Section 7 — Final Interview Checklist

Before moving on, make sure you can confidently answer the following questions without referring to your notes.

- [ ] Explain the TCP/IP networking layers.
- [ ] What is encapsulation?
- [ ] Compare TCP and UDP.
- [ ] Explain the TCP three-way handshake.
- [ ] Why does TCP not preserve message boundaries?
- [ ] What is a socket, and how does a TCP server establish connections?
- [ ] What is the difference between blocking and non-blocking I/O?
- [ ] Compare `select()`, `poll()`, and `epoll()`.
- [ ] Explain level-triggered vs edge-triggered `epoll`.
- [ ] Why should edge-triggered sockets generally be non-blocking?
- [ ] Walk through the Linux network receive path from NIC to application.
- [ ] What are DMA, RX rings, and RSS?
- [ ] Explain interrupt coalescing and NAPI.
- [ ] Why can interrupt coalescing improve throughput but hurt latency?
- [ ] What happens when a UDP receive buffer fills?
- [ ] Why is UDP multicast commonly used for market-data distribution?
- [ ] How does a feed handler detect and recover from sequence gaps?
- [ ] What is kernel bypass, and why can it reduce latency?
- [ ] Compare conventional sockets, DPDK, AF_XDP, and RDMA at a high level.
- [ ] Design a low-latency market-data receive pipeline, from NIC reception to strategy consumption.
