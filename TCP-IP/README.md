# TCP/IP Networking Notes

Study notes and reference materials for TCP/IP networking concepts.

## Contents

| Topic | Directory | Description |
|-------|-----------|-------------|
| **TCP** | [TCP/](TCP/) | TCP connection lifecycle: 3-way handshake, data transfer with byte-level diagrams, flow control, retransmission, 4-way close, all 11 TCP states (RFC 9293) |
| **VLAN** | [VLAN/](VLAN/) | Inter-VLAN routing configuration and concepts |
| **VXLAN** | [VXLAN/](VXLAN/) | Virtual Extensible LAN -- overlay networking for data centers |

### TCP Deep Dives

| Document | Description |
|----------|-------------|
| [TCP Connection Lifecycle](TCP/TCP_Connection.md) | Theory -- 3-way handshake, sequence numbers, golden rule, byte-level diagrams, flow control, retransmission, 4-way close |
| [Wireshark 3-Way Handshake](TCP/Wireshark_3Way_Handshake.md) | **Real packets** -- Frame-by-frame analysis of a live SYN/SYN+ACK/ACK capture (laptop to Microsoft Bing, port 443) with 7 key insights |

### VLAN

| Document | Description |
|----------|-------------|
| [Inter-VLAN Routing (SVI)](VLAN/InterVLAN_Routing.md) | Cisco Nexus 9K inter-VLAN routing with SVI, transit VLAN, static routes, traffic flow, verification, and troubleshooting |
| [InterVlanRouting_1.pdf](VLAN/InterVlanRouting_1.pdf) | Original topology diagram (PDF) |

## Quick Reference

```
OSI Layer 4 (Transport):  TCP, UDP
OSI Layer 3 (Network):    IP, ICMP
OSI Layer 2 (Data Link):  Ethernet, VLAN, VXLAN
```

## References

- [RFC 9293 -- TCP](https://www.ietf.org/rfc/rfc9293.html)
- [RFC 7323 -- TCP Extensions (Window Scale, Timestamps)](https://www.rfc-editor.org/rfc/rfc7323)
- [RFC 5681 -- TCP Congestion Control](https://www.ietf.org/rfc/rfc5681.html)
- [IEEE 802.1Q -- VLANs](https://standards.ieee.org/standard/802_1Q-2018.html)
- [RFC 7348 -- VXLAN](https://www.ietf.org/rfc/rfc7348.html)
