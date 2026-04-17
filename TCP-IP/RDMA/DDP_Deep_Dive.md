# 🔬 DDP Deep Dive — Direct Data Placement Protocol

> **DDP** is the protocol that makes RDMA's "place data directly in application memory" magic possible. This document walks through how DDP works, step by step, with worked examples.

---

## 📖 Table of Contents

1. [What Problem Does DDP Solve?](#-what-problem-does-ddp-solve)
2. [DDP Message Model](#-ddp-message-model)
3. [Tagged vs Untagged Buffers — In Detail](#-tagged-vs-untagged-buffers--in-detail)
4. [DDP Segment Format — Byte by Byte](#-ddp-segment-format--byte-by-byte)
5. [Message Fragmentation — Splitting Large Messages](#-message-fragmentation--splitting-large-messages)
6. [Worked Example: RDMA Write Using DDP](#-worked-example-rdma-write-using-ddp)
7. [Worked Example: RDMA Send Using DDP](#-worked-example-rdma-send-using-ddp)
8. [Error Handling in DDP](#-error-handling-in-ddp)
9. [DDP and MPA — Why Framing Matters](#-ddp-and-mpa--why-framing-matters)
10. [Out-of-Order Placement — DDP's Hidden Superpower](#-out-of-order-placement--ddps-hidden-superpower)
11. [Summary](#-summary)

---

## 🤔 What Problem Does DDP Solve?

TCP delivers data as a **stream of bytes** — it has no concept of messages, buffers, or placement. When your application calls `recv()`, TCP gives you whatever bytes have arrived, and it's up to your application (and the CPU) to figure out where they belong.

DDP solves this by embedding **placement information** in every data segment:

```
The TCP Problem:
════════════════
Application wants to receive data into 3 separate buffers:

    Buffer A (1 KB) — for message headers
    Buffer B (1 MB) — for image data
    Buffer C (256 B) — for checksums

TCP delivers: [...all bytes mixed into one stream...]

    ┌──────────────────────────────────────────────────┐
    │ TCP Receive Buffer: AAABBBBBBBBBBBBBBBBCCC...     │
    └──────────────────────────────────────────────────┘

CPU must:
    1. Read from TCP buffer
    2. Parse protocol to figure out what goes where
    3. Copy header bytes → Buffer A
    4. Copy image bytes  → Buffer B
    5. Copy checksum     → Buffer C
    = Multiple copies + CPU parsing work


The DDP Solution:
═════════════════
Each segment carries its own delivery instructions:

    Segment 1: {dest=Buffer_A, offset=0}  [header data]
    Segment 2: {dest=Buffer_B, offset=0}  [image chunk 1]
    Segment 3: {dest=Buffer_B, offset=8K} [image chunk 2]
    ...
    Segment N: {dest=Buffer_C, offset=0}  [checksum data]

NIC reads the {dest, offset} and places data directly.
    = Zero copies, zero CPU parsing
```

---

## 📦 DDP Message Model

DDP operates on two fundamental concepts:

### Messages and Segments

```
┌────────────────────────────────────────────────────────────┐
│                    DDP Message                              │
│   (Upper layer says: "deliver this complete unit of data") │
│                                                            │
│   Total size: 64 KB                                        │
│   ┌────────────┬────────────┬────────────┬────────────┐   │
│   │ Segment 1  │ Segment 2  │ Segment 3  │ Segment 4  │   │
│   │ (16 KB)    │ (16 KB)    │ (16 KB)    │ (16 KB)    │   │
│   │ L=0        │ L=0        │ L=0        │ L=1        │   │
│   └────────────┴────────────┴────────────┴────────────┘   │
│                                              ▲              │
│                                              │              │
│                                     Last flag = 1           │
│                                  "This is the final piece" │
└────────────────────────────────────────────────────────────┘
```

**Key rules:**
- A **DDP Message** is the complete unit of data from the upper layer (RDMAP)
- A DDP Message may be split into multiple **DDP Segments** for transport
- Each segment fits within one MPA frame (and thus one TCP segment)
- The **Last (L) flag** marks the final segment of a message
- All segments of a message target the same buffer

---

## 🏷️ Tagged vs Untagged Buffers — In Detail

### Tagged Buffers (Used by RDMA Write and RDMA Read)

The sender knows exactly where data should land on the remote side.

```
How Tagged Buffers Work:
═══════════════════════

Setup Phase (one-time):
─────────────────────────
Machine B registers a memory region and gets:
    STag = 0x0000ABCD
    Base address = 0x00007F0000400000
    Length = 1,048,576 bytes (1 MB)

Machine B tells Machine A: "You can write to STag 0xABCD, up to 1 MB"


Data Transfer Phase:
────────────────────
Machine A wants to write 2048 bytes starting at offset 512:

    DDP Segment Header:
    ┌──────────────────────────────────┐
    │ T=1 (tagged), L=1 (last)        │
    │ STag  = 0x0000ABCD              │
    │ TO    = 0x0000000000000200 (512) │
    ├──────────────────────────────────┤
    │ [2048 bytes of payload data]     │
    └──────────────────────────────────┘

Machine B's NIC receives this and:
    1. Looks up STag 0xABCD in its translation table
    2. Finds physical address = base + offset = 0x7F0000400000 + 512
    3. Verifies: 512 + 2048 ≤ 1,048,576? ✓ (within bounds)
    4. Verifies: Write permission granted? ✓
    5. DMA writes 2048 bytes to physical address
    → Done! CPU never involved.
```

### Untagged Buffers (Used by RDMA Send)

The sender doesn't know where data will land — the receiver pre-posts buffers.

```
How Untagged Buffers Work:
═════════════════════════

Setup Phase:
────────────
Machine B pre-posts receive buffers to its Receive Queue:

    Receive Queue:
    ┌──────────────────────────────────────┐
    │ Entry 0: Buffer at 0x7F00...100, 4KB │ ← next to use
    │ Entry 1: Buffer at 0x7F00...200, 4KB │
    │ Entry 2: Buffer at 0x7F00...300, 4KB │
    └──────────────────────────────────────┘

    Queue Number (QN) = 3
    Current Message Sequence Number (MSN) = 0


Data Transfer Phase:
────────────────────
Machine A sends a message (doesn't know remote buffer address):

    DDP Segment Header:
    ┌──────────────────────────────────┐
    │ T=0 (untagged), L=1 (last)      │
    │ QN    = 3 (which receive queue)  │
    │ MSN   = 0 (message number)       │
    │ MO    = 0 (offset within msg)    │
    ├──────────────────────────────────┤
    │ [payload data]                   │
    └──────────────────────────────────┘

Machine B's NIC:
    1. Looks at QN=3, MSN=0
    2. Finds the next posted receive buffer (Entry 0)
    3. Places data into that buffer at offset MO=0
    4. Generates a completion event (CQ entry)
    → Receiver's app polls CQ and knows data arrived.
```

### When to Use Which?

```
┌────────────────┬───────────────────┬───────────────────────────┐
│                │  Tagged Buffer    │  Untagged Buffer          │
├────────────────┼───────────────────┼───────────────────────────┤
│ Sender knows   │  Yes — STag +    │  No — receiver posts      │
│ destination?   │  offset specify   │  buffers blindly          │
│                │  exact location   │                           │
├────────────────┼───────────────────┼───────────────────────────┤
│ Remote CPU     │  NOT notified     │  YES — completion event   │
│ involved?      │                   │  generated                │
├────────────────┼───────────────────┼───────────────────────────┤
│ Used by        │  RDMA Write       │  RDMA Send                │
│                │  RDMA Read        │                           │
├────────────────┼───────────────────┼───────────────────────────┤
│ Typical use    │  Bulk data,       │  Small control messages,  │
│                │  known buffers    │  signaling                │
├────────────────┼───────────────────┼───────────────────────────┤
│ Analogy        │  FedEx to exact   │  Drop-off at reception    │
│                │  room number      │  desk, someone picks up   │
└────────────────┴───────────────────┴───────────────────────────┘
```

---

## 🔍 DDP Segment Format — Byte by Byte

### Tagged Buffer DDP Segment

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│T│L│  Rsvd   │  DV  │       RsvdULP (used by RDMAP)               │
│1│ │         │      │                                              │
├─┴─┴─────────┴──────┴──────────────────────────────────────────────┤
│                          STag (32 bits)                            │
│                    Steering Tag — buffer key                       │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    Tagged Offset — TO (64 bits)                    │
│                    Byte offset within buffer                       │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│                         Payload Data                              │
│                   (variable length)                                │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

Field Breakdown:
─────────────────
T  (1 bit)  = 1 → Tagged buffer model
L  (1 bit)  = Last segment flag (1 = final segment of this DDP message)
Rsvd        = Reserved bits (must be zero)
DV (2 bits) = DDP Version (currently = 01 for version 1)
RsvdULP     = Reserved for upper layer protocol (RDMAP uses this)
STag        = 32-bit key identifying the target memory region
TO          = 64-bit byte offset within the memory region
```

### Untagged Buffer DDP Segment

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│T│L│  Rsvd   │  DV  │       RsvdULP (used by RDMAP)               │
│0│ │         │      │                                              │
├─┴─┴─────────┴──────┴──────────────────────────────────────────────┤
│                    Queue Number — QN (32 bits)                     │
│                    Which receive queue to target                   │
├───────────────────────────────────────────────────────────────────┤
│                 Message Sequence Number — MSN (32 bits)            │
│                 Which message this segment belongs to              │
├───────────────────────────────────────────────────────────────────┤
│                   Message Offset — MO (32 bits)                   │
│                   Byte offset within this message                 │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│                         Payload Data                              │
│                   (variable length)                                │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

Field Breakdown:
─────────────────
T  (1 bit)  = 0 → Untagged buffer model
L  (1 bit)  = Last segment flag
QN          = 32-bit queue number (which receive queue)
MSN         = 32-bit message sequence number (which message)
MO          = 32-bit message offset (where in the message buffer)
```

---

## ✂️ Message Fragmentation — Splitting Large Messages

When a DDP message is larger than the maximum segment size (limited by the MPA frame size and TCP MSS), DDP splits it into multiple segments:

```
Example: Writing 48 KB to remote buffer, max segment payload = 16 KB
═══════════════════════════════════════════════════════════════════

Original RDMA Write request:
    "Write 48 KB to STag=0xABCD, starting at offset 0"

DDP splits into 3 segments:

    Segment 1:                    Segment 2:                    Segment 3:
    ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
    │ T=1, L=0        │          │ T=1, L=0        │          │ T=1, L=1        │
    │ STag = 0xABCD   │          │ STag = 0xABCD   │          │ STag = 0xABCD   │
    │ TO   = 0        │          │ TO   = 16384    │          │ TO   = 32768    │
    │ [16 KB payload]  │          │ [16 KB payload]  │          │ [16 KB payload]  │
    └─────────────────┘          └─────────────────┘          └─────────────────┘
     offset 0─16383              offset 16384─32767            offset 32768─49151
                                                                        ▲
                                                                   L=1 (Last!)

    Note how each segment:
    • Points to the SAME STag (same remote buffer)
    • Has an INCREASING Tagged Offset (TO)
    • Only the LAST segment has L=1
```

### Why This Design Is Brilliant

Each segment is **self-describing** — it contains everything the remote NIC needs to place the data. This means:

1. **Segments can arrive out of order** (due to multipath, retransmission) and still be placed correctly
2. **The NIC doesn't need to buffer** — it can place each segment immediately upon arrival
3. **No reassembly** — there's nothing to reassemble because each piece goes directly to its final destination

---

## 📐 Worked Example: RDMA Write Using DDP

Let's trace a complete RDMA Write operation through the DDP layer:

```
Scenario:
═════════
Machine A wants to write the string "HELLO RDMA WORLD!!" (18 bytes)
to Machine B's memory region at offset 100.

Machine B previously shared:
    STag = 0x00001234
    Buffer base address = 0x00007F0000400000
    Buffer length = 4096 bytes
    Permissions = Remote Write enabled


Step 1: Application on Machine A posts an RDMA Write Work Request
─────────────────────────────────────────────────────────────────

    Work Request:
    {
        operation:    RDMA_WRITE,
        remote_stag:  0x00001234,
        remote_offset: 100,
        local_buffer: pointer to "HELLO RDMA WORLD!!",
        length:       18
    }


Step 2: RDMAP layer creates a DDP message
──────────────────────────────────────────

    RDMAP Header:
    {
        opcode: RDMA_WRITE (maps to DDP Tagged message)
    }


Step 3: DDP creates a segment (message fits in one segment)
──────────────────────────────────────────────────────────

    DDP Tagged Segment:
    ┌────────────────────────────────────────────────────┐
    │ Byte 0:    T=1, L=1, Rsvd=0, DV=01               │
    │ Bytes 1-3: RsvdULP = RDMA_WRITE opcode            │
    │ Bytes 4-7: STag = 0x00001234                       │
    │ Bytes 8-15: TO = 0x0000000000000064 (= 100 dec)   │
    │ Bytes 16-33: "HELLO RDMA WORLD!!"                  │
    └────────────────────────────────────────────────────┘
    Total DDP segment: 14 byte header + 18 byte payload = 32 bytes


Step 4: MPA frames the segment and TCP sends it
────────────────────────────────────────────────

    ┌──────────┬────────────────────────────────┬──────────┐
    │MPA Header│         DDP Segment            │ MPA CRC  │
    │(2B: len) │         (32 bytes)             │  (4B)    │
    └──────────┴────────────────────────────────┴──────────┘
    Wrapped in TCP → IP → Ethernet and sent on the wire.


Step 5: Machine B's NIC receives and processes
──────────────────────────────────────────────

    NIC extracts DDP segment from MPA frame.

    NIC reads header:
        T=1 → Tagged buffer model
        STag = 0x00001234
        TO = 100

    NIC looks up STag 0x1234 in its STag translation table:
        ✓ Valid STag
        ✓ Base physical addr = 0x00007F0000400000
        ✓ Length = 4096
        ✓ Remote Write = enabled

    NIC checks bounds:
        offset(100) + length(18) = 118 ≤ 4096 ✓

    NIC computes physical address:
        0x00007F0000400000 + 100 = 0x00007F0000400064

    NIC performs DMA write:
        Write "HELLO RDMA WORLD!!" to physical address 0x7F0000400064

    ✓ Done! Machine B's CPU was never interrupted.


Step 6: Machine A's NIC posts completion
────────────────────────────────────────

    Completion Queue entry on Machine A:
    {
        work_request_id: (from step 1),
        status: SUCCESS,
        bytes_transferred: 18
    }

    Machine A's application polls CQ and sees: "Write completed ✓"
```

---

## 📐 Worked Example: RDMA Send Using DDP

Now let's trace an RDMA Send (uses untagged buffers):

```
Scenario:
═════════
Machine A wants to send a 100-byte control message to Machine B.
Machine B has pre-posted receive buffers on Queue Number 0.


Step 1: Machine B pre-posts receive buffers
────────────────────────────────────────────

    Receive Queue (QN=0):
    ┌──────────────────────────────────────────┐
    │ WR 0: buffer=0x7F00001000, length=256    │ ← MSN 0 will use this
    │ WR 1: buffer=0x7F00002000, length=256    │
    │ WR 2: buffer=0x7F00003000, length=256    │
    └──────────────────────────────────────────┘


Step 2: Machine A posts an RDMA Send Work Request
──────────────────────────────────────────────────

    Work Request:
    {
        operation:    RDMA_SEND,
        local_buffer: pointer to 100-byte message,
        length:       100
    }
    (Note: No remote address needed! Receiver handles that.)


Step 3: DDP creates an Untagged segment
───────────────────────────────────────

    DDP Untagged Segment:
    ┌────────────────────────────────────────────────────┐
    │ Byte 0:    T=0, L=1, Rsvd=0, DV=01               │
    │ Bytes 1-3: RsvdULP = RDMA_SEND opcode             │
    │ Bytes 4-7: QN = 0 (queue number)                   │
    │ Bytes 8-11: MSN = 0 (message sequence number)      │
    │ Bytes 12-15: MO = 0 (message offset)               │
    │ Bytes 16-115: [100 bytes of message data]           │
    └────────────────────────────────────────────────────┘


Step 4: Machine B's NIC receives
─────────────────────────────────

    NIC reads header:
        T=0 → Untagged buffer model
        QN=0, MSN=0, MO=0

    NIC looks at Receive Queue 0:
        Next posted buffer for MSN 0 = WR 0 at 0x7F00001000

    NIC places 100 bytes at: 0x7F00001000 + MO(0) = 0x7F00001000

    L=1 → Message complete!

    NIC generates Completion Queue entry on Machine B:
    {
        work_request_id: (WR 0),
        status: SUCCESS,
        bytes_received: 100
    }

    Machine B's app polls CQ: "Received 100 bytes in buffer 0 ✓"
```

---

## ⚠️ Error Handling in DDP

DDP defines several error conditions. When an error occurs, the DDP layer reports it to the upper layer (RDMAP), which typically tears down the connection.

```
┌──────────────────────┬────────────────────────────────────────────┐
│ Error                │ What Happened                              │
├──────────────────────┼────────────────────────────────────────────┤
│ Invalid STag         │ The STag in the segment doesn't exist or   │
│                      │ has been invalidated/deregistered           │
├──────────────────────┼────────────────────────────────────────────┤
│ Base/Bounds          │ offset + length exceeds the registered     │
│ Violation            │ memory region size                          │
├──────────────────────┼────────────────────────────────────────────┤
│ Access Rights        │ Operation not permitted (e.g., trying to   │
│ Violation            │ write to a read-only region)                │
├──────────────────────┼────────────────────────────────────────────┤
│ Invalid DDP Version  │ DV field contains an unsupported version   │
├──────────────────────┼────────────────────────────────────────────┤
│ Invalid QN           │ Queue number doesn't exist on receiver     │
├──────────────────────┼────────────────────────────────────────────┤
│ No Receive Buffer    │ Untagged message arrived but no receive    │
│                      │ buffer was posted (queue empty)             │
├──────────────────────┼────────────────────────────────────────────┤
│ Tagged Offset Wrap   │ TO + payload length wraps past 2^64       │
└──────────────────────┴────────────────────────────────────────────┘

Error Severity:
───────────────
Most DDP errors are FATAL for the connection (called "stream errors").
The connection is torn down because continued operation could corrupt
memory or violate security guarantees.

Some are "untagged queue" scoped — only affecting one queue, not the
entire connection, but most implementations treat them as fatal too.
```

---

## 🔗 DDP and MPA — Why Framing Matters

TCP is a **byte stream** — it has no concept of message boundaries. But DDP needs to know where each segment starts and ends. That's MPA's job.

```
The Problem:
════════════
TCP delivers bytes continuously. Where does one DDP segment end
and the next begin?

    TCP byte stream: ...AAAAAAAABBBBBBBBCCCCCCCC...
                      ↑         ↑         ↑
                      Where do segments start??


MPA's Solution:
═══════════════
MPA wraps each DDP segment in a frame with a length header:

    ┌───────┬──────────────────────────┬─────────┐
    │ FPDU  │                          │         │
    │Length │    DDP Segment (N bytes)  │  CRC-32 │
    │(2B)   │                          │  (4B)   │
    └───────┴──────────────────────────┴─────────┘
    │◄──────── MPA Frame (FPDU) ─────────────────►│

    FPDU = Framed Protocol Data Unit

Multiple MPA frames in TCP stream:
    ┌──────────────────┬──────────────────┬──────────────────┐
    │ MPA Frame 1      │ MPA Frame 2      │ MPA Frame 3      │
    │ [len|DDP seg|crc]│ [len|DDP seg|crc]│ [len|DDP seg|crc]│
    └──────────────────┴──────────────────┴──────────────────┘
    │◄───────────── TCP byte stream ─────────────────────────►│


MPA also adds Markers (optional):
══════════════════════════════════
Every 512 bytes in the TCP stream, MPA can insert a 4-byte marker
that points back to the start of the current FPDU.

    Byte 0        Byte 512        Byte 1024
    │              │                │
    ▼              ▼                ▼
    [FPDU data...][MKR][...data...][MKR][...data...]
                   │                │
                   └── "FPDU starts └── "FPDU starts
                       at byte 480"     at byte 990"

Why? If the NIC loses sync (TCP retransmission, etc.), markers let
it find the next FPDU boundary quickly — critical for hardware
implementations that can't afford to rescan the entire stream.
```

---

## 🔀 Out-of-Order Placement — DDP's Hidden Superpower

One of DDP's most important properties is that **segments don't need to arrive in order** for data to be placed correctly.

```
Traditional TCP Receive:
═══════════════════════
If packets arrive out of order, TCP must buffer them and wait:

    Packet 3 arrives first  → buffer it, wait for 1 and 2
    Packet 1 arrives        → buffer it, wait for 2
    Packet 2 arrives        → NOW deliver all three in order
    = Extra buffering + latency


DDP over TCP:
═════════════
Even though TCP delivers in order, in some RDMA implementations
(especially with hardware-offloaded TCP), the NIC can process
segments optimistically:

    Segment 3: {STag=X, TO=2000} [data_C]  → Place at offset 2000 ✓
    Segment 1: {STag=X, TO=0}    [data_A]  → Place at offset 0    ✓
    Segment 2: {STag=X, TO=1000} [data_B]  → Place at offset 1000 ✓

Each segment is self-describing, so the NIC can place it immediately
without waiting for the previous segment. Only the "message complete"
notification (L=1) needs to be delivered in message order.


Why This Matters for Performance:
═════════════════════════════════
    Traditional: [wait][wait][place all three]  → high latency
    DDP:         [place][place][place]           → low latency

This is especially important for large RDMA transfers split
across many segments — each piece lands as soon as it arrives.
```

---

## 📝 Summary

```
┌──────────────────────────────────────────────────────────────┐
│                  DDP — Key Takeaways                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. DDP embeds a "delivery address" in every data segment    │
│     so the NIC knows exactly where to place each byte.       │
│                                                              │
│  2. Tagged buffers (RDMA Write/Read): sender specifies       │
│     STag + offset. Like mailing to an exact room number.     │
│                                                              │
│  3. Untagged buffers (RDMA Send): receiver pre-posts         │
│     buffers. Like a post office pickup counter.              │
│                                                              │
│  4. Large messages are split into segments, each carrying    │
│     its own placement info — no reassembly needed.           │
│                                                              │
│  5. MPA adds framing (length + CRC) to TCP's byte stream    │
│     so the NIC can find DDP segment boundaries.              │
│                                                              │
│  6. Self-describing segments enable out-of-order placement   │
│     — each piece lands as soon as it arrives.                │
│                                                              │
│  7. Security: STag validation, bounds checking, and access   │
│     permissions protect against unauthorized memory access.  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📖 References

- [RFC 5040 — Direct Data Placement over Reliable Transports](https://www.rfc-editor.org/rfc/rfc5040)
- [RFC 5041 — DDP/RDMAP Security](https://www.rfc-editor.org/rfc/rfc5041)
- [RFC 5044 — MPA Framing for TCP](https://www.rfc-editor.org/rfc/rfc5044)
- [RFC 6581 — Enhanced RDMA Connection Establishment](https://www.rfc-editor.org/rfc/rfc6581)

---

> *Every byte knows where it belongs.* 📬
