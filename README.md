# Project 300: Secure Banking Network (Networking Prototype)

> A segmented, secured network for a small banking environment, prototyped in Cisco Packet Tracer and designed to be rebuilt on a single physical rack.

**Module:** Project 300
**Role documented here:** Networking (routing, switching, segmentation, perimeter)
**Team:** Networking: *Adam* | Windows Administration: *Colin* | Database: *Thomas* | Network Security / RADIUS: *Tyler*
**Tool:** Cisco Packet Tracer 8.x
**Status:** Prototype complete and tested (see [Limitations](#10-limitations-and-honest-scope))

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Design Goals](#2-design-goals)
3. [Topology](#3-topology)
4. [Equipment](#4-equipment)
5. [Addressing and VLAN Plan](#5-addressing-and-vlan-plan)
6. [Design Decisions](#6-design-decisions)
7. [Device Configurations](#7-device-configurations)
8. [Testing and Results](#8-testing-and-results)
9. [Security Controls Summary](#9-security-controls-summary)
10. [Limitations and Honest Scope](#10-limitations-and-honest-scope)
11. [Future Work](#11-future-work)
12. [Moving From Packet Tracer to the Physical Rack](#12-moving-from-packet-tracer-to-the-physical-rack)
13. [Repository Structure](#13-repository-structure)

---

## 1. Project Overview

Project 300 is a team build of a small banking system network. The work is split across four roles:

| Role | Responsibility |
|---|---|
| **Networking (this document)** | Rack design, switching, routing, VLAN segmentation, access control, perimeter firewall |
| Windows Administration | Windows Server (admin services) |
| Database | Database server |
| Network Security | RADIUS server and authentication |

The three server roles (Windows admin, database, RADIUS) will run as virtual machines on **one physical server**. The whole network lives in **one rack**, so there are no long cable runs. This document covers the network that all of that sits on.

The prototype was built and tested in Packet Tracer first. The physical rack follows the same design.

## 2. Design Goals

- **Segmentation:** staff, servers and management must be separated, with traffic between them controlled.
- **Least privilege:** staff can reach only what they need (the database port and DNS), nothing else.
- **Controlled routing:** a predictable routing design that rejects unexpected neighbours.
- **Perimeter protection:** a firewall between the bank network and the outside world.
- **Simplicity:** small enough to build, explain and rebuild on real hardware.

## 3. Topology

![Full network topology](images/01-topology.png)
*Figure 1: Full Packet Tracer topology with all devices named and cabled.*

```
 [Internet] ── [FW-01] ── [RTR-EDGE] ── [RTR-CORE]
                 NAT        default         |  802.1Q trunk
              filtering     route/OSPF      |  (router-on-a-stick)
                                         [SW-01]
                                        /   |   \
                                  SRV-HOST  Fa0/2   Fa0/3
                                            LAPTOP (Staff / Management)
```

### Cabling

| From | To | Cable | Purpose |
|---|---|---|---|
| FW-01 Gi1/1 | Outside / ISP | Cross-over | Internet side (not populated in the prototype) |
| FW-01 Gi1/2 | RTR-EDGE Gi0/0 | Cross-over | Transit to firewall |
| RTR-EDGE Gi0/1 | RTR-CORE Gi0/0 | Cross-over | OSPF link |
| RTR-CORE Gi0/1 | SW-01 Gi0/1 | Straight-through | 802.1Q trunk |
| SW-01 Fa0/1 | SRV-HOST | Straight-through | Server (VLAN 10) |
| SW-01 Fa0/2 | LAPTOP | Straight-through | Staff test (VLAN 20) |
| SW-01 Fa0/3 | LAPTOP | Straight-through | Management test (VLAN 30); the laptop's cable is moved here |

## 4. Equipment

| Name | Packet Tracer model | Role |
|---|---|---|
| FW-01 | ASA 5506-X | Perimeter firewall, NAT/PAT, outbound filtering |
| RTR-EDGE | Cisco 2911 | Edge router: default route to the firewall, OSPF |
| RTR-CORE | Cisco 2911 | Inter-VLAN routing, DHCP, ACLs, OSPF |
| SW-01 | Catalyst 2960 (IOS 15) | Layer 2 access switch, VLANs, port security |
| SRV-HOST | Server | Stand-in for the one physical server (will host 3 VMs) |
| LAPTOP | Laptop | Staff and management testing |

## 5. Addressing and VLAN Plan

### VLANs

| VLAN | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| 10 | Servers | 192.168.10.0/24 | 192.168.10.1 | Windows admin, database and RADIUS servers |
| 20 | Staff | 192.168.20.0/24 | 192.168.20.1 | Bank employee machines (DHCP from .21) |
| 30 | Management | 192.168.30.0/24 | 192.168.30.1 | Administration of network devices |
| 99 | Native / Unused | none | none | Native VLAN on trunks, parking for unused ports |

### Point-to-point links and loopbacks

| Link | Subnet | Addresses |
|---|---|---|
| RTR-EDGE to RTR-CORE | 10.0.1.0/30 | .1 RTR-EDGE, .2 RTR-CORE |
| FW-01 to RTR-EDGE | 10.0.2.0/30 | .1 RTR-EDGE, .2 FW-01 |
| FW-01 outside | 203.0.113.0/29 | .2 FW-01 |

| Device | Loopback (OSPF router-ID) |
|---|---|
| RTR-EDGE | 1.1.1.1/32 |
| RTR-CORE | 2.2.2.2/32 |

### Static addresses

| Host | Address |
|---|---|
| SRV-HOST (stand-in; future VMs .10, .20, .30) | 192.168.10.10 |
| SW-01 management SVI | 192.168.30.3 |
| LAPTOP (Management test) | 192.168.30.50 |

## 6. Design Decisions

**Router-on-a-stick with two routers.** RTR-CORE routes between VLANs over a single trunk to SW-01. This avoids needing a Layer 3 switch and keeps the switching layer simple. RTR-EDGE sits between the core and the firewall so the firewall never takes part in the internal routing domain.

**OSPF internally, static routes at the firewall.** OSPF (area 0, point-to-point link) shares the VLAN subnets between the two routers and lets RTR-EDGE hand a default route down to RTR-CORE. The firewall uses static routes only, so the security edge does not exchange dynamic routing with anything.

**`passive-interface default`.** OSPF hellos are only sent on the router-to-router link. A rogue device plugged into a user or server port cannot form a neighbour.

**VLAN segmentation with ACLs.** Staff traffic is checked inbound on its subinterface (STAFF-IN) and server traffic on its own (SERVERS-IN). Staff may reach only DHCP, DNS and the database port.

**Native VLAN 99 and `nonegotiate`.** The trunk does not use VLAN 1 as native and does not negotiate with DTP, which reduces VLAN-hopping risk.

**Port security, PortFast and BPDU Guard** on access ports. Unused ports are shut down and placed in VLAN 99.

**SSH only** for management, with Telnet disabled.

## 7. Device Configurations

The full configuration for each device is in the [`configs/`](configs/) folder. Each file has a header explaining how to paste it, the config itself, the SSH key steps to run by hand, and the verification commands.

| Device | Role | Config file |
|---|---|---|
| SW-01 | Layer 2 access switch: VLANs, trunk, port security | [configs/SW-01.txt](configs/SW-01.txt) |
| RTR-CORE | Inter-VLAN routing, DHCP, ACLs, OSPF | [configs/RTR-CORE.txt](configs/RTR-CORE.txt) |
| RTR-EDGE | Edge router: default route, OSPF | [configs/RTR-EDGE.txt](configs/RTR-EDGE.txt) |
| FW-01 | Perimeter firewall: NAT/PAT, outbound filtering | [configs/FW-01.txt](configs/FW-01.txt) |
| Physical rack | Changes needed when moving off Packet Tracer | [configs/physical-rack-changes.txt](configs/physical-rack-changes.txt) |

**Build order:** SW-01, then RTR-CORE, then RTR-EDGE, then FW-01. For each device, open the CLI, run `enable` and `configure terminal`, then paste the config.

**Packet Tracer note:** after pasting, generate the SSH key with `crypto key generate rsa` and enter `2048` when prompted (the one-line `modulus 2048` form is rejected in Packet Tracer).

**Security note:** the passwords in the config files (`Cisco123!`) are placeholders for the prototype. Change them before using anything real, and do not push real credentials to a public repository.

## 8. Testing and Results

### 8.1 Layer 2: VLANs

![show vlan brief on SW-01](images/02-show-vlan-brief.png)
*Figure 2: `show vlan brief` on SW-01. VLANs 10, 20, 30 and 99 exist, with ports assigned to the correct VLANs.*

### 8.2 Routing: OSPF adjacency

![show ip ospf neighbor on RTR-CORE](images/03-ospf-neighbor.png)
*Figure 3: `show ip ospf neighbor` on RTR-CORE. RTR-EDGE (1.1.1.1) is in state FULL over the point-to-point link.*

### 8.3 Routing: learned default route

![show ip route on RTR-CORE](images/04-ip-route.png)
*Figure 4: `show ip route` on RTR-CORE. The default route is learned from RTR-EDGE via OSPF, and the connected VLAN subnets are present.*

### 8.4 DHCP

![Staff laptop ipconfig](images/05-staff-ipconfig.png)
*Figure 5: The laptop on the Staff port receives an address in 192.168.20.0/24 from RTR-CORE's DHCP pool, with gateway 192.168.20.1.*

### 8.5 ACL enforcement: Staff blocked from the server

![Staff ping to server fails](images/06-staff-ping-blocked.png)
*Figure 6: From the Staff VLAN, `ping 192.168.10.10` fails. This is the intended result: STAFF-IN permits only DHCP, DNS and the database port to the server.*

### 8.6 ACL evidence: match counters

![show access-lists on RTR-CORE](images/07-show-access-lists.png)
*Figure 7: `show access-lists` on RTR-CORE after the blocked ping. The deny entry shows matches, which confirms traffic was dropped by the ACL.*

### 8.7 Management VLAN has full access

![Management laptop pings](images/08-management-pings.png)
*Figure 8: From the Management VLAN, pings to the gateway and the server succeed. VLAN 30 has no ACL, so administrators can reach every segment.*

### Test summary

| # | Test | Expected | Result |
|---|---|---|---|
| 1 | VLANs present on SW-01 | VLANs 10, 20, 30, 99 | Pass |
| 2 | OSPF adjacency RTR-CORE / RTR-EDGE | FULL | Pass |
| 3 | Default route learned by RTR-CORE | `O*E2 0.0.0.0/0` | Pass |
| 4 | Staff laptop gets DHCP address | 192.168.20.x | Pass |
| 5 | Staff pings server | Blocked | Pass |
| 6 | Staff pings Management gateway | Blocked | Pass |
| 7 | Management pings gateway | Reply | Pass |
| 8 | Management pings server | Reply | Pass |
| 9 | Management pings switch (192.168.30.3) | Reply | Pass |
| 10 | Management pings RTR-EDGE (10.0.1.1) | Reply | Pass |
| 11 | Staff reaches external network (NAT) | Reply | **Not tested** (no external device in the prototype) |

## 9. Security Controls Summary

| Control | Where | Purpose |
|---|---|---|
| VLAN segmentation | SW-01, RTR-CORE | Separate staff, servers and management |
| Inbound ACLs per VLAN | RTR-CORE subinterfaces | Least-privilege access between segments |
| Native VLAN 99, DTP off | Trunks | Reduce VLAN-hopping risk |
| Port security (sticky MAC, max 2) | Staff and Management ports | Limit devices per port |
| BPDU Guard and PortFast | Access ports | Block rogue switches |
| Unused ports shut down in VLAN 99 | SW-01 | Reduce physical attack surface |
| `passive-interface default` | Both routers | No OSPF adjacency on user or server ports |
| SSH v2 only, Telnet disabled | All devices | Encrypted management |
| Perimeter firewall, deny-by-default inbound | FW-01 | Protect the internal network |
| NAT/PAT | FW-01 | Hide internal addressing |
| Hashed secrets | Switch and routers | No plain-text credentials |

## 10. Limitations and Honest Scope

This is a prototype. These limits are known and deliberate:

- **Single path.** One switch, two routers, one firewall and one server. A failure in any one device takes the network down. There is no device-level redundancy.
- **Firewall and NAT are configured but not demonstrated end-to-end.** There is no external device in the Packet Tracer model, so outbound NAT could not be shown with a ping or web request. The firewall's interface and routing configuration is in place and can be verified with `show interface ip brief` and `show route`.
- **OSPF authentication was removed.** Packet Tracer's handling of OSPF MD5 authentication was inconsistent, so the prototype runs OSPF without it. On the physical rack it should be enabled (see [physical-rack-changes.txt](configs/physical-rack-changes.txt)).
- **No `log` on ACL deny lines.** Packet Tracer rejected the keyword. Logging should be enabled on real hardware.
- **VMs are not modelled.** The three server roles are represented by a single stand-in server. The ACLs currently point at 192.168.10.10 for DNS, database and RADIUS. When the VM addresses are final, update the ACL lines to match.
- **Not yet implemented:** NTP, syslog, and RADIUS/AAA login on network devices. These depend on the RADIUS server from the security team.

## 11. Future Work

1. **RADIUS/AAA for device logins**, with a local fallback account.
2. **NTP and syslog** so logs share a timeline and can be reviewed centrally.
3. **OSPF MD5 or SHA authentication** on the physical devices.
4. **ACL logging** on deny lines.
5. **Second access switch** with an LACP EtherChannel, separating server and user hardware.
6. **Second core router with HSRP** and a **firewall HA pair**, to remove the remaining single points of failure.
7. **DMZ VLAN** for any public-facing service.
8. **802.1X** for port-based authentication.

## 12. Moving From Packet Tracer to the Physical Rack

The exact commands are in [configs/physical-rack-changes.txt](configs/physical-rack-changes.txt). Summary of what changes:

| Item | Packet Tracer | Physical rack |
|---|---|---|
| Server port (SW-01 Fa0/1) | Access port, VLAN 10 | 802.1Q trunk carrying the VMs' VLANs |
| RSA key | `crypto key generate rsa`, then `2048` | `crypto key generate rsa modulus 2048` |
| ACL deny lines | No `log` | Add `log` |
| OSPF authentication | Removed | Enable MD5 or SHA |
| Interface names | 2911 / 2960 names | Check with `show ip interface brief` and adjust to the real models |

**Before building, confirm:**
- Router and switch models and their port names (`show version`, `show ip interface brief`).
- The switch image supports SSH (the image name contains `k9`).
- Final VM addresses and the database port (SQL Server 1433 assumed; MySQL would be 3306).
- Domain-join ports needed by the Windows team (DNS 53, Kerberos 88, LDAP 389, SMB 445).

## 13. Repository Structure

```
project-300-network/
├── README.md
├── configs/
│   ├── SW-01.txt
│   ├── RTR-CORE.txt
│   ├── RTR-EDGE.txt
│   ├── FW-01.txt
│   └── physical-rack-changes.txt
└── images/
    ├── 01-topology.png
    ├── 02-show-vlan-brief.png
    ├── 03-ospf-neighbor.png
    ├── 04-ip-route.png
    ├── 05-staff-ipconfig.png
    ├── 06-staff-ping-blocked.png
    ├── 07-show-access-lists.png
    └── 08-management-pings.png
```

Rename your screenshots to match the filenames above and drop them in the `images/` folder. Optionally add the Packet Tracer file (`Project300.pkt`) at the top level.

---

*Project 300, Networking component. Prototype built in Cisco Packet Tracer.*
