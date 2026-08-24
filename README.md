# Raspberry Pi Network Monitoring Server

A Raspberry Pi-based monitoring server for my [Active Directory Home Lab](https://github.com/MustafaHas9/Active-Directory-Home-Lab). The Raspberry Pi runs Debian 13 with Uptime Kuma deployed through Docker and monitors hosts and services across my physical network and isolated VirtualBox environment.

## Overview

This project uses a physical Raspberry Pi as a dedicated monitoring server for my virtualized Windows Server environment.

MONITOR01 is connected to my home network, while the Windows Server infrastructure runs on an isolated `192.168.10.0/24` VirtualBox network. Traffic from the Pi is routed through RRAS01 to reach the virtual servers.

Uptime Kuma monitors:

- Internet connectivity
- DNS resolution
- RRAS01 availability
- DC01 availability
- DNS on DC01
- LDAP on DC01
- FS01 availability
- SMB on FS01

## Network Architecture

![Network Architecture](Images/Pi%20Network%20Diagram.png)

| Device | Role | IP Address |
|---|---|---|
| **MONITOR01** | Raspberry Pi / Uptime Kuma | `192.168.6.73` |
| **RRAS01** | Routing and NAT | `192.168.10.1` |
| **RRAS01 Bridged NIC** | Physical network connection | `192.168.6.75` |
| **DC01** | Active Directory / DNS / DHCP | `192.168.10.10` |
| **FS01** | SMB File Server | `192.168.10.20` |
| **CLIENT01** | Domain-joined Windows client | DHCP |

## MONITOR01 Setup

The Raspberry Pi was configured as `MONITOR01` and runs Debian 13 (Trixie) on ARM64.

System information was verified with:

```bash
hostname
hostname -I
uname -m
cat /etc/os-release
```

![MONITOR01 System Information](Images/Pi%20Info.png)

MONITOR01 connects to the physical network over `wlan0` and receives its IP address through DHCP.

Network configuration and connectivity were verified using:

```bash
ip addr
ip route
ping -c 4 8.8.8.8
ping -c 4 google.com
```

![MONITOR01 Network Connectivity](Images/Pi%20IP.png)

The system is administered remotely over SSH:

```bash
ssh mhasnain@192.168.6.73
```

## Docker and Uptime Kuma

Docker was installed on MONITOR01 to run Uptime Kuma in a container.

The Docker installation and service were verified with:

```bash
docker --version
systemctl is-active docker
docker ps
```

Uptime Kuma was deployed as a Docker container and exposed on TCP port `3001`.

The running container can be verified with:

```bash
docker ps
```

![Uptime Kuma Docker Container](Images/Uptime%20Kuma%20Docker.png)

The Uptime Kuma dashboard is accessible from systems on the local network at:

```text
http://192.168.6.73:3001
```

## Routing to the Virtual Network

MONITOR01 is connected to the physical `192.168.4.0/22` network, while the Windows Server environment runs on an isolated `192.168.10.0/24` VirtualBox network.

RRAS01 connects the two networks using:

- An internal adapter on `192.168.10.0/24`
- A bridged adapter connected to the physical network

The bridged interface received `192.168.6.75`, allowing RRAS01 to act as the next hop from MONITOR01 to the virtual network.

A route was added on MONITOR01:

```bash
sudo ip route add 192.168.10.0/24 via 192.168.6.75
```

The resulting path is:

```text
MONITOR01
192.168.6.73
      |
      v
RRAS01 Bridged NIC
192.168.6.75
      |
      | RRAS
      v
192.168.10.0/24
      |
      +---- DC01  192.168.10.10
      |
      +---- FS01  192.168.10.20
```

The routing table shows traffic for `192.168.10.0/24` being forwarded through RRAS01:

```text
192.168.10.0/24 via 192.168.6.75 dev wlan0
```

Connectivity to the isolated network was verified from MONITOR01:

```bash
ping 192.168.10.10
```

![DC01 Routing Test](Images/Ping%20DC01.png)

> **Note:** The RRAS01 bridged interface receives its address through DHCP. Because I do not control the upstream router to create a DHCP reservation, the static route uses the address assigned to RRAS01 in the current lab environment.

## Infrastructure Monitoring

Uptime Kuma was configured with both host-level and service-level monitors.

| Monitor | Check | Target |
|---|---|---|
| Internet Connection | Ping | `8.8.8.8` |
| DNS Resolution | DNS | DNS query |
| Router (RRAS01) | Ping | `192.168.6.75` |
| Domain Controller (DC01) | Ping | `192.168.10.10` |
| DC01 - DNS | TCP | `192.168.10.10:53` |
| DC01 - LDAP | TCP | `192.168.10.10:389` |
| File Server (FS01) | Ping | `192.168.10.20` |
| FS01 - SMB | TCP | `192.168.10.20:445` |

Ping checks verify that each host is reachable, while TCP checks verify that individual services are accepting connections.

For example, DC01 could remain reachable over ICMP while a failed TCP 389 check would indicate that LDAP is unavailable.

![Infrastructure Monitoring](Images/LDAP%20info.png)

## Outage Testing

FS01 was intentionally shut down to test outage detection.

Uptime Kuma detected the loss of connectivity to `192.168.10.20`, marked the server as **Down**, and recorded 100% packet loss during the failed check.

![FS01 Outage Detected](Images/FS01%20Down.png)

FS01 was then restarted. Once connectivity was restored, Uptime Kuma detected the recovery and returned the monitor to **Up**.

![FS01 Recovery](Images/FS01%20Up.png)

This test verified that MONITOR01 could detect both an outage and recovery for a server located on the routed VirtualBox network.

## Technologies

`Raspberry Pi` `Debian 13` `Linux` `Docker` `Uptime Kuma` `SSH` `TCP/IP` `ICMP` `DNS` `LDAP` `SMB` `Windows Server 2025` `RRAS` `VirtualBox`

## Misc

The Windows Server environment monitored in this project was built as part of my Active Directory homelab:

[Active Directory Home Lab](https://github.com/MustafaHas9/Active-Directory-Home-Lab)

I used Ai to upscale some of the low resolution screenshots. As a result, some images may have leftover artifacts.
