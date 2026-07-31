# Cisco AAA Authentication with Ubuntu FreeRADIUS

## Project Goal

The goal of this project was to stop using local usernames on Cisco devices and instead authenticate administrators through a centralized RADIUS server.

Instead of storing a username and password on every router and switch, every login request is sent to the Ubuntu server. The server checks whether the credentials are correct and replies with either an Access-Accept or an Access-Reject message.

The authentication flow now looks like this:

SSH Client
↓
Cisco Router / Switch
↓
Ubuntu FreeRADIUS Server
↓
Access-Accept / Access-Reject

This is how administrator authentication is commonly handled in enterprise environments because all devices rely on a single authentication server.

---

## Lab Topology

Internet

↓

Cloud0 (VMware NAT)

↓

Cisco Router (DHCP Server)

↓

Palo Alto Firewall

↓

Cisco Multilayer Switch

↓

Access Switches

↓

Ubuntu Server (FreeRADIUS)

The Ubuntu server was connected to VLAN 20 (Servers VLAN).

Its static IP address is:

```
192.168.20.21
```

---

# Why Use a Static IP?

Servers should always use static IP addresses.

If the RADIUS server received its address through DHCP, its IP could change after a reboot. Every Cisco device would still try to contact the old address and authentication would stop working.

The Ubuntu server was configured with:

```
IP Address : 192.168.20.21

Subnet Mask: 255.255.255.0

Gateway    : 192.168.20.1

DNS        : 8.8.8.8
```

The default gateway is the Palo Alto subinterface for VLAN 20.

---

# Installing FreeRADIUS

After Ubuntu networking was configured, FreeRADIUS was installed.

```
sudo apt update

sudo apt install freeradius freeradius-utils
```

To verify the installation:

```
freeradius -v
```

To verify that the service is running:

```
sudo systemctl status freeradius
```

---

# Understanding clients.conf

One of the first things I learned is that a RADIUS server does not accept authentication requests from every device.

Every Cisco device that wants to use RADIUS must first be registered inside the file:

```
/etc/freeradius/3.0/clients.conf
```

A client entry looks like this:

```
client internet-router {

    ipaddr = 200.100.100.1

    secret = cisco

    shortname = InternetRouter

}
```

The router sends authentication requests from the interface connected to the Palo Alto.

```
200.100.100.1
```

is therefore the address that FreeRADIUS trusts.

The secret is not the administrator password.

It is simply a shared password between the Cisco device and the RADIUS server.

Both sides must use exactly the same secret or authentication will fail.

In my lab the shared secret is:

```
cisco
```

---

# Creating Users

The actual administrator accounts are stored inside:

```
/etc/freeradius/3.0/users
```

I added:

```
admin Cleartext-Password := "cisco"
```

This means:

Username

```
admin
```

Password

```
cisco
```

When the Cisco router receives an SSH login request, it forwards these credentials to the RADIUS server. FreeRADIUS compares them against the users file and decides whether to allow or deny the login.

---

# Debug Mode

Before configuring Cisco devices I always stopped the service and started FreeRADIUS in debug mode.

```
sudo systemctl stop freeradius

sudo freeradius -X
```

Debug mode prints every authentication request to the terminal.

This makes troubleshooting much easier because it immediately shows whether the request reached the server and why it was accepted or rejected.

When everything loads correctly, the last line should be:

```
Ready to process requests
```

---

# Cisco AAA

Cisco devices normally authenticate users from their own local database.

Enabling AAA changes this behaviour.

```
aaa new-model
```

This command enables the AAA framework.

Without it, every AAA command is ignored.

AAA stands for:

Authentication

Authorization

Accounting

Authentication verifies who the user is.

Authorization decides what the user is allowed to do after logging in.

Accounting records what the user did.

In this lab only Authentication and Authorization were configured.

---

# Configuring the RADIUS Server

The router must know where the RADIUS server is located.

```
radius server FREERADIUS

 address ipv4 192.168.20.21 auth-port 1812 acct-port 1813

 key cisco
```

The IP address points to the Ubuntu server.

Port 1812 is used for authentication.

Port 1813 is used for accounting.

The key must exactly match the secret configured inside clients.conf.

---

# Source Interface

The router has multiple interfaces.

Without specifying one, Cisco may choose a different source address depending on routing.

```
ip radius source-interface GigabitEthernet0/1
```

This forces every RADIUS packet to originate from:

```
200.100.100.1
```

Since that address is registered inside clients.conf, the RADIUS server recognizes the router and processes the authentication request.

---

# Local Backup Account

Even though RADIUS is being used, a local administrator account should always exist.

```
username backupadmin privilege 15 secret Backup123!
```

If the RADIUS server becomes unavailable, the router can still be accessed using the local account.

Without a backup account it is possible to lock yourself out of the device completely.

---

# Authentication Method

```
aaa authentication login default group radius local
```

This tells Cisco what order to use for authentication.

First it asks the RADIUS server.

If the RADIUS server cannot be reached, it falls back to the local user database.

This is the most common enterprise configuration.

---

# Authorization

```
aaa authorization exec default group radius local
```

Authentication only verifies who the user is.

Authorization decides whether the user should receive an EXEC shell after logging in.

Without this command the user might successfully authenticate but still not receive a CLI prompt.

---

# Configuring SSH

To use SSH, RSA keys must first be generated.

```
crypto key generate rsa modulus 2048
```

The router also requires a domain name.

```
ip domain-name lab.local
```

Enable SSH version 2.

```
ip ssh version 2
```

Finally, configure the VTY lines.

```
line vty 0 4

transport input ssh

login authentication default
```

The important command is:

```
login authentication default
```

Instead of checking the local username database, the VTY lines now use the AAA method list called "default", which points to the RADIUS server.

---

# What is an SVI?

SVI stands for Switched Virtual Interface.

It is a virtual Layer 3 interface that belongs to a VLAN.

Creating a VLAN:

```
vlan 30
```

only creates the Layer 2 broadcast domain.

It does not give the switch an IP address.

Creating an SVI:

```
interface vlan 30

ip address 192.168.30.2 255.255.255.0

no shutdown
```

creates a virtual interface inside VLAN 30.

Now the switch itself has an IP address and can be managed remotely using SSH.

In my topology, the Palo Alto performs all routing between VLANs.

The SVI is only used to manage the switch, not to route user traffic.

---

# Testing

The first test is verifying connectivity.

The router should successfully ping:

```
192.168.20.21
```

After that, an SSH session is started to the router.

The username entered is:

```
admin
```

The password is:

```
cisco
```

If everything is configured correctly, the login succeeds and the Ubuntu terminal running:

```
sudo freeradius -X
```

shows an Access-Request followed by an Access-Accept.

This confirms that the router authenticated through the RADIUS server rather than using a locally configured user.

---

## What I Learned

This project helped me understand the difference between local authentication and centralized authentication. I also learned how AAA works, why Cisco devices need to be registered as RADIUS clients, why the shared secret is different from the administrator password, and why specifying the RADIUS source interface is important on devices with multiple interfaces. I also gained a much better understanding of SVIs and how they are used for switch management while the Palo Alto handles routing between VLANs.
