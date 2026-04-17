# 🌐 Computer Networking — The Simple Way

> It is all about computer networking in a simple way. Understand networking concepts like TCP/IP, VLAN, VXLAN, Ethernet, and more — through clear explanations, real-world packet analysis, and hands-on examples.

---

## 📚 What You'll Find Here

This repository breaks down networking concepts into digestible, practical notes — no jargon overload, just clarity.

| Topic | What You'll Learn |
|-------|-------------------|
| [**TCP**](TCP-IP/TCP/) | How connections are established (3-way handshake), how data flows with sequence numbers, flow control, retransmission, and connection teardown |
| [**VLAN**](TCP-IP/VLAN/) | Virtual LANs — what they are, why we need them, 802.1Q packet format, inter-VLAN routing with SVI on Cisco Nexus |
| [**VXLAN**](TCP-IP/VXLAN/) | Overlay networking for data centers — extending Layer 2 over Layer 3 |
| [**RDMA**](TCP-IP/RDMA/) | Remote Direct Memory Access — zero-copy, kernel-bypass networking for high-performance data centers, AI/ML training, and distributed storage |

## 🔬 Highlights

- **Theory + Real Packets** — Every concept is backed by actual Wireshark captures showing real bytes on the wire
- **Sequence Number Deep Dive** — Byte-level diagrams showing exactly how TCP tracks every byte
- **The Golden Rule** — "My sequence number = peer's last ACK" — proven with real traffic
- **Full Configurations** — Copy-paste ready Cisco NX-OS configs with verification commands
- **RDMA from Scratch** — Zero-copy networking explained with ASCII diagrams, DDP protocol deep dive, and side-by-side comparison with traditional TCP

## 📂 Repository Structure

```
Networking/
└── TCP-IP/
    ├── README.md              ← Topic index
    ├── TCP/
    │   ├── TCP_Connection.md  ← TCP lifecycle (theory + worked examples)
    │   └── Wireshark_3Way_Handshake.md  ← Real packet analysis
    ├── VLAN/
    │   ├── VLAN.md              ← VLAN fundamentals & 802.1Q packet format
    │   └── InterVLAN_Routing.md ← SVI routing on Cisco Nexus 9K
    ├── VXLAN/
    │   ├── Basic_VXLAN_Configuration.pdf
    │   └── VXLAN.pdf
    └── RDMA/
        ├── RDMA_Beginners_Guide.md    ← Complete RDMA tutorial (start here)
        ├── DDP_Deep_Dive.md           ← Direct Data Placement protocol walkthrough
        └── RDMA_vs_Traditional.md     ← Side-by-side comparison with TCP
```

## 🚀 Quick Start

New to networking? Start here:

1. **[TCP Connection Lifecycle](TCP-IP/TCP/TCP_Connection.md)** — Understand how two computers establish a connection, send data, and close
2. **[Wireshark 3-Way Handshake](TCP-IP/TCP/Wireshark_3Way_Handshake.md)** — See a real handshake captured from a laptop connecting to Microsoft
3. **[VLAN Fundamentals](TCP-IP/VLAN/VLAN.md)** — What VLANs are, the problems they solve, and the 802.1Q frame format
4. **[Inter-VLAN Routing](TCP-IP/VLAN/InterVLAN_Routing.md)** — Learn how traffic moves between VLANs on Cisco switches
5. **[RDMA Beginner's Guide](TCP-IP/RDMA/RDMA_Beginners_Guide.md)** — What RDMA is, why it exists, and how it eliminates CPU overhead from networking

## 🧠 Key Concepts at a Glance

```
┌─────────────────────────────────────────────────────┐
│                    OSI Model                         │
├──────────┬──────────────────────────────────────────┤
│ Layer 7  │ Application    (HTTP, DNS, SMTP)         │
│ Layer 4  │ Transport      (TCP, UDP)          ← TCP │
│ Layer 3  │ Network        (IP, ICMP, Routing) ← IP  │
│ Layer 2  │ Data Link      (Ethernet, VLAN, VXLAN)   │
│ Layer 1  │ Physical       (Cables, Wi-Fi, Fiber)    │
└──────────┴──────────────────────────────────────────┘
```

## 📖 References

- [RFC 9293 — TCP](https://www.ietf.org/rfc/rfc9293.html)
- [RFC 7323 — TCP Extensions](https://www.rfc-editor.org/rfc/rfc7323)
- [IEEE 802.1Q — VLANs](https://standards.ieee.org/standard/802_1Q-2018.html)
- [RFC 7348 — VXLAN](https://www.ietf.org/rfc/rfc7348.html)
- [RFC 5040 — Direct Data Placement (DDP)](https://www.rfc-editor.org/rfc/rfc5040)
- [RFC 5042 — RDMA Protocol (RDMAP)](https://www.rfc-editor.org/rfc/rfc5042)
- [Wireshark](https://www.wireshark.org/) — Packet capture tool used in this repo

---

> *Built one packet at a time.* 🛠️
