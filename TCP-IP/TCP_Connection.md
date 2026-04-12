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
11. [Key Formulas — Cheat Sheet](#11-key-formulas--cheat-sheet)
12. [References](#12-references)

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

## 11. Key Formulas -- Cheat Sheet

```
Sequence number math:
  next_seq        = current_seq + bytes_sent_in_this_segment
  ack_number      = last_byte_received + 1 (next byte expected)

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

## 12. References

| Document | Topic |
|----------|-------|
| [RFC 9293](https://www.ietf.org/rfc/rfc9293.html) | TCP specification (replaces RFC 793) |
| [RFC 5681](https://www.ietf.org/rfc/rfc5681.html) | TCP Congestion Control |
| [RFC 6298](https://www.ietf.org/rfc/rfc6298.html) | Computing TCP Retransmission Timer |
| [RFC 7323](https://www.ietf.org/rfc/rfc7323.html) | TCP Extensions for High Performance |
| [RFC 2018](https://www.ietf.org/rfc/rfc2018.html) | TCP Selective Acknowledgment (SACK) |
| [RFC 3168](https://www.ietf.org/rfc/rfc3168.html) | Explicit Congestion Notification (ECN) |
| [RFC 4987](https://www.ietf.org/rfc/rfc4987.html) | TCP SYN Flooding Attacks and Mitigations |
