# SecureCorp Enterprise Network
A multi-site enterprise network design and implementation, built in Cisco Packet Tracer, demonstrating CCNA-level routing, switching, redundancy, security, and services configuration across a headquarters, data center, and two branch offices.

## Scenario

SecureCorp operates three connected sites: Headquarters (HQ) with an attached Data Center, Branch Office 1 (BR1), and Branch Office 2 (BR2). This project designs and builds the complete network infrastructure a contracted network engineer would be responsible for: switching, routing, first-hop redundancy, centralized services, security hardening, and WAN/internet connectivity — then documents it to a standard suitable for handoff or review.

## Topology

![Network Topology](screenshots/Topology.png)

- **Core layer:** Core1 & Core2 (Catalyst 3560/3650), redundant via LACP EtherChannel + HSRP
- **Access layer:** Catalyst 2960 switches, dual-homed to both Cores
- **Data Center:** dedicated server segment (DHCP, DNS, Syslog, NTP, TFTP) plus a local management jump host
- **WAN:** serial links from HQ-RTR to BR1-RTR and BR2-RTR (EIGRP AS 100)
- **Edge:** HQ-RTR performs PAT to a simulated ISP segment, with anti-spoofing filtering at the outside interface

## Key Design Decisions

- **STP root and HSRP active are explicitly aligned per VLAN** to avoid inter-Core hairpin traffic
- **Native VLAN (99) carries no traffic**; device management lives on a separate, per-site VLAN 100 to avoid the native-VLAN security anti-pattern
- **Routers use loopback interfaces for management addressing**; switches use their site's VLAN 100 SVI — a router was never a genuine member of any switched VLAN, so it shouldn't borrow an address from one
- **Every VLAN, including the management VLAN, is subnetted per site**, not stretched across WAN links — this was a real bug found and fixed during the build
- **Core2 has no direct WAN-facing link to HQ-RTR.** The ISR 4321 platform used for HQ-RTR only provides 2 onboard Layer 3-capable Gigabit ports, and the NIM module needed to add a third true routed port (NIM-1GE-CU-SFP) was not available in this Packet Tracer version. As a result, HQ-RTR's two onboard ports are used for the Core1 link and the ISP-SIM edge link, leaving none for a direct Core2 link. Core2 instead reaches HQ-RTR's routes — including the EIGRP-redistributed default route — via its EIGRP adjacency with Core1 over VLAN 100.
- **VTY access restricted via `access-class`** to known management subnets only — SSH-only is not treated as sufficient on its own
- **NAT ACL scope is enforced on egress, not assumed.** Since only specific internal VLANs are permitted in the NAT source list, an additional outbound filter on the outside interface permits only traffic already carrying the router's translated (post-NAT) address — anything that reaches that interface still carrying an original internal address (meaning it was never translated) is dropped rather than allowed to leave untranslated

## Repository Structure

```
/configs        - Full running-config for every device
/screenshots    - verification evidence, show-command outputs
/docs           - IP addressing table, VLAN table, show-command outputs
```

## IP Addressing

| Site | VLAN | Purpose | Network | Mask |
|---|---|---|---|---|
| HQ | 10 | Sales | 10.1.1.0 | /24 |
| HQ | 20 | Engineering | 20.1.1.0 | /24 |
| HQ | 30 | Exec | 30.1.1.0 | /24 |
| HQ | 40 | Servers | 40.1.1.0 | /24 (sized for future DC growth; 4 servers currently at .10/.20/.30/.40) |
| HQ | 100 | Mgmt | 100.1.1.0 | /28 |
| BR1 | 10 | Sales | 10.2.1.0 | /24 |
| BR1 | 20 | Engineering | 20.2.1.0 | /24 |
| BR1 | 100 | Mgmt | 100.2.1.0 | /30 |
| BR2 | 10 | Sales | 10.2.2.0 | /24 |
| BR2 | 30 | Exec | 30.2.2.0 | /24 |
| BR2 | 100 | Mgmt | 100.2.2.0 | /30 |

**WAN / Transit Links**

| Link | Network | Mask |
|---|---|---|
| Core1 ↔ HQ-RTR | 200.1.1.0 | /30 |
| HQ-RTR ↔ BR1-RTR (serial) | 200.2.1.0 | /30 |
| HQ-RTR ↔ BR2-RTR (serial) | 200.2.2.0 | /30 |

**Loopbacks (Router Management)**

| Device | Address | Mask |
|---|---|---|
| HQ-RTR | 1.1.1.1 | /32 |
| BR1-RTR | 3.3.3.3 | /32 |
| BR2-RTR | 2.2.2.2 | /32 |

## Verification Performed

- End-to-end reachability across all sites
- HSRP failover (Active shutdown → Standby takeover → preemption on recovery)
- EIGRP neighbor adjacencies and default route propagation (`D EX`)
- DHCP relay functioning per-VLAN, per-site
- NAT/PAT translation confirmed working only for permitted VLANs, with non-permitted VLANs confirmed blocked from reaching the internet
- SSH-only management access, confirmed refused from outside the permitted management subnets
- Password recovery performed live (see /docs) after a real lockout during the build

## Lessons Learned

- A single-octet subnet mask mismatch on one management host produced confusing, intermittent ping symptoms that looked like an ARP or HSRP fault before being isolated to simple addressing error — a reminder to verify addressing consistency across an entire subnet, not just spot-check a couple of devices.
- Reusing the same VLAN 100 subnet across multiple sites broke inter-site management reachability in both directions, since each router's own directly-connected route silently took priority over the correct routed path — the same class of mistake as reusing a VLAN's subnet company-wide instead of per-site.
- NAT/PAT alone does not enforce a security boundary — internal traffic that bypasses translation can still be routed back in if the upstream device has a route to private address space. Addressed by scoping the upstream route correctly and adding an egress filter that only permits already-translated traffic to leave.
- Invalid, discontiguous wildcard masks (bit patterns that don't reduce to a clean subnet boundary) caused two separate, non-obvious issues in this project — once in an EIGRP `network` statement, once in a NAT access list — reinforcing the importance of always deriving a wildcard mask by subtracting the subnet mask from 255.255.255.255, rather than typing a number that looks close.
- Packet Tracer's IOS command support is a reduced subset of real IOS (`logging trap` keyword names, `logging origin-id`, NIM module Layer 3 support) — several design decisions were adapted around these documented limitations rather than treated as configuration failures.

## Known Limitations (Packet Tracer vs. Production)

- **NIM-1GE-CU-SFP module unavailable** — HQ-RTR (ISR 4321) only has 2 onboard Layer 3-capable Gigabit ports, and the module needed for a third true routed port wasn't available in this Packet Tracer version. This is the direct reason Core2 has no direct link to HQ-RTR; see Key Design Decisions above.
- No conditional DNS forwarding available — external hostname resolution simulated with a static record on the internal DNS server rather than a live forwarder
- `logging origin-id` unsupported — syslog entries identified by source IP, cross-referenced against the IP addressing table in /docs rather than self-identifying by hostname
