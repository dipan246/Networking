# Inter-VLAN Routing with SVI (Interface VLAN)

> **Platform**: Cisco Nexus 9000 series  
> **Method**: Layer 3 SVI (Switched Virtual Interface) + Transit VLAN  
> **Goal**: Enable routing between hosts in different VLANs across two switches  

---

## Table of Contents

1. [What is Inter-VLAN Routing?](#1-what-is-inter-vlan-routing)
2. [Network Topology](#2-network-topology)
3. [IP Addressing Plan](#3-ip-addressing-plan)
4. [How Traffic Flows](#4-how-traffic-flows)
5. [Configuration](#5-configuration)
6. [Verification Commands](#6-verification-commands)
7. [Key Concepts Explained](#7-key-concepts-explained)
8. [Common Troubleshooting](#8-common-troubleshooting)
9. [Alternative Methods](#9-alternative-methods)

---

## 1. What is Inter-VLAN Routing?

By default, VLANs **isolate** broadcast domains — hosts in VLAN 20 cannot talk to hosts in VLAN 30. To enable communication between VLANs, you need a **Layer 3 device** (router or L3 switch) to route packets between them.

### Three Common Methods

| Method | Device | Use Case |
|--------|--------|----------|
| **Router-on-a-Stick** | External router with sub-interfaces | Small networks, legacy setups |
| **SVI (Interface VLAN)** | L3 switch with virtual interfaces | **Most common in modern data centers** |
| **Routed Ports** | L3 switch physical ports as routed | Point-to-point L3 links |

This document covers **SVI-based inter-VLAN routing** — the standard approach on Cisco Nexus switches.

---

## 2. Network Topology

```
                        VLAN 40 (Transit Link)
                     192.168.40.0/24
                    Eth1/34 ←────────→ Eth1/34
  ┌──────────────┐                          ┌──────────────┐
  │              │                          │              │
  │  Cisco N9K   │                          │  Cisco N9K   │
  │    SW1       │                          │    SW2       │
  │              │                          │              │
  │ SVI VLAN 20  │                          │ SVI VLAN 30  │
  │ 192.168.20.1 │                          │ 192.168.30.1 │
  │              │                          │              │
  │ SVI VLAN 40  │                          │ SVI VLAN 40  │
  │ 192.168.40.1 │                          │ 192.168.40.2 │
  │              │                          │              │
  └──────┬───────┘                          └──────┬───────┘
         │ Eth1/33                                 │ Eth1/33
         │                                         │
         │ VLAN 20                                 │ VLAN 30
         │ (Access Port)                           │ (Access Port)
         │                                         │
  ┌──────┴───────┐                          ┌──────┴───────┐
  │  Server-1    │                          │  Server-2    │
  │              │                          │              │
  │ 192.168.20.49│                          │ 192.168.30.47│
  │  /24         │                          │  /24         │
  │              │                          │              │
  │ GW:          │                          │ GW:          │
  │ 192.168.20.1 │                          │ 192.168.30.1 │
  └──────────────┘                          └──────────────┘
```

### Why 3 VLANs?

| VLAN | Subnet | Purpose |
|------|--------|---------|
| **VLAN 20** | 192.168.20.0/24 | Server-1's network (on SW1) |
| **VLAN 30** | 192.168.30.0/24 | Server-2's network (on SW2) |
| **VLAN 40** | 192.168.40.0/24 | **Transit VLAN** — interconnects SW1 and SW2 at Layer 3 |

> **VLAN 40 is the bridge**: It creates a Layer 3 path between the two switches so they can route traffic for each other's VLANs.

---

## 3. IP Addressing Plan

| Device | Interface | IP Address | VLAN | Role |
|--------|-----------|------------|------|------|
| **SW1** | SVI VLAN 20 | 192.168.20.1/24 | 20 | Default gateway for Server-1 |
| **SW1** | SVI VLAN 40 | 192.168.40.1/24 | 40 | Transit link (SW1 side) |
| **SW2** | SVI VLAN 30 | 192.168.30.1/24 | 30 | Default gateway for Server-2 |
| **SW2** | SVI VLAN 40 | 192.168.40.2/24 | 40 | Transit link (SW2 side) |
| **Server-1** | NIC | 192.168.20.49/24 | 20 | Host |
| **Server-2** | NIC | 192.168.30.47/24 | 30 | Host |

---

## 4. How Traffic Flows

### Server-1 (192.168.20.49) pings Server-2 (192.168.30.47)

```
Step-by-step packet journey:

 Server-1                SW1                    SW2                Server-2
 192.168.20.49           192.168.40.1           192.168.40.2       192.168.30.47
     │                     │                      │                    │
     │  1. "Dest is not    │                      │                    │
     │  in my subnet       │                      │                    │
     │  (20.0/24 vs 30.47) │                      │                    │
     │  → send to default  │                      │                    │
     │  gateway 20.1"      │                      │                    │
     │                     │                      │                    │
     │ ICMP Echo Request   │                      │                    │
     │ src=20.49 dst=30.47 │                      │                    │
     │────────────────────>│                      │                    │
     │  (VLAN 20, Eth1/33) │                      │                    │
     │                     │                      │                    │
     │                     │  2. SW1 routes:      │                    │
     │                     │  "30.0/24 → next-hop │                    │
     │                     │   192.168.40.2"      │                    │
     │                     │  (static route)      │                    │
     │                     │                      │                    │
     │                     │ ICMP Echo Request    │                    │
     │                     │ src=20.49 dst=30.47  │                    │
     │                     │─────────────────────>│                    │
     │                     │  (VLAN 40, Eth1/34)  │                    │
     │                     │                      │                    │
     │                     │                      │  3. SW2 routes:   │
     │                     │                      │  "30.0/24 is      │
     │                     │                      │   directly        │
     │                     │                      │   connected       │
     │                     │                      │   on VLAN 30"     │
     │                     │                      │                   │
     │                     │                      │ ICMP Echo Request │
     │                     │                      │ src=20.49 dst=30.47
     │                     │                      │──────────────────>│
     │                     │                      │ (VLAN 30, Eth1/33)│
     │                     │                      │                   │
     │                     │                      │    4. Server-2    │
     │                     │                      │    receives ping! │
```

### What Happens at Each Hop

| Step | Device | Action | Source MAC | Dest MAC |
|------|--------|--------|-----------|----------|
| 1 | Server-1 | Sends to default GW (192.168.20.1) | Server-1 MAC | SW1 VLAN 20 MAC |
| 2 | SW1 | Routes via static route → next-hop 192.168.40.2 | SW1 VLAN 40 MAC | SW2 VLAN 40 MAC |
| 3 | SW2 | Delivers to local VLAN 30 | SW2 VLAN 30 MAC | Server-2 MAC |

> **Key insight**: The **IP addresses never change** during the journey (src=20.49, dst=30.47 throughout). Only the **MAC addresses change at each L3 hop** — same principle we saw in the [Wireshark analysis](../TCP/Wireshark_3Way_Handshake.md#insight-2-mac-addresses-never-leave-your-lan)!

---

## 5. Configuration

### SW1 Configuration

```cisco
! Enable IP routing (required for inter-VLAN routing)
feature interface-vlan
ip routing

! Create VLANs
vlan 20
  name SERVER-VLAN-20
vlan 40
  name TRANSIT-LINK

! SVI for Server-1's VLAN — acts as default gateway
interface vlan 20
  description Default-GW-for-VLAN20-Servers
  ip address 192.168.20.1/24
  no shutdown

! SVI for transit link to SW2
interface vlan 40
  description Transit-to-SW2
  ip address 192.168.40.1/24
  no shutdown

! Access port for Server-1
interface Ethernet1/33
  description To-Server-1
  switchport mode access
  switchport access vlan 20
  no shutdown

! Trunk/Access port for transit to SW2
interface Ethernet1/34
  description Transit-to-SW2
  switchport mode access
  switchport access vlan 40
  no shutdown

! Static route: "To reach VLAN 30, go through SW2"
ip route 192.168.30.0/24 192.168.40.2
```

### SW2 Configuration

```cisco
! Enable IP routing
feature interface-vlan
ip routing

! Create VLANs
vlan 30
  name SERVER-VLAN-30
vlan 40
  name TRANSIT-LINK

! SVI for Server-2's VLAN — acts as default gateway
interface vlan 30
  description Default-GW-for-VLAN30-Servers
  ip address 192.168.30.1/24
  no shutdown

! SVI for transit link to SW1
interface vlan 40
  description Transit-to-SW1
  ip address 192.168.40.2/24
  no shutdown

! Access port for Server-2
interface Ethernet1/33
  description To-Server-2
  switchport mode access
  switchport access vlan 30
  no shutdown

! Trunk/Access port for transit to SW1
interface Ethernet1/34
  description Transit-to-SW1
  switchport mode access
  switchport access vlan 40
  no shutdown

! Static route: "To reach VLAN 20, go through SW1"
ip route 192.168.20.0/24 192.168.40.1
```

### Server Configuration

```bash
# Server-1 (Linux)
sudo ip addr add 192.168.20.49/24 dev eth0
sudo ip route add default via 192.168.20.1

# Server-2 (Linux)
sudo ip addr add 192.168.30.47/24 dev eth0
sudo ip route add default via 192.168.30.1
```

---

## 6. Verification Commands

### On Switches

```cisco
! Verify SVIs are up
show interface vlan 20 brief
show interface vlan 30 brief
show interface vlan 40 brief

! Verify routing table
show ip route

! Expected output on SW1:
!   C    192.168.20.0/24  is directly connected, Vlan20
!   C    192.168.40.0/24  is directly connected, Vlan40
!   S    192.168.30.0/24  [1/0] via 192.168.40.2

! Expected output on SW2:
!   C    192.168.30.0/24  is directly connected, Vlan30
!   C    192.168.40.0/24  is directly connected, Vlan40
!   S    192.168.20.0/24  [1/0] via 192.168.40.1

! Verify connectivity
ping 192.168.40.2 source 192.168.40.1    ! Transit link
ping 192.168.30.47 source 192.168.20.1   ! End-to-end

! Verify ARP
show ip arp vlan 20
show ip arp vlan 40

! Verify MAC address table
show mac address-table vlan 20
show mac address-table vlan 40
```

### On Servers

```bash
# Test connectivity
ping 192.168.30.47    # From Server-1
ping 192.168.20.49    # From Server-2

# Trace the path
traceroute 192.168.30.47    # From Server-1
# Expected:
#  1  192.168.20.1   (SW1 - default gateway)
#  2  192.168.40.2   (SW2 - transit hop)
#  3  192.168.30.47  (Server-2 - destination)
```

---

## 7. Key Concepts Explained

### What is an SVI (Switched Virtual Interface)?

An SVI is a **virtual Layer 3 interface** associated with a VLAN. It gives the VLAN an IP address, enabling the switch to:
- Act as the **default gateway** for hosts in that VLAN
- **Route packets** between VLANs (inter-VLAN routing)
- Manage the switch via that VLAN (management SVI)

```
          ┌─────────────────────────┐
          │      L3 Switch          │
          │                         │
          │  ┌─────────────────┐    │
          │  │  Routing Engine │    │
          │  │  (ip routing)   │    │
          │  └──┬──────────┬───┘    │
          │     │          │        │
          │  ┌──┴───┐  ┌───┴──┐    │
          │  │SVI 20│  │SVI 30│    │  ← Virtual L3 interfaces
          │  │.20.1 │  │.30.1 │    │
          │  └──┬───┘  └───┬──┘    │
          │     │          │        │
          │  ┌──┴───┐  ┌───┴──┐    │
          │  │VLAN20│  │VLAN30│    │  ← L2 broadcast domains
          │  │ports │  │ports │    │
          │  └──┬───┘  └───┬──┘    │
          └─────┼──────────┼───────┘
                │          │
            Server-1   Server-2
```

### Why a Transit VLAN?

When two L3 switches need to exchange routes, they need a **common Layer 3 subnet** between them. VLAN 40 serves as this "transit network":

```
SW1 (192.168.40.1) ──── VLAN 40 ──── SW2 (192.168.40.2)
                     192.168.40.0/24
                     (Transit Subnet)
```

Without VLAN 40, SW1 has no next-hop address to reach SW2 and vice versa.

### Static Routes Explained

```
On SW1:  ip route 192.168.30.0/24 192.168.40.2
         ─────────────────────────  ─────────────
         "For any packet going      "Forward it to
          to the 30.0 network..."    SW2 at 40.2"

On SW2:  ip route 192.168.20.0/24 192.168.40.1
         ─────────────────────────  ─────────────
         "For any packet going      "Forward it to
          to the 20.0 network..."    SW1 at 40.1"
```

> In production, you would use a **dynamic routing protocol** (OSPF, BGP, EIGRP) instead of static routes.

---

## 8. Common Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Cannot ping default GW | SVI is down or wrong VLAN | `show interface vlan X` — check status is `up/up` |
| Ping GW works, but not remote server | Missing static route | `show ip route` — verify route to remote subnet exists |
| One-way connectivity | Route missing on one switch | Add return route on the other switch |
| SVI shows "down/down" | No active port in that VLAN | Ensure at least 1 port is `switchport access vlan X` and up |
| "ip routing" not working | Feature not enabled | Run `feature interface-vlan` on Nexus |
| Intermittent connectivity | ARP timeout or duplicate IP | `show ip arp` — look for conflicts |

### Debugging Steps

```cisco
! Step 1: Verify L2 — can server reach its gateway?
ping 192.168.20.1        ! From Server-1

! Step 2: Verify transit — can switches reach each other?
ping 192.168.40.2 source 192.168.40.1    ! From SW1

! Step 3: Verify routing — does the switch know the path?
show ip route 192.168.30.0

! Step 4: Verify end-to-end
ping 192.168.30.47 source 192.168.20.1
traceroute 192.168.30.47
```

---

## 9. Alternative Methods

### Method 1: Router-on-a-Stick

Uses a single physical router with **sub-interfaces**, one per VLAN. The switch trunk port carries all VLANs to the router.

```
            Router
          ┌────────┐
          │ Gi0/0  │
          │  .10   │ ← Sub-interface per VLAN
          │  .20   │
          │  .30   │
          └───┬────┘
              │ Trunk (802.1Q)
          ┌───┴────┐
          │ Switch │
          │        │
          └─┬──┬───┘
            │  │
          V10 V20 ← Access ports
```

**Pros**: Works with L2-only switches  
**Cons**: Router becomes bottleneck (all inter-VLAN traffic goes through one link)

### Method 2: Routed Ports (L3 Point-to-Point)

Convert switch ports to routed interfaces (no switchport):

```cisco
interface Ethernet1/34
  no switchport
  ip address 192.168.40.1/24
```

**Pros**: No VLAN needed for transit  
**Cons**: Uses a physical port per L3 link

### Comparison

| Feature | SVI (this doc) | Router-on-a-Stick | Routed Ports |
|---------|---------------|-------------------|--------------|
| Performance | High (hardware routing) | Low (router bottleneck) | High |
| Scalability | Excellent | Poor | Good |
| Complexity | Medium | Low | Low |
| Requires L3 switch | Yes | No | Yes |
| Transit VLAN needed | Yes | No | No |

---

## References

- [Cisco Nexus 9000 — Configuring Layer 3 Interfaces](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/interfaces/configuration/guide/b_Cisco_Nexus_9000_Series_NX-OS_Interfaces_Configuration_Guide_7x/b_Cisco_Nexus_9000_Series_NX-OS_Interfaces_Configuration_Guide_7x_chapter_010.html)
- [Inter-VLAN Routing on Cisco Switches](https://www.cisco.com/c/en/us/support/docs/lan-switching/inter-vlan-routing/41260-interVLAN-routing.html)
- [IEEE 802.1Q — VLAN Tagging](https://standards.ieee.org/standard/802_1Q-2018.html)

---

> *Redesigned from original PDF diagram. Covers Cisco Nexus 9K inter-VLAN routing with SVI, transit VLAN, static routes, and full verification workflow.*
