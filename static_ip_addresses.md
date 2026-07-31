# Assigning Static IP Addresses

## Why Use Static IP Addresses?

Servers should always use static IP addresses.

If a server uses DHCP, its IP address may change after a reboot or when the DHCP lease expires. Other devices, such as routers, firewalls, DNS servers, or RADIUS clients, would still try to communicate with the old IP address, causing services to fail.

Examples of devices that should normally have static IP addresses include:

- Windows Server
- Linux Servers
- RADIUS Servers
- DNS Servers
- Web Servers
- Firewalls
- Network Switches (Management Interfaces)

Client devices such as laptops and desktops normally use DHCP.

---

# Ubuntu Server

Ubuntu Server uses **Netplan** for network configuration.

First, find the network interface.

```bash
ip addr
```

Typical interface names include:

```
ens33
ens160
enp1s0
eth0
```

Open the Netplan configuration file.

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

(Some systems may use a different filename inside `/etc/netplan/`.)

Example configuration:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.20.21/24
      routes:
        - to: default
          via: 192.168.20.1
      nameservers:
        addresses:
          - 8.8.8.8
```

Apply the configuration.

```bash
sudo netplan apply
```

Verify the new address.

```bash
ip addr
```

Test connectivity.

```bash
ping 192.168.20.1
ping 8.8.8.8
```

---

# Windows Server

Open:

```
Server Manager
```

or

```
Control Panel
→ Network and Internet
→ Network and Sharing Center
→ Change adapter settings
```

Right-click the network adapter.

```
Properties
```

Select:

```
Internet Protocol Version 4 (TCP/IPv4)
```

Click:

```
Properties
```

Choose:

```
Use the following IP address
```

Example:

```
IP Address:
192.168.20.20

Subnet Mask:
255.255.255.0

Default Gateway:
192.168.20.1
```

Configure DNS.

```
Preferred DNS:
8.8.8.8
```

or another DNS server on your network.

Click **OK** to save the configuration.

Open Command Prompt and verify:

```cmd
ipconfig
```

Test connectivity.

```cmd
ping 192.168.20.1
ping 8.8.8.8
```

---

# Things to Remember

A static IP configuration always requires four pieces of information:

```
IP Address
Subnet Mask
Default Gateway
DNS Server
```

If any of these are incorrect, the device may lose network connectivity or Internet access.

For infrastructure devices such as servers, it is good practice to document the assigned IP addresses so they remain consistent throughout the network.
