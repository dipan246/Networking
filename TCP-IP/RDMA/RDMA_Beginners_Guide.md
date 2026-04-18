# 🚀 RDMA — The Beginner's Guide

> **A beginner-friendly rewrite of [RFC 5040](https://www.rfc-editor.org/rfc/rfc5040) — the Remote Direct Memory Access Protocol Specification.**
>
> Before we can understand *Remote* Direct Memory Access, we need to understand **memory** itself — what it is, where it lives, and how the CPU gets data in and out of it. This guide starts from first principles and builds up to RDMA, step by step.

---

## 📖 Table of Contents

1. [Computer Memory — The Foundation](#1--computer-memory--the-foundation)
2. [Storage: HDD and SSD](#2--storage-hdd-and-ssd)
3. [Coherent vs. Non-Coherent Memory](#3--coherent-vs-non-coherent-memory)
4. [What Is DMA?](#4--what-is-dma)
5. [When a Packet Arrives — CPU and DMA in Action](#5--when-a-packet-arrives--cpu-and-dma-in-action)
6. [What Is RDMA? How Does It Work?](#6--what-is-rdma-how-does-it-work)
7. [Invalidation and Key RDMA Terms](#7--invalidation-and-key-rdma-terms)
8. [RDMA Write — The Complete Story](#8--rdma-write--the-complete-story)
9. [References](#9--references)

---

## 1. 🧠 Computer Memory — The Foundation

### What Is Memory?

At its core, **memory** is where a computer stores data it's working with *right now*. Think of your desk: the papers spread out in front of you are your "memory" — easily reachable, quickly referenced. The filing cabinet in the corner? That's your "storage" (we'll get to that next).

Every program running on a computer — your web browser, a database, a game — needs memory to hold:
- The instructions it's executing (code)
- The data it's working on (variables, buffers, structures)
- Temporary results (intermediate calculations)

### The Memory Hierarchy

Computers use a **hierarchy** of memory types, organized by speed and cost:

```
                    ┌──────────┐
                    │ CPU      │  ← Fastest (~0.3 ns access)
                    │ Registers│     Smallest (few hundred bytes)
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ L1 Cache │  ← Very fast (~1 ns)
                    │ (64 KB)  │     Per-core, small
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ L2 Cache │  ← Fast (~4 ns)
                    │ (256 KB) │     Per-core, medium
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ L3 Cache │  ← Pretty fast (~12 ns)
                    │ (16 MB+) │     Shared across cores
                    └────┬─────┘
                         │
               ┌─────────▼─────────┐
               │   Main Memory     │  ← Moderate (~100 ns)
               │   (RAM / DRAM)    │     16 GB – 1 TB typical
               │   DDR4 / DDR5     │     Volatile (loses data on power off)
               └─────────┬─────────┘
                         │
               ┌─────────▼─────────┐
               │   Storage          │  ← Slow (~10,000–10,000,000 ns)
               │   (SSD / HDD)     │     256 GB – many TB
               │                    │     Non-volatile (keeps data on power off)
               └────────────────────┘
```

### Types of Memory

| Type | Full Name | Speed | Volatile? | Use Case |
|------|-----------|-------|-----------|----------|
| **SRAM** | Static RAM | Very fast (~1 ns) | Yes | CPU caches (L1, L2, L3) |
| **DRAM** | Dynamic RAM | Fast (~100 ns) | Yes | Main memory (RAM sticks) |
| **Flash/NAND** | — | Moderate (~50 μs) | No | SSDs, USB drives |
| **Magnetic** | — | Slow (~5-10 ms) | No | Hard disk drives (HDD) |

#### SRAM vs. DRAM — The Key Difference

**SRAM (Static RAM)** uses 6 transistors per bit. It's fast and doesn't need refreshing, but it's expensive and takes up a lot of chip space. That's why CPU caches are only kilobytes to megabytes.

**DRAM (Dynamic RAM)** uses 1 transistor + 1 capacitor per bit. The capacitor leaks charge, so DRAM must be **refreshed** thousands of times per second (hence "dynamic"). It's slower than SRAM, but much cheaper and denser — which is why your computer has gigabytes of DRAM but only megabytes of SRAM cache.

```
SRAM Cell (6 transistors):         DRAM Cell (1T + 1C):
┌─────────────────────┐           ┌──────────────┐
│  ┌───┐     ┌───┐    │           │  ┌───┐       │
│  │ T ├──┬──┤ T │    │           │  │ T ├──┐    │
│  └───┘  │  └───┘    │           │  └───┘  │    │
│  ┌───┐  │  ┌───┐    │           │      ┌──┴──┐ │
│  │ T ├──┴──┤ T │    │           │      │  C  │ │ ← Capacitor leaks!
│  └───┘     └───┘    │           │      │     │ │   Must refresh
│  ┌───┐     ┌───┐    │           │      └─────┘ │   every ~64 ms
│  │ T ├─────┤ T │    │           └──────────────┘
│  └───┘     └───┘    │
└─────────────────────┘
   Fast, no refresh needed           Cheap, dense, needs refresh
```

### Virtual Memory — Every Process Gets Its Own View

Modern operating systems give every process its own **virtual address space**. When your program reads address `0x7F0000`, the CPU's **Memory Management Unit (MMU)** translates that to a physical address in DRAM. This is crucial for RDMA — because RDMA needs to work with *physical* memory addresses to place data directly, bypassing the CPU.

```
Process A sees:             Physical RAM:              Process B sees:
┌────────────────┐         ┌────────────────┐         ┌────────────────┐
│ 0x0000: code   │────────▶│ 0x1A000: A code│◀────────│ 0x0000: code   │
│ 0x1000: data   │────┐    │ 0x1B000: A data│    ┌────│ 0x1000: data   │
│ 0x7F00: buffer │──┐ │    │ 0x2C000: B code│    │ ┌──│ 0x7F00: buffer │
└────────────────┘  │ │    │ 0x3D000: B data│    │ │  └────────────────┘
                    │ └───▶│ 0x4E000: A buf │    │ │
                    │      │ 0x5F000: B buf │◀───┘ │
                    └─────▶│ 0x4E000        │      │
                           │ 0x5F000        │◀─────┘
                           └────────────────┘
                              MMU handles
                              the translation
```

> **Why this matters for RDMA**: When a remote machine wants to write data into your application's buffer, it needs to know where that buffer *actually* is in physical memory — not the virtual address your program sees. This is why RDMA requires **memory registration** (we'll cover that in Section 6).

---

## 2. 💾 Storage: HDD and SSD

Storage is where data lives *permanently* — it survives power cycles. While not directly part of RDMA, understanding storage helps you appreciate *why* RDMA exists: storage is slow, and when distributed systems need to replicate data across machines, the network becomes the bottleneck, not the disk.

### Hard Disk Drives (HDD)

An HDD is a mechanical device. Spinning magnetic platters store bits, and a read/write head moves across the surface to find data.

```
        Top View of an HDD:
        ┌───────────────────────┐
        │      ╭──────────╮     │
        │    ╭─┤ Spinning ├─╮   │
        │   │  │  Platter  │  │  │
        │   │  │ (7200 RPM)│  │  │
        │   │  ╰─────┬─────╯  │  │
        │   │        │        │  │
        │   │   ┌────┴────┐   │  │
        │   │   │  R/W    │   │  │
        │   │   │  Head   │◀──── Moves mechanically
        │   │   └─────────┘   │  │  (seek time: 5-10 ms)
        │   ╰─────────────────╯  │
        └───────────────────────┘
```

**Performance:**
- **Sequential read**: ~150-200 MB/s
- **Random read**: ~0.5-1 MB/s (because the head has to physically *move*)
- **Latency**: 5-10 ms (an eternity for a CPU that runs billions of cycles per second)

### Solid State Drives (SSD)

An SSD has **no moving parts**. It uses NAND flash memory — the same technology in your phone's storage, just much faster and with a smarter controller.

```
        Inside an SSD:
        ┌───────────────────────────────────┐
        │  ┌──────────┐  ┌──────────┐      │
        │  │NAND Flash│  │NAND Flash│ ...  │
        │  │  Chip 0  │  │  Chip 1  │      │
        │  └────┬─────┘  └────┬─────┘      │
        │       │              │             │
        │  ┌────▼──────────────▼────┐       │
        │  │    SSD Controller       │       │
        │  │  (wear leveling, FTL,   │       │
        │  │   garbage collection)   │       │
        │  └────────────┬────────────┘       │
        │               │                    │
        │          PCIe / SATA               │
        │          Interface                 │
        └───────────────────────────────────┘
```

**Performance (NVMe SSD over PCIe):**
- **Sequential read**: 3,000-7,000 MB/s
- **Random read**: 500-1,000 MB/s
- **Latency**: 10-100 μs (100-1000× faster than HDD)

### HDD vs SSD — Quick Comparison

| Feature | HDD | SSD (NVMe) |
|---------|-----|-----------|
| Speed (sequential) | ~200 MB/s | ~5,000 MB/s |
| Latency | 5-10 ms | 10-100 μs |
| Moving parts | Yes (spinning platters) | None |
| Durability | Fragile (drop = death) | Solid |
| Cost per TB | Low (~$15-25) | Higher (~$50-80) |

> **The RDMA connection**: In distributed storage systems (like Ceph, HDFS, or Azure Storage), data is replicated across machines. If your network is slow (traditional TCP copies, kernel overhead, interrupts), it becomes the bottleneck — even with fast SSDs. RDMA removes that bottleneck by making the network path as efficient as a local memory copy.

---

## 3. 🔄 Coherent vs. Non-Coherent Memory

### What Is Cache Coherence?

Remember the memory hierarchy? The CPU doesn't read directly from DRAM every time — it keeps copies of frequently used data in its **caches** (L1, L2, L3). But what happens when multiple entities can modify the same data?

**Cache coherence** is the guarantee that all observers see a consistent view of memory — if one core writes a value, other cores will see the updated value (not a stale cached copy).

```
Example: Two CPU cores share a variable "counter = 5"

  ┌──────────────┐     ┌──────────────┐
  │   Core 0     │     │   Core 1     │
  │              │     │              │
  │  L1 Cache:   │     │  L1 Cache:   │
  │  counter = 5 │     │  counter = 5 │
  └──────┬───────┘     └──────┬───────┘
         │                    │
         └────────┬───────────┘
                  │
         ┌────────▼──────────┐
         │   Shared L3 Cache │
         │   counter = 5     │
         └────────┬──────────┘
                  │
         ┌────────▼──────────┐
         │      DRAM         │
         │   counter = 5     │
         └───────────────────┘

Core 0 writes: counter = 10

  ┌──────────────┐     ┌──────────────┐
  │   Core 0     │     │   Core 1     │
  │              │     │              │
  │  L1 Cache:   │     │  L1 Cache:   │
  │  counter = 10│     │  counter = 5 │ ← STALE! This is the problem.
  └──────────────┘     └──────────────┘

Coherence protocol (e.g., MESI) fixes this:
  → Core 0 says: "I modified 'counter', invalidate your copies!"
  → Core 1 marks its cached copy as Invalid
  → Next time Core 1 reads 'counter', it fetches the updated value (10)
```

### Coherent Memory

**Coherent memory** participates in the cache coherence protocol. When any participant (CPU core, device) writes to coherent memory, all caches are kept in sync automatically.

- ✅ CPU cores always see the latest value
- ✅ No manual "flush" or "invalidate" needed
- ❌ Slower — the coherence protocol adds overhead (snooping, invalidation messages)

### Non-Coherent Memory

**Non-coherent memory** does NOT participate in the cache coherence protocol. A device can write to this memory, but the CPU's caches won't automatically know about the change.

- ✅ Faster — no coherence overhead
- ❌ Requires explicit cache management (flush before device reads, invalidate before CPU reads)
- Common in: GPU memory, some DMA buffers, device memory regions

```
Coherent vs. Non-Coherent:

COHERENT (CPU writes, device reads):
  CPU writes → Cache updated → Coherence protocol → Device sees latest value ✓

NON-COHERENT (Device writes, CPU reads):
  Device writes → DRAM updated → CPU cache still has OLD value ✗
  Fix: CPU must INVALIDATE its cache before reading, to force re-fetch from DRAM

NON-COHERENT (CPU writes, device reads):
  CPU writes → Cache updated, but DRAM might NOT be updated yet ✗
  Fix: CPU must FLUSH its cache to push data to DRAM, so device sees latest value
```

> **Why this matters for RDMA and DMA**: When a NIC writes incoming packet data into system memory via DMA, the CPU's cache might still hold stale data for that address. The system must handle this — either by:
> 1. Using coherent DMA (hardware snoops the cache automatically — more common on modern x86)
> 2. Using non-coherent DMA + explicit cache invalidation (common on ARM, embedded systems)

---

## 4. 🔌 What Is DMA?

### The Problem: CPU as a Data Mover

In the early days of computing, the CPU had to personally move every single byte between devices and memory. Want to read a file from disk? The CPU would:

1. Send a command to the disk controller
2. Wait for each byte/word to be ready
3. Read the byte from the device port
4. Write the byte to memory
5. Repeat for every single byte

This is called **Programmed I/O (PIO)**, and it's terribly wasteful — the CPU spends all its time shuffling bytes instead of doing useful computation.

```
Programmed I/O (PIO) — The Old Way:
════════════════════════════════════

  CPU                  Device              Memory
  ───                  ──────              ──────
   │                     │                   │
   │──── "Give me       │                   │
   │      byte 0" ─────▶│                   │
   │                     │                   │
   │◀─── byte 0 ────────│                   │
   │                     │                   │
   │──── write byte 0 ──┼──────────────────▶│
   │                     │                   │
   │──── "Give me       │                   │
   │      byte 1" ─────▶│                   │
   │                     │                   │
   │◀─── byte 1 ────────│                   │
   │                     │                   │
   │──── write byte 1 ──┼──────────────────▶│
   │                     │                   │
   │   ... hundreds of thousands of times ...│
   │                     │                   │
   ▼ CPU is 100% busy    ▼                   ▼
     doing nothing but
     moving bytes!
```

### DMA — Let the Hardware Do It

**Direct Memory Access (DMA)** solves this by adding a dedicated hardware engine — the **DMA controller** — that can transfer data between devices and memory *without CPU involvement*.

The CPU's role becomes simple:
1. Tell the DMA controller: "Transfer N bytes from device X to memory address Y"
2. Go do useful work
3. Get interrupted when the transfer is done

```
DMA — The Modern Way:
═════════════════════

  CPU                DMA Controller       Device         Memory
  ───                ──────────────       ──────         ──────
   │                      │                 │              │
   │── "Transfer 4KB     │                 │              │
   │    from Device      │                 │              │
   │    to addr 0x1000"─▶│                 │              │
   │                      │                 │              │
   │ (CPU goes to do      │── read data ──▶│              │
   │  useful work!)       │◀── data ───────│              │
   │                      │── write data ──┼─────────────▶│
   │  🔧 Computing...    │                 │              │
   │  🔧 Still working...│── read data ──▶│              │
   │  🔧 More work...    │◀── data ───────│              │
   │                      │── write data ──┼─────────────▶│
   │                      │                 │              │
   │                      │  (repeats until │              │
   │                      │   all 4KB done) │              │
   │                      │                 │              │
   │◀── INTERRUPT:       │                 │              │
   │    "Transfer done!" │                 │              │
   │                      │                 │              │
   ▼ CPU was FREE         ▼                 ▼              ▼
     during the entire
     transfer!
```

### How DMA Works — The Scatter-Gather List

Modern DMA doesn't just transfer one contiguous block. It uses **scatter-gather lists** — a list of (address, length) pairs that tell the DMA engine where to put (or get) chunks of data:

```
Scatter-Gather List:
┌──────────────────────────────┐
│ Entry 0: addr=0x1000, len=512│──▶ Physical page at 0x1000
│ Entry 1: addr=0x5000, len=512│──▶ Physical page at 0x5000
│ Entry 2: addr=0x9000, len=1024│──▶ Physical page at 0x9000
└──────────────────────────────┘

The DMA engine writes incoming data across these
non-contiguous physical pages, reassembling a
contiguous virtual buffer that the OS carved up.
```

This is important because virtual memory means a "contiguous" buffer in your application might be scattered across many physical pages.

### Bus Mastering DMA

In modern systems, the DMA engine is usually built *into the device itself* (like the NIC or disk controller). The device becomes a **bus master** — it can independently read and write system memory over the PCIe bus without going through the CPU.

```
Modern Bus Mastering DMA:
┌────────────────────────────────────────────┐
│                    CPU                      │
│            (not involved in               │
│             data transfer!)                │
└───────────────────┬────────────────────────┘
                    │ (only for setup
                    │  and interrupts)
         ┌──────────▼──────────┐
         │   System Memory     │◀──────────────┐
         │   (DRAM)            │               │
         └──────────┬──────────┘               │
                    │                          │
              ╔═════▼══════════════════════════╗│
              ║       PCIe Bus                 ║│
              ╚═════════╤═════════╤════════════╝│
                        │         │             │
                   ┌────▼───┐ ┌──▼────┐        │
                   │  NIC   │ │ NVMe  │        │
                   │(has own│ │ SSD   │        │
                   │ DMA    │ │(has   │        │
                   │ engine)│ │ own   │────────┘
                   └────────┘ │ DMA)  │  DMA writes
                              └───────┘  directly to
                                         memory!
```

---

## 5. 📡 When a Packet Arrives — CPU and DMA in Action

Now let's bring it all together. What happens when a network packet arrives at your computer? Let's compare the two approaches: **without DMA** and **with DMA**.

### Without DMA (Programmed I/O)

```
Network Packet Arrives Without DMA:
════════════════════════════════════

  Wire ──▶ NIC receives packet into its internal buffer
              │
              ▼
  NIC raises INTERRUPT ──▶ CPU
              │
              ▼
  CPU runs interrupt handler (ISR):
    ┌────────────────────────────────────────────┐
    │  for each byte in packet:                  │
    │    1. CPU reads byte from NIC I/O port     │
    │    2. CPU writes byte to kernel buffer     │
    │                                            │
    │  This is SLOW — CPU is stuck copying       │
    │  every single byte, one at a time.         │
    │  A 1500-byte packet = 1500 read+write ops! │
    └────────────────────────────────────────────┘
              │
              ▼
  CPU processes TCP/IP headers:
    - Verify checksums
    - Match to socket
    - Strip headers
              │
              ▼
  CPU copies payload from kernel buffer
  to application buffer (another copy!)
              │
              ▼
  Application gets the data

  Total CPU involvement: 100% — CPU did ALL the work
  Memory copies: 2 (NIC→kernel, kernel→app)
```

### With DMA (Modern NICs)

```
Network Packet Arrives With DMA:
════════════════════════════════

  SETUP (one-time, before any packets arrive):
  ┌─────────────────────────────────────────────────┐
  │  OS driver allocates a "ring buffer" in memory   │
  │  OS tells NIC: "Here's a list of memory          │
  │    addresses (descriptors) where you can          │
  │    place incoming packets via DMA"                │
  │                                                   │
  │  Ring Buffer (in system memory):                  │
  │  ┌─────┬─────┬─────┬─────┬─────┐                │
  │  │Desc0│Desc1│Desc2│Desc3│Desc4│  Each descriptor │
  │  │addr │addr │addr │addr │addr │  points to a     │
  │  │0x1000│0x2000│0x3000│0x4000│0x5000│  pre-allocated  │
  │  └─────┴─────┴─────┴─────┴─────┘  buffer in memory │
  └─────────────────────────────────────────────────┘

  RUNTIME (when a packet arrives):
  ════════════════════════════════

  1. Wire ──▶ NIC receives packet

  2. NIC's DMA engine reads next descriptor from ring buffer
     NIC knows: "Put this packet at address 0x1000"

  3. NIC DMA-writes the packet directly to address 0x1000
     ┌──────────┐                    ┌──────────────┐
     │   NIC    │═══ PCIe Bus ══════▶│ System Memory │
     │          │    DMA write       │ addr 0x1000:  │
     │ packet:  │    (no CPU!)       │ [packet data] │
     │ [data]   │                    └──────────────┘
     └──────────┘

  4. NIC updates the descriptor: "I wrote 1500 bytes here"

  5. NIC raises INTERRUPT ──▶ CPU
     (or CPU polls in high-performance "busy polling" mode)

  6. CPU runs interrupt handler:
     ┌────────────────────────────────────────────┐
     │  "Oh, there's a packet at 0x1000"          │
     │  - Process TCP/IP headers                   │
     │  - Copy payload to application buffer       │
     │    (kernel → user space copy)               │
     └────────────────────────────────────────────┘

  7. Application gets the data

  Total CPU involvement: ~30% — only header processing + 1 copy
  Memory copies: 1 (kernel→app)  ← DMA eliminated the first copy!
```

### Side-by-Side Comparison

```
            Without DMA                    With DMA
            ═══════════                    ════════
 Packet     NIC ──(PIO)──▶ Kernel ──▶ App  NIC ══(DMA)══▶ Kernel ──▶ App
 arrives    CPU copies      CPU copies      NIC copies     CPU copies
            every byte!     to app          via DMA!       to app

 CPU load:  100%                            ~30%
 Copies:    2 (NIC→kernel, kernel→app)      1 (kernel→app)
 Latency:   High (CPU bottleneck)           Lower

 But still... there's STILL a copy from kernel to app!
 And the CPU STILL processes every packet header!

 Can we do better? ──▶ YES. That's where RDMA comes in.
```

### The Remaining Problem

Even with DMA, traditional networking still has overhead:

1. **One memory copy remains** — kernel buffer → application buffer
2. **CPU processes every packet** — TCP checksum, header parsing, socket lookup
3. **Interrupts** — each packet (or batch) interrupts the CPU
4. **Kernel transitions** — data must cross the kernel/user-space boundary
5. **Context switches** — the application may need to be scheduled to receive data

**RDMA eliminates ALL of these.** Let's see how.

---

## 6. 🌐 What Is RDMA? How Does It Work?

### The Big Idea

**RDMA (Remote Direct Memory Access)** extends the concept of DMA across the network. Instead of the NIC writing packets into a kernel buffer for the CPU to process, the NIC writes data **directly into the application's memory buffer** — on a *remote* machine — without involving either machine's CPU or OS kernel.

```
Traditional TCP/IP:
═══════════════════
App A ──▶ Kernel ──▶ NIC ═══network═══ NIC ──▶ Kernel ──▶ App B
          copy #1    copy #2           copy #3  copy #4
          CPU work   CPU work          CPU work CPU work

RDMA:
═════
App A's memory ══════ NIC ═══network═══ NIC ══════ App B's memory
                no CPU                     no CPU
                no kernel                  no kernel
                no copies                  no copies
```

### The Three Superpowers of RDMA

| Superpower | What It Means |
|------------|---------------|
| **Zero-copy** | Data goes directly from app memory on Machine A to app memory on Machine B. No intermediate buffer copies. |
| **Kernel bypass** | The application talks directly to the RDMA NIC (called an **RNIC** — RDMA-capable NIC), skipping the OS kernel entirely on the data path. |
| **CPU bypass** | The remote CPU is not involved at all. The RNIC on the receiving side places data directly into the application's buffer. The remote application might not even know data arrived until it checks. |

### Memory Registration — How RDMA Knows Where to Put Data

Here's the fundamental challenge: Machine A wants to write data into Machine B's application memory. But how does Machine A know *where* that buffer is?

The answer is **memory registration**:

```
Step 1: Application on Machine B registers a memory region with its RNIC

  Application B:
    buffer = malloc(4096);  // Allocate a 4KB buffer

    // Register this buffer with the RNIC
    mr = rdma_register_memory(buffer, 4096, permissions=READ|WRITE);

  What happens under the hood:
  ┌─────────────────────────────────────────────────────────────┐
  │  1. OS pins the physical pages (prevents swapping to disk)  │
  │  2. OS gives the RNIC the virtual→physical page mappings    │
  │  3. RNIC creates a Memory Region (MR) entry in its tables:  │
  │                                                             │
  │     ┌────────────────────────────────────────────┐          │
  │     │  STag (Steering Tag): 0xABCD               │          │
  │     │  Base Address:        0x7F0000 (virtual)    │          │
  │     │  Length:              4096 bytes             │          │
  │     │  Physical Pages:     [0x1A000, 0x1B000]    │          │
  │     │  Permissions:        Remote Read + Write    │          │
  │     │  Protection Domain:  PD #3                  │          │
  │     └────────────────────────────────────────────┘          │
  │                                                             │
  │  4. RNIC returns an STag (Steering Tag) to the application  │
  │     This STag is like a "key" to this memory region         │
  └─────────────────────────────────────────────────────────────┘
```

```
Step 2: Application B sends the STag, address, and length to Application A
        (using a normal Send message or any out-of-band mechanism)

  Machine B ──────────────────────────────▶ Machine A
              "Hey, you can write to my
               memory using:
               STag   = 0xABCD
               Offset = 0 (start of buffer)
               Length = 4096 bytes"

  This is called ADVERTISEMENT — RFC 5040 Section 2.1 defines it as:
  "The act of informing a Remote Peer that a local RDMA Buffer
   is available to it."
```

```
Step 3: Application A can now do RDMA Write directly into B's buffer

  Machine A's RNIC:                    Machine B's Memory:
  ┌──────────────────┐                ┌──────────────────┐
  │ "Write 1024 bytes│ ═══network═══▶ │ RNIC checks STag │
  │  to STag 0xABCD  │                │ 0xABCD:          │
  │  at offset 0     │                │  ✓ Valid          │
  │  data: [.......]"│                │  ✓ Write allowed  │
  └──────────────────┘                │  ✓ Within bounds  │
                                      │                  │
                                      │ DMA write data   │
                                      │ directly to app  │
                                      │ buffer at 0x7F00 │
                                      │                  │
                                      │ CPU: 😴 sleeping │
                                      │ (not involved!)  │
                                      └──────────────────┘
```

### How an RDMA Connection Is Established — A Simple Example

Let's walk through a real-world RDMA data transfer between a storage client and server:

```
═══════════════════════════════════════════════════════════════
  RDMA Connection Setup: Storage Client ←→ Storage Server
═══════════════════════════════════════════════════════════════

Phase 1: Connection Setup
─────────────────────────

  Client                                         Server
    │                                               │
    │  1. Create RDMA resources:                    │
    │     - Queue Pair (QP) ← send/recv queues     │
    │     - Completion Queue (CQ) ← notifications  │
    │     - Protection Domain (PD) ← access ctrl   │
    │                                               │
    │  2. ──── TCP or CM handshake ────────────────▶│  3. Server also
    │     (exchange RDMA connection params:          │     creates QP, CQ, PD
    │      QP numbers, RNIC addresses, etc.)        │
    │                                               │
    │  4. ◀─── Connection Established ─────────────│
    │                                               │

Phase 2: Memory Registration & Advertisement
─────────────────────────────────────────────

  Server:
    │  5. Register a receive buffer:
    │     buffer = malloc(1MB)
    │     mr = register_memory(buffer, 1MB, REMOTE_WRITE)
    │     // RNIC returns STag = 0xBEEF
    │
    │  6. ──── RDMA Send Message ─────────────────▶ Client
    │     "Write your data to:
    │      STag = 0xBEEF, Offset = 0, Length = 1MB"
    │

Phase 3: RDMA Data Transfer
────────────────────────────

  Client:
    │  7. Client has data to send (e.g., a disk block)
    │     Client posts RDMA Write to its QP:
    │       - Source: local buffer with data
    │       - Dest:   STag=0xBEEF, Offset=0, Len=4KB
    │
    │  8. Client's RNIC reads data from client's memory (DMA)
    │     Client's RNIC sends it over the network
    │                          │
    │                     ═════╪═════ Network ═════
    │                          │
    │                          ▼
    │     Server's RNIC receives the packet
    │     Server's RNIC validates STag 0xBEEF:
    │       ✓ STag is valid and not expired
    │       ✓ Remote Write permission is set
    │       ✓ Offset + Length within bounds
    │     Server's RNIC DMA-writes data directly into
    │       the server application's buffer at 0x7F0000
    │
    │  9. Server CPU: still doing other work, never touched the data!
    │     Server app checks completion queue later and finds the data.
    │

Phase 4: Completion Notification
─────────────────────────────────

  Client:
    │  10. Client's CQ gets a completion entry:
    │      "RDMA Write completed successfully"
    │
    │  11. Client sends RDMA Send (with Solicited Event):
    │      "Hey server, I wrote block #42 to your buffer"
    │
    │                          ▼
    │      Server's CQ gets notification
    │      Server now knows: "Block #42 is in my buffer, ready to use"
```

### The Three RDMA Operations (from RFC 5040)

RFC 5040 defines these data transfer operations:

| Operation | Direction | Description |
|-----------|-----------|-------------|
| **RDMA Write** | Local → Remote | Write data into a remote machine's pre-advertised buffer (Tagged Buffer). Remote CPU is not involved. |
| **RDMA Read** | Remote → Local | Read data from a remote machine's pre-advertised buffer. Remote CPU is not involved. |
| **Send / Receive** | Local → Remote | Send data to the remote machine's *untagged* buffer (remote posted a receive buffer but didn't advertise its address). Remote CPU *is* notified. |

```
RDMA Write:
  Machine A: "Here's data, put it at STag X, offset Y"
  Machine B's RNIC: places data, no CPU involvement
  Machine B's CPU: 😴

RDMA Read:
  Machine A: "Give me data from your STag X, offset Y, length Z"
  Machine B's RNIC: reads data, sends it back, no CPU involvement
  Machine B's CPU: 😴

Send/Receive:
  Machine A: "Here's a message for you"
  Machine B's RNIC: places data in next available receive buffer
  Machine B's CPU: gets notified via Completion Queue
```

---

## 7. 🏷️ Invalidation and Key RDMA Terms

### What Is Invalidation?

When you register a memory region and share its STag with a remote machine, you're giving that machine a *key* to your memory. **Invalidation** is revoking that key.

From RFC 5040:
> **Invalidate STag** — A mechanism used to prevent the Remote Peer from reusing a previous explicitly Advertised STag, until the Local Peer makes it available through a subsequent explicit Advertisement.

```
Why Invalidation Matters:
═════════════════════════

Timeline:
─────────────────────────────────────────────────────────────

  1. Server registers buffer, gets STag = 0xABCD
  2. Server advertises STag 0xABCD to Client
  3. Client does RDMA Write using STag 0xABCD     ✓ Works
  4. Client does another RDMA Write to STag 0xABCD ✓ Still works
  5. Server INVALIDATES STag 0xABCD
  6. Client tries RDMA Write to STag 0xABCD        ✗ REJECTED!
     (STag is no longer valid — the key was revoked)

  This prevents:
  - Stale access to memory that's been freed or repurposed
  - Security vulnerabilities (remote machine accessing memory it shouldn't)
  - Use-after-free bugs in RDMA
```

### Two Ways to Invalidate

1. **Local Invalidation**: The application that owns the memory calls `invalidate(STag)` on its own RNIC. The remote side doesn't know until its next access fails.

2. **Remote Invalidation (Send with Invalidate)**: RFC 5040 defines `Send with Invalidate` — the *sender* includes an STag in the message header. When the message is received and placed, the RNIC at the receiver automatically invalidates that STag. This is powerful because:
   - It combines a data transfer (Send) with an invalidation in one round-trip
   - The remote side initiated the invalidation, saving the local side a step

```
Send with Invalidate Example:
═════════════════════════════

  Server registered STag 0xBEEF for Client to RDMA Write into.
  Client has finished writing. Now:

  Client ──── Send with Invalidate ────▶ Server
               payload: "Write complete"
               invalidate STag: 0xBEEF

  Server's RNIC:
    1. Places the Send payload into the receive buffer
    2. Invalidates STag 0xBEEF (Client can no longer access that buffer)
    3. Delivers completion to server application

  Result: In ONE message, Client both notified the server AND
          relinquished access to the server's memory. Efficient!
```

### Key RDMA Terms You Need to Know

Here's a glossary of the most important terms from RFC 5040, explained simply:

| Term | Definition | Analogy |
|------|-----------|---------|
| **STag (Steering Tag)** | A handle/key that identifies a registered memory region. Shared with the remote side to allow RDMA access. | Like a hotel room key card — it lets a specific person access a specific room. |
| **Tagged Buffer** | A memory buffer that has been registered and its STag advertised to the remote peer. Used for RDMA Read and Write. | A labeled mailbox — the sender knows exactly which box to put mail in. |
| **Untagged Buffer** | A receive buffer that the remote side does NOT know the address of. Used for Send operations — the RNIC picks the next available buffer from a queue. | An "inbox" — mail goes to the next available slot, sender doesn't choose which one. |
| **Memory Region (MR)** | A contiguous block of virtual memory registered with the RNIC, with defined permissions and an STag. | A registered safety deposit box — cataloged, protected, and accessible only with the right credentials. |
| **Protection Domain (PD)** | A logical grouping of RDMA resources (QPs, MRs) that can interact with each other. Resources in different PDs cannot access each other. | Separate buildings in a campus — your keycard works within your building, not others. |
| **Queue Pair (QP)** | The fundamental communication endpoint in RDMA. Contains a Send Queue (SQ) and Receive Queue (RQ). Applications post work requests to QPs. | Like a two-lane road — one lane for outgoing traffic (SQ), one for incoming (RQ). |
| **Completion Queue (CQ)** | A queue where the RNIC posts completion notifications after operations finish. The application polls or waits on the CQ. | A notification bell — rings when your order is ready. |
| **Work Request (WR)** | A request posted by the application to a QP, describing an operation (Send, Receive, RDMA Write, RDMA Read). | An order form — fill it out, submit it to the QP, and the RNIC executes it. |
| **Work Completion (WC)** | An entry in the CQ indicating that a Work Request has completed (successfully or with error). | The receipt — tells you your order was fulfilled (or why it failed). |
| **Advertisement** | The act of sharing a buffer's STag, offset, and length with the remote peer so it can perform RDMA operations on that buffer. | Giving someone your mailing address — now they can send packages directly to you. |
| **Placement** | The act of the RNIC writing received data into the destination memory buffer. Can happen out-of-order for efficiency. | The mailman putting letters in your mailbox — doesn't need to wait for you. |
| **Delivery** | Notifying the application that a complete message has arrived and is ready to use. Happens in order, even if placement was out-of-order. | The "You've got mail" notification — only fires after the full package is assembled. |
| **Solicited Event (SE)** | A flag on Send messages that causes a special notification at the receiver. Used for "important" messages that need immediate attention. | Marking a letter as "URGENT" — the recipient's doorbell rings instead of just filling the mailbox. |
| **Fence** | Blocks the current operation from executing until all prior operations have completed. Ensures ordering when needed. | A "wait for all previous orders" flag — don't start the next one until everything before it is done. |
| **RNIC** | RDMA-capable Network Interface Controller. A smart NIC with DMA engines, memory translation tables, and protocol processing built in. | A self-driving delivery truck — it knows where to go and how to deliver, no driver (CPU) needed. |

### The RDMA Protocol Stack (iWARP from RFC 5040)

RFC 5040 defines RDMA Protocol (RDMAP) as part of the **iWARP** suite, which layers like this:

```
┌─────────────────────────────────────────┐
│         Application / ULP               │  Your code
├─────────────────────────────────────────┤
│    RDMAP (Remote Direct Memory Access   │  RFC 5040
│           Protocol)                     │  Read, Write, Send, Invalidate
├─────────────────────────────────────────┤
│    DDP (Direct Data Placement           │  RFC 5041
│         Protocol)                       │  Places data into tagged/untagged buffers
├─────────────────────────────────────────┤
│    MPA (Marker PDU Aligned Framing)     │  RFC 5044
│         — only over TCP                 │  Frames RDMA messages within TCP stream
├────────────────┬────────────────────────┤
│      TCP       │        SCTP            │  Reliable transport
├────────────────┴────────────────────────┤
│              IP                          │
├──────────────────────────────────────────┤
│           Ethernet                       │
└──────────────────────────────────────────┘
```

> **Note**: iWARP (Internet Wide Area RDMA Protocol) runs RDMA over TCP/IP. There are other RDMA transports too:
> - **InfiniBand** — purpose-built RDMA fabric (fastest, used in HPC/AI clusters)
> - **RoCE (RDMA over Converged Ethernet)** — RDMA over Ethernet, skipping TCP (lower latency than iWARP)
> - **iWARP** — RDMA over TCP/IP (works over any IP network, including WANs)

---

## 8. ✍️ RDMA Write — The Complete Story

RDMA Write is the most commonly used RDMA operation in practice. Let's trace exactly what happens, byte by byte.

### What Is RDMA Write?

From RFC 5040 Section 5.1:
> An RDMA Write operation uses an RDMA Write Message to transfer data from the Data Source to a previously Advertised Buffer at the Data Sink.

In plain English: **"I have data. You told me where to put it (STag + offset). I'm writing it directly into your memory. Your CPU doesn't need to do anything."**

### The Complete RDMA Write Flow

```
═══════════════════════════════════════════════════════════════
  RDMA Write: Machine A writes "HELLO" into Machine B's buffer
═══════════════════════════════════════════════════════════════

PRE-CONDITION:
  Machine B has:
    - Registered a 4KB buffer, got STag = 0xCAFE
    - Advertised to Machine A: {STag=0xCAFE, Offset=0, Length=4096}

STEP 1: Application A Posts RDMA Write Work Request
────────────────────────────────────────────────────

  App A calls: rdma_post_write(
    local_buffer  = "HELLO" (5 bytes at addr 0x3000),
    remote_stag   = 0xCAFE,
    remote_offset = 0,
    length        = 5
  );

  This creates a Work Request (WR) on the Send Queue (SQ):
  ┌───────────────────────────────────────────┐
  │  WR #17:                                  │
  │    Type:          RDMA_WRITE              │
  │    Local Buffer:  addr=0x3000, len=5      │
  │    Remote STag:   0xCAFE                  │
  │    Remote Offset: 0                       │
  │    Remote Length:  5                       │
  └───────────────────────────────────────────┘

STEP 2: RNIC A Processes the Work Request
─────────────────────────────────────────

  RNIC A (on Machine A):
    1. Reads the WR from the Send Queue
    2. DMA-reads "HELLO" from application memory at 0x3000
    3. Constructs an RDMA Write Message:

       ┌──────────────────────────────────────────────┐
       │ TCP Header:                                   │
       │   src_port=5999, dst_port=5999, seq, ack...  │
       ├──────────────────────────────────────────────┤
       │ MPA Header:                                   │
       │   ULPDU_Length = (DDP+RDMA+payload size)      │
       ├──────────────────────────────────────────────┤
       │ DDP Header (Tagged Buffer Model):             │
       │   Tagged flag = 1                             │
       │   STag        = 0xCAFE                        │
       │   Tagged Offset = 0                           │
       │   DDP Message Sequence Number                 │
       ├──────────────────────────────────────────────┤
       │ RDMAP Header:                                 │
       │   Opcode = 0000 (RDMA Write)                  │
       │   (no additional RDMAP fields for Write)      │
       ├──────────────────────────────────────────────┤
       │ Payload: "HELLO" (5 bytes)                    │
       ├──────────────────────────────────────────────┤
       │ MPA CRC (for data integrity)                  │
       └──────────────────────────────────────────────┘

    4. RNIC A transmits the packet onto the wire

STEP 3: Network Transit
───────────────────────

  Machine A ════════════════════════════▶ Machine B
             Ethernet → Switches → Ethernet
             (standard network, nothing special)

STEP 4: RNIC B Receives and Processes
──────────────────────────────────────

  RNIC B (on Machine B):
    1. Receives the packet from the wire
    2. Validates TCP (sequence number, checksum)
    3. Strips MPA framing, validates CRC
    4. Reads DDP header:
       - Tagged flag = 1 → This is a Tagged Buffer write
       - STag = 0xCAFE
    5. Looks up STag 0xCAFE in its memory translation table:
       ┌────────────────────────────────────────┐
       │  STag 0xCAFE:                          │
       │    Valid:       YES                     │
       │    Permissions: REMOTE_WRITE            │ ✓
       │    Base vaddr:  0x7F0000               │
       │    Length:      4096                    │
       │    Phys pages:  [0x1A000]              │
       │    Prot Domain: PD #3                   │
       └────────────────────────────────────────┘
    6. Validates:
       ✓ STag is valid (not invalidated)
       ✓ REMOTE_WRITE permission is set
       ✓ Offset(0) + Length(5) ≤ Buffer Length(4096)
       ✓ QP belongs to Protection Domain #3
    7. Translates virtual address to physical:
       vaddr = 0x7F0000 + 0 = 0x7F0000 → phys 0x1A000
    8. DMA-WRITES "HELLO" to physical address 0x1A000

STEP 5: Data Is in Machine B's Application Memory!
───────────────────────────────────────────────────

  Machine B's Memory:
  ┌──────────────────────────────────────┐
  │ Physical addr 0x1A000:               │
  │   Before: [00 00 00 00 00 ...]       │
  │   After:  [48 45 4C 4C 4F ...]       │
  │            H  E  L  L  O              │
  │                                      │
  │ Machine B's CPU: 😴                  │
  │   (was never interrupted,            │
  │    never context-switched,           │
  │    never processed a header,         │
  │    never copied a byte)              │
  └──────────────────────────────────────┘
```

### What Happens After the RDMA Write?

The write is complete at the hardware level. But there are important follow-up considerations:

```
After RDMA Write Completes:
═══════════════════════════

On Machine A (the writer):
──────────────────────────
  1. RNIC A posts a Work Completion (WC) to the Completion Queue (CQ)
     ┌──────────────────────────────┐
     │  WC for WR #17:             │
     │    Status: SUCCESS           │
     │    Bytes transferred: 5      │
     │    Opcode: RDMA_WRITE        │
     └──────────────────────────────┘

  2. Application A polls CQ and sees: "My RDMA Write completed!"

  ⚠️  IMPORTANT: This only means Machine A's RNIC sent the data
     AND received a TCP ACK. It does NOT necessarily mean Machine B's
     application has *consumed* the data.

On Machine B (the receiver):
────────────────────────────
  3. Machine B's application does NOT get a completion event
     for RDMA Write! (This is by design — CPU bypass means no notification)

  4. So how does Machine B know data arrived?
     Two common patterns:

     Pattern A: "Send after Write" (most common)
     ─────────────────────────────────────────────
       After the RDMA Write, Machine A sends a regular RDMA Send:
       "Hey, I wrote 5 bytes at offset 0 of your buffer"

       Machine B's RNIC places this Send into B's receive queue
       Machine B gets a CQ completion for the Send
       Machine B reads the notification → knows to look at its buffer

     Pattern B: "Polling" (lowest latency)
     ──────────────────────────────────────
       Machine B's application continuously polls a known memory
       location for a "flag" or "sequence number" that Machine A
       writes as the last byte of the RDMA Write.

       while (buffer[LAST_BYTE] != EXPECTED_MAGIC) {
           // spin-wait — checking memory directly
       }
       // Data is ready!

  5. RFC 5040 ordering guarantee (Section 5.5):
     "All RDMA Write Messages on the same stream are guaranteed to
      be Placed in order."

     This means if Machine A does:
       RDMA Write #1 (data)
       RDMA Write #2 (flag/notification)

     Machine B is GUARANTEED to see #1's data before #2's flag.
     This enables the polling pattern above.
```

### RDMA Write with Immediate Data (Beyond RFC 5040)

Some RDMA implementations extend Write with an **immediate data** field — a small (4 bytes) value sent alongside the write that *does* generate a completion at the receiver. This combines the efficiency of RDMA Write with the notification of Send:

```
RDMA Write with Immediate:
  Machine A ──── RDMA Write + Imm(0x42) ────▶ Machine B
                                                │
                 Data placed in tagged buffer ───┘
                 AND CQ completion generated with imm_data=0x42

  Machine B's CQ: "RDMA Write arrived with immediate data 0x42"
  (Best of both worlds: zero-copy + notification!)
```

### Summary: The Life of an RDMA Write

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Register memory (once)           ← Both sides      │
│  2. Advertise STag + offset + len    ← Receiver → Sender│
│  3. Post RDMA Write work request     ← Sender app      │
│  4. RNIC reads data via local DMA    ← Sender RNIC     │
│  5. RNIC builds & sends packet       ← Sender RNIC     │
│  6. Remote RNIC validates STag       ← Receiver RNIC   │
│  7. Remote RNIC DMA-writes to app buf← Receiver RNIC   │
│  8. Sender gets CQ completion        ← Sender app      │
│  9. Notify receiver (Send or poll)   ← App-level       │
│ 10. Optionally invalidate STag       ← Either side     │
│                                                         │
│  CPU involvement: Steps 1, 2, 3, 8, 9 only             │
│  Steps 4-7: PURE HARDWARE — no CPU, no kernel, no copy │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 📚 References

- **[RFC 5040](https://www.rfc-editor.org/rfc/rfc5040)** — A Remote Direct Memory Access Protocol Specification (RDMAP) — the basis for this guide
- **[RFC 5041](https://www.rfc-editor.org/rfc/rfc5041)** — Direct Data Placement over Reliable Transports (DDP)
- **[RFC 5042](https://www.rfc-editor.org/rfc/rfc5042)** — Direct Data Placement Protocol (DDP) / Remote Direct Memory Access Protocol (RDMAP) Security
- **[RFC 5044](https://www.rfc-editor.org/rfc/rfc5044)** — Marker PDU Aligned Framing for TCP Specification (MPA)
- **[RFC 7146](https://www.rfc-editor.org/rfc/rfc7146)** — Updates to RFC 5765 and RFC 5040 (iWARP updates)
- **[InfiniBand Architecture Specification](https://www.infinibandta.org/)** — The original RDMA transport
- **[RDMA Consortium](https://www.rdmaconsortium.org/)** — Industry group for RDMA standards

---

> *Understanding RDMA starts with understanding memory. Once you see that the CPU has always been the bottleneck — copying bytes it doesn't care about, processing headers it doesn't need to read, getting interrupted for packets that aren't its concern — RDMA's design becomes obvious: let the hardware do what it's good at, and let the CPU do what it's good at.* 🧠
