# VLAN — Virtual Local Area Network (IEEE 802.1Q)

> **Standard**: IEEE 802.1Q-2022  
> **Layer**: 2 (Data Link)  
> **Tag size**: 4 bytes inserted into Ethernet frame  
> **Max VLANs**: 4,094 (12-bit VLAN ID, IDs 0 and 4095 are reserved)

---

## Table of Contents

1. [What is a VLAN?](#1-what-is-a-vlan)
2. [What Problem Are We Solving?](#2-what-problem-are-we-solving)
3. [IEEE 802.1Q Packet Format](#3-ieee-8021q-packet-format)
4. [VLAN Tag Fields — Bit-Level Breakdown](#4-vlan-tag-fields--bit-level-breakdown)
5. [Tagged vs Untagged Frames](#5-tagged-vs-untagged-frames)
6. [Port Types: Access vs Trunk](#6-port-types-access-vs-trunk)
7. [Real Wireshark Example](#7-real-wireshark-example)
8. [Q-in-Q (802.1ad) — Double Tagging](#8-q-in-q-8021ad--double-tagging)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is a VLAN?

A **Virtual Local Area Network (VLAN)** is a method of creating **logically separate broadcast domains** on the same physical network infrastructure. Devices in the same VLAN can communicate as if they are on the same physical LAN segment — even if they are connected to different switches in different buildings.

```
  Physical Reality:                 Logical Reality (VLANs):

  ┌──────────────┐                  ┌─────── VLAN 10 ─────────┐
  │   Switch     │                  │  PC-A ←──→ PC-C         │
  │              │                  │  (same broadcast domain) │
  │ Port1: PC-A  │                  └──────────────────────────┘
  │ Port2: PC-B  │
  │ Port3: PC-C  │                  ┌─────── VLAN 20 ─────────┐
  │ Port4: PC-D  │                  │  PC-B ←──→ PC-D         │
  └──────────────┘                  │  (same broadcast domain) │
                                    └──────────────────────────┘
  All on ONE switch,
  but isolated into                 PC-A CANNOT reach PC-B
  two VLANs                        (different VLANs = different networks)
```

### Key Characteristics

| Property | Description |
|----------|-------------|
| **Broadcast isolation** | Broadcast frames stay within the VLAN — they never leak to other VLANs |
| **Logical grouping** | Group users by function (Engineering, HR, Finance) regardless of physical location |
| **Unique ID** | Each VLAN is identified by a 12-bit VLAN ID (1–4094) |
| **Layer 2 boundary** | VLANs operate at Layer 2; communication **between** VLANs requires a Layer 3 device (router or L3 switch) |

> **Think of it this way**: A VLAN turns one physical switch into multiple *virtual* switches. Each virtual switch is completely isolated from the others.

---

## 2. What Problem Are We Solving?

### The Flat Network Problem

Without VLANs, every device on a switch (or group of switches) belongs to **one big broadcast domain**. This causes real problems as the network grows:

```
  WITHOUT VLANs — Flat Network:

  ┌──────────────────────────────────────────────────┐
  │              ONE Broadcast Domain                 │
  │                                                   │
  │  Engineering   HR        Finance     Printers     │
  │  ┌───┐ ┌───┐  ┌───┐    ┌───┐ ┌───┐  ┌───┐       │
  │  │PC1│ │PC2│  │PC3│    │PC4│ │PC5│  │PRN│       │
  │  └───┘ └───┘  └───┘    └───┘ └───┘  └───┘       │
  │                                                   │
  │  🔊 ARP broadcast from PC1 → reaches ALL devices │
  │  🔊 DHCP discover from PC3 → reaches ALL devices │
  │  🔊 Any broadcast → floods EVERYWHERE             │
  └──────────────────────────────────────────────────┘

  Problems:
  ✗ PC1's ARP request hits PC3 (HR), PC4 (Finance) — wasted bandwidth
  ✗ Anyone can sniff traffic from any department
  ✗ A broadcast storm from one device takes down EVERYTHING
  ✗ Network slows as more devices are added
```

### The Five Problems VLANs Solve

| # | Problem | Without VLANs | With VLANs |
|---|---------|---------------|------------|
| 1 | **Broadcast storms** | One misbehaving device floods the entire network | Broadcast is contained within the VLAN |
| 2 | **Security** | HR payroll data visible to Engineering sniffers | Traffic is isolated; cross-VLAN access requires explicit routing + ACLs |
| 3 | **Performance** | Every device processes every broadcast frame | Devices only see broadcasts from their own VLAN |
| 4 | **Flexibility** | Moving a user to a different department = re-cabling | Change the port's VLAN assignment — no physical change needed |
| 5 | **Scalability** | Flat networks don't scale past ~500 devices | VLANs break the network into manageable segments |

### With VLANs — Segmented Network

```
  WITH VLANs — Segmented:

  ┌─────── VLAN 10 (Engineering) ──┐   ┌──── VLAN 20 (HR) ────────┐
  │  ┌───┐ ┌───┐                   │   │  ┌───┐                   │
  │  │PC1│ │PC2│                   │   │  │PC3│                   │
  │  └───┘ └───┘                   │   │  └───┘                   │
  │  🔊 Broadcast stays HERE only  │   │  🔊 Broadcast stays HERE │
  └────────────────────────────────┘   └──────────────────────────┘

  ┌─────── VLAN 30 (Finance) ──────┐   ┌── VLAN 99 (Printers) ───┐
  │  ┌───┐ ┌───┐                   │   │  ┌───┐                   │
  │  │PC4│ │PC5│                   │   │  │PRN│                   │
  │  └───┘ └───┘                   │   │  └───┘                   │
  └────────────────────────────────┘   └──────────────────────────┘

  ✓ PC1's ARP only reaches PC2 (same VLAN 10)
  ✓ HR data (VLAN 20) invisible to Engineering (VLAN 10)
  ✓ Broadcast storm in VLAN 30 doesn't affect other VLANs
  ✓ Move PC4 to Engineering? Just change the port to VLAN 10
```

### Real-World Analogy

> Imagine an office building with one open floor. Everyone hears everyone's conversations, and sensitive HR discussions are overheard by the entire company. **VLANs are like adding walls and doors** — each department gets its own room (broadcast domain). People in the same room communicate freely, but to talk to another room you must go through a door (router/L3 switch) that has access controls.

---

## 3. IEEE 802.1Q Packet Format

IEEE 802.1Q defines how VLAN membership is carried **inside the Ethernet frame** by inserting a 4-byte tag between the Source MAC address and the original EtherType/Length field.

### Standard Ethernet Frame (Untagged)

```
 ┌──────────┬──────────┬──────────┬─────────────────────┬─────┐
 │ Dest MAC │ Src MAC  │ EtherType│      Payload        │ FCS │
 │ 6 bytes  │ 6 bytes  │ 2 bytes  │   46–1500 bytes     │ 4 B │
 └──────────┴──────────┴──────────┴─────────────────────┴─────┘
                                   Total max: 1518 bytes
```

### 802.1Q Tagged Ethernet Frame

```
 ┌──────────┬──────────┬──────┬──────────┬──────────┬─────────────────────┬─────┐
 │ Dest MAC │ Src MAC  │ TPID │   TCI    │ EtherType│      Payload        │ FCS │
 │ 6 bytes  │ 6 bytes  │ 2 B  │   2 B    │ 2 bytes  │   46–1500 bytes     │ 4 B │
 └──────────┴──────────┴──────┴──────────┴──────────┴─────────────────────┴─────┘
                        ├─── 802.1Q Tag ──┤
                        │    (4 bytes)     │            Total max: 1522 bytes
                        └─────────────────┘

 TPID = 0x8100 (identifies this as a VLAN-tagged frame)
 TCI  = Tag Control Information (Priority + DEI + VLAN ID)
```

### Side-by-Side Comparison

```
  Untagged Frame:
  ┌──────────┬──────────┬──────────┬──────────────┬─────┐
  │ Dst MAC  │ Src MAC  │ 0x0800   │ IP Packet... │ FCS │
  │          │          │ (IPv4)   │              │     │
  └──────────┴──────────┴──────────┴──────────────┴─────┘
                         ↑
                         EtherType at offset 12

  Tagged Frame (802.1Q):
  ┌──────────┬──────────┬──────────┬──────┬──────────┬──────────────┬─────┐
  │ Dst MAC  │ Src MAC  │  0x8100  │ TCI  │  0x0800  │ IP Packet... │ FCS │
  │          │          │  (TPID)  │      │  (IPv4)  │              │     │
  └──────────┴──────────┴──────────┴──────┴──────────┴──────────────┴─────┘
                         ↑                  ↑
                         Tag inserted        Original EtherType
                         at offset 12        shifted to offset 16

  📌 The 4-byte tag (TPID + TCI) is INSERTED — the rest of the frame shifts right
  📌 Max frame size increases from 1518 → 1522 bytes
```

---

## 4. VLAN Tag Fields — Bit-Level Breakdown

The 4-byte 802.1Q tag is composed of two 16-bit fields:

```
                    802.1Q Tag (32 bits / 4 bytes)
  ┌────────────────────────────┬────────────────────────────┐
  │         TPID (16 bits)     │         TCI (16 bits)      │
  │         0x8100             │                            │
  └────────────────────────────┴────────────────────────────┘

  TCI Breakdown:
  ┌───────────┬─────┬──────────────────────────────────────┐
  │ PCP (3b)  │DEI  │           VID (12 bits)              │
  │           │(1b) │                                      │
  └───────────┴─────┴──────────────────────────────────────┘
   Bits: 15-13  12            11-0
```

### Field Details

| Field | Bits | Description |
|-------|------|-------------|
| **TPID** (Tag Protocol Identifier) | 16 | Always `0x8100` for 802.1Q. This is what tells the receiver "this frame is VLAN-tagged." Appears at the same offset as the EtherType in an untagged frame |
| **PCP** (Priority Code Point) | 3 | Frame priority level (0–7). Used by IEEE 802.1p for Quality of Service (QoS). Higher values = higher priority |
| **DEI** (Drop Eligible Indicator) | 1 | If `1`, this frame can be dropped first during congestion. Originally called **CFI** (Canonical Format Indicator) in older standards — used to indicate Token Ring frame format. Renamed to DEI in IEEE 802.1Q-2011 |
| **VID** (VLAN Identifier) | 12 | The VLAN number (0–4095). IDs `0` and `4095` are reserved. Usable range: **1–4094** |

### PCP Priority Values (IEEE 802.1p)

| PCP Value | Priority | Traffic Type |
|-----------|----------|-------------|
| 0 | Best Effort | Default — normal data traffic |
| 1 | Background | Bulk data, backups |
| 2 | Excellent Effort | Business-critical applications |
| 3 | Critical Applications | Signaling (SIP, H.323) |
| 4 | Video | Video streaming (<100ms latency) |
| 5 | Voice | VoIP (<10ms latency) |
| 6 | Internetwork Control | Routing protocols (OSPF, BGP) |
| 7 | Network Control | Highest — STP BPDUs, critical network control |

### Reserved VLAN IDs

| VID | Usage |
|-----|-------|
| **0** | Priority-tagged frame only (carries QoS info but no VLAN membership) |
| **1** | Default VLAN on most switches (native VLAN) |
| **4095** | Reserved — must not be used |

---

## 5. Tagged vs Untagged Frames

Understanding when frames carry the 802.1Q tag is critical:

```
  Access Port (Untagged)              Trunk Port (Tagged)
  ┌────────┐                         ┌────────┐
  │        │   No tag — the switch   │        │   Tag added — tells
  │ Server │   knows the VLAN from   │ Switch ├──── the other switch
  │        │   the port config       │    A   │   which VLAN this
  └───┬────┘                         └───┬────┘   frame belongs to
      │                                  │
      │  ┌─────────────────┐             │  ┌───────────────────────┐
      └──│ Untagged Frame  │             └──│ Tagged Frame (0x8100) │
         │ (no 802.1Q tag) │                │ VLAN 10, PCP 0       │
         └─────────────────┘                └───────────────────────┘
```

| Scenario | Tag Present? | Why? |
|----------|-------------|------|
| PC → Access port | **No** | End devices don't know about VLANs; the switch assigns the VLAN based on port config |
| Switch → Switch (trunk) | **Yes** | The receiving switch needs to know which VLAN the frame belongs to |
| Switch → Access port → PC | **No** | Switch strips the tag before delivering to the end device |
| Native VLAN on trunk | **No** | Frames on the native VLAN are sent untagged on trunk ports (configurable) |

---

## 6. Port Types: Access vs Trunk

| Feature | Access Port | Trunk Port |
|---------|-------------|------------|
| **VLANs carried** | Exactly 1 | Multiple (or all) |
| **Tagging** | Frames are untagged | Frames are tagged with 802.1Q |
| **Typical connection** | End device (PC, server, printer) | Switch-to-switch, switch-to-router |
| **Native VLAN** | N/A — the access VLAN IS the native | Configurable — untagged frames belong to this VLAN |

### Trunk Link — How Multiple VLANs Share One Cable

```
        Switch A                              Switch B
  ┌────────────────┐                    ┌────────────────┐
  │ VLAN 10: Ports │                    │ VLAN 10: Ports │
  │   1, 2, 3      │    Trunk Link      │   1, 2, 3      │
  │                │  (carries ALL      │                │
  │ VLAN 20: Ports ├────VLANs with──────┤ VLAN 20: Ports │
  │   4, 5, 6      │   802.1Q tags)     │   4, 5, 6      │
  │                │                    │                │
  │ VLAN 30: Ports │  VLAN 10 frame →   │ VLAN 30: Ports │
  │   7, 8         │  Tag: VID=10       │   7, 8         │
  └────────────────┘                    └────────────────┘

  The trunk port tags each frame with the VLAN ID so the remote
  switch delivers it to the correct VLAN's ports.
```

---

## 7. Real Wireshark Example

Here's a real VLAN-tagged frame as seen in Wireshark (from [Wireshark VLAN wiki](https://wiki.wireshark.org/VLAN)):

```
 Frame 53 (70 bytes on wire, 70 bytes captured)
 Ethernet II, Src: 00:40:05:40:ef:24, Dst: 00:60:08:9f:b1:f3
 802.1Q Virtual LAN
    000. .... .... .... = Priority: 0        ← PCP = 0 (Best Effort)
    ...0 .... .... .... = DEI: 0             ← Not drop-eligible
    .... 0000 0010 0000 = ID: 32             ← VLAN 32
    Type: IP (0x0800)                        ← Inner EtherType (IPv4)
 Internet Protocol, Src: 131.151.32.129, Dst: 131.151.32.21
 Transmission Control Protocol, Src Port: 1173, Dst Port: 6000
```

### Reading the Hex

```
  Byte offset:  00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11
                ─────────── ─────────── ───── ───── ─────
                Dest MAC     Src MAC     TPID   TCI  EtherType
                                        81 00 00 20 08 00
                                              │  │
                                              │  └─ VID = 0x020 = 32
                                              └─ PCP=0, DEI=0

  TCI = 0x0020 = 0000 0000 0010 0000 (binary)
                  ─── ─ ────────────
                  PCP DEI   VID
                   0   0     32
```

### Wireshark Display Filters

| Filter | What It Shows |
|--------|---------------|
| `vlan` | All VLAN-tagged traffic |
| `vlan.id == 10` | Only frames in VLAN 10 |
| `vlan.priority == 5` | Only voice-priority frames |
| `vlan.id == 10 && ip.addr == 10.0.10.5` | VLAN 10 traffic to/from a specific host |

### Wireshark Capture Filters

```bash
# Capture only VLAN-tagged traffic
vlan

# Capture only VLAN 17 traffic
vlan 17

# Capture both tagged and untagged traffic to a host
host 10.0.10.5 or (vlan and host 10.0.10.5)
```

> ⚠️ **Important**: The `vlan` keyword in capture filters changes decoding offsets for the rest of the filter expression. Place it carefully — see [Wireshark VLAN wiki](https://wiki.wireshark.org/VLAN) for details.

---

## 8. Q-in-Q (802.1ad) — Double Tagging

Service providers use **Q-in-Q** (IEEE 802.1ad) to stack a second VLAN tag, allowing them to carry customer VLANs transparently across their network:

```
  Double-Tagged Frame (Q-in-Q):

  ┌──────────┬──────────┬─────────┬─────────┬─────────┬─────────┬──────────┬─────┐
  │ Dst MAC  │ Src MAC  │ S-TPID  │ S-TCI   │ C-TPID  │ C-TCI   │ EtherType│ ... │
  │ 6 bytes  │ 6 bytes  │ 0x88A8  │ 2 bytes │ 0x8100  │ 2 bytes │ 2 bytes  │     │
  └──────────┴──────────┴─────────┴─────────┴─────────┴─────────┴──────────┴─────┘
                         ├── Outer (Service) ──┤├── Inner (Customer) ──┤
                         │  Provider's VLAN    ││  Customer's VLAN     │
                         └─────────────────────┘└──────────────────────┘

  S-TPID = 0x88A8 (Service tag — provider)
  C-TPID = 0x8100 (Customer tag — original VLAN)
```

| Tag | TPID | Purpose | Who Adds It |
|-----|------|---------|-------------|
| **Outer (S-Tag)** | `0x88A8` | Service provider's VLAN — routes the frame through the provider network | Provider edge switch |
| **Inner (C-Tag)** | `0x8100` | Customer's original VLAN — preserved end-to-end | Customer switch |

---

## 9. Key Takeaways

| Concept | One-Line Summary |
|---------|-----------------|
| **VLAN** | Logically divides one physical network into isolated broadcast domains |
| **802.1Q** | The IEEE standard that defines the 4-byte tag inserted into Ethernet frames |
| **TPID** | `0x8100` — the magic number that says "this frame is VLAN-tagged" |
| **VID** | 12-bit VLAN ID (1–4094) — identifies which VLAN the frame belongs to |
| **PCP** | 3-bit priority (0–7) — enables QoS on Layer 2 |
| **DEI** | 1 bit — marks frames that can be dropped first during congestion |
| **Access port** | Connects end devices; carries one VLAN; frames are untagged |
| **Trunk port** | Connects switches; carries multiple VLANs; frames are tagged |
| **Native VLAN** | The one VLAN on a trunk that is sent untagged |
| **Q-in-Q** | Double tagging (0x88A8 + 0x8100) — providers carry customer VLANs transparently |

### What's Next?

Now that you understand VLAN fundamentals and the 802.1Q frame format, see:
- **[Inter-VLAN Routing with SVI](InterVLAN_Routing.md)** — How traffic moves between VLANs using Layer 3 SVIs on Cisco Nexus switches

---

## References

- [IEEE 802.1Q-2022 — Bridges and Bridged Networks](https://standards.ieee.org/standard/802_1Q-2022.html) — the definitive standard for VLANs
- [IEEE 802.1p — Traffic Class Expediting](https://standards.ieee.org/standard/802_1p-1998.html) — PCP priority values
- [IEEE 802.1ad — Provider Bridges (Q-in-Q)](https://standards.ieee.org/standard/802_1ad-2005.html) — double tagging
- [RFC 5765 — Security Vulnerabilities of IEEE 802.1Q](https://www.rfc-editor.org/rfc/rfc5765) — VLAN security considerations
- [Wireshark VLAN Wiki](https://wiki.wireshark.org/VLAN) — capture filters, dissector, sample captures
- [Wireshark VLAN Display Filter Reference](https://www.wireshark.org/docs/dfref/v/vlan.html) — all filterable fields

---

> *From flat networks to segmented sanity — one VLAN tag at a time.* 🏷️
