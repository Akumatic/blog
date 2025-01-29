---
title: "Homelab"
slug: "homelab"
menu:
    main:
        weight: 10
        params: 
            icon: server
readingTime: false
---

## Introduction

My Homelab - an environment to provide the local network, host services and, above all, to learn. 

It all started many years ago with a single Raspberry Pi 1 Model B+, although that was more of a gimmick for me at the time. I did set up one or two services and made my first attempts with Linux, but it was never anything permanent. It wasn't until I was studying computer science that I got more involved with Linux and started using it actively.

When some friends wanted to start a new Minecraft server in 2021, I decided to take matters into my own hands. Since power consumption was a factor and I wanted a device with a small footprint that still delivered enough performance, I ordered an Intel NUC with an Intel i5-8259U.

Since then, I've been hosting services and have continued to expand and upgrade the environment - and there's still a lot to learn, try out and implement. 

## Current State

### Rack

The core of the Homelab is located in a network rack:

![Current rack configuration](img/rack.jpg)

From top to bottom & left to right:

#### Router

The router is **OPNsense** on a **ThinkCentre Tiny M720q**, equipped with an Intel Core i3-8100T, 8 GiB DDR4 and an Intel i340-T4 network card.

In addition to several networks including VLANs and the usual services such as DNS and DHCP, other useful services such as VPN servers and DynDNS updaters are also provided via this.

#### Patchpanels & Core Switch

A **MikroTik CSS326-24G-2S+RM** with SwOS serves as the core switch. 

The patch panels are equipped with RJ45 Keystone modules on the left-hand side and connected with colored patch cables:

| Color | Meaning |
| -- | -- |
| Red | Upink (WAN) |
| Orange | Uplink (Router) |
| Blue (Links) | Access Points |
| Green | Server |
| Blue (Right) | Clients |

A handful of HDMI keystone modules are inserted on the right-hand side, providing easy access to the HDMI slots of the devices installed in the rack.

#### NAS 1

The main NAS is an **UGREEN NASync DXP4800**, equipped with an Intel N100 and 16 GiB DDR5, running **TrueNAS Scale**.

Two pools are available for data: One MIRROR consisting of 2 hard disks of 16 TiB each and one MIRROR consisting of 2 hard disks of 6 TiB each.

#### NAS 2

This NAS serves as a backup system for the main NAS and is a **UGREEN NASync DXP2800**. 

Like the main NAS, it is equipped with an N100 and is operated with TrueNAS Scale, although it only has 8 GiB DDR5 and the pools do not consist of a MIRROR, but only of individual disks.

#### Server

A **Geekom A5** with an AMD Ryzen 7 5825U and 64 GiB DDR4 is used as the host for most of the services that I host locally. The computer itself runs **Proxmox Virtual Environment** as a hypervisor, the services are run in VMs or LXC.

#### Basic PDU

A simple power strip that is screwed into the rack. In addition to the devices in the rack, PoE injectors for the access points are also connected and can be found behind the PDU.

### External Components

#### Access Points

Two access points from **Ubiquiti** are used to provide Wi-Fi: An **UAP-AC-Pro** and an **UAP-AC-Lite**.
