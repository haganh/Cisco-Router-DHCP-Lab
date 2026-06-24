#  Router DHCP Lab

## Overview

In this lab, I configured a small multi-router network in Cisco Packet Tracer consisting of three routers — **R1**, **R2**, and **ISP** — with two end-user PCs (PC-A and PC-B) connected through switches to R1.

The goal was to build a fully functional network where the PCs automatically receive IP addresses via DHCP, routed through multiple hops across the topology.

Here's a breakdown of what was accomplished:

- **Base hardening** – Applied consistent security settings across all routers including encrypted passwords, MOTD banners, and console/VTY login requirements.
- **IP addressing** – Assigned IPv4 addresses to all router interfaces (Gigabit Ethernet and Serial) according to the topology, and set clock rates on DCE serial links.
- **Dynamic routing (RIPv2)** – Configured RIPv2 on R1 and R2 so internal networks could discover each other automatically, with `no auto-summary` to prevent classful route summarization issues.
- **Default & static routing** – Added a default static route on R2 pointing to the ISP, redistributed it via `default-information originate`, and configured a summary static route on ISP to reach all internal subnets (`192.168.0.0/22`).
- **DHCPv4 server on R2** – Set up two DHCP pools on R2 (one per LAN subnet on R1), excluded the first 9 addresses in each pool for static assignments, and configured DNS and domain name options.
- **DHCP relay on R1** – Since R2 is the DHCP server but the PCs sit behind R1, I configured `ip helper-address` on both of R1's LAN interfaces to forward DHCP broadcasts across the routed link to R2.
- **End-to-end verification** – Confirmed full connectivity with pings between all routers and verified DHCP lease assignments on the PCs using `ipconfig /all`.

---

## Topology

<img width="700" height="683" alt="Screenshot 2026-06-23 172414" src="https://github.com/user-attachments/assets/a717fdeb-f251-4779-974a-eb54f6bb8e45" />



| Device | Interface | IP Address | Subnet Mask |
|--------|-----------|------------|-------------|
| R1 | G0/0 | 192.168.0.1 | 255.255.255.0 |
| R1 | G0/1 | 192.168.1.1 | 255.255.255.0 |
| R1 | S0/0/0 (DCE) | 192.168.2.253 | 255.255.255.252 |
| R2 | S0/0/0 | 192.168.2.254 | 255.255.255.252 |
| R2 | S0/0/1 (DCE) | 209.165.200.226 | 255.255.255.224 |
| ISP | S0/0/1 | 209.165.200.225 | 255.255.255.224 |

---

## Part 1 – Basic Router Configuration

Apply the following base configuration to **each router**:

```
no ip domain-lookup
service password-encryption
enable secret class
banner motd #
Unauthorized access is strictly prohibited. #
line con 0
 password cyber
 login
 logging synchronous
line vty 0 4
 password cyber
 login
```

- Configure the **hostname** on each router as shown in the topology.
- Configure **IPv4 addresses** on all interfaces per the topology table.
- Set **clock rate 128000** on all DCE serial interfaces.
- Issue `no shutdown` on all interfaces.

---

## Part 2 – Routing Configuration

### R1 – RIPv2

```
R1(config)# router rip
R1(config-router)# version 2
R1(config-router)# network 192.168.0.0
R1(config-router)# network 192.168.1.0
R1(config-router)# network 192.168.2.252
R1(config-router)# no auto-summary
```

### R2 – RIPv2 + Default Static Route to ISP

```
R2(config)# router rip
R2(config-router)# version 2
R2(config-router)# network 192.168.2.252
R2(config-router)# default-information originate
R2(config-router)# exit
R2(config)# ip route 0.0.0.0 0.0.0.0 209.165.200.225
```

### ISP – Summary Static Route

```
ISP(config)# ip route 192.168.0.0 255.255.252.0 209.165.200.226
```

---

## Part 3 – Save Configurations

Copy the running configuration to the startup configuration on **all routers**:

```
Router# copy running-config startup-config
```

---

## Part 4 – Verify Connectivity

### From R1, ping:
- `192.168.2.254`
- `209.165.200.225`
- `209.165.200.226`

### From R2, ping:
- `192.168.0.1`
- `192.168.1.1`
- `192.168.2.253`

---

## Part 5 – DHCPv4 Server Configuration on R2

```
R2(config)# ip dhcp excluded-address 192.168.0.1 192.168.0.9
R2(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.9

R2(config)# ip dhcp pool R1G1
R2(dhcp-config)# network 192.168.1.0 255.255.255.0
R2(dhcp-config)# default-router 192.168.1.1
R2(dhcp-config)# dns-server 209.165.200.225
R2(dhcp-config)# domain-name cyber-lab.com
R2(dhcp-config)# exit

R2(config)# ip dhcp pool R1G0
R2(dhcp-config)# network 192.168.0.0 255.255.255.0
R2(dhcp-config)# default-router 192.168.0.1
R2(dhcp-config)# dns-server 209.165.200.225
R2(dhcp-config)# domain-name cyber-lab.com
```

---

## Part 6 – DHCP Relay Agent Configuration on R1

```
R1(config)# interface g0/0
R1(config-if)# ip helper-address 192.168.2.254
R1(config-if)# exit

R1(config)# interface g0/1
R1(config-if)# ip helper-address 192.168.2.254
```

---

## Part 7 – Configure PCs for DHCP

On each PC (PC-A and PC-B):

1. Click the **PC** -> **Desktop** tab -> **IP Configuration**
2. Select **Obtain an IP Address Automatically**

To verify, open the **Command Prompt** on each PC and run:

```
ipconfig /all
```

---

## Part 8 – Verify DHCP on R2

```
R2# show ip dhcp server statistics
R2# show ip dhcp pool
```
