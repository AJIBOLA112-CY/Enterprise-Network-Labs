# Enterprise Network Design and Implementation Using Cisco Packet Tracer

## Project Overview

This project demonstrates the design, configuration, implementation and troubleshooting of a multi-VLAN enterprise network using Cisco Packet Tracer.

The network simulates an office environment with three departments: Sales, IT and Management. Each department operates on a separate VLAN and IP subnet.

The project covers VLSM subnetting, VLANs, trunking, inter-VLAN routing, DHCP, static routing, NAT/PAT and extended Access Control Lists (ACLs).

## Network Topology

The network consists of:
- 2 Cisco 2911 routers (Core Router and ISP Router)
- 1 Cisco 2960 switch
- 6 end-user PCs
- ![Enterprise Network Topology](Screenshot%20(1022).png)

### Departmental Structure

| Department | PCs | VLAN |
|---|---|---|
| Sales | PC0, PC1 | 10 |
| IT | PC2, PC3 | 20 |
| Management | PC4, PC5 | 30 |

## IP Addressing and VLSM

The base network is `192.168.10.0/24`.

| Department | VLAN | Network | Subnet Mask | Gateway |
|---|---:|---|---|---|
| Sales | 10 | 192.168.10.0/26 | 255.255.255.192 | 192.168.10.1 |
| IT | 20 | 192.168.10.64/27 | 255.255.255.224 | 192.168.10.65 |
| Management | 30 | 192.168.10.96/28 | 255.255.255.240 | 192.168.10.97 |

### WAN Addressing

| Device | Interface | IP Address |
|---|---|---|
| Core Router | G0/0 | 203.0.113.1/30 |
| ISP Router | G0/0 | 203.0.113.2/30 |
| ISP Router | Loopback0 | 8.8.8.8/32 |

## VLAN and Switch Configuration

Three VLANs were created on the Cisco 2960 switch.

```bash
vlan 10
name Sales

vlan 20
name IT

vlan 30
name Management
```

Access port assignments:

| Switch Ports | Department | VLAN |
|---|---|---:|
| Fa0/1–Fa0/2 | Sales | 10 |
| Fa0/3–Fa0/4 | IT | 20 |
| Fa0/5–Fa0/6 | Management | 30 |

The uplink to the Core Router was configured as an 802.1Q trunk.

```bash
interface gigabitEthernet0/1
switchport mode trunk
```

PortFast was enabled on the PC-facing access ports.

## Inter-VLAN Routing

Router-on-a-Stick was implemented using subinterfaces on the Core Router.

```bash
interface gigabitEthernet0/2.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.192

interface gigabitEthernet0/2.20
encapsulation dot1Q 20
ip address 192.168.10.65 255.255.255.224

interface gigabitEthernet0/2.30
encapsulation dot1Q 30
ip address 192.168.10.97 255.255.255.240
```

Each subinterface acts as the default gateway for its respective VLAN.

## DHCP Configuration

The Core Router was configured as the DHCP server to automatically assign IP addresses to departmental PCs.

Gateway addresses were excluded from DHCP allocation.

```bash
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.65
ip dhcp excluded-address 192.168.10.97
```

### Sales DHCP Pool

```bash
ip dhcp pool SALES
network 192.168.10.0 255.255.255.192
default-router 192.168.10.1
dns-server 8.8.8.8
```

### IT DHCP Pool

```bash
ip dhcp pool IT
network 192.168.10.64 255.255.255.224
default-router 192.168.10.65
dns-server 8.8.8.8
```

### Management DHCP Pool

```bash
ip dhcp pool Management
network 192.168.10.96 255.255.255.240
default-router 192.168.10.97
dns-server 8.8.8.8
```

## Static Default Routing

The Core Router was configured with a default static route pointing towards the ISP Router.

```bash
ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

## NAT/PAT Configuration

NAT Overload (PAT) was configured to allow multiple internal devices to share the Core Router's outside IP address.

### NAT Interface Configuration

The ISP-facing interface was configured as outside:

```bash
interface gigabitEthernet0/0
ip nat outside
```

The LAN-facing physical interface and VLAN subinterfaces were configured as inside:

```bash
interface gigabitEthernet0/2
ip nat inside

interface gigabitEthernet0/2.10
ip nat inside

interface gigabitEthernet0/2.20
ip nat inside

interface gigabitEthernet0/2.30
ip nat inside
```

### NAT Access List

```bash
access-list 1 permit 192.168.10.0 0.0.0.255
```

### PAT Overload Rule

```bash
ip nat inside source list 1 interface gigabitEthernet0/0 overload
```

This allows multiple internal devices to share the outside IP address `203.0.113.1`.

## Extended ACL Configuration

Extended ACL 101 was configured to prevent Sales from initiating ICMP echo requests to Management while permitting other departmental traffic.

```bash
access-list 101 deny icmp 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15 echo

access-list 101 permit ip 192.168.10.0 0.0.0.63 any

access-list 101 permit ip 192.168.10.64 0.0.0.31 any

access-list 101 permit ip 192.168.10.96 0.0.0.15 any
```

The ACL was applied inbound on the Sales subinterface.

```bash
interface gigabitEthernet0/2.10
ip access-group 101 in
```

### Security Policy

- Sales cannot initiate ping requests to Management.
- Sales can still access the simulated internet.
- IT can communicate with all departments.
- Management can communicate with Sales.

The ACL specifically restricts ICMP echo requests; it does not block every type of communication between Sales and Management.

## Troubleshooting Experience


During implementation, the Sales PC initially failed to ping the simulated internet destination `8.8.8.8`.

Troubleshooting included:
1. Checking the PC's IPv4 configuration using `ipconfig`.
2. Verifying the default gateway using `ipconfig /all`.
3. Testing connectivity to the default gateway.
4. Inspecting NAT translations and NAT statistics.
5. Checking the NAT access list and PAT configuration.
6. Verifying router interfaces and routing tables.
7. Configuring `ip nat inside` on the VLAN subinterfaces.

After configuring NAT inside on the VLAN subinterfaces, Sales PC internet connectivity was successfully established.

This demonstrated the importance of understanding the actual traffic path when troubleshooting NAT in a Router-on-a-Stick environment.

### DHCP Failure Caused by Extended ACL

**Problem:** After saving and reopening the Packet Tracer project, the Sales PCs were unable to obtain IP addresses through DHCP. PC0 received an APIPA address (`169.254.x.x`).

**Investigation:**
- Verified the Sales VLAN assignment on the access switch.
- Checked the trunk configuration and confirmed VLAN 10 was active.
- Verified the router subinterface and DHCP pool configuration.
- Identified that Extended ACL 101 was applied inbound on the Sales subinterface.

**Root Cause:** DHCP Discover packets initially use source IP address `0.0.0.0`. The original ACL permitted traffic from the Sales subnet (`192.168.10.0/26`) but did not explicitly permit DHCP client requests. These requests were therefore blocked by the ACL's implicit deny.

**Solution:** Added a DHCP permit rule before the existing security rules:

```cisco
access-list 101 permit udp any eq 68 any eq 67
```

Restored ACL 101 on the Sales subinterface:

```cisco
interface gigabitEthernet0/2.10
ip access-group 101 in
```

**Verification:**
- Sales PC successfully renewed its DHCP lease.
- Sales received the correct IP address, subnet mask, gateway and DNS server.
- Sales-to-Management ICMP traffic remained blocked.
- IT-to-Management and Management-to-Sales connectivity remained successful.

**Lesson Learned:** Extended ACLs must account for essential network services such as DHCP. Rule order matters because traffic that does not match an explicit permit or deny rule is subject to the implicit deny at the end of the ACL.

## Verification and Testing

| Test | Result |
|---|---|
| Sales PC to default gateway | Passed |
| Core Router to 8.8.8.8 | Passed |
| Sales to 8.8.8.8 | Passed |
| IT to 8.8.8.8 | Passed |
| Management to 8.8.8.8 | Passed |
| Sales to Management ping | Blocked as intended |
| IT to Management ping | Passed |
| Management to Sales ping | Passed |

The ACL deny rule recorded four matches during the Sales-to-Management ping test.

The Core Router routing table confirmed all three departmental networks as directly connected and the default route through `203.0.113.2`.

## Useful Verification Commands

| Command | Purpose |
|---|---|
| `show vlan brief` | Verify VLANs and port assignments |
| `show interfaces trunk` | Verify trunk configuration |
| `show ip interface brief` | Check interface addresses and status |
| `show ip route` | Inspect the routing table |
| `show ip dhcp binding` | View DHCP leases |
| `show access-lists` | Inspect ACLs and match counters |
| `show ip nat statistics` | Inspect NAT statistics |
| `show ip nat translations` | View NAT translations |
| `ping` | Test connectivity |

## Technologies and Skills Demonstrated

- Cisco Packet Tracer
- IPv4 addressing and VLSM subnetting
- VLAN configuration
- IEEE 802.1Q trunking
- Router-on-a-Stick
- DHCP
- Static routing
- NAT and PAT
- Standard and extended ACLs
- Network troubleshooting and verification

## Project Files

The repository is intended to contain:

```text
Enterprise-Network-Labs/
├── README.md
├── Packet-Tracer/
│   └── Enterprise-Network-Labs-one.pkt
└── Screenshots/
    ├── network-topology.png
    ├── vlan-configuration.png
    ├── nat-configuration.png
    └── acl-testing.png
```

## Conclusion

This project provided hands-on experience in designing, configuring, securing and troubleshooting a multi-VLAN enterprise network.

It strengthened my understanding of Cisco networking concepts, including inter-VLAN routing, DHCP, NAT/PAT, static routing and Access Control Lists.
