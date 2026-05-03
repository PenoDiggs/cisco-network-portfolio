# Bank Branch LAN with VLAN Segmentation, Inter-VLAN Routing, ACLs and SSH

**Tool:** Cisco Packet Tracer | **Industry:** Banking | **Difficulty:** Foundation

---

## Why I Built This

I was reading about how Nigerian banks structure their internal networks and one thing kept coming up: isolation. The ATM network cannot talk to the teller network. The guest WiFi cannot reach anything internal. Management systems sit completely apart from everything else. This is not just good practice, it is a CBN regulatory requirement.

So I asked myself: what does that actually look like on a Cisco switch? How do you physically enforce that an ATM machine cannot send traffic to a teller PC even though they are in the same building, connected to the same switch?

This project is my answer to that question. I built a single bank branch network from scratch in Cisco Packet Tracer and implemented every layer of that isolation using VLANs, inter-VLAN routing, and Extended ACLs. By the end, I could prove with ping tests that the ATM device was completely cut off from every other department, exactly the way a real bank branch would be configured.

---

## What the Network Looks Like


![Bank Branch LAN Topology](Network%20Topology.png)
```

---

## IP Addressing Plan

| Device | VLAN | IP Address | Subnet Mask | Default Gateway |
|--------|------|------------|-------------|-----------------|
| L3 Switch SVI (Tellers) | 10 | 192.168.10.1 | 255.255.255.0 | n/a |
| L3 Switch SVI (Management) | 20 | 192.168.20.1 | 255.255.255.0 | n/a |
| L3 Switch SVI (ATM) | 30 | 192.168.30.1 | 255.255.255.0 | n/a |
| L3 Switch SVI (Guest) | 40 | 192.168.40.1 | 255.255.255.0 | n/a |
| PC-Teller1 | 10 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC-Teller2 | 10 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 |
| Manager-PC | 20 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| Server | 20 | 192.168.20.20 | 255.255.255.0 | 192.168.20.1 |
| ATM-Device | 30 | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| Laptop-Guest | 40 | 192.168.40.10 | 255.255.255.0 | 192.168.40.1 |
| Router G0/0 | n/a | 192.168.0.2 | 255.255.255.252 | n/a |
| L3 Switch G0/1 | n/a | 192.168.0.1 | 255.255.255.252 | n/a |

---

## Full Configuration

### Layer 2 Switch (Cisco 2960)

**Basic setup and VLAN creation**
```
enable
configure terminal
no ip domain-lookup
hostname L2-Switch

vlan 10
 name Tellers
vlan 20
 name Management
vlan 30
 name ATM
vlan 40
 name Guest
exit
```

**Assigning access ports to VLANs with port security**
```
interface range fa0/1-2
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

interface range fa0/3-4
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

interface fa0/5
 switchport mode access
 switchport access vlan 30
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

interface fa0/6
 switchport mode access
 switchport access vlan 40
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit
```

**Trunk link to L3 switch**
```
interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
exit
```

**SSH configuration**
```
ip domain-name bank.local
username admin privilege 15 secret Cisco123!
crypto key generate rsa
 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
exit
enable secret Cisco123!
service password-encryption
end
write memory
```

---

### Layer 3 Switch (Cisco 3560)

**Basic setup, enable routing, create VLANs**
```
enable
configure terminal
no ip domain-lookup
hostname L3-Switch
ip routing

vlan 10
 name Tellers
vlan 20
 name Management
vlan 30
 name ATM
vlan 40
 name Guest
exit
```

**Create SVIs (these become the default gateways for each VLAN)**
```
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit

interface vlan 40
 ip address 192.168.40.1 255.255.255.0
 no shutdown
exit
```

**Trunk port facing L2 switch and routed port facing router**
```
interface g0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
exit

interface g0/1
 no switchport
 ip address 192.168.0.1 255.255.255.252
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 192.168.0.2
```

**Extended ACLs to isolate ATM and Guest VLANs**
```
ip access-list extended BLOCK-ATM
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip any any
exit

ip access-list extended BLOCK-GUEST
 deny ip 192.168.40.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip any any
exit

interface vlan 30
 ip access-group BLOCK-ATM in
exit

interface vlan 40
 ip access-group BLOCK-GUEST in
exit
```

**SSH configuration**
```
ip domain-name bank.local
username admin privilege 15 secret Cisco123!
crypto key generate rsa
 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
exit
enable secret Cisco123!
service password-encryption
end
write memory
```

---

### Router (Cisco 2911)

```
enable
configure terminal
no ip domain-lookup
hostname Bank-Router

interface g0/0
 ip address 192.168.0.2 255.255.255.252
 no shutdown
exit

ip route 192.168.10.0 255.255.255.0 192.168.0.1
ip route 192.168.20.0 255.255.255.0 192.168.0.1
ip route 192.168.30.0 255.255.255.0 192.168.0.1
ip route 192.168.40.0 255.255.255.0 192.168.0.1

ip domain-name bank.local
username admin privilege 15 secret Cisco123!
crypto key generate rsa
 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login local
exit
enable secret Cisco123!
service password-encryption
end
write memory
```

---

## Verification Commands

These are the commands I used to confirm everything was working:

```
! On L3 Switch
show ip route
show ip interface brief
show ip access-lists
show interfaces trunk
show vlan brief

! On L2 Switch
show vlan brief
show interfaces trunk
show interfaces fa0/1 status
show port-security
show mac address-table
```

---

## Test Results

| Test | Source | Destination | Expected | Actual |
|------|--------|-------------|----------|--------|
| Same VLAN ping | PC-Teller1 | PC-Teller2 | Pass | Pass |
| Inter-VLAN routing | PC-Teller1 | Manager-PC | Pass | Pass |
| ACL blocking ATM | ATM-Device | PC-Teller1 | Blocked | Blocked |
| ACL blocking ATM | ATM-Device | Manager-PC | Blocked | Blocked |
| Guest isolation | Laptop-Guest | PC-Teller1 | Blocked | Blocked |

The ATM device got back "Destination host unreachable" on every blocked ping, which means the ACL is firing at the SVI level before the packet even gets routed. That is exactly what you want in a real bank network.

---

## Challenges I Ran Into (and How I Fixed Them)

This section is probably the most honest part of this README. Building this was not straightforward and I think documenting the problems is just as important as documenting the solution.

**1. Port security shut down my access ports before the PCs were connected**

I configured port security with sticky MAC learning before I had finished cabling all the devices. When I later plugged the PCs in, the switch detected a new MAC on a port that had already learned a different MAC from when I was reconnecting cables, and it triggered a violation, putting the ports into err-disabled state. The fix was to remove port security completely, reset the ports with shutdown followed by no shutdown, then re-add port security after all devices were properly connected and cabled. Lesson learned: finish all physical connections before configuring port security.

**2. The trunk link was not establishing because the cable was on the wrong port**

I configured the trunk on G0/2 of the L3 switch but the physical cable was actually plugged into Fa0/2. Because of this, show interfaces trunk returned nothing even though the trunk configuration looked correct. Running show cdp neighbors showed the connection was on Fa0/2 instead of Gig0/2, which made it obvious. I deleted the cable and reconnected it to the correct Gig ports on both switches and the trunk came up immediately. CDP is genuinely useful for catching this kind of physical mismatch.

**3. I forgot that ip routing has to be explicitly enabled on the L3 switch**

On a regular Layer 2 switch there is no routing. On a Cisco 3560 multilayer switch, routing capability exists but it is turned off by default. I had created all the SVIs and assigned IP addresses but inter-VLAN traffic was still not moving because I had not typed ip routing. One command fixed everything. This is easy to forget and probably catches a lot of people out.

**4. ACLs disappeared after a session**

At one point I noticed show ip access-lists returned nothing even though I had configured the ACLs earlier. This happened because I had not run write memory after applying them, so when the simulated session refreshed, the running config was not saved to startup config. Always end a configuration session with write memory.

**5. First ping always shows 25% packet loss**

This one confused me initially. Every time I pinged between two VLANs for the first time, one packet would time out and the other three would succeed. This is normal and expected. The first packet is lost because the switch needs to run ARP to resolve the destination MAC address before it can forward the packet. Once ARP completes and the MAC is in the table, every subsequent ping is 100% successful. This is not a bug, it is just how Ethernet works.

---

## What I Learned

The biggest thing this project taught me is that network configuration is one thing and network verification is another. You can type every command correctly and still have nothing working because of a cable in the wrong port, or a port in err-disabled state, or a command that was not saved. The show commands (show ip route, show interfaces trunk, show vlan brief, show port-security) are just as important as the configuration commands. In a real job, that troubleshooting instinct is what separates someone who can configure from someone who can actually operate a network.

---

## Skills Covered

- VLAN design and 802.1Q trunking
- Layer 3 SVI inter-VLAN routing
- IP subnetting (both /24 and /30)
- Extended Named ACLs applied to SVIs
- Port security with sticky MAC learning
- SSH v2 hardening and Telnet removal
- Cisco IOS CLI across three device types (2911, 3560, 2960)
- Network troubleshooting using show commands

---

## Files in This Repository

| File | Description |
|------|-------------|
| `BankBranchLAN.pkt` | Cisco Packet Tracer project file |
| `README.md` | This file |

---

*Part of my Cisco Packet Tracer network engineering portfolio. Building toward CCNA. Projects cover banking, finance, and telecom infrastructure.*
