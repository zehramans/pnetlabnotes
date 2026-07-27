# VLANs, Trunking and Router-on-a-Stick with Palo Alto

## Overview

The goal of this lab was to move from a flat network into a segmented enterprise network. Instead of having every device in the same broadcast domain, the network was divided into multiple VLANs while still allowing all clients to obtain IP addresses from a single DHCP server.

The topology consists of an Internet router connected to the Palo Alto firewall. The firewall is connected to a Cisco IOSvL2 switch acting as the distribution switch, which then connects to two access switches. Endpoint devices are connected to the access switches.

```
                    Internet
                        |
                  Internet Router
                        |
                 Palo Alto Firewall
                        |
                 802.1Q Trunk Link
                        |
            Distribution Switch (IOSvL2)
                 /                  \
            Trunk                    Trunk
             /                          \
      Access Switch 1             Access Switch 2
```

The Palo Alto performs all Layer 3 routing while every switch operates only at Layer 2.

---

# Why VLANs?

Without VLANs, every device connected to a switch belongs to the same Layer 2 network.

For example:

```
PC1
PC2
Linux
Printer
Server
```

All devices receive addresses from the same subnet.

```
192.168.10.0/24
```

Every broadcast is sent to every device.

As the network grows this becomes inefficient and difficult to secure.

VLANs allow one physical switch to be divided into multiple logical switches.

Example:

```
VLAN 10
Users

192.168.10.0/24
```

```
VLAN 20
Servers

192.168.20.0/24
```

```
VLAN 30
Management

192.168.30.0/24
```

Although all devices are connected to the same switch, each VLAN is its own independent broadcast domain.

---

# Access Ports

Ports connected to endpoint devices are configured as access ports.

Example:

```
PC

|

Ethernet0/0
```

```
interface Ethernet0/0

switchport mode access

switchport access vlan 10
```

This means the port belongs to exactly one VLAN.

A PC connected to this port becomes a member of VLAN 10.

The PC never knows VLANs exist because the switch removes VLAN tags before sending traffic to the endpoint.

---

# Why Trunks Are Needed

The connection between switches must carry traffic from multiple VLANs simultaneously.

Suppose Switch 1 contains:

```
VLAN 10
VLAN 20
VLAN 30
```

Switch 2 also contains:

```
VLAN 10
VLAN 20
VLAN 30
```

If the link between them were an access port, only one VLAN could cross.

Instead, the link is configured as a trunk.

```
Switch1
    ||
  Trunk
    ||
Switch2
```

A trunk carries traffic for multiple VLANs over a single physical cable.

The switch inserts an IEEE 802.1Q VLAN tag into every Ethernet frame so the receiving switch knows which VLAN the frame belongs to.

---

# Trunk Configuration

The links between networking devices were configured as trunks.

```
interface GigabitEthernet0/0

switchport mode trunk

switchport trunk allowed vlan 10,20,30
```

The `switchport mode trunk` command tells the switch to expect VLAN-tagged traffic.

The `switchport trunk allowed vlan` command limits the trunk to only the VLANs that are required.

Only links between networking devices should be trunks.

Links connected directly to endpoint devices remain access ports.

---

# Distribution Switch

The IOSvL2 switch sits in the middle of the network.

Its purpose is simply to forward Layer 2 traffic between switches and the firewall.

Even though it is called a multilayer switch, Layer 3 routing was intentionally left disabled in this lab.

Keeping routing on the Palo Alto allows all inter-VLAN traffic to pass through the firewall where security policies can be applied.

---

# Router-on-a-Stick

Originally the Palo Alto had a single interface:

```
ethernet1/2

192.168.10.1/24
```

This worked when only one network existed.

Once VLANs were introduced, that interface became a trunk.

Instead of assigning an IP address to the physical interface, separate subinterfaces were created.

```
ethernet1/2.10

Tag 10

192.168.10.1/24
```

```
ethernet1/2.20

Tag 20

192.168.20.1/24
```

```
ethernet1/2.30

Tag 30

192.168.30.1/24
```

Each subinterface represents the default gateway for one VLAN.

The physical interface simply carries tagged traffic.

This design is known as Router-on-a-Stick because one physical interface provides routing for multiple VLANs.

---

# DHCP Relay

The DHCP server exists in another subnet.

```
10.250.40.2
```

Clients send DHCP Discover packets as broadcasts.

Broadcasts do not cross Layer 3 boundaries.

To solve this problem, DHCP Relay was enabled on each Palo Alto subinterface.

```
ethernet1/2.10

DHCP Relay

10.250.40.2
```

```
ethernet1/2.20

DHCP Relay

10.250.40.2
```

```
ethernet1/2.30

DHCP Relay

10.250.40.2
```

Each subinterface forwards DHCP requests to the centralized DHCP server.

Only one DHCP server is required regardless of how many VLANs exist.

---

# Why DHCP Relay Is Configured Per Subinterface

The physical interface no longer belongs to any IP network.

Instead, every VLAN has its own Layer 3 gateway.

```
ethernet1/2.10

192.168.10.1
```

```
ethernet1/2.20

192.168.20.1
```

```
ethernet1/2.30

192.168.30.1
```

When a DHCP request arrives, the Palo Alto already knows which VLAN received it because it arrives on the corresponding subinterface.

The relay agent forwards the request using the gateway address of that VLAN so the DHCP server knows which address pool to use.

---

# Security Zones

Each VLAN was assigned its own security zone.

```
Users
```

```
Servers
```

```
Management
```

Instead of simply routing packets, the Palo Alto checks every packet that moves between zones.

For example:

```
Users

↓

Servers
```

or

```
Users

↓

Outside
```

Without a matching security policy, the firewall drops the traffic.

This also applies to DHCP Relay traffic because it travels from the Users zone to the DMZ zone.

During this lab, DHCP initially failed because no security policy existed allowing the relay traffic to reach the DHCP server.

Once the Users → DMZ policy was added, clients immediately began receiving IP addresses.

---

# Traffic Flow

A client in VLAN 10 follows this path:

```
PC

↓

Access Switch

↓

Trunk

↓

Distribution Switch

↓

Trunk

↓

Palo Alto

↓

DHCP Relay

↓

DHCP Server
```

Internet traffic follows a similar path.

```
PC

↓

Access Switch

↓

Distribution Switch

↓

Palo Alto

↓

NAT

↓

Internet Router

↓

Internet
```

The firewall performs routing, applies security policies and translates private addresses before forwarding traffic to the Internet.

---

# What I Learned

This lab introduced enterprise switching concepts and demonstrated how VLANs reduce broadcast domains while improving network organization.

I learned how access ports differ from trunk ports, how IEEE 802.1Q tagging allows multiple VLANs to traverse a single physical connection, and how Router-on-a-Stick allows one firewall interface to act as the default gateway for multiple networks.

I also learned that DHCP broadcasts cannot cross Layer 3 boundaries, making DHCP Relay necessary when the DHCP server is located in another subnet.

Finally, I gained a much better understanding of how Palo Alto security zones affect traffic flow. Unlike a traditional router, a Palo Alto firewall requires explicit security policies for traffic moving between zones, including DHCP relay traffic.
