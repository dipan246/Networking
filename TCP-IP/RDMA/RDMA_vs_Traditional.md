# ⚡ RDMA vs Traditional Networking — A Side-by-Side Comparison

> Why does RDMA exist when TCP/IP already works? This document walks through the architectural differences, shows where traditional networking wastes time, and explains exactly how RDMA eliminates each bottleneck.

---

## 📖 Table of Contents

1. [The Big Picture](#-the-big-picture)
2. [Data Path Comparison — Step by Step](#-data-path-comparison--step-by-step)
3. [Where Traditional Networking Spends Time](#-where-traditional-networking-spends-time)
4. [How RDMA Eliminates Each Bottleneck](#-how-rdma-eliminates-each-bottleneck)
5. [Protocol Stack Comparison](#-protocol-stack-comparison)
6. [Connection Setup: Sockets vs Queue Pairs](#-connection-setup-sockets-vs-queue-pairs)
7. [The Memory Copy Problem — Visualized](#-the-memory-copy-problem--visualized)
8. [Kernel Bypass — What It Really Means](#-kernel-bypass--what-it-really-means)
9. [When to Use What](#-when-to-use-what)
10. [Common Misconceptions](#-common-misconceptions)
11. [Summary Table](#-summary-table)

---

## 🌐 The Big Picture

```
┌─────────────────────── Traditional TCP/IP ──────────────────────┐
│                                                                  │
│  App ──▶ Kernel ──▶ NIC ──── wire ────▶ NIC ──▶ Kernel ──▶ App │
│       copy  process  copy            copy  process  copy        │
│                                                                  │
│  Every byte passes through the CPU and kernel TWICE on each     │
│  side. The CPU is the bottleneck, not the network.              │
└──────────────────────────────────────────────────────────────────┘

┌────────────────────────── RDMA ─────────────────────────────────┐
│                                                                  │
│  App ──────────────── NIC ──── wire ────▶ NIC ─────────────▶ App │
│         DMA read      smart            smart    DMA write       │
│                                                                  │
│  Data goes directly from app memory to app memory.              │
│  The CPU and kernel are out of the picture.                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Data Path Comparison — Step by Step

### Sending 1 MB of Data: Traditional TCP

```
Step│ What Happens                               │ Who Does It  │ Time Cost
────┼────────────────────────────────────────────┼──────────────┼──────────
 1  │ App calls send(socket, buffer, 1MB)        │ App (CPU)    │ ~0.1 μs
 2  │ Trap into kernel (system call)             │ CPU          │ ~1 μs
 3  │ Copy data: user buffer → kernel buffer     │ CPU (memcpy) │ ~50 μs
 4  │ TCP: segment data, compute checksums       │ CPU          │ ~30 μs
 5  │ IP: add headers, routing lookup            │ CPU          │ ~5 μs
 6  │ Copy data: kernel buffer → NIC TX ring     │ CPU / DMA    │ ~10 μs
 7  │ NIC transmits on wire                      │ NIC          │ ~80 μs
 8  │ Return to user space (context switch)      │ CPU          │ ~1 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL SEND SIDE                            │              │ ~177 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
 9  │ NIC receives, raises interrupt             │ NIC → CPU    │ ~5 μs
 10 │ Interrupt handler runs                     │ CPU          │ ~3 μs
 11 │ Copy data: NIC RX ring → kernel buffer     │ CPU / DMA    │ ~10 μs
 12 │ TCP: verify checksums, reassemble          │ CPU          │ ~30 μs
 13 │ IP: validate headers                       │ CPU          │ ~2 μs
 14 │ App calls recv() — trap into kernel        │ CPU          │ ~1 μs
 15 │ Copy data: kernel buffer → user buffer     │ CPU (memcpy) │ ~50 μs
 16 │ Return to user space                       │ CPU          │ ~1 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL RECEIVE SIDE                         │              │ ~102 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL END-TO-END                           │              │ ~279 μs
    │ CPU time (both sides combined)             │              │ ~198 μs
    │ (71% of total time is CPU overhead!)       │              │
```

### Sending 1 MB of Data: RDMA Write

```
Step│ What Happens                               │ Who Does It  │ Time Cost
────┼────────────────────────────────────────────┼──────────────┼──────────
 1  │ App posts RDMA Write work request to SQ    │ App (CPU)    │ ~0.2 μs
 2  │ Ring doorbell (MMIO write to NIC)          │ CPU          │ ~0.1 μs
 3  │ NIC reads data from app memory via DMA     │ NIC (DMA)    │ ~5 μs
 4  │ NIC builds headers (iWARP/RoCE/IB)        │ NIC          │ ~0.5 μs
 5  │ NIC transmits on wire                      │ NIC          │ ~80 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL SEND SIDE                            │              │ ~86 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
 6  │ Remote NIC receives data                   │ NIC          │ ~0.5 μs
 7  │ NIC validates STag, checks bounds          │ NIC          │ ~0.2 μs
 8  │ NIC writes data to app memory via DMA      │ NIC (DMA)    │ ~5 μs
 9  │ (No interrupt, no CPU involvement)         │ —            │ 0 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL RECEIVE SIDE                         │              │ ~6 μs
────┼────────────────────────────────────────────┼──────────────┼──────────
    │ TOTAL END-TO-END                           │              │ ~92 μs
    │ CPU time (both sides combined)             │              │ ~0.3 μs
    │ (0.3% of total time is CPU overhead!)      │              │
```

### Side by Side

```
                    Traditional TCP        RDMA Write      Improvement
                    ───────────────        ──────────      ───────────
End-to-end:         ~279 μs                ~92 μs          3x faster
CPU time:           ~198 μs                ~0.3 μs         660x less CPU
Memory copies:      4                      0               100% eliminated
System calls:       2 (send + recv)        0               100% eliminated
Interrupts:         1+ (per completion)    0 (batched)     ~95% fewer
Context switches:   2                      0               100% eliminated
```

---

## ⏱️ Where Traditional Networking Spends Time

Let's break down where CPU cycles actually go in a traditional TCP send/receive:

```
CPU Time Breakdown for 1 MB TCP Transfer (both sides):
══════════════════════════════════════════════════════

Memory copies (user↔kernel)     │████████████████████████████████│  51%
TCP/IP processing               │██████████████████████│           34%
  ├─ Checksum computation       │██████████│                       15%
  ├─ Segmentation              │████████│                          12%
  └─ Header build/parse        │████│                               7%
System call overhead            │████│                               6%
Interrupt handling              │████│                               6%
Context switch cost             │██│                                 3%
                                ────────────────────────────────────────
                                0%                              100%

The #1 cost is COPYING DATA that doesn't need to be copied.
The #2 cost is PROCESSING that the NIC could do in hardware.
```

### The Hidden Cost: Cache Pollution

```
Something benchmarks often miss:

When the CPU copies 1 MB of data through memcpy(), it:
  1. Reads 1 MB into L1/L2/L3 cache (evicting useful data!)
  2. Writes 1 MB from cache to destination
  3. The application's working data has been evicted from cache

Before the copy:                    After the copy:
┌────────────────────┐              ┌────────────────────┐
│ L3 Cache (32 MB)   │              │ L3 Cache (32 MB)   │
│                    │              │                    │
│ [App's hot data]   │              │ [Network data]     │
│ [App's hot data]   │              │ [Network data]     │
│ [App's hot data]   │              │ [More net data]    │
│ [More useful stuff]│              │ [App data EVICTED] │
└────────────────────┘              └────────────────────┘

Now the app's next memory access is a CACHE MISS.
This indirect cost can add 10-50% overhead to the application.

RDMA avoids this entirely — the NIC uses DMA, which bypasses CPU caches.
```

---

## 🛠️ How RDMA Eliminates Each Bottleneck

```
Bottleneck              │ TCP/IP Approach          │ RDMA Approach
════════════════════════╪══════════════════════════╪═══════════════════════
Memory copies           │ 4 copies per message     │ 0 copies — NIC does
                        │ (user↔kernel, kernel↔NIC)│ DMA to/from app memory
────────────────────────┼──────────────────────────┼───────────────────────
Kernel involvement      │ Every send/recv is a     │ Kernel bypass — app
                        │ system call through      │ talks directly to NIC
                        │ the kernel               │ via memory-mapped I/O
────────────────────────┼──────────────────────────┼───────────────────────
Protocol processing     │ CPU builds/parses TCP/IP │ NIC handles protocol
                        │ headers and checksums    │ in hardware (offload)
────────────────────────┼──────────────────────────┼───────────────────────
Interrupts              │ One interrupt per packet  │ Polling (no interrupts)
                        │ or per batch             │ or batched completions
────────────────────────┼──────────────────────────┼───────────────────────
Remote CPU involvement  │ Remote CPU processes     │ RDMA Write/Read: remote
                        │ every received packet    │ CPU does ZERO work
────────────────────────┼──────────────────────────┼───────────────────────
Context switches        │ 2 per operation (user→   │ 0 — everything happens
                        │ kernel→user)             │ in user space
────────────────────────┼──────────────────────────┼───────────────────────
Cache pollution         │ CPU touches all data     │ DMA bypasses CPU cache
                        │ (pollutes L1/L2/L3)      │ entirely
```

---

## 📚 Protocol Stack Comparison

```
Traditional TCP/IP:                    iWARP (RDMA over TCP):
═══════════════════                    ═════════════════════

┌───────────────────┐                  ┌───────────────────┐
│   Application     │                  │   Application     │
│   (send/recv)     │                  │   (RDMA Verbs)    │
├───────────────────┤                  ├───────────────────┤
│   Socket API      │ ←── in kernel    │   RDMAP           │ ←── in NIC
├───────────────────┤                  ├───────────────────┤     hardware
│   TCP             │ ←── in kernel    │   DDP             │ ←── (offloaded)
├───────────────────┤                  ├───────────────────┤
│   IP              │ ←── in kernel    │   MPA             │
├───────────────────┤                  ├───────────────────┤
│   Ethernet Driver │ ←── in kernel    │   TCP (offloaded) │
├───────────────────┤                  ├───────────────────┤
│   NIC Hardware    │                  │   IP (offloaded)  │
└───────────────────┘                  ├───────────────────┤
                                       │   NIC Hardware    │
                                       └───────────────────┘

Key difference: In TCP/IP, layers run on the CPU in the kernel.
In RDMA, layers are offloaded to NIC hardware.
The application bypasses the kernel entirely.
```

---

## 🔌 Connection Setup: Sockets vs Queue Pairs

### TCP Socket Workflow

```python
# SERVER                              # CLIENT
sock = socket(AF_INET, SOCK_STREAM)   sock = socket(AF_INET, SOCK_STREAM)
bind(sock, ("0.0.0.0", 8080))
listen(sock)
                                      connect(sock, ("server", 8080))
conn = accept(sock)                   # 3-way handshake happens here

# Data transfer
send(conn, data)                      recv(sock, buffer)

# All goes through kernel
# Every send/recv = system call
# Data is copied in and out of kernel buffers
```

### RDMA Queue Pair Workflow

```python
# SERVER                              # CLIENT
pd = alloc_pd(device)                 pd = alloc_pd(device)
cq = create_cq(device, depth=100)     cq = create_cq(device, depth=100)
qp = create_qp(pd, cq, type=RC)      qp = create_qp(pd, cq, type=RC)

# Register memory regions
mr = reg_mr(pd, buffer, size,         mr = reg_mr(pd, buffer, size,
            IBV_ACCESS_REMOTE_WRITE)              IBV_ACCESS_LOCAL_WRITE)

# Exchange connection info (via TCP or CM)
exchange_qp_info(qp, remote_info)     exchange_qp_info(qp, remote_info)

# Connect QPs
modify_qp(qp, state=RTR)             modify_qp(qp, state=RTR)
modify_qp(qp, state=RTS)             modify_qp(qp, state=RTS)

# Data transfer (RDMA Write — one-sided!)
post_send(qp, {                       # Client does NOTHING here!
    opcode: RDMA_WRITE,               # Data appears in client's
    remote_addr: client_mr.addr,      # buffer magically.
    rkey: client_mr.rkey,             #
    local_addr: my_data,              # Client's CPU can be doing
    length: 1MB                       # other work the whole time.
})

poll_cq(cq)  # Wait for completion

# No system calls during data transfer!
# No kernel involvement!
# No copies!
```

### Key Differences

```
┌────────────────────┬──────────────────┬─────────────────────────┐
│ Aspect             │ TCP Sockets      │ RDMA Queue Pairs        │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Setup complexity   │ Simple           │ More complex            │
│                    │ (6 system calls) │ (15+ setup calls)       │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Data path          │ Through kernel   │ Direct to NIC hardware  │
│                    │ on every I/O     │ (kernel only at setup)  │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Data model         │ Byte stream      │ Messages + direct       │
│                    │ (no boundaries)  │ memory access            │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Notification       │ Always           │ Optional (RDMA Write    │
│                    │ (recv returns)   │ doesn't notify remote)  │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Error handling     │ errno, graceful  │ QP moves to error state │
│                    │ partial reads    │ — connection is dead     │
├────────────────────┼──────────────────┼─────────────────────────┤
│ Programming model  │ Familiar, easy   │ Steeper learning curve  │
│                    │ (streams/datagrams)│ (verbs, WRs, CQs)    │
└────────────────────┴──────────────────┴─────────────────────────┘
```

---

## 📋 The Memory Copy Problem — Visualized

This is the single biggest reason RDMA exists. Let's see exactly where copies happen:

```
Traditional TCP Send (App A → App B):
═════════════════════════════════════

App A memory:  [DATA] ──── copy 1 ────▶ Kernel TX buffer: [DATA]
                                                │
                                           copy 2
                                                │
                                                ▼
                                        NIC TX ring: [DATA]
                                                │
                                            (on wire)
                                                │
                                                ▼
                                        NIC RX ring: [DATA]
                                                │
                                           copy 3
                                                │
                                                ▼
                                        Kernel RX buffer: [DATA]
                                                │
                                           copy 4
                                                │
                                                ▼
App B memory:  [DATA] ◀────────────────────────┘

Total: 4 copies, all done by CPU
For 1 GB transfer = 4 GB of memory bandwidth wasted


RDMA Write (App A → App B):
════════════════════════════

App A memory:  [DATA] ─── DMA read ────▶ NIC: [DATA]
                        (NIC reads                │
                         directly!)           (on wire)
                                                  │
                                                  ▼
                                        NIC: [DATA] ─── DMA write ───▶ App B memory: [DATA]
                                                      (NIC writes
                                                       directly!)

Total: 0 CPU copies (DMA is done by the NIC's DMA engine, not the CPU)
For 1 GB transfer = 0 GB of CPU memory bandwidth wasted
```

---

## 🚫 Kernel Bypass — What It Really Means

```
Traditional I/O Path:
═════════════════════

User Space          │ Kernel Space           │ Hardware
────────────────────┼────────────────────────┼──────────────
                    │                        │
App calls send() ──▶│ Kernel receives data   │
  (system call)     │ TCP/IP processing      │
  (privilege switch)│ Copy to NIC buffer ────┼──▶ NIC sends
  (TLB flush)       │                        │
                    │ NIC interrupt ◀─────────┼── NIC done
App calls recv() ──▶│ Copy from NIC buffer   │
  (system call)     │ TCP/IP processing      │
  (privilege switch)│ Return data to app     │
                    │                        │

Every I/O operation crosses the user/kernel boundary TWICE.
Each crossing costs ~1000 CPU cycles (TLB flush, cache cold).


RDMA Kernel Bypass:
═══════════════════

User Space                                  │ Hardware
────────────────────────────────────────────┼──────────────
                                            │
App writes Work Request to QP ──────────────┼──▶ NIC reads WR
  (memory-mapped I/O — no system call!)     │    NIC does DMA
                                            │    NIC sends data
App polls Completion Queue ◀────────────────┼── NIC posts CQ entry
  (direct memory read — no system call!)    │
                                            │

The kernel is ONLY involved during setup (creating QPs, registering
memory). After that, ALL data operations happen in user space.

┌─────────────────────────────────────────────────────────────┐
│  Why does this matter?                                       │
│                                                             │
│  System call cost:     ~1000-5000 cycles (~0.5-2.5 μs)     │
│  MMIO doorbell ring:   ~100-200 cycles  (~0.05-0.1 μs)     │
│                                                             │
│  For 1 million messages/sec:                                 │
│    TCP:  1M × 2 syscalls × 2 μs = 4 seconds of CPU time    │
│    RDMA: 1M × 1 doorbell × 0.1 μs = 0.1 seconds           │
│                                                             │
│  That's 40x less CPU overhead for the same message rate.    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🤔 When to Use What

RDMA isn't always the right choice. Here's a decision framework:

```
                          ┌────────────────────┐
                          │ Do you need low     │
                          │ latency (<10 μs)?   │
                          └─────────┬──────────┘
                           Yes      │      No
                          ┌─────────┘      └──────────┐
                          ▼                            ▼
                 ┌──────────────────┐        ┌──────────────────┐
                 │ Is data transfer │        │ Is CPU usage a   │
                 │ on a fast LAN?   │        │ bottleneck?      │
                 └────────┬─────────┘        └────────┬─────────┘
                  Yes     │    No              Yes    │    No
                  │       │                    │      │
                  ▼       ▼                    ▼      ▼
              ┌──────┐  ┌──────┐          ┌──────┐  ┌──────┐
              │ RDMA │  │ TCP  │          │ RDMA │  │ TCP  │
              │  ✓   │  │ (WAN │          │  ✓   │  │ is   │
              │      │  │ loss)│          │      │  │ fine │
              └──────┘  └──────┘          └──────┘  └──────┘
```

### Use RDMA When:

| Scenario | Why RDMA Wins |
|----------|---------------|
| **HPC / Scientific computing** | MPI message passing needs μs latency |
| **Distributed storage** (NVMe-oF, Ceph) | Storage I/O at near-local-disk latency |
| **AI/ML training** (multi-GPU) | Gradient sync across nodes is the bottleneck |
| **In-memory databases** | Replication and distributed queries |
| **Financial trading** | Every microsecond of latency = money |
| **100+ Gbps throughput** | TCP can't saturate 100G without burning all CPUs |

### Stick with TCP When:

| Scenario | Why TCP Is Better |
|----------|-------------------|
| **Internet / WAN traffic** | Packet loss and long RTT negate RDMA benefits |
| **Simple client-server apps** | Socket API is much simpler to program |
| **Low throughput needs** | At 1 Gbps, TCP overhead is negligible |
| **Broad compatibility** | Every device speaks TCP; RDMA needs special NICs |
| **Dynamic connections** | RDMA connection setup is expensive — bad for short-lived connections |

---

## ❌ Common Misconceptions

```
Misconception #1: "RDMA is just faster TCP"
═══════════════════════════════════════════
Reality: RDMA is a fundamentally different programming model.
It's not a drop-in replacement — you redesign your application
around one-sided operations, memory registration, and queue pairs.


Misconception #2: "RDMA doesn't use TCP"
═══════════════════════════════════════════
Reality: iWARP RDMA runs ON TOP of TCP!
RoCE uses UDP. Only InfiniBand uses its own transport.
RDMA is about HOW data is handled, not which transport carries it.


Misconception #3: "RDMA means zero latency"
═══════════════════════════════════════════
Reality: RDMA eliminates SOFTWARE overhead.
You still have:
  - Speed-of-light propagation delay (5 μs per km of fiber)
  - Serialization delay (time to push bits on the wire)
  - Switch/router queuing delay
RDMA just removes the ~100 μs of CPU processing overhead.


Misconception #4: "RDMA is only for supercomputers"
═══════════════════════════════════════════════════
Reality: RDMA is everywhere in modern data centers:
  - Azure, AWS, GCP all use RDMA internally
  - Windows SMB Direct uses RDMA for file sharing
  - Most modern storage arrays use NVMe-oF over RDMA


Misconception #5: "The remote CPU does nothing with RDMA"
═══════════════════════════════════════════════════════════
Reality: Depends on the operation:
  - RDMA Write: correct, remote CPU is not involved
  - RDMA Read: correct, remote CPU is not involved  
  - RDMA Send: remote CPU IS notified via completion event
  - Setup/teardown always involves both CPUs
```

---

## 📊 Summary Table

```
┌────────────────────────┬───────────────────────┬───────────────────────┐
│ Characteristic         │ Traditional TCP/IP    │ RDMA                  │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ Memory copies          │ 4 per message         │ 0 (zero-copy)        │
│ Kernel involvement     │ Every I/O operation   │ Setup only           │
│ CPU overhead           │ High (all processing) │ Near zero            │
│ Latency                │ 50-100+ μs            │ 1-5 μs              │
│ Throughput (per core)  │ ~10-25 Gbps           │ 100-400 Gbps        │
│ Remote CPU notified    │ Always                │ Only for Send        │
│ Programming model      │ Byte stream (easy)    │ Verbs API (complex)  │
│ Connection setup       │ Fast (3-way handshake)│ Slow (QP setup)     │
│ Error handling         │ Graceful (partial)    │ Fatal (QP destroyed) │
│ Network requirements   │ Any IP network        │ Lossless (RoCE) or   │
│                        │                       │ TCP-based (iWARP)    │
│ Hardware requirements  │ Any NIC               │ RDMA-capable NIC     │
│ Operating systems      │ Everything            │ Linux, Windows, ESXi │
│ Ecosystem maturity     │ 40+ years             │ 20+ years (growing)  │
│ Best for               │ General networking    │ High-perf data center│
└────────────────────────┴───────────────────────┴───────────────────────┘
```

---

## 📖 References

- [RFC 5040 — Direct Data Placement over Reliable Transports](https://www.rfc-editor.org/rfc/rfc5040)
- [RFC 5042 — Remote Direct Memory Access Protocol (RDMAP)](https://www.rfc-editor.org/rfc/rfc5042)
- [RFC 5044 — Marker PDU Aligned Framing for TCP (MPA)](https://www.rfc-editor.org/rfc/rfc5044)
- [RDMA Aware Networks Programming User Manual (Mellanox)](https://docs.nvidia.com/networking/display/rdmacoreprogrammingmanual/)

---

> *Same wire, completely different game.* ⚡
