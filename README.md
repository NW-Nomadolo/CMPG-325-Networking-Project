# CMPG-325 Networking Project: Enterprise LAN Implementation & Verification

**Course:** CMPG 325 Computer Networks  
**Project ID:** CMPG325-2026-106  
**Client:** North-West Agricultural Research Station (Potchefstroom)  
**Prepared By:** Neo William Nomadolo (Student ID: 46428267)  
**Assigned Subnet:** `192.168.47.0/24`  
**GitHub Repository:** [NW-Nomadolo/CMPG-325 Networking Project](https://github.com/NW-Nomadolo/CMPG-325)  

---

## 1. Executive Summary & Overview

This project documents the enterprise-grade Local Area Network (LAN) design, physical/logical implementation, and end-to-end verification for the **North-West Agricultural Research Station (Potchefstroom)**[cite: 1].

Built in Cisco Packet Tracer, the topology implements:
- **Variable Length Subnet Masking (VLSM)** to optimize the single `/24` network allocation[cite: 1].
- **Extended Star Topology** anchored by a central Multilayer Switch and dual Cisco 2911 ISR Core Routers[cite: 1].
- **Router-on-a-Stick Inter-VLAN Routing** with isolated administrative and operational VLANs[cite: 1].
- **Multi-Router Static Path Control** segregating internal routing from edge NAT/internet traffic[cite: 1].
- **Centralized DHCP Relay Services** across VLAN boundaries (`ip helper-address`)[cite: 1].
- **End-to-End Application Testing** verifying Layer 2 switching up through Layer 7 HTTP/DNS resolution[cite: 1].

---

## 2. Network Architecture & Physical Topology

### 2.1 Hardware Specification Matrix
| Device Name | Model / Hardware | Qty | Functional Role |
| :--- | :--- | :---: | :--- |
| **Core-Router-1** | Cisco 2911 ISR | 1 | Edge Router; handles default routing to Cloud-PT Internet and internal static summaries[cite: 1]. |
| **Core-Router-2** | Cisco 2911 ISR | 1 | Internal Gateway; performs Router-on-a-Stick Inter-VLAN routing and DHCP Relay[cite: 1]. |
| **Multilayer-Switch-1** | Cisco Catalyst 3560-24PS | 1 | Central Core Switch; aggregates 802.1Q trunk links from all floor switches[cite: 1]. |
| **Floor1-SW** | Cisco Catalyst 2960 | 1 | Access switch servicing Staff PCs (`PC0`, `PC1`)[cite: 1]. |
| **Floor2-SW** | Cisco Catalyst 2960 | 1 | Access switch servicing Research Labs (`PC2`), Central DHCP Server, and DB Server[cite: 1]. |
| **CR2-SW** | Cisco Catalyst 2960 | 1 | Access switch servicing Change Request 2 expansion floor (`PC3`, `PC4`)[cite: 1]. |
| **Mgmt-SW** | Cisco Catalyst 2960 | 1 | Out-of-band management switch servicing administrative hardware (`Laptop0`)[cite: 1]. |

### 2.2 Physical Topology Diagram
<img src="Physical Topology" alt="Physical Topology Diagram" width="100%">

---

## 3. Implemented VLSM IP Addressing Scheme

Base Assigned Network: `192.168.47.0/24`[cite: 1]

| Department / Purpose | VLAN | Subnet Address | CIDR | Subnet Mask | Usable Host Range | Default Gateway |
| :--- | :---: | :--- | :---: | :--- | :--- | :--- |
| **Management** | `10` | `192.168.47.0` | `/27` | `255.255.255.224` | `192.168.47.1 – .30` | `192.168.47.1` |
| **Staff Network** | `20` | `192.168.47.32` | `/26` | `255.255.255.192` | `192.168.47.33 – .94` | `192.168.47.33` |
| **Research Labs** | `30` | `192.168.47.96` | `/26` | `255.255.255.192` | `192.168.47.97 – .158` | `192.168.47.97` |
| **CR2 Expansion Floor** | `40` | `192.168.47.160` | `/27` | `255.255.255.224` | `192.168.47.161 – .190` | `192.168.47.161` |
| **Infrastructure / Mgmt** | `99` | `192.168.47.192` | `/28` | `255.255.255.240` | `192.168.47.193 – .206` | `192.168.47.193` |
| **Point-to-Point WAN Link**| `N/A`| `192.168.47.208` | `/30` | `255.255.255.252` | `192.168.47.209 – .210` | `N/A` |

---

## 4. Technical Configuration Highlights

### 4.1 Core Static Routing Integration
- **Core-Router-2 (Outbound Flow):** Configured with a default static route directing all non-local outbound traffic across the `/30` WAN link toward `192.168.47.209` (Core-Router-1)[cite: 1].
- **Core-Router-1 (Inbound Flow):** Configured with a summary static route `192.168.47.0/24` pointing back to `192.168.47.210` (Core-Router-2) for inbound reachability[cite: 1]. External traffic routes via `GigabitEthernet0/0` to ISP `10.0.0.2`[cite: 1].

### 4.2 Change Request 2 (CR2 Expansion) & DHCP Relay
- **Sub-Interface Provisioning:** Created `GigabitEthernet0/0.40` on **Core-Router-2** configured for 802.1Q encapsulation on VLAN 40 (`192.168.47.161/27`)[cite: 1].
- **DHCP Relay Service:** Configured `ip helper-address 192.168.47.35` on sub-interface `G0/0.40`[cite: 1]. Broadcast `DHCPDISCOVER` frames from VLAN 40 hosts are forwarded as unicast requests to the centralized DHCP server on VLAN 20[cite: 1].

---

## 5. Test Verification & Results Analysis

### 5.1 Test Summary Matrix
| Test ID | System / Function Tested | Input Command / Action | Observed Output / Status | Verdict |
| :---: | :--- | :--- | :--- | :---: |
| **TC-01** | Static Path Control Audit | `show ip route` on Core Routers | Active route to `192.168.47.0/24` via `.210` on R1; Default route `0.0.0.0/0` via `.209` on R2[cite: 1]. | **PASSED** |
| **TC-02** | DHCP Relay Allocation | Toggled IP Config to DHCP on Laptop0 | Received IP `192.168.47.163` (`/27`), GW `192.168.47.161`, DNS `192.168.47.66`[cite: 1]. | **PASSED** |
| **TC-03a**| Local Gateway ICMP | `ping 192.168.47.161` | 4/4 replies received, <1ms response time, TTL 255[cite: 1]. | **PASSED** |
| **TC-03b**| Inter-VLAN ICMP Reachability| `ping 192.168.47.1` | 4/4 replies received, Inter-VLAN routing confirmed across sub-interfaces[cite: 1]. | **PASSED** |
| **TC-03c**| Multi-Hop Static WAN Link | `ping 192.168.47.209` | 3/4 replies received (initial packet loss due to ARP resolution), TTL 254[cite: 1]. | **PASSED** |
| **TC-04** | Application Layer Integration| Web Browser: `http://cisco.com` | Resolved domain via DNS (`192.168.47.66`) and retrieved HTTP landing page[cite: 1]. | **PASSED** |

---

### 5.2 Verification Screenshots

#### Test 1: Static Routing Table Audit
*Core-Router-1 CLI:*
<img src="Static Routing Table Audit test (Core router 1 CLI)" alt="Core Router 1 Routing Audit" width="100%">

*Core-Router-2 CLI:*
<img src="Static Routing Table Audit test (Core router CLI)" alt="Core Router 2 Routing Audit" width="100%">

---

#### Test 2: Dynamic IP Address Allocation (DHCP Relay)
<img src="Dynamic IP Address Allocation TEST" alt="DHCP Relay Verification" width="100%">

---

#### Test 3: Local & Inter-VLAN Gateway ICMP Reachability
<img src="Local %26 Inter-VLAN Gateway ICMP Reachability TEST" alt="ICMP Ping Verification" width="100%">

---

#### Test 4: Application Layer Services (DNS & HTTP)
<img src="Application Layer TEST" alt="Application Layer Web Verification" width="100%">
