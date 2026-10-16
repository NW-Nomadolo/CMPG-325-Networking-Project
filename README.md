# CMPG-325 Networking Project: Enterprise LAN Final Technical Design & Implementation

**Course:** CMPG 325 Computer Networks  
**Project ID:** CMPG325-2026-106  
**Client:** North-West Agricultural Research Station (Potchefstroom)  
**Prepared By:** Neo William Nomadolo (Student ID: 46428267)  
**Assigned Subnet:** `192.168.47.0/24`  
**Date:** 16 October 2026  
**GitHub Repository:** [NW-Nomadolo/CMPG-325-Networking-Project](https://github.com/NW-Nomadolo/CMPG-325-Networking-Project)  

---

## 1. Executive Summary & Client Requirement Analysis

This repository documents the complete technical design, physical implementation, and empirical verification for the modernized network at the **North-West Agricultural Research Station (Potchefstroom)**[cite: 10]. Designed and simulated in Cisco Packet Tracer, the network satisfies five core operational mandates[cite: 10]:

### 1.1 IP Addressing Efficiency & Minimal Wastage
- **Client Need:** Fit all station departments into a single assigned IPv4 block (`192.168.47.0/24`) without address exhaustion or wastage[cite: 10].
- **Implemented Solution:** Variable Length Subnet Masking (VLSM)[cite: 10]. Tailored subnets range from `/26` (62 usable hosts for Staff and Research) and `/27` (30 usable hosts for Management and CR2 Expansion) down to `/28` (14 usable hosts for Infrastructure) and `/30` (2 hosts for point-to-point links)[cite: 10].
- **Technical Advantage:** Prevents over 200 wasted IPv4 addresses compared to traditional classful allocations while containing broadcast domains at the subnet level[cite: 10].

### 1.2 Unrestricted Executive Internet Path & Security Isolation
- **Client Need:** Executive Management (VLAN 10) requires dedicated, high-priority, uninterrupted internet access isolated from staff bandwidth throttling or scheduled blackouts[cite: 10].
- **Implemented Solution:** Logical VLAN segmentation paired with policy-based outbound ACL rules on Core-Router-2 and Core-Router-1 that bypass staff filters[cite: 10].
- **Technical Advantage:** Eliminates bandwidth competition between heavy background staff traffic and executive operations without requiring physical cable re-routing[cite: 10].

### 1.3 Multi-Router Path Control & Static Routing Challenge
- **Client Need:** Demonstrate multi-hop path control across multiple core routers using explicit static routing[cite: 10].
- **Implemented Solution:** Dual-router core implementation consisting of **Core-Router-1** (Perimeter Edge/NAT/ISP Gateway) and **Core-Router-2** (Internal Gateway/Inter-VLAN Routing) interconnected via a `/30` point-to-point WAN link (`192.168.47.208/30`)[cite: 10].
- **Technical Advantage:** Distributes processing strain across separate router CPUs, eliminates dynamic routing protocol overhead, and hardens the core against routing table poisoning[cite: 10].

### 1.4 Change Request Integration (CR2 Expansion Floor)
- **Client Need:** Integrate a newly acquired building floor (CR2 Expansion) into the operational network without service interruption or IP re-addressing[cite: 10].
- **Implemented Solution:** Extended Star Topology expansion[cite: 10]. Provisioned VLAN 40 (`192.168.47.160/27`), connected a new access switch (CR2-SW) via an 802.1Q trunk to the central core switch, and assigned sub-interface `G0/0.40` on Core-Router-2[cite: 10].
- **Technical Advantage:** Enables modular "plug-and-play" infrastructure growth without modifying existing access layer switch configurations[cite: 10].

### 1.5 Infrastructure Protection & Out-of-Band Management
- **Client Need:** Protect switch and router administrative interfaces against unauthorized user access or packet interception[cite: 10].
- **Implemented Solution:** Native/Infrastructure Management Isolation (VLAN 99)[cite: 10]. All Switch Virtual Interfaces (SVIs) and administrative sessions are bound exclusively to VLAN 99 (`192.168.47.192/28`) on dedicated switch `Mgmt-SW`[cite: 10].
- **Technical Advantage:** Renders management interfaces invisible and inaccessible from standard employee workstations, enforcing Zero-Trust security[cite: 10].

---

## 2. Hardware Specification & Physical Topology Architecture

### 2.1 Hardware Specification Matrix
| Device Name | Hardware Model | Qty | Physical Ports & Media | Functional Role in Architecture |
| :--- | :--- | :---: | :--- | :--- |
| **Core Router 1** | Cisco 2911 ISR | 1 | GigabitEthernet / Serial WAN | Perimeter Edge Router; interfaces directly with ISP Cloud, handling outbound NAT and default routes[cite: 10]. |
| **Core Router 2** | Cisco 2911 ISR | 1 | GigabitEthernet | Internal Gateway; handles Router-on-a-Stick Inter-VLAN routing and cross-VLAN DHCP relaying[cite: 10]. |
| **Core Switch** | Cisco Catalyst 3560 (L2/L3) | 1 | GigabitEthernet (802.1Q) | Central Distribution Switch; aggregates trunk links from all floor access switches[cite: 10]. |
| **Access Switches** | Cisco Catalyst 2960-24TT | 3 | FastEthernet (Access) / Gigabit (Trunk) | Floor Access Switches (`Floor1-SW`, `Floor2-SW`, `CR2-SW`) enforcing departmental VLAN port rules[cite: 10]. |
| **Mgmt Switch** | Cisco Catalyst 2960-24TT | 1 | FastEthernet (Access) / Gigabit (Trunk) | Dedicated switch isolated strictly for VLAN 99 administrative terminals (`Laptop0`)[cite: 10]. |

### 2.2 Cabling & Media Selection Strategy
- **Copper Straight-Through (Cat6 UTP):** Used for connections between unlike devices (e.g., Access Switch FastEthernet ports to workstations/servers)[cite: 10].
- **Copper Crossover / Gigabit Trunk Media:** Used for high-bandwidth backbone uplinks connecting access switches to the central Catalyst 3560 Core Switch[cite: 10].
- **GigabitEthernet Point-to-Point WAN Link:** Interconnects Core-Router-2 (`G0/0`) and Core-Router-1 (`G0/1`) over a `/30` link (`192.168.47.208/30`)[cite: 10].

### 2.3 Physical Extended Star Topology Diagram
<img src="Physical Topology" alt="Physical Topology Diagram" width="100%">

---

## 3. Logical Topology & Master VLSM IP Addressing Scheme

Base Assigned Network: `192.168.47.0/24` (256 Total Addresses)[cite: 10]

| Department / Purpose | VLAN | Subnet Address | CIDR | Subnet Mask | Usable Host Range | Default Gateway | Allocation Method |
| :--- | :---: | :--- | :---: | :--- | :--- | :--- | :--- |
| **Management** | `10` | `192.168.47.0` | `/27` | `255.255.255.224` | `192.168.47.1 – .30` | `192.168.47.1` | Static (`.2–.10`) / DHCP (`.11–.30`)[cite: 10] |
| **Staff Network** | `20` | `192.168.47.32` | `/26` | `255.255.255.192` | `192.168.47.33 – .94` | `192.168.47.33` | Central DHCP Scope (`.34–.94`)[cite: 10] |
| **Research Labs** | `30` | `192.168.47.96` | `/26` | `255.255.255.192` | `192.168.47.97 – .158` | `192.168.47.97` | Central DHCP Scope (`.98–.158`)[cite: 10] |
| **CR2 Expansion** | `40` | `192.168.47.160` | `/27` | `255.255.255.224` | `192.168.47.161 – .190` | `192.168.47.161` | DHCP Relay via `ip helper` (`.162–.190`)[cite: 10] |
| **Infrastructure / Mgmt** | `99` | `192.168.47.192` | `/28` | `255.255.255.240` | `192.168.47.193 – .206` | `192.168.47.193` | Strictly Statically Assigned[cite: 10] |
| **Point-to-Point WAN** | `N/A` | `192.168.47.208` | `/30` | `255.255.255.252` | `192.168.47.209 – .210` | `N/A` | Static Router Interfaces[cite: 10] |

---

## 4. Technical Configurations & Implementation Highlights

### 4.1 Multi-Router Static Path Control Setup
```text
Core-Router-2 (Internal Gateway Outbound Default Route):
Core-Router-2(config)# ip route 0.0.0.0 0.0.0.0 192.168.47.209

Core-Router-1 (Perimeter Edge Inbound Summary Route):
Core-Router-1(config)# ip route 192.168.47.0 255.255.255.0 192.168.47.210

Core-Router-1 (Internet Gateway Route to ISP):
Core-Router-1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
```[cite: 10]

### 4.2 Cross-VLAN DHCP Relay Implementation
The central DHCP server is located on VLAN 20 at `192.168.47.35`[cite: 10]. To forward dynamic allocation requests across subnets for CR2 Expansion hosts (VLAN 40), an IP helper address was configured on Core-Router-2[cite: 10]:
```text
Core-Router-2(config)# interface GigabitEthernet0/0.40
Core-Router-2(config-subif)# encapsulation dot1Q 40
Core-Router-2(config-subif)# ip address 192.168.47.161 255.255.255.224
Core-Router-2(config-subif)# ip helper-address 192.168.47.35
```[cite: 10]

---

## 5. Empirical Verification & Test Results Analysis

### 5.1 Verification Summary Matrix
| Test ID | System / Feature Tested | Input Command / Action | Expected Result | Verdict | Observed Evidence Details |
| :---: | :--- | :--- | :--- | :---: | :--- |
| **TC-01** | Static Path Control Audit | `show ip route` on Core Routers | Active static entries installed in routing tables[cite: 10] | **PASSED** | `S* 0.0.0.0/0` active on Core-2; `S 192.168.47.0/24` active on Core-1[cite: 10]. |
| **TC-02** | DHCP Relay Service | Toggle IP Config to DHCP on `Laptop0` | Auto-assignment from central DHCP scope[cite: 10] | **PASSED** | Assigned IP `192.168.47.163` (`/27`), GW `.161`, DNS `.66`[cite: 10]. |
| **TC-03a**| Local Gateway ICMP | `ping 192.168.47.161` | 0% packet loss to local sub-interface[cite: 10] | **PASSED** | 4/4 replies received, 0% loss, TTL=255, time < 1ms[cite: 10]. |
| **TC-03b**| Inter-VLAN ICMP Reachability| `ping 192.168.47.1` | Cross-VLAN reachability to Management GW[cite: 10] | **PASSED** | 4/4 replies received, 0% loss, TTL=255, time < 1ms[cite: 10]. |
| **TC-03c**| Multi-Hop Static WAN Link | `ping 192.168.47.209` | Packet forwarding across static WAN link[cite: 10] | **PASSED** | 3/4 replies received (1st packet ARP timeout), TTL=254[cite: 10]. |
| **TC-04** | Application Services | Web Browser: `http://cisco.com` | DNS resolution and HTTP web page render[cite: 10] | **PASSED** | Domain resolved to `192.168.47.66`; web page loaded successfully[cite: 10]. |

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

---

## 6. Troubleshooting & Engineering Lessons Learned
- **Inter-VLAN Trunking Omission:** Initial ping failures between VLAN 40 and VLAN 20 occurred because VLAN 40 was omitted from the trunk link configuration on the central Core Switch[cite: 10]. Executing `switchport trunk allowed vlan 10,20,30,40,99` resolved trunk forwarding[cite: 10].
- **ARP Table Population Loss:** Initial ICMP pings across the WAN link dropped the first packet due to standard Layer 2 Address Resolution Protocol (ARP) table population over the `/30` link rather than a routing configuration fault[cite: 10].
#### Test 4: Application Layer Services (DNS & HTTP)
<img src="Application Layer TEST" alt="Application Layer Web Verification" width="100%">
