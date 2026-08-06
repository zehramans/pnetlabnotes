# Centralized Syslog Server Lab

## Objective

The goal of this lab is to configure a centralized Syslog server using **rsyslog** on Ubuntu and configure Cisco networking devices to forward their logs to it.

Centralized logging is a fundamental component of enterprise networks because it allows administrators to collect logs from all infrastructure devices in one place instead of logging into each device individually.

---

# What is Syslog?

Syslog is a standard protocol used by network devices, servers, firewalls and applications to send log messages to a central logging server.

Instead of storing logs only locally, devices transmit them over the network to a Syslog server.

Example:

```text
Switch --------\
Router ---------\
Firewall --------> Syslog Server
Linux Server ---/
```

Without centralized logging:
-Every device stores its own logs
-Logs are lost after a reboot
-Difficult to investigate incidents

With centralized logging:

-One location for every log
-Easier troubleshooting
-Security monitoring
-Long-term log retention
-Required in many enterprise environments

---

# Why use Syslog?

Imagine a user reports that the network was unavailable for five minutes.

Without Syslog you would have to:

-SSH into every switch
-SSH into every router
-Check firewall logs
-Correlate timestamps manually

With Syslog:

Everything is already stored on one server.

---

# How Syslog Works

```text
             UDP 514
Cisco Switch --------\
                       \
Cisco Router -----------> Ubuntu Syslog Server
                       /
Palo Alto Firewall ----/
```

Each device generates events such as:

-Interface goes down
-User logs in
-Configuration changes
-Routing changes
-Authentication failures

These events are sent over UDP port **514** by default.

---

# UDP vs TCP

Syslog traditionally uses UDP 514

Modern implementations also support TCP 514

---

# Lab Topology

```text
                    Internet
                        |
                 Cisco Router
                        |
                 Palo Alto Firewall
                        |
                Multilayer Switch
                        |
          ----------------------------
          |            |            |
       VLAN10       VLAN20      VLAN30
        Users       Servers        IT
                        |
                Ubuntu Server
                (FreeRADIUS + Syslog)
```

Ubuntu IP:

```
192.168.20.21
```

Cisco MLS:

```
192.168.20.2
```

---

# Installing rsyslog

Most Ubuntu installations already include rsyslog.

Check its status:

```bash
sudo systemctl status rsyslog
```

Expected:

```text
Active: active (running)
```

If not installed:

```bash
sudo apt update
sudo apt install rsyslog -y
```

Enable it:

```bash
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

---

# Configuring rsyslog

Open:

```bash
sudo nano /etc/rsyslog.conf
```

Enable UDP:

```text
module(load="imudp")
input(type="imudp" port="514")
```

Enable TCP:

```text
module(load="imtcp")
input(type="imtcp" port="514")
```

Restart:

```bash
sudo systemctl restart rsyslog
```

---

# Understanding the Configuration

## module(load="imudp")

Loads the UDP input module.

Without this module the server cannot receive Syslog packets.

---

## input(type="imudp" port="514")

Creates a UDP listener.

Meaning type = UDP input port = 514

Now Ubuntu waits for incoming Syslog packets.

---

## module(load="imtcp")

Loads TCP support.

---

## input(type="imtcp" port="514")

Creates a TCP listener.

Useful if devices send logs over TCP.

---

# Verify the Listener

Check UDP:

```bash
sudo ss -lun
```

Expected:

```text
UNCONN
0.0.0.0:514
```

Meaning:

```
0.0.0.0
```

The server is listening on every interface.

```
514
```

Syslog port.

```
UNCONN
```

UDP is connectionless, so there is no established session.

---

# Cisco Configuration

Enter configuration mode:

```cisco
conf t
```

Enable timestamps:

```cisco
service timestamps log datetime msec
```

Explanation:

Adds date and milliseconds to every log.

Instead of:

```
LINK UP
```

You get:

```
Aug 6 11:41:55.997 LINK UP
```

Much easier to investigate incidents.

---

Configure the Syslog server:

```cisco
logging host 192.168.20.21
```

Meaning:

Send every Syslog message to the Ubuntu server.

---

Choose severity:

```cisco
logging trap informational
```

Cisco has eight severity levels:

| Level | Name | Description |
|--------|------|-------------|
| 0 | Emergencies | System unusable |
| 1 | Alerts | Immediate action required |
| 2 | Critical | Critical conditions |
| 3 | Errors | Errors |
| 4 | Warnings | Warning messages |
| 5 | Notifications | Normal but important |
| 6 | Informational | Informational events |
| 7 | Debugging | Debug messages |

Using:

```cisco
logging trap informational
```

means:

Send levels **0 through 6**.

Debug messages (7) are excluded.

---

Enable logging:

```cisco
logging on
```

Without this command the device will not transmit Syslog messages.

---

Save:

```cisco
end
write memory
```

---

# Verifying Cisco Configuration

Display logging status:

```cisco
show logging
```

Useful information includes:

```
Logging to 192.168.20.21
```

This confirms the configured destination.

```
Trap logging: informational
```

Shows the selected severity.

```
message lines logged
```

Indicates that messages have been generated and sent.

---

# Testing

Generate logs:

```cisco
conf t
interface g0/0
shutdown
no shutdown
```

or

```cisco
hostname TEST
```

Every configuration change generates Syslog messages.

---

# Watching Incoming Logs

View live logs:

```bash
sudo tail -f /var/log/syslog
```

Example:

```text
192.168.20.2 %LINK-5-CHANGED
192.168.20.2 %LINEPROTO-5-UPDOWN
192.168.20.2 %SYS-5-CONFIG_I
```

Explanation:

```
192.168.20.2
```

Device that generated the log.

```
LINK-5-CHANGED
```

Interface status changed.

```
CONFIG_I
```

Configuration modified.

---

# Troubleshooting

## Verify rsyslog

```bash
sudo systemctl status rsyslog
```

---

## Verify syntax

```bash
sudo rsyslogd -N1
```

Checks the configuration without restarting the service.

---

## Verify listeners

```bash
sudo ss -lun
```

Confirms UDP port 514 is open.

---

## Verify packets

```bash
sudo tcpdump -ni any udp port 514
```

Example:

```text
192.168.20.2 > 192.168.20.21 SYSLOG local7.notice
```

This confirms
Cisco transmitted the packet.
The packet reached Ubuntu.
The network path is working.

If packets appear in `tcpdump` but not in the log file, the issue is with rsyslog processing rather than network connectivity.

---

# Separating Cisco Logs

Instead of mixing Cisco logs with Linux logs, create:

```bash
sudo nano /etc/rsyslog.d/30-cisco.conf
```

Example:

```text
if $fromhost-ip == '192.168.20.2' then /var/log/cisco.log
& stop
```

Restart:

```bash
sudo systemctl restart rsyslog
```

View:

```bash
sudo tail -f /var/log/cisco.log
```

This stores Cisco logs in a dedicated file.

---

# Skills Learned
- What Syslog is
- Why centralized logging is important
- UDP vs TCP Syslog
- Configuring rsyslog
- Configuring Cisco Syslog
- Understanding Syslog severity levels
- Verifying listeners with `ss`
- Verifying network traffic with `tcpdump`
- Troubleshooting Syslog issues
- Separating logs into dedicated files

---

# Key Takeaways

This lab demonstrates how enterprise networks centralize logs for monitoring, troubleshooting, and incident response. Cisco devices forward events to an Ubuntu server running **rsyslog**, which listens on UDP/TCP port 514 and stores incoming messages. During troubleshooting, verifying the listener (`ss`), validating the configuration (`rsyslogd -N1`), capturing packets (`tcpdump`), and confirming logs in `/var/log/syslog` provides a structured workflow for isolating issues. Centralized logging is a foundational service that supports security monitoring, compliance, and day-to-day network operations.
