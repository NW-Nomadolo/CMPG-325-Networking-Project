# CMPG-325-Networking-Project
This project designs and implements an enterprise-grade Local Area Network (LAN) for the North-West Agricultural Research Station (Potchefstroom) under a single 192.168.47.0/24 subnet block.

# Network Architecture & Client Requirement Analysis
**Course:** CMPG 325 Computer Networks  
**Project ID:** CMPG325-2026-106  
**Client:** North-West Agricultural Research Station (Potchefstroom)  
**Prepared By:** Neo William Nomadolo (Student ID: 46428267)  
**Assigned Subnet:** `192.168.47.0/24`  
**Milestone 1 (Report) Due Date:** October 16, 2026  

---

## 1. Executive Summary & Client Requirement Analysis

The **North-West Agricultural Research Station (Potchefstroom)** operates critical agricultural data infrastructure, real-time research laboratories, administrative offices, and executive leadership facilities. To modernize its network infrastructure, five core operational mandates were established for Milestone 1.

### 1.1 IP Addressing Efficiency & Minimal Wastage
* **Client Need:** Fit all station departments (Management, Staff, Research, CR2 Expansion, and Infrastructure) into a single assigned classless IPv4 address block (`192.168.47.0/24`) without running out of host addresses or wasting unassigned IP space.
* **Implemented Solution:** Variable Length Subnet Masking (VLSM). The `/24` block was subdivided into tailored variable subnets ranging from `/26` (62 hosts) to `/27` (30 hosts) down to `/30` (2 hosts for point-to-point links).
* **Technical Advantage:** Matches subnet boundaries directly to host requirements rather than using rigid classful allocations, preventing address exhaustion and containing broadcast domains.
* **Future Impact:** Preserves unallocated address space for future physical site additions without requiring network re-addressing.

### 1.2 Unrestricted Executive Internet Path & Security Isolation
* **Client Need:** Executive Management (VLAN 10) must retain dedicated, high-priority, uninterrupted internet access, completely isolated from general staff bandwidth throttling, access-control lists (ACLs), or scheduled outages.
* **Implemented Solution:** Dedicated VLAN segmentation (VLAN 10) coupled with sub-interface routing and direct Quality of Service (QoS) routing policies.
* **Technical Advantage:** Physical and logical traffic separation ensures executive traffic bypasses restrictive filter rules applied to regular user subnets.
* **Future Impact:** Guarantees uptime for critical administrative operations during heavy network load or active perimeter security enforcement.

### 1.3 Multi-Router Path Control & Static Routing
* **Client Need:** Demonstrate multi-hop path control and traffic forwarding using static routing across multiple interconnected core routers.
* **Implemented Solution:** Dual-router core implementation consisting of **Core Router 1** (Perimeter Edge/NAT) and **Core Router 2** (Internal Gateway/Inter-VLAN Routing) connected via a `/30` point-to-point link.
* **Technical Advantage:** Offloads Inter-VLAN routing processing from the edge firewall/router, reducing CPU utilization and providing explicit administrative control over packet forwarding paths.
* **Future Impact:** Simplifies edge firewall security policies and allows simple integration of redundant backup WAN links via floating static routes.

### 1.4 Change Request Integration (CR2 Expansion Floor)
* **Client Need:** Incorporate a newly acquired building floor (CR2 Expansion) into the operational network without interrupting existing active subnets or requiring downtime.
* **Implemented Solution:** Extended Star Topology integration. Provisioned a dedicated subnet on VLAN 40 (`192.168.47.160/27`), connected a new access switch via 802.1Q trunking to the central core switch, and assigned sub-interface `G0/0.40` on Core Router 2.
* **Technical Advantage:** The modular Extended Star framework permits new switches to be added to the distribution tier without altering existing access layer configurations.
* **Future Impact:** Establishes a scalable deployment pattern for future physical facility additions.

### 1.5 Infrastructure Protection & Out-of-Band Management
* **Client Need:** Isolate switch and router management interfaces from general user traffic to protect network devices against unauthorized access or interception.
* **Implemented Solution:** Isolated Management VLAN (VLAN 99). All Switch Virtual Interfaces (SVIs) and administrative SSH sessions are assigned exclusively to VLAN 99 (`192.168.47.192/28`).
* **Technical Advantage:** Prevents host devices in user VLANs (10, 20, 30, 40) from reaching switch management IPs, mitigating localized privilege escalation attacks.
* **Future Impact:** Meets enterprise zero-trust baseline standards for network device management.

---

## 2. Physical Topology Architecture

### 2.1 Hardware Specification Matrix
| Device Name | Model / Hardware | Qty | Functional Role |
| :--- | :--- | :---: | :--- |
| **Core Router 1** | Cisco 2911 ISR | 1 | Edge Router; handles outbound NAT and default routing to the ISP. |
| **Core Router 2** | Cisco 2911 ISR | 1 | Internal Gateway; performs Inter-VLAN routing via sub-interfaces. |
| **Core Switch** | Cisco Catalyst 3560 | 1 | Distribution/Core Switch; aggregates 802.1Q trunks from access switches. |
| **Access Switches** | Cisco Catalyst 2960 | 3 | Floor Switches (Staff, Research, CR2 Expansion) enforcing access port security. |
| **Mgmt Switch** | Cisco Catalyst 2960 | 1 | Dedicated switch for VLAN 99 administrative device management. |

### 2.2 Physical Topology Diagram
```text
                            [ ISP Cloud / Internet ]
                                       │
                                       │ (GigabitEthernet 0/0)
                                       ▼
                              [ CORE ROUTER 1 ] (Edge Router)
                               (Cisco 2911 ISR)
                                       │
                                       │ ⚡ Point-to-Point WAN Link
                                       │    IP Subnet: 192.168.47.208/30
                                       ▼
                              [ CORE ROUTER 2 ] (Internal Gateway)
                               (Cisco 2911 ISR)
                                       │
                                       │ (802.1Q Gigabit Trunk Link)
                                       ▼
                            [ CENTRAL CORE SWITCH ]
                            (Cisco Catalyst 3560)
                                       │
   ┌───────────────────┬───────────────┴───────────────┬───────────────────┐
   │ Trunk (VLAN 10,99)│ Trunk (VLAN 20,30)            │ Trunk (VLAN 40)   │ Trunk (VLAN 99)
   ▼                   ▼                               ▼                   ▼
[ FLOOR 1 SWITCH ]  [ FLOOR 2 SWITCH ]              [ CR2 FLOOR SWITCH ][ MGMT SWITCH ]
 (Catalyst 2960)     (Catalyst 2960)                 (Catalyst 2960)     (Catalyst 2960)
   │         │         │         │                     │         │           │         │
   ▼         ▼         ▼         ▼                     ▼         ▼           ▼         ▼
[PC 1]    [PC 2]    [PC 3]    [Server 1]            [PC 4]    [PC 5]      [Admin Laptop]
(Mgmt)    (Staff)   (Lab)     (Research Data)       (CR2)     (CR2)       (Secure Terminal)
 ```

Logical Topology & Departmental Segmentation
Logical segmentation is enforced at Layer 2 via VLANs and at Layer 3 via Router-on-a-Stick sub-interfaces configured on Core Router 2.
```text
                              [ CORE ROUTER 2 ]
                         (Inter-VLAN Sub-Interfaces)
                                       │
  ┌─────────────┬──────────────────────┼──────────────────────┬─────────────┐
  │ .10         │ .20                  │ .30                  │ .40         │ .99
  ▼             ▼                      ▼                      ▼             ▼
[ VLAN 10 ]   [ VLAN 20 ]            [ VLAN 30 ]            [ VLAN 40 ]   [ VLAN 99 ]
Management    Staff Network          Research Labs          CR2 Expansion Infrastructure
192.168.47.0/27 192.168.47.32/26     192.168.47.96/26       192.168.47.160/27 192.168.47.192/28
Gateway: .1    Gateway: .33           Gateway: .97           Gateway: .161 Gateway: .193
```
### IP Addressing Scheme Summary Table

Base Assigned Network: `192.168.47.0/24`

| Device / VLAN | Subnet Address | CIDR | Subnet Mask | Usable Host Range | Default Gateway | Allocation Type |
| :--- | :--- | :---: | :--- | :--- | :--- | :--- |
| **VLAN 10: Executive Management** | `192.168.47.0` | `/27` | `255.255.255.224` | `192.168.47.1 - 192.168.47.30` | `192.168.47.1` | Static / Dynamic |
| **VLAN 20: General Staff** | `192.168.47.32` | `/26` | `255.255.255.192` | `192.168.47.33 - 192.168.47.94` | `192.168.47.33` | Dynamic (DHCP) |
| **VLAN 30: Research Labs** | `192.168.47.96` | `/26` | `255.255.255.192` | `192.168.47.97 - 192.168.47.158` | `192.168.47.97` | Dynamic (DHCP) |
| **VLAN 40: CR2 Expansion Floor** | `192.168.47.160` | `/27` | `255.255.255.224` | `192.168.47.161 - 192.168.47.190` | `192.168.47.161` | Dynamic (DHCP) |
| **VLAN 99: Infrastructure Mgmt** | `192.168.47.192` | `/28` | `255.255.255.240` | `192.168.47.193 - 192.168.47.206` | `192.168.47.193` | Static Only |
| **Core Link (Core R1 ↔ Core R2)** | `192.168.47.208` | `/30` | `255.255.255.252` | `192.168.47.209 - 192.168.47.210` | N/A | Static |
