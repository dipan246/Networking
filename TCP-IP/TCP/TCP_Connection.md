# TCP Connection — Deep Dive (RFC 9293)

> **Reference**: [RFC 9293 — Transmission Control Protocol](https://www.ietf.org/rfc/rfc9293.html) (Obsoletes RFC 793)

TCP is a **reliable, in-order, byte-stream** transport protocol. A connection is identified by a 4-tuple: `(src IP, src port, dst IP, dst port)`.

---

## Table of Contents

1. [TCP vs UDP — When to Use What](#1-tcp-vs-udp--when-to-use-what)
2. [TCP Header Format](#2-tcp-header-format)
3. [Connection Establishment — 3-Way Handshake](#3-connection-establishment--3-way-handshake)
4. [Data Transfer — Worked Example](#4-data-transfer--worked-example)
5. [Flow Control — Sliding Window](#5-flow-control--sliding-window)
6. [Retransmission — Handling Packet Loss](#6-retransmission--handling-packet-loss)
7. [Connection Termination — 4-Way Handshake](#7-connection-termination--4-way-handshake)
8. [TCP State Machine (All 11 States)](#8-tcp-state-machine-all-11-states)
9. [Special Scenarios](#9-special-scenarios)
10. [Key Reliability Mechanisms](#10-key-reliability-mechanisms)
11. [ACK Deep Dive — Cumulative ACKs, Delayed ACKs & When TCP Sends ACKs](#11-ack-deep-dive--cumulative-acks-delayed-acks--when-tcp-sends-acks)
12. [SACK Deep Dive — Selective Acknowledgment](#12-sack-deep-dive--selective-acknowledgment)
13. [Congestion Control Deep Dive — Slow Start, AIMD & CUBIC](#13-congestion-control-deep-dive--slow-start-aimd--cubic)
14. [Key Formulas — Cheat Sheet](#14-key-formulas--cheat-sheet)
15. [References](#15-references)

---

## 1. TCP vs UDP -- When to Use What

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake) | Connectionless |
| **Reliability** | Guaranteed delivery with ACKs and retransmission | Best-effort, no guarantees |
| **Ordering** | In-order delivery via sequence numbers | No ordering |
| **Flow Control** | Yes (sliding window) | No |
| **Congestion Control** | Yes (slow start, AIMD) | No |
| **Header Size** | 20-60 bytes | 8 bytes |
| **Speed** | Slower (overhead) | Faster (minimal overhead) |
| **Use Cases** | HTTP, SSH, FTP, SMTP, databases | DNS, VoIP, video streaming, gaming |

**Rule of thumb**: Use TCP when you need reliability; use UDP when you need speed and can tolerate loss.

---

## 2. TCP Header Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| DOffset | Rsrvd |C|E|U|A|P|R|S|F|            Window           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Key Header Fields

| Field | Bits | Purpose |
|-------|------|---------|
| **Source Port** | 16 | Sender's port number |
| **Destination Port** | 16 | Receiver's port number |
| **Sequence Number** | 32 | Byte position of first data byte in this segment |
| **Acknowledgment Number** | 32 | Next byte the sender expects to receive |
| **Data Offset** | 4 | Length of TCP header in 32-bit words (min 5 = 20 bytes) |
| **Flags** | 8 | Control bits: CWR, ECE, URG, ACK, PSH, RST, SYN, FIN |
| **Window Size** | 16 | Receiver's available buffer space in bytes (flow control) |
| **Checksum** | 16 | Error detection covering header + data + pseudo-header |
| **Urgent Pointer** | 16 | Offset to urgent data (only valid when URG flag is set) |

### Control Flags

| Flag | Meaning | When Used |
|------|---------|-----------|
| **SYN** | Synchronize sequence numbers | Connection setup |
| **ACK** | Acknowledgment field is valid | Almost every segment after SYN |
| **FIN** | No more data from sender | Connection teardown |
| **RST** | Reset -- abort immediately | Error / rejected connection |
| **PSH** | Push data to application now | Interactive data (e.g., keystrokes) |
| **URG** | Urgent pointer is significant | Out-of-band data (rarely used) |
| **ECE** | ECN-Echo | Congestion notification |
| **CWR** | Congestion Window Reduced | Response to ECE |

> **Note**: SYN and FIN each consume **1 sequence number**, even though they carry no user data.

---

## 3. Connection Establishment -- 3-Way Handshake

**Example**: Client `192.168.1.10:54321` connects to Server `10.0.0.1:80`

```
  Client (CLOSED)                          Server (LISTEN)
      |                                        |
      |  1. SYN  seq=100                       |
      |--------------------------------------->|
      |     (Client -> SYN-SENT)               |  (Server -> SYN-RECEIVED)
      |                                        |
      |  2. SYN+ACK  seq=300, ack=101          |
      |<---------------------------------------|
      |                                        |
      |  3. ACK  seq=101, ack=301              |
      |--------------------------------------->|
      |     (Client -> ESTABLISHED)            |  (Server -> ESTABLISHED)
```

### Step-by-Step Breakdown

| Step | Sender | Flags | Seq | Ack | What It Means |
|------|--------|-------|-----|-----|---------------|
| 1 | Client | SYN | 100 (ISN) | -- | "I want to connect. My starting sequence is 100." |
| 2 | Server | SYN+ACK | 300 (ISN) | 101 | "OK. My starting sequence is 300. I expect your byte 101 next." |
| 3 | Client | ACK | 101 | 301 | "Got it. I expect your byte 301 next. Connection ready." |

### Why exactly 3 steps?

- **Step 1**: Client proves it can send.
- **Step 2**: Server proves it can receive AND send. Also synchronizes its own ISN.
- **Step 3**: Client proves it can receive. Without this, the server can't confirm the client got its ISN.

**ISN Selection**: Each ISN is chosen pseudo-randomly (RFC 9293 Section 3.4.1) to prevent old duplicate segments from prior connections being accepted.

---

## 4. Data Transfer -- Worked Example

**Scenario**: After handshake, Client sends **20 bytes** (2 x 10B blocks), Server replies with **30 bytes** (3 x 10B blocks).

- Client ISN = 100, Server ISN = 300
- First data byte from client = seq 101, first data byte from server = seq 301

```
  Client                                              Server
     |                                                    |
     |  [1] SYN  seq=100                                  |
     |--------------------------------------------------->|
     |                                                    |
     |  [2] SYN+ACK  seq=300, ack=101                     |
     |<---------------------------------------------------|
     |                                                    |
     |  [3] ACK  seq=101, ack=301                         |
     |--------------------------------------------------->|
     |            ~~~ ESTABLISHED both sides ~~~           |
     |                                                    |
     |  [4] PSH+ACK seq=101, ack=301, data[10B] "Block1" |
     |--------------------------------------------------->|
     |                                                    |
     |  [5] PSH+ACK seq=111, ack=301, data[10B] "Block2" |
     |--------------------------------------------------->|
     |                                                    |
     |  [6] ACK  seq=301, ack=121                         |
     |<---------------------------------------------------|  (ACKs all 20B from client)
     |                                                    |
     |  [7] PSH+ACK seq=301, ack=121, data[10B] "Reply1" |
     |<---------------------------------------------------|
     |                                                    |
     |  [8] PSH+ACK seq=311, ack=121, data[10B] "Reply2" |
     |<---------------------------------------------------|
     |                                                    |
     |  [9] PSH+ACK seq=321, ack=121, data[10B] "Reply3" |
     |<---------------------------------------------------|
     |                                                    |
     |  [10] ACK  seq=121, ack=331                        |
     |--------------------------------------------------->|  (ACKs all 30B from server)
     |                                                    |
     |            ~~~ DATA TRANSFER COMPLETE ~~~           |
     |                                                    |
     |  [11] FIN+ACK  seq=121, ack=331                    |
     |--------------------------------------------------->|  (Client closes)
     |                                                    |
     |  [12] ACK  seq=331, ack=122                        |
     |<---------------------------------------------------|
     |                                                    |
     |  [13] FIN+ACK  seq=331, ack=122                    |
     |<---------------------------------------------------|  (Server closes)
     |                                                    |
     |  [14] ACK  seq=122, ack=332                        |
     |--------------------------------------------------->|
     |                                                    |
     |     (Client -> TIME-WAIT, 2xMSL -> CLOSED)         |
     |                        (Server -> CLOSED)           |
```

### Byte-Level Block Diagram

Each box represents **one byte** and shows its sequence number.

**Pkt 4 — Client sends "Block1" (10 bytes, seq=101):**

```
seq=101                                              seq=111 (next)
  |                                                    |
  v                                                    v
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
| 101 | 102 | 103 | 104 | 105 | 106 | 107 | 108 | 109 | 110 |
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
  B[0]  B[1]  B[2]  B[3]  B[4]  B[5]  B[6]  B[7]  B[8]  B[9]
|<---------------------- 10 bytes ---------------------->|
```

**Pkt 5 — Client sends "Block2" (10 bytes, seq=111):**

```
seq=111                                              seq=121 (next)
  |                                                    |
  v                                                    v
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
| 111 | 112 | 113 | 114 | 115 | 116 | 117 | 118 | 119 | 120 |
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
  B[0]  B[1]  B[2]  B[3]  B[4]  B[5]  B[6]  B[7]  B[8]  B[9]
|<---------------------- 10 bytes ---------------------->|
```

**Pkt 6 — Server ACKs with ack=121** (last byte was 120, next expected = 121):

```
CLIENT'S BYTE STREAM (complete picture):

  SYN                   Block 1                            Block 2                         FIN
   |                      |                                  |                              |
+-----+ +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+ +-----+
| 100 | | 101 | 102 | 103 | 104 | 105 | 106 | 107 | 108 | 109 | 110 | 111 | 112 | 113 | 114 | 115 | 116 | 117 | 118 | 119 | 120 | | 121 |
+-----+ +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+ +-----+
  SYN     |<------------- Pkt 4 (10B) ------------>|  |<------------- Pkt 5 (10B) ------------->|    FIN
(1 seq)                                                                                           (1 seq)
          Server ack=121: "I got bytes 101-120, send me 121 next"
```

**Pkt 7, 8, 9 — Server sends "Reply1", "Reply2", "Reply3" (3 x 10 bytes):**

```
SERVER'S BYTE STREAM:

  SYN                   Reply 1                            Reply 2                            Reply 3                         FIN
   |                      |                                  |                                  |                              |
+-----+ +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+ +-----+
| 300 | | 301 | 302 | 303 | 304 | 305 | 306 | 307 | 308 | 309 | 310 | 311 | 312 | 313 | 314 | 315 | 316 | 317 | 318 | 319 | 320 | 321 | 322 | 323 | 324 | 325 | 326 | 327 | 328 | 329 | 330 | | 331 |
+-----+ +-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+ +-----+
  SYN     |<----------- Pkt 7 (10B) ---------->|  |<----------- Pkt 8 (10B) ---------->|  |<----------- Pkt 9 (10B) ---------->|    FIN
(1 seq)                                                                                                                          (1 seq)
          Client ack=331: "I got bytes 301-330, send me 331 next"
```

### How ack=121 Is Calculated (Not 122)

```
  first data byte:   101
  total bytes sent:   20
  last byte sent:    120  (= 101 + 20 - 1)
  next expected:     121  (= 101 + 20)  <-- this becomes the ACK
```

### Sequence Number Tracker (All 14 Packets)

| Pkt | Dir | Flags | seq | ack | Data | Explanation |
|-----|-----|-------|-----|-----|------|-------------|
| 1 | C->S | SYN | 100 | -- | 0B | Client ISN = 100 |
| 2 | S->C | SYN+ACK | 300 | 101 | 0B | Server ISN = 300; expects client byte 101 |
| 3 | C->S | ACK | 101 | 301 | 0B | Handshake complete; expects server byte 301 |
| 4 | C->S | PSH+ACK | 101 | 301 | 10B | Client sends bytes 101-110 |
| 5 | C->S | PSH+ACK | 111 | 301 | 10B | Client sends bytes 111-120 |
| 6 | S->C | ACK | 301 | 121 | 0B | Server ACKs all 20 bytes: "next send byte 121" |
| 7 | S->C | PSH+ACK | 301 | 121 | 10B | Server sends bytes 301-310 |
| 8 | S->C | PSH+ACK | 311 | 121 | 10B | Server sends bytes 311-320 |
| 9 | S->C | PSH+ACK | 321 | 121 | 10B | Server sends bytes 321-330 |
| 10 | C->S | ACK | 121 | 331 | 0B | Client ACKs all 30 bytes: "next send byte 331" |
| 11 | C->S | FIN+ACK | 121 | 331 | 0B | Client requests close (FIN uses 1 seq) |
| 12 | S->C | ACK | 331 | 122 | 0B | Server ACKs the FIN (121 + 1 = 122) |
| 13 | S->C | FIN+ACK | 331 | 122 | 0B | Server also closes (FIN uses 1 seq) |
| 14 | C->S | ACK | 122 | 332 | 0B | Client ACKs server's FIN (331 + 1 = 332) |

### Key Rules

```
next_seq        = current_seq + bytes_sent_in_this_segment
ack_number      = next byte you EXPECT from the other side
                = last_byte_received + 1
```

### Golden Rule: My Sequence Number = Peer's Last ACK

> **Your `seq` always equals the last `ack` your peer sent you.**

The peer's ACK says *"send me byte X next"* — so your next segment uses `seq = X`.

**Proof from the example above:**

| Peer sent me | Then I send | Match? |
|--------------|-------------|--------|
| Server `ack=101` (pkt 2) | Client `seq=101` (pkt 3) | Yes |
| Client `ack=301` (pkt 3) | Server `seq=301` (pkt 6) | Yes |
| Server `ack=121` (pkt 6) | Client `seq=121` (pkt 10) | Yes |
| Client `ack=331` (pkt 10) | Server `seq=331` (pkt 12) | Yes |

> **Exception**: During **retransmission**, the sender reuses an older seq (the lost byte's position), not the peer's latest ack.

---

## 5. Flow Control -- Sliding Window

TCP uses a **sliding window** to prevent the sender from overwhelming the receiver.

### How It Works

1. Receiver advertises a **window size** (in bytes) in every ACK.
2. Sender can only have `window_size` bytes of unacknowledged data in flight.
3. As the receiver processes data, the window "slides" forward.

### Example: Window Size = 20 bytes

```
  Sender                                              Receiver
     |                                                    |
     |   Window = 20B advertised                          |
     |                                                    |
     |  seq=1, 10B data  -------------------------------->|  (10B in flight, 10B window left)
     |  seq=11, 10B data -------------------------------->|  (20B in flight, window full!)
     |                                                    |
     |  ** SENDER MUST WAIT -- window full **             |
     |                                                    |
     |  <--- ACK=11, window=20 ---------------------------|  (processed 10B, window reopens)
     |                                                    |
     |  seq=21, 10B data -------------------------------->|  (can send again)
     |                                                    |
     |  <--- ACK=21, window=0 ----------------------------|  (receiver buffer full!)
     |                                                    |
     |  ** ZERO WINDOW -- sender probes periodically **   |
     |                                                    |
     |  <--- ACK=21, window=20 ---------------------------|  (buffer freed, window reopens)
```

### Zero Window

When receiver advertises `window=0`, the sender enters a **persist** state and sends tiny **window probe** segments periodically until the window reopens.

---

## 6. Retransmission -- Handling Packet Loss

When a segment is lost, TCP detects it and retransmits.

### Example: Packet 2 Lost

```
  Sender                                              Receiver
     |                                                    |
     |  [1] seq=1, 10B   -------------------------------->|  ACK=11
     |  [2] seq=11, 10B  --------X  (LOST!)              |
     |  [3] seq=21, 10B  -------------------------------->|  ACK=11 (duplicate!)
     |  [4] seq=31, 10B  -------------------------------->|  ACK=11 (duplicate!)
     |  [5] seq=41, 10B  -------------------------------->|  ACK=11 (duplicate!)
     |                                                    |
     |  3 duplicate ACKs -> FAST RETRANSMIT               |
     |                                                    |
     |  [2'] seq=11, 10B (retransmit) ------------------->|  ACK=51 (cumulative!)
     |                                                    |
     |  Receiver had buffered [3],[4],[5] out of order.   |
     |  Once gap is filled, ACKs everything up to 51.     |
```

### Two Retransmission Triggers

| Trigger | How It Works |
|---------|-------------|
| **Timeout (RTO)** | If no ACK arrives within the Retransmission Timeout, resend. RTO is computed dynamically from measured RTT. |
| **Fast Retransmit** | If 3 duplicate ACKs arrive for the same sequence number, assume that segment was lost and retransmit immediately (without waiting for timeout). |

---

## 7. Connection Termination -- 4-Way Handshake

Either side can initiate close. TCP supports **half-close** (one direction closes while the other keeps sending).

```
  Client (ESTABLISHED)                     Server (ESTABLISHED)
      |                                        |
      |  1. FIN+ACK  seq=121, ack=331         |
      |--------------------------------------->|
      |     (Client -> FIN-WAIT-1)             |  (Server -> CLOSE-WAIT)
      |                                        |
      |  2. ACK  seq=331, ack=122             |
      |<---------------------------------------|
      |     (Client -> FIN-WAIT-2)             |  Server can still send data here!
      |                                        |
      |  3. FIN+ACK  seq=331, ack=122         |
      |<---------------------------------------|  (Server -> LAST-ACK)
      |                                        |
      |  4. ACK  seq=122, ack=332             |
      |--------------------------------------->|
      |     (Client -> TIME-WAIT)              |  (Server -> CLOSED)
      |                                        |
      |     ... waits 2xMSL (typically 60s)...|
      |     (Client -> CLOSED)                 |
```

### Why TIME-WAIT?

- **Ensures the final ACK arrives**: If the last ACK (step 4) is lost, the server will retransmit its FIN, and the client needs to be around to re-ACK it.
- **Lets old segments expire**: Prevents stale segments from a prior connection being confused with a new connection on the same 4-tuple.
- **Duration**: 2 x MSL (Maximum Segment Lifetime). Typically MSL = 30s, so TIME-WAIT = 60s.

### Half-Close

Between steps 2 and 3, the server is in CLOSE-WAIT and **can still send data**. This is useful when the client says "I'm done sending" but the server still has a response to deliver.

---

## 8. TCP State Machine (All 11 States)

```
                              +---------+
                              |  CLOSED |
                              +---------+
                        passive OPEN |    active OPEN: send SYN
                                     v               |
                              +---------+             v
                              |  LISTEN |       +-----------+
                              +---------+       | SYN-SENT  |
                         rcv SYN: |              +-----------+
                        send SYN+ACK            rcv SYN+ACK: |
                                  v             send ACK      v
                          +----------+        +-----------+
                          |SYN-RCVD  |------->|ESTABLISHED|
                          +----------+ rcv ACK+-----------+
                                              CLOSE: send FIN
                                                     |
                          +----------+               v
              CLOSE:      |CLOSE-WAIT|        +-----------+
              send FIN    +----------+        |FIN-WAIT-1 |
                   |       rcv FIN:           +-----------+
                   v       send ACK            rcv ACK |
             +----------+                            v
             | LAST-ACK |                    +-----------+
             +----------+                   |FIN-WAIT-2 |
              rcv ACK: |                    +-----------+
              -> CLOSED |                    rcv FIN:    |
                        v                    send ACK    v
                   +---------+             +-----------+
                   |  CLOSED |             | TIME-WAIT | -> waits 2xMSL -> CLOSED
                   +---------+             +-----------+
```

### State Reference Table

| # | State | Role | Description |
|---|-------|------|-------------|
| 1 | **CLOSED** | Both | No connection exists |
| 2 | **LISTEN** | Server | Waiting for incoming SYN |
| 3 | **SYN-SENT** | Client | Sent SYN, waiting for SYN+ACK |
| 4 | **SYN-RECEIVED** | Server | Received SYN, sent SYN+ACK, waiting for ACK |
| 5 | **ESTABLISHED** | Both | Connection open, data flows bidirectionally |
| 6 | **FIN-WAIT-1** | Closer | Sent FIN, waiting for ACK or FIN |
| 7 | **FIN-WAIT-2** | Closer | Our FIN was ACKed, waiting for peer's FIN |
| 8 | **CLOSE-WAIT** | Peer | Received FIN, waiting for local app to close |
| 9 | **CLOSING** | Both | Both sides sent FIN simultaneously |
| 10 | **LAST-ACK** | Peer | Sent FIN (after receiving FIN), waiting for final ACK |
| 11 | **TIME-WAIT** | Closer | Waiting 2xMSL before fully closing |

---

## 9. Special Scenarios

### 9.1 Simultaneous Open (Both Sides SYN at Same Time)

Rare, but TCP handles it. Both sides transition through SYN-SENT -> SYN-RECEIVED -> ESTABLISHED.

```
  Host A                                   Host B
    |  SYN seq=100                           |
    |--------------------------------------->|
    |                    SYN seq=200          |
    |<---------------------------------------|
    |  (both -> SYN-SENT -> SYN-RECEIVED)    |
    |                                        |
    |  SYN+ACK seq=100, ack=201             |
    |--------------------------------------->|
    |           SYN+ACK seq=200, ack=101     |
    |<---------------------------------------|
    |  (both -> ESTABLISHED)                 |
```

### 9.2 RST (Reset) -- Immediate Abort

Sent when:
- A segment arrives for a **non-existent connection**
- An application wants to **abort** (not gracefully close)
- A SYN arrives on a port with **no listener**

The receiver immediately transitions to **CLOSED** -- no handshake needed.

### 9.3 SYN Flood Attack

An attacker sends many SYNs without completing the handshake, filling the server's SYN queue. Mitigations:
- **SYN Cookies** (RFC 4987): Server encodes state in the ISN, avoiding queue allocation
- **SYN Queue tuning**: Increase backlog size
- **Firewall rate limiting**: Drop excessive SYNs from single sources

---

## 10. Key Reliability Mechanisms

| Mechanism | Purpose | RFC |
|-----------|---------|-----|
| **Sequence Numbers** | Order bytes, detect duplicates, detect loss | 9293 |
| **Checksums** | Detect bit errors in transit | 9293 |
| **Cumulative ACKs** | Confirm receipt; one ACK can cover multiple segments | 9293 |
| **Selective ACKs (SACK)** | Report non-contiguous blocks received (avoids unnecessary retransmit) | 2018 |
| **Retransmission Timer (RTO)** | Resend lost segments; dynamically computed from SRTT | 6298 |
| **Fast Retransmit** | 3 duplicate ACKs trigger immediate retransmit | 5681 |
| **Sliding Window** | Flow control -- receiver limits sender's rate | 9293 |
| **Congestion Control** | Sender limits rate to avoid network overload | 5681 |
| **ECN** | Routers signal congestion without dropping packets | 3168 |

---

## 11. ACK Deep Dive -- Cumulative ACKs, Delayed ACKs & When TCP Sends ACKs

### 11.1 Cumulative ACKs — Always On

TCP ACKs are **always cumulative** — this is fundamental to the protocol (RFC 793/9293), not an optional mode.

The **Acknowledgment Number** in the TCP header means:

> "I have successfully received all bytes up to N-1. Send me byte N next."

A single ACK can confirm multiple segments at once:

```
  Sender                                              Receiver
     |                                                    |
     |  [1] seq=1, 10B   -------------------------------->|
     |  [2] seq=11, 10B  -------------------------------->|
     |  [3] seq=21, 10B  -------------------------------->|
     |  [4] seq=31, 10B  -------------------------------->|
     |  [5] seq=41, 10B  -------------------------------->|
     |                                                    |
     |  <----------  ACK=51  -----------------------------|  (one ACK covers ALL 5 segments!)
     |                                                    |
```

ACK=51 means "I received bytes 1-50, send me byte 51 next." The receiver doesn't need to ACK each segment individually — one cumulative ACK covers everything received so far.

### 11.2 Delayed ACK — RFC 1122

Sending an ACK for every single segment wastes bandwidth. **Delayed ACK** (RFC 1122) reduces overhead by batching:

| Rule | RFC | Requirement Level |
|------|-----|-------------------|
| ACK within **500ms** of receiving a segment | RFC 1122 | **MUST** |
| ACK at least every **2nd full-size segment** | RFC 1122 | **SHOULD** |
| Piggyback ACK with data if available | RFC 1122 | **SHOULD** |

> **In practice**, most implementations (Linux, Windows) use a **~200ms** delayed ACK timer — more responsive than the 500ms maximum allowed by RFC 1122.

### 11.3 When Does TCP Send an ACK?

| # | Trigger | ACK Behavior | RFC | Why |
|---|---------|-------------|-----|-----|
| 1 | **Every 2nd full-size segment** | ACK immediately | 1122 | Delayed ACK default — don't let too many segments go unacknowledged |
| 2 | **Delayed ACK timer expires (~200ms)** | ACK fires on timeout | 1122 | Only 1 segment arrived; waited long enough |
| 3 | **Out-of-order segment arrives** | ACK immediately (duplicate) | 5681 | Signal a gap to trigger fast retransmit at the sender |
| 4 | **Gap-filling segment arrives** | ACK immediately | 5681 | Received the missing segment — update cumulative ACK to cover everything |
| 5 | **Receiver has data to send back** | Piggyback ACK with data | 1122 | Efficient — combine ACK + response in one segment (no extra packet) |
| 6 | **Window update needed** | ACK immediately | 9293 | Receiver freed buffer space — tell sender the new window size |

### 11.4 Worked Example — 5 In-Order Packets

```
  Sender                                              Receiver
     |                                                    |
     |  [1] seq=1, 1460B  ------------------------------->|  (starts delayed ACK timer)
     |  [2] seq=1461, 1460B ------------------------------>|  ACK=2921 (2nd segment → ACK now!)
     |  [3] seq=2921, 1460B ------------------------------>|  (starts delayed ACK timer)
     |  [4] seq=4381, 1460B ------------------------------>|  ACK=5841 (2nd segment → ACK now!)
     |  [5] seq=5841, 1460B ------------------------------>|  (starts delayed ACK timer)
     |                                                    |
     |         ... 200ms passes ...                       |
     |                                                    |
     |  <----------  ACK=7301  ----------------------------|  (timer fires → ACK for pkt 5)
     |                                                    |
```

**Result**: 3 ACKs for 5 packets (ACK every 2nd + timer for the odd one).

### 11.5 Out-of-Order Example — Packet 3 Lost

When a packet is lost, the receiver sends **immediate duplicate ACKs** for every subsequent packet — no delay allowed (RFC 5681):

```
  Sender                                              Receiver
     |                                                    |
     |  [1] seq=1, 1460B  ------------------------------->|  (starts delayed ACK timer)
     |  [2] seq=1461, 1460B ------------------------------>|  ACK=2921 (2nd segment)
     |  [3] seq=2921, 1460B  --------X  (LOST!)          |
     |  [4] seq=4381, 1460B ------------------------------>|  ACK=2921 (dup! out-of-order → immediate)
     |  [5] seq=5841, 1460B ------------------------------>|  ACK=2921 (dup! out-of-order → immediate)
     |                                                    |
     |  Sender sees duplicate ACKs → suspects loss        |
     |                                                    |
```

The cumulative ACK stays at 2921 because that's the next byte the receiver is missing. Even though packets 4 and 5 were received, the ACK number doesn't advance past the gap.

### 11.6 Piggybacking — Free ACKs

When the receiver has data to send back (e.g., an HTTP response), the ACK is included in the data segment for free — no separate ACK packet needed:

```
  Client                                              Server
     |                                                    |
     |  [1] PSH+ACK seq=1, data="GET /index.html"  ----->|
     |                                                    |
     |  <--- PSH+ACK seq=1, ack=18, data="<html>..."  ---|  (ACK piggybacked with response!)
     |                                                    |
```

**One packet carries both the ACK and the response data** — this is why the ACK flag is set on almost every TCP segment after the handshake.

---

## 12. SACK Deep Dive -- Selective Acknowledgment

### 12.1 The Problem SACK Solves

With only cumulative ACKs, the sender knows there's a gap but doesn't know exactly what the receiver has beyond the gap:

```
Without SACK — sender knows:     "Receiver is missing byte 2921"
With SACK    — sender knows:     "Receiver is missing byte 2921, but HAS 4381-7300"
```

Without SACK, the sender must either:
- **Wait for RTO** and retransmit everything from the gap, or
- **Retransmit conservatively** and potentially resend data already received

### 12.2 How SACK Works

SACK is a **TCP option** negotiated during the 3-way handshake:

1. **SYN**: Client includes `SACK-Permitted` option → "I support SACK"
2. **SYN+ACK**: Server includes `SACK-Permitted` option → "Me too"
3. **Data transfer**: Receiver includes `SACK blocks` in ACKs when there are gaps

A SACK block is a pair: `(left_edge, right_edge)` — meaning "I have bytes from left_edge to right_edge-1."

### 12.3 Worked Example — Packet 3 Lost, SACK Enabled

```
  Sender                                              Receiver
     |                                                    |
     |  [1] seq=1, 1460B  ------------------------------->|
     |  [2] seq=1461, 1460B ------------------------------>|  ACK=2921
     |  [3] seq=2921, 1460B  --------X  (LOST!)          |
     |  [4] seq=4381, 1460B ------------------------------>|  ACK=2921, SACK=[4381-5841]
     |  [5] seq=5841, 1460B ------------------------------>|  ACK=2921, SACK=[4381-7301]
     |                                                    |
     |  Sender knows: "Missing 2921-4380, has 4381-7300"  |
     |  Retransmits ONLY packet 3:                        |
     |                                                    |
     |  [3'] seq=2921, 1460B (retransmit) --------------->|  ACK=7301 (cumulative! gap filled)
     |                                                    |
```

**Without SACK**: Sender would retransmit packets 3, 4, AND 5 — wasting bandwidth on 4 and 5 which were already received.

**With SACK**: Sender retransmits **only packet 3** — saving 2920 bytes of unnecessary retransmission.

### 12.4 Multiple Gaps

SACK supports up to **3-4 SACK blocks** per ACK (limited by TCP option space — 40 bytes max). With multiple losses, the receiver reports all non-contiguous ranges:

```
  Received: [1-1460], [4381-5840], [8761-10220]
  Missing:  [1461-4380], [5841-8760]

  ACK=1461, SACK=[4381-5841][8761-10221]
  
  Sender retransmits: only the missing ranges (1461-4380, 5841-8760)
```

### 12.5 SACK vs No SACK — Comparison

| Scenario: Packets 1-5 sent, packet 3 lost | Without SACK | With SACK |
|--------------------------------------------|-------------|-----------|
| Receiver sends | ACK=2921, ACK=2921, ACK=2921 | ACK=2921 SACK[4381-5841], ACK=2921 SACK[4381-7301] |
| Sender retransmits | Packets 3, 4, 5 (3 segments) | Packet 3 only (1 segment) |
| Wasted bandwidth | 2 segments (4 and 5 already received) | None |
| Recovery time | Longer | Shorter |

### 12.6 Key RFCs

| RFC | Topic |
|-----|-------|
| [RFC 2018](https://www.ietf.org/rfc/rfc2018.html) | TCP SACK option definition |
| [RFC 2883](https://www.ietf.org/rfc/rfc2883.html) | D-SACK — Duplicate SACK (reports duplicate segments received) |
| [RFC 3517](https://www.ietf.org/rfc/rfc3517.html) | Conservative SACK-based loss recovery algorithm |

---

## 13. Congestion Control Deep Dive -- Slow Start, AIMD & CUBIC

### 13.1 Why Congestion Control?

Flow control (sliding window) prevents overwhelming the **receiver**. Congestion control prevents overwhelming the **network**. Without it, senders would flood the network, causing massive packet loss and collapse.

The sender maintains a **congestion window (cwnd)** — the maximum bytes it can have in flight, independent of the receiver's window:

```
effective_window = min(cwnd, receiver_window)
can_send         = effective_window - bytes_in_flight
```

### 13.2 The Four Phases

```
    cwnd
     ^
     |                          * * *
     |                      *           *       <- Congestion Avoidance (linear)
     |                   *                 *
     |                *                       *  <- Loss detected!
     |             *                          |
     |          *   <- Slow Start             |  cwnd cut (multiplicative decrease)
     |       *      (exponential)             |
     |    *                                   v
     | *                                   *
     |*                                 *      <- Recovery + Congestion Avoidance again
     +----------------------------------------> time
              ssthresh
```

#### Phase 1: Slow Start (Exponential Growth)

- **Initial cwnd** = 1 MSS (or 10 MSS per RFC 6928)
- **On each ACK**: cwnd += 1 MSS → effectively **doubles cwnd every RTT**
- **Until**: cwnd reaches `ssthresh` (slow start threshold), then switch to congestion avoidance

```
RTT 1: cwnd = 1 MSS  →  send 1 segment   →  1 ACK  →  cwnd = 2
RTT 2: cwnd = 2 MSS  →  send 2 segments  →  2 ACKs →  cwnd = 4
RTT 3: cwnd = 4 MSS  →  send 4 segments  →  4 ACKs →  cwnd = 8
RTT 4: cwnd = 8 MSS  →  send 8 segments  →  8 ACKs →  cwnd = 16
```

#### Phase 2: Congestion Avoidance (Linear Growth — AIMD)

Once cwnd ≥ ssthresh, growth slows to **Additive Increase**:

- **On each ACK**: cwnd += MSS × (MSS / cwnd) → approximately **+1 MSS per RTT**
- This is the "AI" in AIMD (Additive Increase, Multiplicative Decrease)

#### Phase 3: Loss Detection → Multiplicative Decrease

| Loss Signal | Response | New cwnd | New ssthresh |
|------------|----------|----------|-------------|
| **3 duplicate ACKs** (fast retransmit) | Mild reduction | cwnd / 2 | cwnd / 2 |
| **RTO timeout** | Severe reduction | 1 MSS | cwnd / 2 |

This is the "MD" in AIMD — **halve the window** on loss.

#### Phase 4: Fast Recovery (RFC 5681)

After fast retransmit (3 dup ACKs):
1. Set ssthresh = cwnd / 2
2. Set cwnd = ssthresh + 3 MSS (inflate for the 3 dup ACKs)
3. For each additional dup ACK: cwnd += 1 MSS
4. When new data is ACKed: cwnd = ssthresh (deflate, enter congestion avoidance)

### 13.3 Reno vs NewReno vs CUBIC

| Algorithm | Growth Pattern | Loss Response | Key Feature | RFC |
|-----------|---------------|---------------|-------------|-----|
| **Reno** | Linear (AIMD) | cwnd / 2 | Basic fast recovery — exits on first new ACK | 5681 |
| **NewReno** | Linear (AIMD) | cwnd / 2 | Improved fast recovery — handles multiple losses in one window | 6582 |
| **CUBIC** | Cubic function of time | cwnd × 0.7 (β=0.7) | RTT-fair, time-based growth curve | 8312 |

### 13.4 CUBIC — The Modern Default

CUBIC is the default congestion control in **Linux** (since 2.6.19) and **Windows** (since Windows 10). It models cwnd growth as a **cubic function of time** since the last congestion event.

#### The Cubic Curve

```
    cwnd
     ^
     |         Wmax ------>  * * * * * * * * * *
     |                   *                       *
     |                *         Plateau            *
     |             *       (cautious near Wmax)      *
     |          *                                      *
     |       *  Concave                     Convex       *
     |    *     (fast ramp)              (aggressive       *
     | *                                  probing)           *
     +---------------------------------------------------------> time
                        K (time to reach Wmax)
```

#### The Formula

```
W(t) = C × (t - K)³ + Wmax

Where:
  W(t)  = congestion window at time t
  C     = scaling constant (0.4)
  K     = ∛(Wmax × β / C)     (time to reach Wmax)
  Wmax  = window size before last loss
  β     = multiplicative decrease factor (0.7 for CUBIC, 0.5 for Reno)
  t     = time since last congestion event
```

#### Why CUBIC Is Better Than Reno

| Property | Reno (AIMD) | CUBIC |
|----------|------------|-------|
| **Growth basis** | ACK-clocked (per RTT) | Time-based (wall clock) |
| **RTT fairness** | Unfair — low RTT flows grow faster | Fair — growth is time-based, RTT doesn't matter |
| **High BDP networks** | Slow to fill large pipes | Fast — aggressive probing past Wmax |
| **Near Wmax** | Linear approach (may overshoot) | Cautious plateau (cubic flattens near Wmax) |
| **After loss** | cwnd / 2 (50% cut) | cwnd × 0.7 (30% cut — less aggressive) |

#### CUBIC in Action — Example

```
1. Connection running at cwnd = 100 MSS (this is Wmax)
2. Packet loss detected!
3. New cwnd = 100 × 0.7 = 70 MSS
4. CUBIC recovery:
   - Phase 1 (concave):  Fast growth from 70 toward 100 (far from Wmax)
   - Phase 2 (plateau):  Slow growth near 100 (cautious — don't cause another loss)
   - Phase 3 (convex):   Accelerating growth past 100 (probing for more capacity)
```

### 13.5 ECN — Congestion Without Drops

**Explicit Congestion Notification** (RFC 3168) lets routers signal congestion **without dropping packets**:

```
  Sender                    Router                    Receiver
     |  IP: ECT(0)            |                          |
     |----------------------->|  Queue filling...        |
     |                        |  Marks: ECT(0) → CE     |
     |                        |------------------------->|
     |                        |                          |
     |  <--- TCP: ECE flag ---|--------------------------|  (receiver echoes congestion)
     |                        |                          |
     |  TCP: CWR flag ------->|                          |  (sender reduces cwnd)
     |  (cwnd reduced)        |                          |
```

- **ECT** (ECN-Capable Transport): Sender marks packets as ECN-capable
- **CE** (Congestion Experienced): Router marks packets when queue is building
- **ECE** (ECN-Echo): Receiver tells sender "congestion happened"
- **CWR** (Congestion Window Reduced): Sender confirms it reduced cwnd

**Benefit**: Congestion signal arrives **before** packets are dropped — sender reduces rate proactively, avoiding loss and retransmission.

### 13.6 Key RFCs

| RFC | Topic |
|-----|-------|
| [RFC 5681](https://www.ietf.org/rfc/rfc5681.html) | TCP Congestion Control (slow start, AIMD, fast retransmit/recovery) |
| [RFC 6582](https://www.ietf.org/rfc/rfc6582.html) | NewReno — improved fast recovery for multiple losses |
| [RFC 6928](https://www.ietf.org/rfc/rfc6928.html) | Increasing TCP Initial Window to 10 MSS |
| [RFC 8312](https://www.ietf.org/rfc/rfc8312.html) | CUBIC congestion control algorithm |
| [RFC 3168](https://www.ietf.org/rfc/rfc3168.html) | Explicit Congestion Notification (ECN) |

---

## 14. Key Formulas -- Cheat Sheet

```
Sequence number math:
  next_seq        = current_seq + bytes_sent_in_this_segment
  ack_number      = last_byte_received + 1 (next byte expected)

Golden rule:
  my_seq          = peer's last ack to me
                    (peer said "send me byte X" -> I send with seq=X)
                    (Exception: retransmission uses the lost byte's seq)

SYN/FIN consume 1 sequence number:
  After SYN with seq=X  -> next seq = X + 1
  After FIN with seq=Y  -> peer ACKs with ack = Y + 1

Window:
  bytes_in_flight = SND.NXT - SND.UNA
  can_send        = min(receiver_window, congestion_window) - bytes_in_flight

RTO (Retransmission Timeout):
  SRTT    = (1 - alpha) * SRTT + alpha * RTT_sample     (alpha = 1/8)
  RTTVAR  = (1 - beta) * RTTVAR + beta * |SRTT - RTT|   (beta = 1/4)
  RTO     = SRTT + 4 * RTTVAR
```

---

## 15. References

| Document | Topic |
|----------|-------|
| [RFC 9293](https://www.ietf.org/rfc/rfc9293.html) | TCP specification (replaces RFC 793) |
| [RFC 1122](https://www.ietf.org/rfc/rfc1122.html) | Host Requirements — Delayed ACK rules |
| [RFC 5681](https://www.ietf.org/rfc/rfc5681.html) | TCP Congestion Control |
| [RFC 6298](https://www.ietf.org/rfc/rfc6298.html) | Computing TCP Retransmission Timer |
| [RFC 6582](https://www.ietf.org/rfc/rfc6582.html) | NewReno Fast Recovery |
| [RFC 6928](https://www.ietf.org/rfc/rfc6928.html) | Increasing TCP Initial Window |
| [RFC 7323](https://www.ietf.org/rfc/rfc7323.html) | TCP Extensions for High Performance |
| [RFC 2018](https://www.ietf.org/rfc/rfc2018.html) | TCP Selective Acknowledgment (SACK) |
| [RFC 2883](https://www.ietf.org/rfc/rfc2883.html) | D-SACK — Duplicate SACK |
| [RFC 3517](https://www.ietf.org/rfc/rfc3517.html) | Conservative SACK-based Loss Recovery |
| [RFC 8312](https://www.ietf.org/rfc/rfc8312.html) | CUBIC Congestion Control |
| [RFC 3168](https://www.ietf.org/rfc/rfc3168.html) | Explicit Congestion Notification (ECN) |
| [RFC 4987](https://www.ietf.org/rfc/rfc4987.html) | TCP SYN Flooding Attacks and Mitigations |
