# 🔬 Real-World TCP 3-Way Handshake — Wireshark Packet Analysis

> **Capture source**: macOS Wi-Fi interface (en0)  
> **Connection**: Laptop (192.168.1.3) → Microsoft Bing server (204.79.197.203:443)  
> **Captured**: April 12, 2026 — 3 packets forming a complete TCP handshake  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Packet 1 — SYN (Frame 128)](#2-packet-1--syn-frame-128)
3. [Packet 2 — SYN+ACK (Frame 132)](#3-packet-2--synack-frame-132)
4. [Packet 3 — ACK (Frame 133)](#4-packet-3--ack-frame-133)
5. [Complete Handshake Diagram](#5-complete-handshake-diagram)
6. [Golden Rule Proof with Real Data](#6-golden-rule-proof-with-real-data)
7. [Negotiated Parameters Summary](#7-negotiated-parameters-summary)
8. [Packet Size Comparison](#8-packet-size-comparison)
9. [Timing Analysis](#9-timing-analysis)
10. [Key Insights](#10-key-insights)

---

## 1. Overview

This document dissects a **real TCP 3-way handshake** captured with Wireshark between a laptop and Microsoft's Bing server (`204.79.197.203`). The connection is to port **443 (HTTPS)** — after the TCP handshake completes, a TLS handshake would follow.

### Connection Identity (5-Tuple)

| Field | Value |
|-------|-------|
| Protocol | TCP (IP Protocol 6) |
| Source IP | 192.168.1.3 (laptop on home Wi-Fi) |
| Source Port | 53760 (ephemeral port) |
| Destination IP | 204.79.197.203 (Microsoft/Bing) |
| Destination Port | 443 (HTTPS) |

### Wireshark Stream Info

| Property | Value |
|----------|-------|
| Ethernet Stream Index | 0 |
| IP Stream Index | 9 |
| TCP Stream Index | 8 |

---

## 2. Packet 1 — SYN (Frame 128)

> **Client → Server**: "I want to connect. Here is my initial sequence number and capabilities."

### Raw Capture Data

```
Frame 128: 78 bytes on wire (624 bits), 78 bytes captured
Arrival Time: Apr 12, 2026 13:26:47.878495000 IST
```

### Layer-by-Layer Breakdown

#### Layer 2 — Ethernet II (14 bytes)

| Field | Value | Insight |
|-------|-------|---------|
| Destination MAC | `00:04:95:e7:30:c8` | TejasNetwork — this is the **home router** |
| Source MAC | `5a:ad:8a:bd:70:8b` | Laptop Wi-Fi MAC (**locally administered** — LG bit = 1) |
| EtherType | `0x0800` | IPv4 |

> 💡 **Why locally administered MAC?** macOS uses **Private Wi-Fi Address** (MAC randomization) for privacy. The `LG bit = 1` confirms this is NOT the factory-burned MAC.

#### Layer 3 — IPv4 (20 bytes)

| Field | Value | Insight |
|-------|-------|---------|
| Version | 4 | IPv4 |
| Header Length | 20 bytes (5 x 4) | Minimum — no IP options |
| DSCP | CS0 (0x00) | Default / Best Effort — no QoS marking from client |
| Total Length | 64 bytes | 20 (IP) + 44 (TCP with options) |
| Identification | 0x0000 | Not used (DF is set, no fragmentation) |
| Don't Fragment | **Set** | Tells routers: don't fragment, use Path MTU Discovery instead |
| TTL | **64** | macOS default — decrements by 1 per router hop |
| Protocol | 6 | TCP |
| Source IP | 192.168.1.3 | Private IP (RFC 1918) — NAT will translate this |
| Destination IP | 204.79.197.203 | Microsoft public IP |

> 💡 **TTL = 64** is the default for macOS/Linux. Windows uses 128. You can fingerprint the OS from TTL!

#### Layer 4 — TCP (44 bytes = 20 header + 24 options)

| Field | Value | Insight |
|-------|-------|---------|
| Source Port | 53760 | Ephemeral port chosen by OS |
| Destination Port | 443 | HTTPS |
| **Sequence Number (raw)** | **1,756,649,201** | Client ISN — pseudo-random to prevent hijacking |
| Sequence Number (relative) | 0 | Wireshark normalizes to 0 for readability |
| Acknowledgment Number | 0 | Not meaningful — ACK flag is NOT set |
| Header Length | 44 bytes (11 x 4) | 20 base + 24 bytes of options |
| **Flags** | **0x002 (SYN)** | Only SYN is set — connection request |
| Window Size | 65535 | Raw window (scale not applied yet) |

#### TCP Options (24 bytes)

| Option | Value | Purpose |
|--------|-------|---------|
| **MSS** | **1460 bytes** | "My network can handle 1460-byte segments" (standard for Ethernet: 1500 MTU - 20 IP - 20 TCP) |
| **Window Scale** | **6** (multiply by 64) | Effective window = 65535 x 64 = **~4 MB** |
| **Timestamps** | TSval=2882275842, TSecr=0 | Client offers timestamp-based RTT measurement |
| **SACK Permitted** | Yes | Client supports Selective ACK for loss recovery |
| NOP x 3 | -- | Padding for 4-byte alignment |
| EOL x 2 | -- | End of options |

> 💡 **Options are ONLY negotiated during SYN and SYN+ACK.** After the handshake, TCP headers carry zero options (just the 20-byte minimum).

---

## 3. Packet 2 — SYN+ACK (Frame 132)

> **Server → Client**: "I accept. Here is MY initial sequence number. I acknowledge your SYN."

### Raw Capture Data

```
Frame 132: 66 bytes on wire (528 bits), 66 bytes captured
Arrival Time: Apr 12, 2026 13:26:47.896762000 IST
Time since SYN: 18.267 ms
```

### Layer-by-Layer Breakdown

#### Layer 2 — Ethernet II (14 bytes)

| Field | Value | Insight |
|-------|-------|---------|
| Destination MAC | `5a:ad:8a:bd:70:8b` | Now **reversed** — going to your laptop |
| Source MAC | `00:04:95:e7:30:c8` | Router MAC (frames arriving from internet always have router MAC) |

> 💡 **MACs are always the LAST hop.** Even though the packet came from Microsoft, the Ethernet source is your local router — because MACs are Layer 2 (local segment only).

#### Layer 3 — IPv4 (20 bytes)

| Field | Value | Insight |
|-------|-------|---------|
| DSCP | **EF (Expedited Forwarding)** = 0xB8 | Microsoft marks their traffic as **premium/low-latency** |
| Total Length | 52 bytes | 20 (IP) + 32 (TCP with options) — smaller than SYN |
| TTL | **117** | Started at 128 (Windows server), crossed **11 router hops** (128 - 117 = 11) |
| Source IP | 204.79.197.203 | Microsoft responding |
| Destination IP | 192.168.1.3 | Back to your laptop |

> 💡 **DSCP = EF (46)**: Microsoft uses Expedited Forwarding, a QoS marking that tells routers "treat this as real-time/premium traffic." Your laptop sent CS0 (best effort). This asymmetry is common — big companies pay for QoS.

> 💡 **TTL forensics**: Server TTL=117, started at 128 → **11 hops away**. Your SYN had TTL=64 → also travelled ~11 hops to reach server (arriving with TTL ~ 53). The hop count is roughly symmetric.

#### Layer 4 — TCP (32 bytes = 20 header + 12 options)

| Field | Value | Insight |
|-------|-------|---------|
| Source Port | 443 | Server responding |
| Destination Port | 53760 | Back to client ephemeral port |
| **Sequence Number (raw)** | **712,760,935** | Server ISN — completely different random number |
| **Acknowledgment Number (raw)** | **1,756,649,202** | = Client ISN (1,756,649,201) + 1 |
| **Flags** | **0x012 (SYN + ACK)** | Both SYN and ACK bits set |
| Window Size | 65535 | Raw window |

> 💡 **The ACK proves SYN consumes 1 sequence number**: Server ACKs `1,756,649,202` = client ISN + 1. The SYN itself occupies sequence number 1,756,649,201 even though it carries zero bytes of data.

#### TCP Options (12 bytes) — Server Counter-Offer

| Option | Client Offered | Server Responds | **Negotiated Result** |
|--------|---------------|-----------------|----------------------|
| **MSS** | 1460 | **1250** | **1250** (minimum wins) |
| **Window Scale** | 6 (x64) | **8 (x256)** | Each side uses own |
| **SACK** | Yes | **Yes** | Enabled |
| **Timestamps** | Yes (TSval sent) | **Not echoed** | Disabled |

> 💡 **MSS 1250 < 1460**: The server (or a middlebox) has a smaller MTU path. Both sides will now cap segments at 1250 bytes. This often happens with VPNs, tunnels, or cloud load balancers that add encapsulation headers.

> 💡 **Timestamps declined**: Server did not echo the Timestamp option. This means TCP Timestamps (RFC 7323) will NOT be used for this connection. RTT will be estimated from ACK timing instead.

---

## 4. Packet 3 — ACK (Frame 133)

> **Client → Server**: "Confirmed. I acknowledge your SYN. Let us go!"

### Raw Capture Data

```
Frame 133: 54 bytes on wire (432 bits), 54 bytes captured
Arrival Time: Apr 12, 2026 13:26:47.896983000 IST
Time since SYN: 18.488 ms
Time since SYN+ACK: 0.221 ms (221 microseconds!)
```

### Layer 4 — TCP (20 bytes — MINIMUM HEADER)

| Field | Value | Insight |
|-------|-------|---------|
| **Sequence Number (raw)** | **1,756,649,202** | = Server ACK from Packet 2 (**Golden Rule!**) |
| **Acknowledgment Number (raw)** | **712,760,936** | = Server ISN (712,760,935) + 1 |
| Header Length | **20 bytes** | Absolute minimum — **zero options** |
| **Flags** | **0x010 (ACK only)** | Pure acknowledgment, no SYN |
| Window Size | 4096 x 64 = **262,144 bytes** | Window scale now applied! |

> 💡 **20-byte header = no options**: After the handshake, TCP options are not needed. The SYN/SYN+ACK already negotiated everything. This is why Packet 3 is the **smallest** of the three (54 bytes total).

> 💡 **Window scale is now active**: Raw window = 4096, but with the negotiated scale factor of 6 (x64), the effective window is **262,144 bytes (256 KB)**. This is the client saying "I can buffer 256 KB of data before you need to slow down."

---

## 5. Complete Handshake Diagram

```
  192.168.1.3:53760                              204.79.197.203:443
  (Your Laptop, macOS)                           (Microsoft Bing, Windows)
       |                                               |
  Frame 128                                            |
  78 bytes  | SYN  seq=1756649201                      |   t = 0.000 ms
  TTL=64    |  MSS=1460, WS=6, SACK, Timestamps       |
  DSCP=CS0  |--------------------------------------------->|
            |                                              |
            |                                         Frame 132
            |       SYN+ACK  seq=712760935            66 bytes
            |                ack=1756649202            TTL=117
            |       MSS=1250, WS=8, SACK              DSCP=EF
            |<---------------------------------------------|   t = 18.267 ms
            |                                              |
  Frame 133 |                                              |
  54 bytes  | ACK  seq=1756649202, ack=712760936           |   t = 18.488 ms
  TTL=64    |  window=262144 (4096 x 64)                   |
  DSCP=CS0  |--------------------------------------------->|
            |                                              |
            |     ~~~ ESTABLISHED — Connection Ready ~~~   |
            |     Next: TLS ClientHello (port 443)         |
```

---

## 6. Golden Rule Proof with Real Data

> **Golden Rule**: My sequence number = the peer's last ACK to me.

### Verification Table

| Packet | Who Sends | Seq (raw) | How Derived | Match? |
|--------|-----------|-----------|-------------|--------|
| **Pkt 1** (SYN) | Client | 1,756,649,201 | ISN (random, OS-generated) | -- |
| **Pkt 2** (SYN+ACK) | Server | 712,760,935 | ISN (random, OS-generated) | -- |
| **Pkt 2** (SYN+ACK) | Server | ack = 1,756,649,202 | Client ISN + 1 (SYN consumed 1 seq) | -- |
| **Pkt 3** (ACK) | Client | **1,756,649,202** | = Server ack from Pkt 2 | YES |
| **Pkt 3** (ACK) | Client | ack = **712,760,936** | = Server ISN + 1 (SYN consumed 1 seq) | YES |

### The Math

```
Client Pkt 3 seq = Server Pkt 2 ack
  1,756,649,202  =  1,756,649,202     ✅  GOLDEN RULE PROVEN

Client Pkt 3 ack = Server Pkt 2 seq + 1  (SYN = 1 phantom byte)
    712,760,936  =  712,760,935 + 1   ✅  SYN CONSUMES 1 SEQ NUMBER
```

---

## 7. Negotiated Parameters Summary

After the 3-way handshake, these connection parameters are locked in:

| Parameter | Client Value | Server Value | Negotiated Result |
|-----------|-------------|-------------|-------------------|
| **MSS** | 1460 bytes | 1250 bytes | **1250** (min of both) |
| **Window Scale** | 6 (x64) | 8 (x256) | Each side uses own |
| **Effective Window** | ~4 MB | ~16 MB | Each side uses own |
| **SACK** | Offered | Agreed | **Enabled** |
| **Timestamps** | Offered | Declined | **Disabled** |
| **Initial Seq (Client)** | 1,756,649,201 | -- | -- |
| **Initial Seq (Server)** | -- | 712,760,935 | -- |

### Why MSS = 1250?

```
Standard Ethernet MTU:    1500 bytes
 - IP Header:              - 20 bytes
 - TCP Header:             - 20 bytes
 = Standard MSS:           1460 bytes  <-- Client offered this

Server offered 1250 --> likely a middlebox (load balancer, tunnel, CDN)
adds extra headers that reduce the available payload space.

Result: Both sides cap payload at 1250 bytes/segment.
```

### Why Different Window Scales?

```
Client:  window=65535, scale=6 --> 65535 x 64  =   4,194,240 bytes (~4 MB)
Server:  window=65535, scale=8 --> 65535 x 256  =  16,776,960 bytes (~16 MB)

The server advertises a larger receive buffer because it handles
thousands of connections and has more memory allocated for TCP.
Your laptop is more conservative with a 4 MB buffer.
```

---

## 8. Packet Size Comparison

| Packet | Frame Bytes | Ethernet | IPv4 | TCP Header | TCP Options | Data | Total |
|--------|------------|----------|------|------------|-------------|------|-------|
| **Pkt 1** SYN | 78 | 14 | 20 | 20 | **24** | 0 | 78 |
| **Pkt 2** SYN+ACK | 66 | 14 | 20 | 20 | **12** | 0 | 66 |
| **Pkt 3** ACK | 54 | 14 | 20 | 20 | **0** | 0 | 54 |

### Why the Size Difference?

- **SYN (78 bytes)**: Largest — client sends ALL capabilities (MSS + WS + TS + SACK + padding = 24 bytes of options)
- **SYN+ACK (66 bytes)**: Medium — server responds with fewer options (MSS + WS + SACK = 12 bytes, no Timestamps)
- **ACK (54 bytes)**: Smallest — pure acknowledgment, **zero options** needed (negotiation is complete)

> 💡 **54 bytes is the absolute minimum for a TCP/IPv4 packet on Ethernet**: 14 (Eth) + 20 (IP) + 20 (TCP) = 54 bytes. You cannot go smaller while still being a valid TCP segment.

---

## 9. Timing Analysis

```
Timeline (milliseconds from SYN):

0.000 ms --- Frame 128: SYN sent ----------------------------+
                                                              | 18.267 ms
                                                              | (Network Round Trip)
18.267 ms -- Frame 132: SYN+ACK received -------------------+
                                                              | 0.221 ms
                                                              | (Kernel Processing)
18.488 ms -- Frame 133: ACK sent ---------------------------+

Total Handshake Duration: 18.488 ms
```

| Metric | Value | What It Means |
|--------|-------|---------------|
| **Network RTT** | 18.267 ms | Time for packet to travel laptop -> Microsoft -> laptop |
| **ACK Processing** | 0.221 ms (221 us) | Time for your OS kernel to process SYN+ACK and respond |
| **Total Handshake** | 18.488 ms | Connection ready in under 19 milliseconds |
| **iRTT** | 18.488 ms | Wireshark initial RTT estimate for this connection |

> 💡 **0.221 ms ACK delay** is nearly instant — your kernel does not need to "think" about the ACK; it just sends it immediately. The 18.267 ms is pure network latency (your ISP -> internet backbone -> Microsoft datacenter -> back).

---

## 10. Key Insights

### Insight 1: SYN Consumes Exactly 1 Sequence Number

```
Client ISN:     1,756,649,201
Server ACKs:    1,756,649,202  (ISN + 1)

The SYN flag occupies 1 "phantom byte" in the sequence space,
even though it carries zero data. This is by design (RFC 9293)
so the SYN itself can be acknowledged and retransmitted if lost.
```

### Insight 2: MAC Addresses Never Leave Your LAN

```
Packet 1 (outbound):  Your MAC -> Router MAC
Packet 2 (inbound):   Router MAC -> Your MAC

Even though the IP destination is Microsoft (204.79.197.203),
the Ethernet destination is always your local router. MACs are
Layer 2 — they get rewritten at every hop.
```

### Insight 3: OS Fingerprinting from TTL

```
Your laptop:   TTL = 64  -> macOS/Linux default
Microsoft:     TTL = 117 -> started at 128 (Windows default), 11 hops away

Common defaults:
  Linux/macOS:  64
  Windows:      128
  Cisco/network: 255
```

### Insight 4: QoS Asymmetry (DSCP)

```
Your traffic:       DSCP = CS0 (Best Effort, 0x00)
Microsoft traffic:  DSCP = EF  (Expedited Forwarding, 0xB8)

Microsoft marks their packets as high-priority. ISPs and backbone
routers that honor QoS will prioritize these packets. Your laptop
sends default best-effort traffic — typical for consumer endpoints.
```

### Insight 5: Options Negotiation is a One-Shot Deal

```
SYN:      Client proposes  (MSS=1460, WS=6, SACK, Timestamps)
SYN+ACK:  Server responds  (MSS=1250, WS=8, SACK, no Timestamps)
ACK:      Nothing to negotiate — 0 bytes of options

After the handshake, TCP options are locked in for the lifetime
of the connection. The ACK packet proves this with its minimal
20-byte header (no option bytes at all).
```

### Insight 6: Window Scale Changes Everything

```
Raw window in ACK:   4,096
Scale factor:        x 64 (from negotiated WS=6)
Effective window:    262,144 bytes = 256 KB

Without window scaling (RFC 7323), TCP windows max out at 65,535 bytes
(16-bit field). With scaling, windows can reach up to ~1 GB, enabling
high-throughput connections on fast links.
```

### Insight 7: Privacy via MAC Randomization

```
Your MAC: 5a:ad:8a:bd:70:8b
LG bit = 1 -> Locally Administered (randomized)

macOS/iOS randomize Wi-Fi MAC addresses to prevent tracking.
The second hex digit being odd (a = 1010, LG bit is bit 1 = 1)
confirms this is a private/random MAC, not your hardware MAC.
```

---

## References

- [RFC 9293 — Transmission Control Protocol (TCP)](https://www.ietf.org/rfc/rfc9293.html) — TCP specification
- [RFC 7323 — TCP Extensions for High Performance](https://www.rfc-editor.org/rfc/rfc7323) — Window Scale, Timestamps
- [RFC 2474 — DiffServ (DSCP)](https://www.rfc-editor.org/rfc/rfc2474) — QoS field definitions
- [RFC 2018 — SACK](https://www.rfc-editor.org/rfc/rfc2018) — Selective Acknowledgment
- [Wireshark Documentation](https://www.wireshark.org/docs/) — Packet capture and analysis

---

> *Captured and analyzed on April 12, 2026 — a practical exercise in mapping RFC 9293 theory to real network traffic.*
