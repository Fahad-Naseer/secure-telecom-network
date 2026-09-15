# Secure Telecom Network — Cisco Packet Tracer Project

A segmented, routed, and secured enterprise network designed and configured in Cisco Packet Tracer for a fictional telecom company. The network separates four departments into their own VLANs, routes between them through a two-tier distribution layer, provides automatic addressing and name resolution, connects out to an ISP with NAT/PAT, and restricts device management to the network admin team only.

## Overview

| | |
|---|---|
| **Tool** | Cisco Packet Tracer |
| **Routing protocol** | OSPF (single area) |
| **Segmentation** | 4 VLANs, router-on-a-stick across 2 distribution switches |
| **Internet access** | NAT/PAT (overload) via a simulated ISP router |
| **Services** | DHCP (per-VLAN pools via `ip helper-address`), DNS, HTTP |
| **Security** | Standard ACL restricting router management (Telnet) to the Admin VLAN only |

## Topology

```
                              ISP Router
                                  |
                            (WAN link, /30)
                                  |
                             Core Router (2911)
                          Gi0/0            Gi0/2
                       (trunk)             (trunk)
                          |                    |
                 Distribution SW 1      Distribution SW 2
                  (VLAN 10, 20)          (VLAN 30, 40)
                    /        \              /        \
              SW-ADMIN   SW-SUPPORT   SW-SALES   SW-SERVERS
               (VLAN10)   (VLAN20)    (VLAN30)    (VLAN40)
                 |            |          |         |  |  |
                PCs          PCs        PCs      DHCP DNS Web
```

Inter-VLAN routing is handled entirely by the Core Router using sub-interfaces (router-on-a-stick), one pair of sub-interfaces per distribution switch.

## IP Addressing Plan (VLSM, base block `192.168.10.0/24`)

| Department | VLAN | Hosts needed | Subnet | Usable range | Gateway |
|---|---|---|---|---|---|
| Support | 20 | 50 | `192.168.10.0/26` | .1 – .62 | .1 |
| Admin / HR | 10 | 20 | `192.168.10.64/27` | .65 – .94 | .65 |
| Sales | 30 | 20 | `192.168.10.96/27` | .97 – .126 | .97 |
| Server farm | 40 | 10 | `192.168.10.128/28` | .129 – .142 | .129 |
| Router ↔ ISP link | — | 2 | `192.168.10.144/30` | .145 – .146 | — |

Server farm static assignments: DHCP server `.130`, DNS server `.131`, Web server `.132`.

## Devices

- 1× Router 2911 — Core Router (inter-VLAN routing, NAT/PAT, OSPF, ACL)
- 1× Router 2911 — ISP Router (simulated internet edge)
- 2× Switch 2960 — Distribution switches (trunking between core and access layer)
- 4× Switch 2960 — Access switches, one per department
- 3× Server — DHCP, DNS, HTTP (Server farm VLAN)
- Multiple PCs across the Admin, Support, and Sales VLANs

## Key Configuration

**VLAN sub-interfaces on the Core Router (router-on-a-stick):**
```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.65 255.255.255.224
 ip helper-address 192.168.10.130
 ip nat inside

interface GigabitEthernet0/2.40
 encapsulation dot1Q 40
 ip address 192.168.10.129 255.255.255.240
 ip nat inside
```
*(repeated per VLAN sub-interface; `ip helper-address` relays DHCP broadcasts across VLANs to the DHCP server)*

**NAT/PAT — all internal VLANs share the router's public-facing IP:**
```
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

**OSPF — advertises every internal subnet:**
```
router ospf 1
 network 192.168.10.0   0.0.0.63 area 0
 network 192.168.10.64  0.0.0.31 area 0
 network 192.168.10.96  0.0.0.31 area 0
 network 192.168.10.128 0.0.0.15 area 0
 network 192.168.10.144 0.0.0.3  area 0
```

**Distribution switch trunking:**
```
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

**Management-plane security (only Admin VLAN can Telnet into the router):**
```
access-list 10 permit 192.168.10.64 0.0.0.31
access-list 10 deny any

line vty 0 4
 access-class 10 in
 password cisco
 login
```

## Services

- **DHCP:** one pool per department VLAN, each with its own default gateway, DNS server, and address range.
- **DNS:** internal record `intranet.telecom.local` → `192.168.10.132` (the Web server).
- **HTTP:** Web server reachable by name from every department once DNS + DHCP are working.

## Testing & Verification

- ✅ Each department PC receives an address automatically from its own DHCP pool
- ✅ Each PC can ping its own default gateway
- ✅ Cross-VLAN connectivity confirmed (e.g. Admin ↔ Sales)
- ✅ Internal DNS resolves `intranet.telecom.local` and the page loads in-browser
- ✅ `show ip nat translations` confirms private-to-public translation toward the ISP
- ✅ Telnet to the Core Router succeeds from the Admin VLAN and is refused from every other VLAN

## What I'd Add Next

- Extended ACLs for finer-grained service-level restrictions (e.g. limiting which VLANs can reach which server ports)
- A redundant link between the two distribution switches with Spanning Tree managing the loop
- IPv6 addressing on at least one link (dual-stack)

## How to Open

Requires [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (free via Cisco Networking Academy). Open `secure-telecom-network.pkt` directly.

---
Built while studying Computer Networks (CNC601400) — Superior University, Lahore.
