# TCP Connection — Deep Dive (RFC 9293)

> **Reference**: [RFC 9293 — Transmission Control Protocol](https://www.ietf.org/rfc/rfc9293.html)

TCP is a **reliable, in-order, byte-stream** transport protocol. A connection is identified by a 4-tuple: `(src IP, src port, dst IP, dst port)`.

---

## 1. TCP Header Format

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
```

### Key Header Fields

| Field | Bits | Purpose |
|-------|------|---------|
| **Source Port** | 16 | Sender's port number |
| **Destination Port** | 16 | Receiver's port number |
| **Sequence Number** | 32 | Byte position of first data byte in segment |
| **Acknowledgment Number** | 32 | Next byte the sender expects to receive |
| **Data Offset** | 4 | Length of TCP header in 32-bit words |
| **Flags** | 8 | Control bits: CWR, ECE, URG, ACK, PSH, RST, SYN, FIN |
| **Window** | 16 | Receiver's available buffer space (flow control) |
| **Checksum** | 16 | Error detection for header + data |

### Important Flags

| Flag | Meaning |
|------|---------|
| **SYN** | Synchronize sequence numbers (connection setup) |
| **ACK** | Acknowledgment field is valid |
| **FIN** | No more data from sender (connection teardown) |
| **RST** | Reset — abort the connection immediately |
| **PSH** | Push — deliver data to application immediately |
| **URG** | Urgent pointer field is significant |

---

## 2. Connection Establishment — 3-Way Handshake

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

### What happens at each step

| Step | Sender | Flags | Seq | Ack | Purpose |
|------|--------|-------|-----|-----|---------|
| 1 | Client | SYN | 100 (ISN) | — | "I want to connect. My starting sequence is 100." |
| 2 | Server | SYN+ACK | 300 (ISN) | 101 | "OK. My starting sequence is 300. I expect byte 101 next." |
| 3 | Client | ACK | 101 | 301 | "Got it. I expect byte 301 next." |

**Why 3-way?** It ensures **both sides** agree on initial sequence numbers (ISNs). Each ISN is chosen pseudo-randomly (RFC 9293 S3.4.1) to prevent old duplicate segments from being confused with new ones.

> **Note**: SYN and FIN each consume **1 sequence number**, even though they carry no data.

---

## 3. Data Transfer — Worked Example

**Scenario**: Client sends **20 bytes** (2x10B blocks), Server sends **30 bytes** (3x10B blocks).

Client ISN = 100, Server ISN = 300.

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

| Pkt | Dir | seq | ack | Bytes | What |
|-----|-----|-----|-----|-------|------|
| 1 | C->S | 100 | — | 0 | SYN (ISN=100) |
| 2 | S->C | 300 | 101 | 0 | SYN+ACK (ISN=300) |
| 3 | C->S | 101 | 301 | 0 | ACK (handshake done) |
| 4 | C->S | 101 | 301 | 10 | Client bytes 101-110 |
| 5 | C->S | 111 | 301 | 10 | Client bytes 111-120 |
| 6 | S->C | 301 | 121 | 0 | "Got all 20B, next=121" |
| 7 | S->C | 301 | 121 | 10 | Server bytes 301-310 |
| 8 | S->C | 311 | 121 | 10 | Server bytes 311-320 |
| 9 | S->C | 321 | 121 | 10 | Server bytes 321-330 |
| 10 | C->S | 121 | 331 | 0 | "Got all 30B, next=331" |
| 11 | C->S | 121 | 331 | 0 | FIN (close request) |
| 12 | S->C | 331 | 122 | 0 | ACK the FIN |
| 13 | S->C | 331 | 122 | 0 | FIN (server closes) |
| 14 | C->S | 122 | 332 | 0 | ACK the FIN (done) |

### Key Rule

```
next_seq = current_seq + bytes_sent
ack_number = next byte you EXPECT from the other side
```

---

## 4. Connection Termination — 4-Way Handshake

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
      |     (Client -> FIN-WAIT-2)             |  Server can still send data!
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

**TIME-WAIT (2xMSL)**: Client waits to ensure its final ACK arrived and to let old duplicate segments expire. MSL = Maximum Segment Lifetime (typically 30s, so TIME-WAIT = 60s).

---

## 5. TCP State Machine

The **11 states** of a TCP connection:

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

### All 11 States

| State | Description |
|-------|-------------|
| **CLOSED** | No connection exists |
| **LISTEN** | Server waiting for incoming SYN |
| **SYN-SENT** | Client sent SYN, waiting for SYN+ACK |
| **SYN-RECEIVED** | Server received SYN, sent SYN+ACK, waiting for ACK |
| **ESTABLISHED** | Connection open, data can flow both ways |
| **FIN-WAIT-1** | Sent FIN, waiting for ACK or FIN |
| **FIN-WAIT-2** | FIN acknowledged, waiting for peer's FIN |
| **CLOSE-WAIT** | Received FIN, waiting for local application to close |
| **CLOSING** | Both sides sent FIN simultaneously |
| **LAST-ACK** | Sent FIN after receiving FIN, waiting for final ACK |
| **TIME-WAIT** | Waiting 2xMSL before fully closing |

---

## 6. Special Scenarios

### Simultaneous Open (both sides send SYN at same time)

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

### RST (Reset) — Immediate Abort

Sent when a segment arrives for a non-existent connection or to abort a connection. The receiver immediately transitions to **CLOSED** — no handshake needed.

---

## 7. Key Reliability Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| **Sequence Numbers** | Order bytes and detect duplicates |
| **Checksums** | Detect bit errors in transit |
| **ACKs (Cumulative)** | Confirm receipt of data; one ACK can cover multiple segments |
| **Retransmission Timer** | Resend lost segments (RTO computed via Smoothed RTT algorithm) |
| **Window (Flow Control)** | Receiver limits how fast sender can transmit |
| **Congestion Control** | Sender limits rate to avoid network overload (RFC 5681) |

---

## 8. Key Formulas

```
Sequence Number of next segment = current_seq + bytes_sent_in_this_segment
Acknowledgment Number           = last_byte_received + 1  (i.e., next expected byte)
SYN consumes 1 seq number       -> after SYN with seq=X, next seq = X+1
FIN consumes 1 seq number       -> after FIN with seq=Y, peer ACKs with ack=Y+1
```

---

## References

- [RFC 9293 — Transmission Control Protocol (TCP)](https://www.ietf.org/rfc/rfc9293.html) — Obsoletes RFC 793
- [RFC 5681 — TCP Congestion Control](https://www.ietf.org/rfc/rfc5681.html)
- [RFC 7323 — TCP Extensions for High Performance](https://www.ietf.org/rfc/rfc7323.html)
