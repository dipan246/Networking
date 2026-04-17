# TCP/IP Networking Notes

Study notes and reference materials for TCP/IP networking concepts.

## Contents

| Topic | Directory | Description |
|-------|-----------|-------------|
| **TCP** | [TCP/](TCP/) | TCP connection lifecycle: 3-way handshake, data transfer with byte-level diagrams, flow control, retransmission, 4-way close, all 11 TCP states (RFC 9293) |
| **VLAN** | [VLAN/](VLAN/) | Inter-VLAN routing configuration and concepts |
| **VXLAN** | [VXLAN/](VXLAN/) | Virtual Extensible LAN -- overlay networking for data centers |
| **RDMA** | [RDMA/](RDMA/) | Remote Direct Memory Access -- zero-copy, kernel-bypass networking with DDP protocol deep dive |

### TCP Deep Dives

| Document | Description |
|----------|-------------|
| [TCP Connection Lifecycle](TCP/TCP_Connection.md) | Theory -- 3-way handshake, sequence numbers, golden rule, byte-level diagrams, flow control, retransmission, 4-way close |
| [Wireshark 3-Way Handshake](TCP/Wireshark_3Way_Handshake.md) | **Real packets** -- Frame-by-frame analysis of a live SYN/SYN+ACK/ACK capture (laptop to Microsoft Bing, port 443) with 7 key insights |

### VLAN

| Document | Description |
|----------|-------------|
| [VLAN Fundamentals (802.1Q)](VLAN/VLAN.md) | What is a VLAN, the problems it solves, IEEE 802.1Q packet format with bit-level breakdown, tagged vs untagged frames, access vs trunk ports, Q-in-Q |
| [Inter-VLAN Routing (SVI)](VLAN/InterVLAN_Routing.md) | Cisco Nexus 9K inter-VLAN routing with SVI, transit VLAN, static routes, traffic flow, verification, and troubleshooting |

### RDMA

| Document | Description |
|----------|-------------|
| [RDMA Beginner's Guide](RDMA/RDMA_Beginners_Guide.md) | Complete tutorial -- what RDMA is, three flavors (InfiniBand, RoCE, iWARP), operations (Send/Write/Read/Atomic), queue pairs, memory registration, security model, and real-world use cases |
| [DDP Deep Dive](RDMA/DDP_Deep_Dive.md) | Direct Data Placement protocol walkthrough -- tagged vs untagged buffers, segment format byte-by-byte, message fragmentation, worked examples with RDMA Write and Send |
| [RDMA vs Traditional Networking](RDMA/RDMA_vs_Traditional.md) | Side-by-side architectural comparison -- data path step-by-step, memory copy visualization, kernel bypass explained, when to use RDMA vs TCP |

## Quick Reference

```
OSI Layer 4 (Transport):  TCP, UDP
OSI Layer 3 (Network):    IP, ICMP
OSI Layer 2 (Data Link):  Ethernet, VLAN, VXLAN
RDMA (cross-layer):       InfiniBand, RoCE v2 (L2-L4), iWARP (L4-L7)
```

## References

- [RFC 9293 -- TCP](https://www.ietf.org/rfc/rfc9293.html)
- [RFC 7323 -- TCP Extensions (Window Scale, Timestamps)](https://www.rfc-editor.org/rfc/rfc7323)
- [RFC 5681 -- TCP Congestion Control](https://www.ietf.org/rfc/rfc5681.html)
- [IEEE 802.1Q -- VLANs](https://standards.ieee.org/standard/802_1Q-2018.html)
- [RFC 7348 -- VXLAN](https://www.ietf.org/rfc/rfc7348.html)
- [RFC 5040 -- Direct Data Placement (DDP)](https://www.rfc-editor.org/rfc/rfc5040)
- [RFC 5042 -- RDMA Protocol (RDMAP)](https://www.rfc-editor.org/rfc/rfc5042)
- [RFC 5044 -- MPA Framing for TCP](https://www.rfc-editor.org/rfc/rfc5044)
