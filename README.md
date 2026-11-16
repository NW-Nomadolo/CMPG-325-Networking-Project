# CMPG 325 Networking Project: Small-Enterprise LAN Design & Verification

**Course:** CMPG 325 Computer Networks (NWU)
**Client (case study):** North-West Agricultural Research Station (Potchefstroom)
**Prepared by:** Neo William Nomadolo
**Assigned address block:** `192.168.47.0/24`
**Tool:** Cisco Packet Tracer (simulation only, no physical hardware)

---

## 1. Overview

This project designs, builds and tests a segmented LAN for a case-study client with executive, staff, research lab, expansion-floor and management areas. Everything was built and verified in Cisco Packet Tracer.

What the design implements:

- **VLAN segmentation** with 802.1Q trunking between the core switch and access switches
- **Router-on-a-stick inter-VLAN routing** on Core-Router-2
- **VLSM addressing** carved from one `/24` block
- **Two-router core with static routing** in both directions (internal gateway and edge router)
- **DHCP relay** (`ip helper-address`) so hosts on a new floor receive addresses from a central DHCP server on another VLAN
- **A dedicated management VLAN (VLAN 99)** for switch and router administration
- **End-to-end checks** from Layer 2 switching through DNS and HTTP at Layer 7

---

## 2. Network Architecture & Physical Topology

### 2.1 Hardware

| Device | Model | Role |
| :--- | :--- | :--- |
| Core-Router-1 | Cisco 2911 ISR | Edge router: default route to the simulated Internet (Cloud-PT) and a summary route back to the internal network |
| Core-Router-2 | Cisco 2911 ISR | Internal gateway: router-on-a-stick inter-VLAN routing and DHCP relay |
| Multilayer-Switch-1 | Cisco Catalyst 3560-24PS | Core switch: aggregates 802.1Q trunks from the access switches |
| Floor1-SW | Cisco Catalyst 2960 | Access switch for staff PCs (`PC0`, `PC1`) |
| Floor2-SW | Cisco Catalyst 2960 | Access switch for the research PC (`PC2`), the central DHCP server and the research database server |
| CR2-SW | Cisco Catalyst 2960 | Access switch for the Change Request 2 expansion floor (`PC3`, `PC4`) |
| Mgmt-SW | Cisco Catalyst 2960 | Management switch for administrative access (`Laptop0`) |

[CONFIRM: say where Laptop0 was connected when it was used for TC-02 below. If it was moved to CR2-SW for that test, write that here. Delete this line when done.]

### 2.2 Physical Topology

<img src="Physical Topology" alt="Physical Topology Diagram" width="100%">

The network uses an extended star: the Catalyst 3560 sits in the centre and each access switch connects back to it with one trunk link. Adding a floor means adding one switch and one trunk, without changing the existing access switches.

---

## 3. VLSM IP Addressing Plan

Base network: `192.168.47.0/24`

| Department / Purpose | VLAN | Subnet | CIDR | Subnet Mask | Usable Hosts | Default Gateway |
| :--- | :---: | :--- | :---: | :--- | :--- | :--- |
| Management (executive) | 10 | `192.168.47.0` | /27 | `255.255.255.224` | `.1 - .30` | `192.168.47.1` |
| Staff | 20 | `192.168.47.32` | /27 | `255.255.255.224` | `.33 - .62` | `192.168.47.33` |
| Research Labs | 30 | `192.168.47.64` | /27 | `255.255.255.224` | `.65 - .94` | `192.168.47.65` |
| CR2 Expansion Floor | 40 | `192.168.47.160` | /27 | `255.255.255.224` | `.161 - .190` | `192.168.47.161` |
| Infrastructure / Management | 99 | `192.168.47.192` | /28 | `255.255.255.240` | `.193 - .206` | `192.168.47.193` |
| Router-to-router WAN link | n/a | `192.168.47.208` | /30 | `255.255.255.252` | `.209 - .210` | n/a |

`192.168.47.96 - .159` is left unallocated for future growth.

Key servers: central DHCP server `192.168.47.35` (VLAN 20), DNS server `192.168.47.66`.

---

## 4. Configuration Highlights

### 4.1 Static routing across two routers

- **Core-Router-2 (internal gateway):** default route toward Core-Router-1 across the `/30` link.
  `ip route 0.0.0.0 0.0.0.0 192.168.47.209`
- **Core-Router-1 (edge):** summary route back to the internal network, plus a default route to the simulated ISP.
  `ip route 192.168.47.0 255.255.255.0 192.168.47.210`
  `ip route 0.0.0.0 0.0.0.0 10.0.0.2`

### 4.2 Change Request 2: new floor and DHCP relay

- A new sub-interface for VLAN 40 was created on Core-Router-2 (`GigabitEthernet0/1.40`, `192.168.47.161/27`, 802.1Q encapsulation `dot1Q 40`).
- `ip helper-address 192.168.47.35` on that sub-interface relays DHCP broadcasts from VLAN 40 to the central DHCP server on VLAN 20 as unicast.

```
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 192.168.47.161 255.255.255.224
 ip helper-address 192.168.47.35
```

Full device configurations are in [`configs/`](configs/).

---

## 5. Test Verification

### 5.1 Summary

| ID | Function tested | Action | Observed result | Verdict |
| :---: | :--- | :--- | :--- | :---: |
| TC-01 | Static routing | `show ip route` on both core routers | R1 has `192.168.47.0/24` via `.210`; R2 has default route `0.0.0.0/0` via `.209` | Passed |
| TC-02 | DHCP relay | Switched Laptop0 from static to DHCP | Received `192.168.47.163/27`, gateway `192.168.47.161`, DNS `192.168.47.66` | Passed |
| TC-03a | Local gateway | `ping 192.168.47.161` | 4/4 replies, under 1 ms, TTL 255 | Passed |
| TC-03b | Inter-VLAN gateway reachability | `ping 192.168.47.1` from a VLAN 40 host | 4/4 replies, TTL 255 | Passed |
| TC-03c | Multi-hop WAN link | `ping 192.168.47.209` | 3/4 replies (first request timed out during ARP resolution), TTL 254 | Passed |
| TC-04 | DNS and HTTP | Browse to `http://cisco.com` | Name resolved via `192.168.47.66`; Packet Tracer landing page loaded | Passed |

### 5.2 Evidence

#### Test 1: Static Routing Table Audit*Core-Router-1 CLI:*<img src="Static Routing Table Audit test (Core router 1 CLI)" alt="Core Router 1 Routing Audit" width="100%">

*Core-Router-2 CLI:*<img src="Static Routing Table Audit test (Core router CLI)" alt="Core Router 2 Routing Audit" width="100%">

#### Test 2: Dynamic IP Address Allocation (DHCP Relay)<img src="Dynamic IP Address Allocation TEST" alt="DHCP Relay Verification" width="100%">

#### Test 3: Local & Inter-VLAN Gateway ICMP Reachability<img src="Local %26 Inter-VLAN Gateway ICMP Reachability TEST" alt="ICMP Ping Verification" width="100%">

#### Test 4: Application Layer Services (DNS & HTTP)<img src="Application Layer TEST" alt="Application Layer Web Verification" width="100%">

---

## 6. Troubleshooting & Lessons Learned

1. **Trunk did not allow the new VLAN.** Pings from the CR2 floor failed at first. The cause was that VLAN 40 was not permitted on the core switch's uplink trunk. Allowing it fixed the problem:
   `switchport trunk allowed vlan 10,20,30,40,99`
2. **First ping to the edge router always dropped.** Packet Tracer's simulation mode showed this was ARP resolving on the `/30` link, not a routing fault. Later pings returned normally.
3. **Documentation must match the running config.** Checking the address plan against `show ip route` showed the subnet table had to match what the routers actually run. Always verify documentation against the device output.

---

## 7. Scope & Limitations

- This is a **simulation** built for a university assignment, not a deployed network.
- The tests above cover routing, DHCP relay, reachability and DNS/HTTP. **Not tested here:** ACL filtering between VLANs, NAT, QoS, and whether the management VLAN blocks access from user VLANs.
- **Next steps:** add ACLs to block user VLANs from reaching VLAN 99, test them (a staff PC pinging `192.168.47.193` should fail), and add NAT on the edge router with `show ip nat translations` as evidence.

---
