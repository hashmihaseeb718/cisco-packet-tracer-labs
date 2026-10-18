# Multi-Area OSPF Enterprise Network (Cisco Packet Tracer)

A small enterprise network lab with a multi-area OSPF core, Layer 3 switching, inter-VLAN routing, NAT/PAT, ACL-based VLAN filtering and a simulated internet. Built in Cisco Packet Tracer.

## Overview

The network has an Area 0 backbone with two edge routers and two multilayer switches, plus two Area 1 branches connected through their own L3 switches. The edge routers connect to an ISP router with redundant links. A web server behind the ISP stands in for the internet.

## Features

| Area | What is configured |
|------|--------------------|
| Routing | OSPF multi-area (Area 0 backbone, Area 1 branches) |
| Switching | Multilayer (L3) switches, VLAN10 and VLAN20, inter-VLAN routing via SVIs |
| Redundancy | EtherChannel between the core switches, STP, dual edge routers |
| Addressing | DHCP for both VLANs |
| Internet access | Static default routes to the ISP, default route propagated into OSPF, NAT/PAT (overload) |
| Security | Extended ACLs blocking VLAN10 from VLAN20, SSHv2 for remote management |
| Simulated internet | HTTP server at 203.1.1.10 behind the ISP router |

## Topology

| Device | Role |
|--------|------|
| Router0 | ISP router, NAT/PAT and static routes to the internal networks |
| Router1, Router2 | Edge routers, Area 0 |
| Multilayer Switch0, Multilayer Switch1 | Core L3 switches, Area 0 |
| Multilayer Switch2, Multilayer Switch3 | Branch L3 switches, Area 1 |
| Switch0 to Switch3 | Access switches |
| PC0 to PC7 | End hosts (VLAN10 and VLAN20) |
| Server0 | Simulated internet web server |

## Addressing

All links are /24.

| Subnet | Link |
|--------|------|
| 192.168.1.0 | Router0 (ISP) to Router2 |
| 192.168.2.0 | Router0 (ISP) to Router1 |
| 192.168.3.0 | Router2 to Router1 |
| 192.168.4.0 | Router2 to Multilayer Switch0 |
| 192.168.5.0 | Router1 to Multilayer Switch1 |
| 192.168.6.0 | Router2 to Multilayer Switch1 |
| 192.168.7.0 | Router1 to Multilayer Switch0 |
| 192.168.8.0 / 192.168.9.0 | Multilayer Switch0 VLAN10 / VLAN20 |
| 192.168.10.0 / 192.168.11.0 | Multilayer Switch1 VLAN10 / VLAN20 |
| 192.168.20.0 | Router2 to Multilayer Switch2 |
| 192.168.21.0 / 192.168.22.0 | Multilayer Switch2 VLAN10 / VLAN20 |
| 192.168.30.0 | Router1 to Multilayer Switch3 |
| 192.168.31.0 / 192.168.32.0 | Multilayer Switch3 VLAN10 / VLAN20 |
| 192.168.40.0 | Multilayer Switch2 to Multilayer Switch3 |
| 203.1.1.0 | ISP to internet server (203.1.1.10) |

## Key Configurations

### ISP router: static routes and NAT/PAT

```
interface g0/0
 ip address 203.1.1.1 255.255.255.0
 ip nat outside
interface s0/0/0
 ip nat inside
interface s0/0/1
 ip nat inside

access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface g0/0 overload

ip route 192.168.0.0 255.255.0.0 192.168.1.2
ip route 192.168.0.0 255.255.0.0 192.168.2.2 10
```

The second route is a floating static route. It only activates if the primary link through Router2 fails.

### Edge routers: default route into OSPF

```
! Router2
ip route 0.0.0.0 0.0.0.0 192.168.1.1
router ospf 1
 default-information originate

! Router1
ip route 0.0.0.0 0.0.0.0 192.168.2.1
router ospf 1
 default-information originate
```

`default-information originate` advertises the default route to the L3 switches, so hosts in every VLAN learn the path to the ISP.

### ACL: block VLAN10 from VLAN20 (Multilayer Switch0)

```
ip access-list extended BLOCK-V10-TO-V20
 deny ip 192.168.8.0 0.0.0.255 192.168.9.0 0.0.0.255
 deny ip 192.168.8.0 0.0.0.255 192.168.11.0 0.0.0.255
 deny ip 192.168.8.0 0.0.0.255 192.168.22.0 0.0.0.255
 deny ip 192.168.8.0 0.0.0.255 192.168.32.0 0.0.0.255
 permit ip any any
interface vlan 10
 ip access-group BLOCK-V10-TO-V20 in
```

The same ACL is applied on the other three L3 switches, changing only the VLAN10 source subnet (192.168.10.0, 192.168.21.0 and 192.168.31.0).

### SSHv2

```
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin secret <password>
line vty 0 15
 transport input ssh
 login local
```

## Verification

| Test | Expected result |
|------|-----------------|
| PC0 pings PC1 (VLAN10 to VLAN20) | Fails (blocked by ACL) |
| PC0 pings PC2 (VLAN10 to VLAN10) | Succeeds |
| Any PC browses `http://203.1.1.10` | Web page loads through NAT |
| `show ip nat translations` on the ISP | Private addresses mapped to 203.1.1.1 |
| `show ip route` on an L3 switch | `O*E2 0.0.0.0/0` default route present |
| `show access-lists` on an L3 switch | Match counters increase on the deny lines |
| `show ip ospf neighbor` | Adjacencies Full on all OSPF links |
| `show etherchannel summary` | Bundle shows `SU` |
| Shut ISP link on Router2, then ping 203.1.1.10 | Traffic fails over through Router1 |

## How to Run

1. Install Cisco Packet Tracer.
2. Open `Multi-Area-OSPF-NAT-ACL.pkt` from this folder.
3. Open the Desktop tab on any PC and use the Web Browser to visit `http://203.1.1.10`.
4. Use the Command Prompt on a PC to test pings between VLANs.

## What I Learned

- A "destination unreachable" reply from the first L3 switch usually means the default route never reached it, so check `default-information originate`.
- NAT only works if the ISP has a route back to the inside networks. The translation can be correct and the replies still get dropped.
- Every ACL ends with an implicit deny, so a `permit ip any any` line is needed when you only want to block specific traffic.
- `show ip nat translations` and `show access-lists` make it easy to see NAT and ACLs working in real time.

## Design Notes

NAT runs on the ISP router to keep the lab simple. In a real network, NAT belongs on the customer's own edge routers or firewall, and the ISP routes only public addresses.

## Future Improvements

- Move NAT/PAT to the edge routers
- Add a firewall between the edge and the ISP
- OSPF authentication
- DHCP snooping and port security
- HSRP for gateway redundancy

## Tools

Cisco Packet Tracer, Cisco IOS CLI

## Author

**Haseeb** | AI Automation Engineer and BS IT student at NUML

CCNA completed at Corvit.

🔗 LinkedIn: [https://www.linkedin.com/in/haseeb-hashmi-9bb3293a6]
