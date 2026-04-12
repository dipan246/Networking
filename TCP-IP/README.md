# TCP/IP Networking Notes

Study notes and reference materials for TCP/IP networking concepts.

## Contents

| Topic | File | Description |
|-------|------|-------------|
| **TCP Connections** | [TCP_Connection.md](TCP_Connection.md) | Deep dive into TCP connection lifecycle: 3-way handshake, data transfer with worked examples, flow control, retransmission, 4-way close, all 11 TCP states (RFC 9293) |
| **VLAN** | [VLAN/](VLAN/) | Inter-VLAN routing configuration and concepts |
| **VXLAN** | [VXLAN/](VXLAN/) | Virtual Extensible LAN -- overlay networking for data centers |

## Quick Reference

```
OSI Layer 4 (Transport):  TCP, UDP
OSI Layer 3 (Network):    IP, ICMP
OSI Layer 2 (Data Link):  Ethernet, VLAN, VXLAN
```

## References

- [RFC 9293 -- TCP](https://www.ietf.org/rfc/rfc9293.html)
- [RFC 5681 -- TCP Congestion Control](https://www.ietf.org/rfc/rfc5681.html)
- [IEEE 802.1Q -- VLANs](https://standards.ieee.org/standard/802_1Q-2018.html)
- [RFC 7348 -- VXLAN](https://www.ietf.org/rfc/rfc7348.html)
