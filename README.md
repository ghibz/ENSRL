# Enterprise Network Security & Resilience Lab (ENSRL)

A two-site enterprise-style network built in **Cisco Packet Tracer 8.2.2** for a fictional company, **Northstar Manufacturing**. The project covers segmentation, Layer 3 switching, dynamic routing, gateway redundancy, NAT, least-privilege ACLs, hardened device management, and structured testing and troubleshooting.

> **Status: work in progress.** This README only claims what has been built and tested. Anything not yet verified is marked *Planned*. Plan version: ENSRL v3.0.

---

## Contents

1. [Scenario and objectives](#1-scenario-and-objectives)
2. [Honesty and evidence rules](#2-honesty-and-evidence-rules)
3. [Topology](#3-topology)
4. [IP and VLAN design](#4-ip-and-vlan-design)
5. [Design decisions](#5-design-decisions)
6. [Feature status](#6-feature-status)
7. [Test results](#7-test-results)
8. [Limitations](#8-limitations)
9. [Repository layout](#9-repository-layout)
10. [How to open and reproduce](#10-how-to-open-and-reproduce)
11. [Future work](#11-future-work)

---

## 1. Scenario and objectives

Northstar Manufacturing is a fictional company with a headquarters (HQ), a branch office, staff and finance users, IT administrators, internal application servers, a public-facing web server and a simulated ISP. A compromised workstation has shown the need for segmentation, restricted administration, controlled public access and resilient connectivity.

**Objectives**

- Build a two-site network with clear IP and VLAN documentation.
- Practise switching, inter-VLAN routing, OSPF, gateway and link redundancy, and NAT.
- Enforce least-privilege traffic flows with ACLs and restricted device management.
- Validate every control with positive and negative tests, not pings alone.
- Introduce faults deliberately, troubleshoot them and write short incident reports.
- Publish diagrams, sanitized configs, limitations and evidence.

---

## 2. Honesty and evidence rules

- A command being accepted is **not** proof a feature works. Each feature is marked as working only after a test demonstrates the intended outcome.
- Simulated controls are labelled as simulated. Router ACLs are stateless filters, **not** a next-generation or stateful firewall.
- Packet Tracer uses simplified IOS and protocol models. Behaviour here may differ from real hardware.
- Each test is recorded as **Pass**, **Fail** or **Limitation**, with an evidence file.

---

## 3. Topology

**Simulator:** Cisco Packet Tracer 8.2.2

**Current build (HQ LAN, edge router and routing):**

```
                      R-HQ  (edge router, 2911)
                     /      \
        routed /30  /        \  routed /30
                   /          \
              CORE-1 ======= CORE-2        3560-24PS multilayer switches
              (2-link LACP EtherChannel, Po1)
              |   \          /   |
              |    \        /    |         dual-homed trunks
         ACC-STAFF   ACC-SERVERS           2960-24TT access switches
         |   |   |        |
   IT-ADMIN STAFF FINANCE  DNS-DHCP        endpoints
```

**Target design** (planned, added in stages): a simulated ISP and external network, a DMZ segment, and a branch office connected over a WAN link, bringing the total to about 24 devices.

### Device inventory (current build)

| Role | Model | Name |
|---|---|---|
| Core switches | Catalyst 3560-24PS | CORE-1, CORE-2 |
| Access switches | Catalyst 2960-24TT | ACC-STAFF, ACC-SERVERS |
| Edge router | Cisco 2911 | R-HQ |
| PCs | Generic PC | IT-ADMIN, STAFF, FINANCE |
| Servers | Generic Server | DNS-DHCP |

### Links (current build)

| From | To | Type |
|---|---|---|
| CORE-1 Fa0/23-24 | CORE-2 Fa0/23-24 | LACP EtherChannel (Po1), trunk |
| ACC-STAFF Gi0/1 / Gi0/2 | CORE-1 Fa0/1 / CORE-2 Fa0/1 | Trunks |
| ACC-SERVERS Gi0/1 / Gi0/2 | CORE-1 Fa0/2 / CORE-2 Fa0/2 | Trunks |
| R-HQ Gi0/0 | CORE-1 Fa0/3 | Routed /30 |
| R-HQ Gi0/1 | CORE-2 Fa0/3 | Routed /30 |
| IT-ADMIN, STAFF, FINANCE | ACC-STAFF Fa0/1, Fa0/2, Fa0/3 | Access ports |
| DNS-DHCP | ACC-SERVERS Fa0/1 | Access port |

> FINANCE is temporarily on ACC-STAFF. A dedicated ACC-FINANCE switch is planned.

The full interface inventory is in [`design/interface-inventory.md`](design/interface-inventory.md).

---

## 4. IP and VLAN design

### VLANs (HQ)

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | MANAGEMENT | 10.10.10.0/24 | Network device management |
| 20 | IT-ADMIN | 10.10.20.0/24 | Administrator workstations |
| 30 | STAFF | 10.10.30.0/24 | Employee workstations |
| 40 | FINANCE | 10.10.40.0/24 | Finance endpoints, isolated from STAFF |
| 50 | SERVERS | 10.10.50.0/24 | DNS/DHCP and internal application servers |
| 998 | NATIVE-BLACKHOLE | none | Native VLAN on trunks, no endpoints |
| 999 | PARKING | none | Unused ports, shut down |
| 60, 70, 80 | DMZ, GUEST, LOGGING | 10.10.60/70/80.0/24 | *Planned* |

### Gateway convention

Each HQ VLAN uses the virtual gateway **.1** (HSRP), with **CORE-1 = .2** and **CORE-2 = .3** as real SVI addresses.

### Transit and loopback addressing

| Link / item | Addressing |
|---|---|
| CORE-1 to R-HQ | 10.255.1.0/30 (R-HQ .1, CORE-1 .2) |
| CORE-2 to R-HQ | 10.255.1.4/30 (R-HQ .5, CORE-2 .6) |
| Loopbacks (/32) | R-HQ 10.255.255.1, CORE-1 10.255.255.11, CORE-2 10.255.255.12 |

*Planned:* R-HQ to R-ISP on 203.0.113.0/29, external network 198.51.100.0/24 and a branch at 10.20.x.x. These are documentation-only address ranges.

Full tables: [`design/ip-addressing.md`](design/ip-addressing.md) and [`design/vlan-inventory.md`](design/vlan-inventory.md).

---

## 5. Design decisions

| Decision | Reason |
|---|---|
| Three-layer layout (core, access, edge) | Standard enterprise design; no single switch failure removes all paths |
| Dual-homed access switches with RPVST+ | Redundant uplinks, with spanning tree blocking the second path until needed. CORE-1 is root, CORE-2 secondary |
| LACP EtherChannel between cores | More bandwidth and survives a single member failure |
| Native VLAN 998, explicit allowed-VLAN lists | Avoids VLAN 1 on trunks; limits what each trunk carries |
| SVIs on the cores | One gateway address per VLAN across all trunked access switches |
| Routed /30 links from cores to R-HQ | Point-to-point links to a single router; no VLANs needed |
| Single-area OSPF with explicit router IDs and loopbacks | Stable router IDs; simple and verifiable first |
| HSRP with preempt (CORE-1 active) | Removes the single-gateway point of failure |
| Passive interfaces on VLAN SVIs | No routing traffic sent toward end devices |

---

## 6. Feature status

| Feature | Status |
|---|---|
| VLANs, access ports, 802.1Q trunks, native VLAN 998 | Implemented |
| RPVST+ with CORE-1 root, CORE-2 secondary | Implemented |
| LACP EtherChannel (Po1) between cores | Implemented |
| SVIs and inter-VLAN routing on cores | Implemented |
| Routed /30 links, single-area OSPF | Implemented |
| HSRP gateway redundancy (VLANs 20, 30, 40, 50) | Implemented, failover test passed |
| ISP, external network, default route, PAT | In progress |
| DMZ and static NAT | Planned |
| Branch site and WAN link | Planned |
| DHCP, DNS, application services, DHCP relay | Planned |
| Extended ACL policy (STAFF, FINANCE, GUEST, DMZ, branch) | Planned |
| SSH-only management, VTY restriction, Telnet disabled | Planned |
| Port security, unused ports shut in VLAN 999 | Planned |
| Syslog and NTP | Optional, planned |
| DAI, DHCP snooping, IPsec, ASA, IDS/SIEM | Out of scope for Packet Tracer; see [Limitations](#8-limitations) |

Statuses are updated only when a test passes.

---

## 7. Test results

Full register: [`testing/test-register.md`](testing/test-register.md). Evidence: [`evidence/`](evidence/).

| ID | Test | Result | Evidence |
|---|---|---|---|
| NET-01 | Same-VLAN connectivity across switches | Pass | *add filename* |
| NET-02 | Inter-VLAN routing before ACLs | Pass | *add filename* |
| RES-03 | Active HSRP gateway fails, standby takes over | Pass | *add filename* |
| NET-03, NET-04 | DHCP lease and DNS resolution | Not yet run | |
| NET-07 | Inside client reaches external service via PAT | Not yet run | |
| SEC-01 to SEC-12 | ACL allow and deny tests | Not yet run | |
| RES-01, RES-02, RES-04, RES-05 | Link, uplink and WAN failure | Not yet run | |
| TRB-01 to TRB-05 | Fault injection | Not yet run | |

> Do not leave a row as **Pass** unless an evidence file exists.

---

## 8. Limitations

Recorded in [`design/feature-limitations.md`](design/feature-limitations.md).

- Packet Tracer is a simulator with simplified IOS models. A command being accepted does not prove real-world behaviour.
- ACLs, when added, will be **stateless**. They are not stateful inspection and not a firewall.
- DMZ separation will be VLAN and ACL separation only, not a separate physical security zone.
- Not demonstrated in Packet Tracer and excluded from claims: Dynamic ARP Inspection, IP Source Guard, reliable DHCP snooping, IPS/IDS, deep packet inspection and realistic packet capture.
- Single points of failure remain: each endpoint connects to one access switch over one cable.
- *Add any behaviour that differed from the plan as you find it.*

---

## 9. Repository layout

```
enterprise-network-security-lab/
  README.md
  topology/
    enterprise-network.pkt
    topology.png
  design/
    requirements.md
    interface-inventory.md
    ip-addressing.md
    vlan-inventory.md
    security-policy.md
    threat-model.md
    feature-limitations.md
  configs/
    CORE-1.txt  CORE-2.txt  R-HQ.txt  (R-BRANCH.txt, R-ISP.txt when built)
  testing/
    test-register.md
    connectivity-tests.md
    security-tests.md
    failover-tests.md
    regression-tests.md
  incidents/
    INC-01-lateral-movement.md
    INC-02-dmz-isolation.md
    INC-03-core-wan-failure.md
  evidence/
    routing/  security/  resilience/
  defect-log.md
```

Configs are sanitized: no passwords, keys or personal data.

---

## 10. How to open and reproduce

1. Install **Cisco Packet Tracer 8.2.2**. Files saved in a newer version may not open in an older one.
2. Open `topology/enterprise-network.pkt`.
3. Wait a few seconds for spanning tree to settle. Link lights should turn green.
4. Reproduce key checks from the CLI (`enable`, then):

| Check | Command | Expected |
|---|---|---|
| VLANs | `show vlan brief` | VLANs 10, 20, 30, 40, 50, 998, 999 |
| Trunks | `show interfaces trunk` | Native VLAN 998, allowed 10,20,30,40,50 |
| EtherChannel | `show etherchannel summary` | Po1 (SU), Fa0/23 and Fa0/24 (P) |
| Spanning tree | `show spanning-tree vlan 30` | CORE-1 is root |
| OSPF | `show ip ospf neighbor` (on R-HQ) | Both cores FULL |
| Routes | `show ip route ospf` (on R-HQ) | 10.10.x.0 subnets, two paths each |
| HSRP | `show standby brief` (on cores) | CORE-1 Active, CORE-2 Standby |

5. **Failover test (RES-03):** from the STAFF PC run `ping -t 10.255.255.1`, shut `interface vlan 30` on CORE-1, and watch CORE-2 become Active. Re-enable it afterwards.

Addressing and credentials: endpoints use static IPs (see `design/ip-addressing.md`). Lab credentials, once SSH is configured, are not published.

---

## 11. Future work

- Finish the ISP, PAT, DMZ static NAT, branch site and WAN link.
- Complete ACL policy, SSH hardening and port security, then the full test register.
- Write the three required incident reports and run at least three fault-injection scenarios.
- Build a separate **Linux, Wireshark and IDS lab**, or use **GNS3/EVE-NG** with a suitable legally obtained image, for controls Packet Tracer cannot validate (DAI, DHCP snooping, stateful firewall, IPsec, real packet capture).
