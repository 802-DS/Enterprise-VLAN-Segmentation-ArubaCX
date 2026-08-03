# Enterprise VLAN Segmentation Project
### Securing and Organizing Network Traffic for a Growing Office

---

## 1. Problem Statement

A small company is expanding into a larger office space and growing its headcount across the **HR**, **Sales**, and general/**Guest** (visitor Wi-Fi) user groups. Currently, all devices sit on a single flat network. This creates several problems:

- **Security risk:** Guests connecting to office Wi-Fi are on the same broadcast domain as HR workstations, which handle sensitive payroll and personnel data.
- **No traffic isolation:** A broadcast storm, misconfigured device, or malware outbreak in one department can affect every other department.
- **Poor scalability:** As the office grows, a single flat /24 (or larger) network becomes harder to manage, troubleshoot, and secure.
- **No policy enforcement:** There's no way to restrict Sales from accessing HR file shares, or to stop Guest devices from reaching internal resources at all.

**Goal:** Redesign the LAN using VLAN segmentation so that HR, Sales, and Guest traffic are logically separated, security policies can be applied per department, and the network is easier to manage as the company grows. See Section 5 for the full topology diagram.

### Logical Topology Diagram
![Network Topology Diagram](images/vlan1.drawio.svg)

---

## 2. Learning Objectives

By completing this project, you will be able to:

1. Explain the purpose of VLANs and how they create separate broadcast domains on shared physical infrastructure.
2. Design an IP addressing scheme that maps cleanly to VLANs.
3. Configure VLANs, trunk ports, and access ports on an ArubaOS-CX core switch and Cisco access switch.
4. Configure **inter-VLAN routing** using a Layer 3 switch (Switched Virtual Interfaces).
5. Apply **Access Control Lists (ACLs)** to enforce department-level security policy (e.g., isolating Guest traffic).
6. Configure DHCP scopes per VLAN so each department gets addressing automatically.
7. Verify VLAN and routing configuration using standard show commands and ping/traceroute tests.
8. Diagnose and resolve common VLAN/trunking misconfigurations across multi-vendor networks.

---

## 3. Requirements

### 3.1 Functional Requirements
| ID | Requirement |
|----|-------------|
| R1 | HR, Sales, and Guest devices must be placed in separate VLANs. |
| R2 | HR and Sales must be able to reach shared internal resources (e.g., a file/print server) but **not** each other's departmental subnets unless explicitly required. |
| R3 | Guest devices must reach the Internet only — no access to HR, Sales, or internal server subnets. |
| R4 | Each VLAN must have its own DHCP scope so devices auto-configure IP addressing. |
| R5 | Network administrators must retain a dedicated Management VLAN, separate from user traffic, to manage switches/routers. |
| R6 | The design must support adding new VLANs (e.g., a future "Finance" or "IoT" VLAN) without re-architecting the network. |

### 3.2 Technical Requirements
- At least one Layer 3–capable device (multilayer switch or router) to perform inter-VLAN routing.
- Access switch(es) supporting 802.1Q trunking.
- ACL support (standard or extended) on the routing device.
- DHCP service (can be the L3 switch/router itself, or a dedicated server reachable via DHCP relay).

---

## 4. Assumptions

- This is a **single-site** office (no WAN/branch links involved in this phase).
- The company has an **ArubaCX Layer 3 switch** acting as the routing core.
- Internet access is provided via an existing edge router/firewall; NAT/firewall configuration is out of scope for this project (only the internal VLAN boundary is covered).
- Physical cabling and switch placement have already been decided; this project focuses on the **logical** network design.
- No existing VLANs are configured — this is treated as a greenfield VLAN design.
- Wireless APs are "dumb" (unmanaged) or configured to tag Guest SSID traffic into the Guest VLAN via trunked uplink; wireless controller configuration is out of scope.

---

## 5. Topology

### 5.1 Logical Network Diagram

```text
                                  ┌───────────────┐
                                  │      ISP      │
                                  └───────┬───────┘
                                          │ Gi0/0
                                          │
                                       (1/1/1)
                              ┌───────────────────────┐
                              │   ArubaCX-Core-L3     │
                              │   (Core / L3 Switch)  │
                              └───────────┬───────────┘
                                       (1/1/2)
                                          │
                                        Gi0/0
                              ┌───────────────────────┐
                              │    Cisco-Access-L2    │
                              │    (Access Switch)    │
                              └───┬───┬───────┬───┬───┘
                       Gi0/1─────┘   │       │   └───Gi1/1
                            │       Gi0/2   Gi1/0      │
                         Gi0/3───────┼───────┤         │
                            │        │       │         │
                          eth0     eth0    eth0      eth0
                            │        │       │         │
                       ┌────┴───┐ ┌──┴───┐ ┌─┴────┐ ┌──┴──────┐
                       │ HR-PC1 │ │HR-PC2│ │Sales1│ │Guest-WAP│
                       └────────┘ └──────┘ └──────┘ └─────────┘
                                             │
                                          ┌──┴───┐
                                          │Sales2│
                                          └──────┘
```

### 5.2 IP Addressing Plan

| VLAN ID | Name | Subnet | Gateway (SVI) | DHCP Range |
| :--- | :--- | :--- | :--- | :--- |
| **10** | HR | `192.168.10.0/24` | `192.168.10.1` | `.100 - .200` |
| **20** | Sales | `192.168.20.0/24` | `192.168.20.1` | `.100 - .200` |
| **30** | Guest | `192.168.30.0/24` | `192.168.30.1` | `.100 - .200` |
| **99** | Management | `192.168.99.0/24` | `192.168.99.1` | Static only (no DHCP) |

---

## 6. Design Decisions

| Decision | Rationale |
| :--- | :--- |
| **Use an ArubaOS-CX Layer 3 switch as the core router** | Higher throughput for inter-VLAN traffic via hardware ASIC switching; native support for modern CLI architecture. |
| **One VLAN per department + one dedicated Guest VLAN + one Management VLAN** | Keeps broadcast domains small and maps directly to security policy boundaries. Four VLANs is simple enough to manage but demonstrates the full segmentation pattern. |
| **802.1Q trunking using `vlan trunk allowed`** | Limits broadcast traffic across uplinks by explicitly specifying allowed VLANs. |
| **Native VLAN moved off VLAN 1 (e.g., VLAN 999)** | VLAN 1 is the default/well-known VLAN and a common target for VLAN hopping attacks. Moving native VLAN off VLAN 1 is a standard hardening practice. |
| **Guest VLAN isolated via extended ACL on the SVI** | Guests need Internet only. An ACL applied inbound on the Guest SVI blocks traffic destined to internal RFC1918 ranges while permitting everything else (i.e., the Internet-bound path). |
| **DHCP scopes configured directly on ArubaCX Core** | Simplest to demonstrate in a lab/small-office context; per-VLAN pools provided directly on the core router. |
| **Separate out-of-band Management VLAN (99), not reachable by user VLANs** | Prevents end-user devices from ever reaching switch/router management interfaces, reducing the risk of unauthorized device administration. |

---

## 7. Configuration

> **Note:** Core switch syntax is ArubaOS-CX. Access switch syntax is Cisco IOS-style. Only the commands that establish or change something meaningful are shown below — repetitive per-VLAN/per-port lines are represented once, with a note to repeat the pattern.

### 7.1 Core Switch (ArubaCX) — VLANs and SVIs

```text
ArubaCX-Core-L3(config)# vlan 10
ArubaCX-Core-L3(config-vlan-10)# name HR
! Repeat the vlan/name pattern for 20 (SALES), 30 (GUEST), 99 (MGMT), 999 (NATIVE_UNUSED)

ArubaCX-Core-L3(config)# interface vlan 10
ArubaCX-Core-L3(config-if-vlan)# ip address 192.168.10.1/24
ArubaCX-Core-L3(config-if-vlan)# no shutdown
! Repeat per VLAN using the gateway IPs from the addressing table in Section 5.2
```

### 7.2 Core Switch (ArubaCX) — Trunk to Access Switch

```text
ArubaCX-Core-L3(config)# interface 1/1/2
ArubaCX-Core-L3(config-if)# no routing
ArubaCX-Core-L3(config-if)# vlan trunk native 999
ArubaCX-Core-L3(config-if)# vlan trunk allowed 10,20,30,99
```

### 7.3 Access Switch (Cisco) — Trunk to Core

```text
ACCESS-SW(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW(config-if)# switchport mode trunk
ACCESS-SW(config-if)# switchport trunk native vlan 999
ACCESS-SW(config-if)# switchport trunk allowed vlan 10,20,30,99
```

*Native VLAN and allowed VLAN list must match exactly on both ends of the trunk (Section 9 covers what happens when they don't).*

### 7.4 Access Switch (Cisco) — Access Ports

```text
ACCESS-SW(config-if)# switchport mode access
ACCESS-SW(config-if)# switchport access vlan 10
ACCESS-SW(config-if)# spanning-tree portfast
! Same pattern per port, just swap the VLAN ID: 20 for Sales ports, 30 for the Guest AP port
```

### 7.5 Core Switch (ArubaCX) — DHCP Pool

```text
ArubaCX-Core-L3(config-dhcp-server)# pool HR_POOL
ArubaCX-Core-L3(config-dhcp-server-pool)# range 192.168.10.100 192.168.10.200 netmask 255.255.255.0
ArubaCX-Core-L3(config-dhcp-server-pool)# default-router 192.168.10.1
ArubaCX-Core-L3(config-dhcp-server-pool)# dns-server 8.8.8.8
! Repeat for SALES_POOL and GUEST_POOL using their own subnet/gateway; no pool needed for Management (static only)
ArubaCX-Core-L3(config-dhcp-server)# enable
```

### 7.6 Core Switch (ArubaCX) — Guest Isolation ACL

```text
ArubaCX-Core-L3(config)# access-list ip GUEST_RESTRICT
ArubaCX-Core-L3(config-acl-ip)# 10 deny ip 192.168.30.0/24 192.168.10.0/24
ArubaCX-Core-L3(config-acl-ip)# 20 deny ip 192.168.30.0/24 192.168.20.0/24
ArubaCX-Core-L3(config-acl-ip)# 30 deny ip 192.168.30.0/24 192.168.99.0/24
ArubaCX-Core-L3(config-acl-ip)# 40 permit ip any any

ArubaCX-Core-L3(config)# interface vlan 30
ArubaCX-Core-L3(config-if-vlan)# apply access-list ip GUEST_RESTRICT in
```

*Permits Guest → Internet while explicitly blocking Guest → HR, Sales, and Management subnets. The `permit any any` line is required, or Guest devices lose Internet access too.*

---

## 8. Verification

| Goal | Command | Expected Result |
| :--- | :--- | :--- |
| **Confirm VLANs exist** | `show vlan` | VLANs 10, 20, 30, 99, 999 listed on Core Switch. |
| **Confirm trunk is passing correct VLANs** | `show interface 1/1/2 switchport` | Trunk mode enabled, native VLAN 999, allowed VLANs 10,20,30,99. |
| **Confirm SVIs are up** | `show interface brief` | Vlan10, Vlan20, Vlan30, Vlan99 show up/up with correct IPs. |
| **Confirm routing table has all VLAN subnets** | `show ip route` | Connected routes listed for each VLAN subnet. |
| **Confirm HR device gets correct IP** | `ipconfig` (on HR PC) | Address in 192.168.10.100–.200 range, gateway 192.168.10.1. |
| **Confirm Guest is blocked from HR/Sales** | `ping 192.168.10.1` from a Guest device | Request times out / unreachable. |
| **Confirm Guest can still reach Internet** | `ping 8.8.8.8` from a Guest device | Replies received. |
| **Confirm ACL is being hit** | `show access-list GUEST_RESTRICT` | Match counters increment on the deny lines when a Guest device tries to reach HR/Sales. |
| **Confirm DHCP is leasing correctly** | `show dhcp-server leases` | Leases listed per VLAN pool with correct client MACs and IPs. |

---

## 9. Troubleshooting

| Symptom | Likely Cause | Fix |
| :--- | :--- | :--- |
| **Devices in the same VLAN can't reach each other** | Access port assigned to wrong VLAN, or port still in VLAN 1 | Verify with `show vlan brief`; correct with `switchport access vlan <id>`. |
| **Interface 1/1/2 acts as L3 instead of L2 trunk** | Port is operating in routed mode | Run `no routing` inside `interface 1/1/2` before applying trunk settings. |
| **SVI shows down** | VLAN doesn't exist yet, or interface shut down | Create the VLAN (`vlan <id>`) and ensure `no shutdown` is executed on the SVI. |
| **Trunk not passing a particular VLAN** | VLAN not included in `vlan trunk allowed` list | Add it: `vlan trunk allowed 10,20,30,99` under the interface. |
| **Intermittent connectivity / broadcast storms across VLANs** | Native VLAN mismatch between the two ends of the trunk | Ensure both trunk ends specify the same native VLAN. |
| **Guest device can reach HR/Sales despite ACL** | ACL applied in the wrong direction, or applied to the wrong interface | Confirm ACL is applied using `apply access-list ip GUEST_RESTRICT in` on `interface vlan 30`. |
| **Guest device also blocked from the Internet** | `40 permit ip any any` rule missing or ordered incorrectly | Ensure the ACL ends with an explicit `permit ip any any` after the specific deny statements. |
| **Devices not receiving an IP address at all** | DHCP pool range statement doesn't match VLAN subnet or DHCP not enabled | Recheck `dhcp-server pool` config; confirm global `enable` command was executed under `dhcp-server`. |
| **Can't manage the switch remotely** | Management VLAN SVI IP not reachable from admin subnet | Confirm `vlan 99` SVI is up and that the admin PC's subnet has a route to `192.168.99.0/24`. |

---

## 10. Engineering Notes

- Kept VLAN numbering **identical on both the Aruba core and Cisco access switch** (10/20/30/99) — mismatched IDs across vendors are a common source of "invisible" outages that only show up in packet captures.
- Reserved **VLAN 99 for management** and kept it off the user-facing trunk logic entirely, so a misconfigured ACL or DHCP scope on a user VLAN can never accidentally expose switch management.
- Moved the **native VLAN to an unused ID (999)** instead of leaving it as VLAN 1, closing off a well-known VLAN-hopping vector for close to zero extra effort.
- Enabled **`spanning-tree portfast`** only on access ports connecting to end hosts (PCs, AP) — never on the trunk — so devices come up fast without triggering unnecessary topology change notifications.
- Applied the Guest-isolation ACL **on the SVI, not the trunk port**, so the policy follows the VLAN regardless of which physical port or switch a guest device lands on.
- Left **VLAN ranges 40–98 and 100+ unused on purpose** — an intentional gap for future departments (Finance, IoT, additional guest tiers) without renumbering anything already deployed.
- Chose a **Layer 3 core + Layer 2 access split** rather than routing on the access switch, so all policy (ACLs, DHCP, inter-VLAN routing) lives in one place and the access layer stays simple to swap or scale out.

---

## 11. Lessons Learned

Building this as a lab/documentation exercise surfaced a few things that would need more attention in a real production rollout:

- **Cross-vendor consistency is the biggest risk, not the biggest complexity.** The Aruba and Cisco CLIs differ in syntax, but the actual failure mode in practice is a human one — someone updates the allowed-VLAN list on one switch and forgets the other. In production this would be handled with a **single source of truth** (e.g., a network automation tool like Ansible/NetBox) pushing config to both platforms instead of manual parallel changes.
- **A single core switch is a single point of failure.** This design is fine for a small office, but at production scale I'd introduce a **redundant core pair** (Aruba VSX or similar) so a core switch failure doesn't take down every VLAN's gateway and DHCP service at once.
- **On-switch DHCP doesn't scale well.** Running DHCP directly on the core switch is convenient for a small office, but a production environment would benefit from a **centralized DHCP server** with `ip helper-address`-style relay, giving centralized logging, failover, and reservation management that switch-based DHCP doesn't offer.
- **ACLs alone are a coarse tool.** The Guest-isolation ACL works, but it's static and subnet-based. In production I'd look at **dynamic segmentation (e.g., 802.1X + RADIUS-assigned VLANs, or role-based policies)** so device identity — not just IP subnet — drives access decisions.
- **Documentation drift is a real risk.** Writing this project doc alongside the config was useful, but in production the diagram, addressing plan, and ACLs need to live somewhere that's automatically kept in sync with the running config (e.g., generated from the network source of truth) rather than maintained by hand.
