# Palo Alto DHCP Relay Lab (PNETLab + VMware)

## Introduction

The purpose of this lab was to build a small enterprise-style network in PNETLab using a Palo Alto firewall, two Cisco routers, a Layer 2 switch and multiple clients. The objective was to have the clients receive their IP addresses from a DHCP server located on a different subnet while still being able to access the Internet through the Palo Alto firewall.

Unlike a simple home network, the DHCP server is not located on the same subnet as the clients. Because DHCP broadcasts cannot cross routers, DHCP Relay must be configured on the firewall.

The completed topology looks like this:

```
                 Internet
                     │
               VMware NAT
                     │
                 Cloud0 (PNETLab)
                     │
            Internet Router
      G0/0 (DHCP)     G0/1 (200.100.100.1)
                     │
             Palo Alto Firewall
      eth1/1               eth1/2             eth1/4
 200.100.100.10       192.168.10.1       10.250.40.1
                           │                   │
                        Switch          DHCP Router
                    VPC / Linux      10.250.40.2
```

---

# IP Addressing

| Device | Interface | IP Address |
|---------|-----------|------------|
| Internet Router | G0/0 | DHCP |
| Internet Router | G0/1 | 200.100.100.1/24 |
| Palo Alto | ethernet1/1 | 200.100.100.10/24 |
| Palo Alto | ethernet1/2 | 192.168.10.1/24 |
| Palo Alto | ethernet1/4 | 10.250.40.1/24 |
| DHCP Router | G0/0 | 10.250.40.2/24 |
| Clients | DHCP | 192.168.10.0/24 |

---

# Configuring the Internet Router

The Internet router acts as the gateway between the lab and VMware's NAT network.

The outside interface receives an IP address automatically from VMware.

```cisco
interface g0/0
 ip address dhcp
 ip nat outside
 no shutdown
```

The inside interface connects directly to the Palo Alto.

```cisco
interface g0/1
 ip address 200.100.100.1 255.255.255.0
 ip nat inside
 no shutdown
```

Since the router only knows about networks directly connected to it, it also needs a route telling it where the client network is located.

```cisco
ip route 192.168.10.0 255.255.255.0 200.100.100.10
```

Without this route, packets returning from the Internet would be dropped because the router would not know how to reach the client subnet.

---

# Configuring NAT on the Internet Router

The Internet router performs the second layer of NAT before traffic leaves VMware.

Initially I made the mistake of creating the ACL like this:

```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
```

Although this looked correct, it never matched any traffic.

The reason is that the Cisco router never sees the original client IP addresses.

Before traffic reaches the router, the Palo Alto has already translated:

```
192.168.10.21
        ↓
200.100.100.10
```

Therefore the ACL must match the translated network instead.

```cisco
access-list 1 permit 200.100.100.0 0.0.0.255

ip nat inside source list 1 interface g0/0 overload
```

Once this was corrected, Internet connectivity immediately started working.

---

# Configuring the DHCP Router

The DHCP router only provides DHCP services.

It does not perform any routing or NAT.

The interface connected to the Palo Alto was configured as:

```cisco
interface g0/0
 ip address 10.250.40.2 255.255.255.0
 no shutdown
```

A default route was added so the router could send replies back through the firewall.

```cisco
ip route 0.0.0.0 0.0.0.0 10.250.40.1
```

Finally, the DHCP pool was configured.

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20

ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

Notice that although the DHCP server itself lives on the 10.250.40.0 network, it hands out addresses from the 192.168.10.0 network. This is possible because DHCP Relay forwards the requests.

---

# Configuring the Switch

The switch required almost no configuration.

Each interface was configured as an access port.

```cisco
interface range g0/1-3
 switchport mode access
 spanning-tree portfast
 no shutdown
```

---

# Configuring the Palo Alto Interfaces

The firewall has three Layer 3 interfaces.

**ethernet1/1**

```
IP Address
200.100.100.10/24

Zone
Outside
```

**ethernet1/2**

```
IP Address
192.168.10.1/24

Zone
Inside
```

**ethernet1/4**

```
IP Address
10.250.40.1/24

Zone
DMZ
```

Each interface was added to the default virtual router.

---

# Configuring the Default Route

The Palo Alto requires a default route pointing toward the Internet router.

```
Destination
0.0.0.0/0

Next Hop
200.100.100.1

Interface
ethernet1/1
```

Without this route the firewall would not know where to forward Internet traffic.

---

# Configuring DHCP Relay

The clients and DHCP server are located on different networks.

Because DHCP uses broadcasts, the requests would normally never reach the DHCP server.

DHCP Relay solves this problem.

A relay agent was configured on the Inside interface.

```
Interface

ethernet1/2

DHCP Server

10.250.40.2
```

When a client broadcasts a DHCP Discover packet, the Palo Alto forwards it directly to the DHCP server instead of dropping it.

---

# Configuring NAT

A Dynamic IP and Port NAT rule was created.

Original Packet

```
Source Zone
Inside

Destination Zone
Outside
```

Translated Packet

```
Dynamic IP and Port

Interface Address

ethernet1/1
```

Every client leaving the Inside network is translated to the firewall's outside IP address.

```
192.168.10.21

↓

200.100.100.10
```

---

# Security Policies

Only a few policies were required.

Inside → Outside was configured to allow Internet access.

Inside → DMZ allows clients to communicate with the DHCP server.

DMZ → Inside allows DHCP replies back to the clients.

---

# Packet Flow

When a client requests an IP address, the communication looks like this:

```
Client

↓

Broadcast DHCP Discover

↓

Palo Alto DHCP Relay

↓

DHCP Router

↓

DHCP Offer

↓

Palo Alto

↓

Client receives IP address
```

Internet traffic follows a different path.

```
Client

192.168.10.21

↓

Palo Alto

Source NAT

↓

200.100.100.10

↓

Internet Router

Source NAT

↓

10.0.137.x

↓

VMware NAT

↓

Internet
```

---

# Troubleshooting

One of the biggest problems during this lab was that client traffic reached the Palo Alto but never reached the Internet.

The traffic logs constantly showed:

```
Session End Reason

aged-out
```

Using the Palo Alto session table revealed that NAT was working correctly.

```
show session all filter destination 8.8.8.8
```

The output showed that traffic was being translated from:

```
192.168.10.21

↓

200.100.100.10
```

This confirmed that the firewall was not the problem.

The next step was checking the Cisco router.

```
show ip nat statistics
```

The output showed:

```
Hits: 0

Translations: 0
```

This immediately indicated that the router was not performing NAT.

The problem turned out to be the NAT ACL.

I originally permitted the client subnet:

```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
```

However, by the time traffic reached the router, the Palo Alto had already translated every client into 200.100.100.10.

Changing the ACL to:

```cisco
access-list 1 permit 200.100.100.0 0.0.0.255
```

fixed the issue immediately.

---

# Conclusion

This lab demonstrated several enterprise networking concepts in a single topology. I configured DHCP Relay to allow clients on one subnet to receive addresses from a DHCP server on another subnet, implemented source NAT on both the Palo Alto firewall and the Cisco Internet router, configured static routing between devices, and verified the complete end-to-end packet flow from the client to the Internet.

The most valuable lesson from this lab was understanding how packets change as they move through multiple devices. By following the packet step by step and checking routing tables, session tables and NAT translations, it became much easier to identify where the communication was failing instead of assuming the firewall was always the problem.
