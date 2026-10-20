# Bokone Community Network (Vryburg) — CMPG 325 Individual Project

## Project Overview
This repository contains the complete design, implementation, and testing
portfolio for **Bokone Community Network (Vryburg)**, a telecommunications
client scenario assigned as part of the CMPG 325 Computer Networks individual
semester project.

The assigned networking challenge for this project is **EtherChannel
(Link Aggregation)**, implemented using LACP between the core and
distribution switches.

## Project Details
| Field | Value |
|---|---|
| Student | Siyamthanda Xenza |
| Student Number | 48975842 |
| Project ID | CMPG325-2026-148 |
| Client ID | CLI-148 |
| Assigned Address Block | 172.30.100.0/23 |
| Module | CMPG 325 – Computer Networks |
| Institution | North-West University |

## Client Summary
Bokone Community Network is a telecommunications organisation based in
Vryburg. The network design provides reliable, logically segmented
connectivity across staff, technical/NOC, and management functions, while
accommodating two forward-looking requirements:
- A possible **additional floor** next financial year (reserved VLAN/subnet)
- A planned **branch office** (Change Request CR6 — addressing and design
  accommodation only, no second-site build)

## Objectives
1. Design a network that is reliable, segmented by function, efficiently addressed and scalable.
2. Build it in Cisco Packet Tracer with working VLANs, inter-VLAN routing and DHCP.
3. Implement EtherChannel (LACP) between two switches and verify it, including behaviour when a member link fails.
4. Document the design, configuration, testing and troubleshooting in this repository.

**Out of scope:** physical build of the future floor and branch office (addressing only), and a simulated ISP/Internet connection, which was not a stated client requirement.

## Network Design

### Topology
The network is a hierarchical (tree) design with three layers. The core routes between VLANs, the distribution layer aggregates the access layer, and the access layer connects end devices.

| Device | Model | Role |
|---|---|---|
| BOKONE-EDGE-RTR01 | Cisco ISR4321 | Edge router, routed point-to-point link to the core |
| BOKONE-CORE-SW01 | Cisco 3560-24PS (Layer 3) | Inter-VLAN routing (SVIs), DHCP server for VLAN 20, EtherChannel end |
| BOKONE-DIST-SW01 | Cisco 3560-24PS | EtherChannel end, aggregates the access layer |
| BOKONE-ACC-SW01 | Cisco 3560-24PS | Connects PC1, PC2, PC3 and SRV1 |

| Link | Ports | Type |
|---|---|---|
| Router to core | Router Gi0/0/1 to Core Fa0/24 | Routed point-to-point, 172.30.101.128/30 |
| Core to distribution | Core Gi0/1, Gi0/2 to Dist Gi0/1, Gi0/2 | EtherChannel (Po1, LACP), 802.1Q trunk |
| Distribution to access | Dist Fa0/24 to Acc Gi0/1 | 802.1Q trunk, VLANs 20 and 30 |
| Access to end devices | Acc Fa0/1-2 (PC1, PC2), Fa0/3-4 (PC3, SRV1) | Access ports |

### VLANs and Addressing
The 172.30.100.0/23 block is divided into /26 subnets (62 usable hosts each).

| VLAN | Name | Subnet | Gateway (SVI) | Addressing | Status |
|---|---|---|---|---|---|
| 10 | MGMT | 172.30.100.0/26 | 172.30.100.1 | Static | Active |
| 20 | STAFF | 172.30.100.64/26 | 172.30.100.65 | DHCP | Active |
| 30 | NOC_SRV | 172.30.100.128/26 | 172.30.100.129 | Static | Active |
| 50 | FUTURE_FLOOR | 172.30.101.0/26 | — | — | Reserved |
| 60 | BRANCH (CR6) | 172.30.101.64/26 | — | — | Reserved |
| — | Router–core link | 172.30.101.128/30 | .129 router, .130 core | Static | Active |

Static hosts in VLAN 30: PC3 172.30.100.130, SRV1 172.30.100.131 (gateway 172.30.100.129).

### Key Design Decisions
- **Layer 3 core switch with SVIs** for inter-VLAN routing: faster than router-on-a-stick and needs no extra hardware.
- **/26 subnets and a /30 router link:** no wasted addresses, with whole subnets left free for growth.
- **DHCP for staff, static for servers:** convenience for users, predictable addresses for servers.
- **NOC and servers share VLAN 30:** simple for this scale, and can be split later because spare space exists.
- **VLAN 10 not carried to the access layer:** trunks allow only the VLANs that are needed.

## Implementation Summary
| Device | What was configured |
|---|---|
| Core | `ip routing`, VLANs 10/20/30, one SVI per VLAN, DHCP pool for VLAN 20, routed port Fa0/24 (172.30.101.130/30), Gi0/1-2 trunks in `channel-group 1 mode active`, default route to the router |
| Router | Gi0/0/1 172.30.101.129/30, static route `172.30.100.0 255.255.254.0 172.30.101.130` |
| Distribution | VLANs 10/20/30, Gi0/1-2 identical to the core in `channel-group 1 mode active`, Fa0/24 trunk (VLANs 20, 30) |
| Access | VLANs 20/30, Gi0/1 trunk, Fa0/1-2 access VLAN 20, Fa0/3-4 access VLAN 30 |

## EtherChannel (Assigned Challenge)
**Problem:** two parallel links between switches make Spanning Tree block one, wasting bandwidth and leaving a single active path.

**Solution:** EtherChannel bundles the two Gigabit links into one logical interface, Port-channel 1. LACP (IEEE 802.3ad) negotiates the bundle, so ports join only when their settings match. Both ends use `mode active`.

**Benefits:** more bandwidth, redundancy if a link fails, and one interface to manage.

**Rule:** all member ports must match in speed, duplex, trunk mode and allowed VLANs.

**Verification:** `show etherchannel summary` on both switches shows `Po1(SU)` with `Gig0/1(P)` and `Gig0/2(P)`, protocol LACP.

## Testing and Verification
| ID | Test | Result |
|---|---|---|
| T1 | Ping PC1 (VLAN 20) to 172.30.100.129 | 4/4 replies, 0% loss |
| T2 | Same ping with one core-to-distribution cable removed | 4/4 replies, 0% loss |
| T3 | DHCP for VLAN 20 (`show ip dhcp binding`, `ipconfig /renew`) | PC1 172.30.100.71, PC2 172.30.100.72, gateway .65, DNS 8.8.8.8 |
| T4 | `tracert 172.30.100.129` from PC1 | One hop, core routes between VLANs |
| T5 | Ping from PC3 and SRV1 to 172.30.100.129 | 4/4 replies each, 0% loss |
| T6 | `show etherchannel summary` on core and distribution | Po1(SU), Gig0/1(P), Gig0/2(P), LACP |
| T7 | Ping from the router to 172.30.100.65 | 4/5 (first packet lost to ARP), then no loss |
| T8 | Ping from PC1 to the router, 172.30.101.129 | Successful |

The first packet of a ping can time out while the device resolves the next hop's address (ARP). This is normal.

## Troubleshooting Log
| Issue | Cause | Fix |
|---|---|---|
| Router had no route back to the VLANs; router and core default routes pointed at each other, which would loop unknown traffic | Default routes in both directions | Removed the router's default route and added a specific route, `ip route 172.30.100.0 255.255.254.0 172.30.101.130`. Router then reached 172.30.100.65 and PC1 reached the router |
| Risk of EtherChannel not forming | Member ports must match on both ends | Applied identical settings on both switches and confirmed with `show etherchannel summary` |

## Scalability and Future Growth
- **Additional floor:** create VLAN 50, add an SVI (172.30.101.1), and add VLAN 50 to the trunks. No readdressing needed.
- **Branch office (CR6):** the same approach using VLAN 60 and 172.30.101.64/26.
- **Router:** the static route already covers the whole /23, so no router change is needed.
- **Spare space:** 172.30.100.192/26 and 172.30.101.192/26 remain unallocated.

## Limitations and Future Improvements
- One edge router is a single point of failure; add a second router with HSRP.
- The WAN/ISP link is not simulated; add an ISP connection with NAT if external access is required.
- Add ACLs between VLANs and port security on access ports.
- Add redundant uplinks and switches as the network grows.

## Repository Structure
> Adjust folder names below to match your repository.

```
CMPG325-Bokone-Community-Network/
├── README.md
├── 01-client-requirements/     Client requirements document
├── 02-topologies/              Physical and logical topology diagrams
├── 03-addressing-plan/         IP addressing plan
├── 04-packet-tracer/           Bokone_community_network.pkt
├── 05-configs/                 Running configurations for all four devices
├── 06-testing-evidence/        Screenshots and outputs for tests T1-T8
└── 07-documentation/           Milestone 2 documentation, technical report
```

## Deliverables Checklist
- [x] Packet Tracer project (`.pkt`)
- [x] GitHub portfolio of evidence (this repository)
- [x] Technical report
- [x] Any additional files required

## Project Milestones
| Milestone | Scope | Status |
|---|---|---|
| Milestone 1 | Client requirements, physical and logical topologies, IP addressing plan | ✅ Completed |
| Milestone 2 | Client implementation review, EtherChannel (LACP) implementation, testing evidence | ✅ Completed |
| Final submission | Packet Tracer project, GitHub portfolio, technical report | ✅ Completed |

## How to Use This Repository
1. Open `04-packet-tracer/Bokone_community_network.pkt` in Cisco Packet Tracer.
2. Use **Realtime** mode. Open each device's CLI tab to run the commands below.
3. Follow the test order in the Testing section.

| Command | Purpose |
|---|---|
| `show vlan brief` | VLANs and their ports |
| `show interfaces trunk` | Trunk ports and carried VLANs |
| `show ip interface brief` | Interface addresses and status, including SVIs |
| `show ip route` | Routing table |
| `show ip dhcp binding` | Addresses issued by DHCP |
| `show etherchannel summary` | Bundle status, protocol and member flags |

## References
- Cisco Systems, Inc. (2023) *Understand EtherChannel load balance and redundancy on Catalyst switches.* https://www.cisco.com/c/en/us/support/docs/lan-switching/etherchannel/12023-4.html
- Cisco Systems, Inc. (2024) *Configure Inter-VLAN Routing with Catalyst Switches.*
- Internet Engineering Task Force (1996) *RFC 1918: Address Allocation for Private Internets.*
