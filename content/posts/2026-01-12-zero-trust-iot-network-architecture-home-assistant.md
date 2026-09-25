---
title: "Zero-Trust IoT Network Architecture for Home Assistant"
date: 2026-01-12
slug: zero-trust-iot-network-architecture-home-assistant
summary: "A practical zero-trust network architecture for Home Assistant and IoT using VLAN isolation, centralized DNS, controlled discovery, and explicit trust boundaries."
categories: [Infrastructure]
tags: [iot]
jumbotron:
  meta: true
---

## Overview

Smart-home devices are useful, but they should not automatically be trusted simply because they are inside the home network.

Many IoT devices have limited security controls, depend on vendor cloud services, expose local APIs, or use proprietary discovery mechanisms. Putting them on the same network as workstations and infrastructure makes those devices part of the trusted network by default.

Instead, I built my smart-home network around a simple principle:

> **IoT devices are useful, but they are not trusted.**

The network uses VLAN isolation, centralized DNS, Home Assistant as the primary controller, and explicit communication paths between security zones.

This is a production homelab design that evolved through actually running Matter, HomeKit, TP-Link Kasa, Sonos, ESP-based devices, and other smart-home equipment.

---

## Network Segmentation

The network is divided into separate security zones:

| VLAN            | Purpose                                                    |
| --------------- | ---------------------------------------------------------- |
| Management      | Proxmox, infrastructure administration, SSH                |
| Trusted / Admin | Personal computers and trusted clients                     |
| IoT             | Matter, HomeKit, Kasa, Sonos, ESPHome, Tasmota, WLED, etc. |
| Infrastructure  | Pi-hole and recursive DNS                                  |
| Home Assistant  | Dedicated HA host or subnet                                |

The important part is not the number of VLANs. It is that **there is no implicit trust between them**.

Inter-VLAN routing is controlled by the firewall, and the default policy is deny.

That means adding a device to the IoT VLAN does not automatically give it access to:

* Proxmox
* NAS services
* administrative interfaces
* workstations
* other internal VLANs

---

## The Trust Model

Home Assistant is treated differently from ordinary IoT devices.

It is the trusted controller for the smart-home network:

```text
Trusted / Admin
       │
       │ administration
       ▼
Infrastructure ───── Home Assistant
       ▲                    │
       │ DNS                │ control
       │                    ▼
       └────────────────── IoT
```

The IoT network is deliberately restricted.

An IoT device can reach the services it requires, but it does not receive general access to the rest of the network.

Home Assistant, on the other hand, is allowed to initiate the connections required to control those devices.

This creates an intentionally asymmetric relationship:

```text
Home Assistant ────────> IoT
       control

IoT ─────────────X─────> Trusted networks
```

If a smart plug or light bulb is compromised, the firewall should prevent that device from using its network position as a straightforward path toward the rest of the infrastructure.

---

## Pi-hole as the DNS Authority

DNS is another important part of the design.

All clients use Pi-hole as their DNS server:

```text
IoT device
    │
    │ DNS
    ▼
  Pi-hole
    │
    ▼
Recursive resolver
    │
    ▼
Internet
```

Pi-hole therefore becomes both the DNS service and an important network policy boundary.

The firewall permits clients to query Pi-hole while blocking direct DNS access to the Internet.

Conceptually:

```text
Client → Pi-hole       TCP/UDP 53    ALLOW
Client → Internet DNS  TCP/UDP 53    DENY
```

The recursive resolver is the component that performs external DNS resolution.

I also block DNS-over-TLS on port 853 from the IoT network.

DNS-over-HTTPS is more difficult to block comprehensively because it uses normal HTTPS traffic. Blocking a handful of known DoH endpoints can help, but it should not be treated as a complete solution.

The objective is therefore not to claim that encrypted DNS can never be used. It is to make the normal and expected DNS path explicit and enforceable.

---

## Discovery Without Opening the Network

VLAN isolation introduces a problem with smart-home protocols.

Matter, HomeKit, ESP-based devices, and other systems frequently rely on multicast DNS.

Simply blocking all multicast breaks discovery.

The solution is to treat discovery as a specific capability rather than opening the entire network.

mDNS uses:

```text
UDP 5353
```

I allow it only between the networks that actually require it, primarily Home Assistant and the IoT VLAN.

Conceptually:

```text
Home Assistant ←→ IoT
             UDP 5353
```

Other VLANs do not receive general mDNS access.

This is an important distinction:

> **Allowing discovery does not mean allowing general communication.**

A device can advertise itself through mDNS without receiving permission to communicate with every host on the network.

---

## Home Assistant Access

Home Assistant itself is not directly exposed to the Internet.

External access is provided through a Cloudflare Tunnel:

```text
Internet
   │
   ▼
Cloudflare
   │
   ▼
Tunnel
   │
   ▼
Reverse Proxy
   │
   ▼
Home Assistant
```

This keeps inbound Internet access separate from the IoT network.

Home Assistant still has the network access it needs to control local devices, but IoT devices are not given equivalent access back into the trusted network.

---

## Administrative Access

There is an operational problem with highly restricted IoT networks: eventually you need to configure something manually.

Devices such as Tasmota and WLED commonly expose local web interfaces. Permanently allowing the trusted network to reach every IoT device would make administration easier, but it also creates a permanent path into the least-trusted network.

Instead, administrative access is temporary.

For example:

```text
Admin → IoT TCP 80/443     DISABLED
Admin → IoT mDNS           DISABLED
```

When maintenance is required, I enable the appropriate rule, make the change, and disable it again.

This keeps the normal network state restrictive without making device administration impractical.

---

## Adding New Devices

The architecture also changes how I add IoT devices.

I don't start with a broad rule such as:

```text
Home Assistant → IoT → ALL
```

Instead, the process is:

1. Put the device into the IoT VLAN.
2. Keep the default-deny policy in place.
3. Configure the Home Assistant integration.
4. Watch firewall logs for blocked traffic.
5. Identify the required protocol and destination.
6. Create the smallest rule necessary.
7. Test the integration again.

This is particularly useful with vendor-specific devices because their actual network requirements are often different from what their documentation suggests.

---

## What the Architecture Provides

The design provides several useful security properties.

An IoT device does not automatically gain access to:

* management interfaces
* Proxmox
* NAS services
* trusted computers
* other internal VLANs
* arbitrary DNS servers

At the same time, Home Assistant can still perform its primary job: controlling the smart home.

There is an accepted trust relationship here. If Home Assistant itself is compromised, an attacker could potentially use its permitted access to control IoT devices.

That is a deliberate tradeoff.

The objective is not perfect isolation between Home Assistant and IoT. The objective is to prevent **IoT compromise from becoming general network compromise**.

---

## The Durable Part of the Design

Individual devices and protocols will change.

A future smart-home device may use different ports from today's devices. A vendor may change its discovery mechanism. A Home Assistant integration may introduce another dependency.

Those details belong in the firewall configuration.

The architectural boundaries should remain much more stable:

```text
Management
    │
    X
    │
Trusted ─── Infrastructure
                 │
                 │ DNS
                 ▼
            Home Assistant
                 │
                 │ controlled access
                 ▼
                IoT
```

The important rules are:

1. IoT lives in its own security zone.
2. Inter-VLAN traffic is denied by default.
3. DNS goes through the designated resolver.
4. Home Assistant gets the access required to control IoT.
5. Discovery is explicitly allowed where required.
6. Administrative access is temporary.
7. New access is added based on observed requirements rather than broad assumptions.

The result is a network where adding a smart-home device does not mean automatically trusting it.

The vendors and protocols will change. The trust boundaries do not have to.
