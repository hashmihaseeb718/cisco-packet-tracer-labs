# Multi-Area OSPF Enterprise Network Lab
A five-site enterprise network built in Cisco Packet Tracer. It combines **multi-area OSPF** with a **virtual link**, **inter-VLAN routing (router-on-a-stick)**, **ACL-based traffic isolation** between departments, and **Telnet** remote management.


## What this lab demonstrates

| Topic | Implementation |
|---|---|
| Dynamic routing | Multi-area OSPF across areas 0, 1 and 2 |
| Area design | ABRs at R1 (areas 0/1) and R2 (areas 1/2); R2 reaches the backbone through a virtual link |
| Segmentation | VLAN 10 (IT) and VLAN 20 (HR) at every site |
| Inter-VLAN routing | Router-on-a-stick: 802.1Q trunk plus one sub-interface per VLAN |
| Security policy | Extended ACLs block IT-to-HR traffic in both directions |
| Management | Telnet on the VTY lines of each router |

## Addressing

### LAN sub-interfaces (default gateways)

| Router | Gi0/0.10 (VLAN 10, IT) | Gi0/0.20 (VLAN 20, HR) | Switch | PCs |
|---|---|---|---|---|
| R0 | 20.1.1.1/16 | 20.2.1.1/16 | SW0 | PC0 (IT), PC1 (HR), PC2 (IT) |
| R1 | 30.1.1.1/16 | 30.2.1.1/16 | SW1 | PC6 (IT), PC7 (HR), PC8 (HR) |
| R2 | 50.1.1.1/16 | 50.2.1.1/16 | SW2 | PC3 (IT), PC4 (IT), PC5 (HR) |
| R3 | 70.1.1.1/16 | 70.2.1.1/16 | SW3 | PC9 (IT), PC10 (HR), PC11 (IT) |
| R4 | 90.1.1.1/16 | 90.2.1.1/16 | SW4 | PC12 (IT), PC13 (HR), PC14 (HR) |

### Serial (WAN) links

| Link | Endpoints | OSPF area |
|---|---|---|
| R0 - R1 | 10.1.1.1 / 10.1.1.2 | 0 |
| R1 - R2 | 40.1.1.1 / 40.1.1.2 | 1 |
| R2 - R3 | 60.1.1.1 / 60.1.1.2 | 2 |
| R3 - R4 | 80.1.1.1 / 80.1.1.2 | 2 |

## OSPF design

- **Area 0** is the backbone (R0-R1 link).
- **Area 1** sits between R1 and R2. R1 is the ABR for areas 0 and 1.
- **Area 2** covers R2 to R4. R2 is the ABR for areas 1 and 2.
- Area 2 has no physical link to area 0, so a **virtual link between R1 and R2** (transit area 1) attaches R2 to the backbone.

## Key configuration

> Replace passwords and values with your own. Never commit real credentials.

<details>
<summary><b>Router-on-a-stick (example: R0)</b></summary>

```
interface GigabitEthernet0/0
 no shutdown
!
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 20.1.1.1 255.255.0.0
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 20.2.1.1 255.255.0.0
```
</details>

<details>
<summary><b>Switch VLANs and trunk (example: SW0)</b></summary>

```
vlan 10
 name IT
vlan 20
 name HR
!
interface FastEthernet0/1
 switchport mode trunk
!
interface FastEthernet1/1
 switchport mode access
 switchport access vlan 10
```
</details>

<details>
<summary><b>Virtual link (R1 and R2)</b></summary>

```
! R1
router ospf 1
 area 1 virtual-link 3.3.3.3

! R2
router ospf 1
 area 1 virtual-link 2.2.2.2
```
</details>

<details>
<summary><b>ACL: block IT to HR (example: R0)</b></summary>

```
ip access-list extended BLOCK-IT-TO-HR
 deny ip 20.1.0.0 0.0.255.255 20.2.0.0 0.0.255.255
 deny ip 20.1.0.0 0.0.255.255 30.2.0.0 0.0.255.255
 deny ip 20.1.0.0 0.0.255.255 50.2.0.0 0.0.255.255
 deny ip 20.1.0.0 0.0.255.255 70.2.0.0 0.0.255.255
 deny ip 20.1.0.0 0.0.255.255 90.2.0.0 0.0.255.255
 permit ip any any
!
interface GigabitEthernet0/0.10
 ip access-group BLOCK-IT-TO-HR in
```

A mirror ACL (HR as source, IT subnets as destination) is applied inbound on `Gi0/0.20`, and the same pattern is repeated on every router.
</details>

<details>
<summary><b>Telnet on VTY lines</b></summary>

```
line vty 0 4
 password <your-password>
 login
 transport input telnet
```
</details>

## Verification

| Test | Expected result |
|---|---|
| IT PC to IT PC at another site | Success |
| HR PC to HR PC at another site | Success |
| IT PC to HR PC (same site) | Blocked |
| IT PC to HR PC (other site) | Blocked |
| HR PC to IT PC (any site) | Blocked |
| `telnet <router-ip>` from a PC | Login prompt |

Useful commands: `show ip route`, `show ip ospf neighbor`, `show ip ospf virtual-links`, `show access-lists`, `show vlan brief`, `show interfaces trunk`.

<!-- Add your screenshots to /images and uncomment:
![Allowed ping](images/ping-allowed.png)
![Blocked ping](images/ping-blocked.png)
![ACL hit counters](images/show-access-lists.png)
-->

## Troubleshooting case study

**Symptom:** the area 1 networks could not communicate with the far end of the topology, although every other path worked.

**Diagnosis:** `show ip route` on the far router showed no routes back to area 1. A ping reached its destination, but the reply was dropped for lack of a return route.

**Root cause:** the last link was placed in a separate area, so the router at the edge was learning area 1 routes through a chain of virtual links that Packet Tracer did not propagate correctly.

**Fix:** re-assigned the final link to area 2, which removed the second virtual link and gave the far-end router a direct path to the area 1 routes.

**Lesson:** area design matters as much as configuration. Every area needs a clean path to the backbone.

## Limitations and next steps

- **Telnet is unencrypted.** Production networks should use SSH with local or AAA authentication.
- **Router-on-a-stick does not scale.** A Layer 3 switch with SVIs is the production approach.
- **No redundancy.** Each site has a single path, so a link failure partitions the network.
- Possible extensions: DHCP per VLAN, port security, SSH, a redundant WAN link.


## Author

**Haseeb** - BS Information Technology, NUML | Networking & Cybersecurity
(Connect on [LinkedIn](www.linkedin.com/in/haseeb-hashmi-9bb3293a6))
