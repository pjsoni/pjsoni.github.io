---

layout: post
slug: zerotrust-iot-homeassistant-firewall
title: "Zero-Trust IoT & Home Assistant Firewall Architecture"
date: 2026-01-12
categories: [networking, security, homelab]
tags: [firewall, iot, homeassistant, zerotrust, dns]
----------------------------------------------------

## Overview

This post documents a **vendor-agnostic, zero-trust firewall design** for a modern smart home and homelab. The goal is to securely operate IoT devices and Home Assistant (HA) while maintaining **strict VLAN isolation**, **single-source DNS**, and **explicit trust boundaries**.

This is not a theoretical design — it is a **battle-tested, production homelab configuration** built iteratively after real-world issues with Matter, HomeKit, TP-Link Kasa, Sonos, ESP-based devices, and mDNS discovery.

---

## Who This Is For (and Not For)

### This post **is for you** if:

* You run **Home Assistant** (bare metal, VM, or appliance)
* You want **strong IoT isolation** without breaking Matter or HomeKit
* You use **Pi-hole** (or similar) and want DNS to be a control plane
* You prefer **explicit firewall rules** over flat trusted networks
* You want a design that survives vendor changes

### This post **is not for you** if:

* All devices live on a single flat LAN
* You rely entirely on vendor cloud apps
* You are not using VLANs or inter-VLAN firewalls
* You want a "plug-and-play" consumer router setup

---

## Design Principles

1. **Default Deny Everywhere** – No inter-VLAN communication unless explicitly allowed
2. **Single Controller Model** – Home Assistant is the only IoT control plane
3. **Single DNS Authority** – One recursive DNS resolver for the entire network
4. **Discovery Without Trust** – mDNS allowed only where required
5. **Maintenance Is Temporary** – Admin access is disabled by default

---

## Network Segmentation

| VLAN            | Purpose                                                 |
| --------------- | ------------------------------------------------------- |
| IoT VLAN        | Smart devices (Matter, HomeKit, ESP, Kasa, Sonos, etc.) |
| Trusted / Admin | Admin workstation                                       |
| Infrastructure  | Pi-hole + recursive DNS resolver                        |
| Home Assistant  | Dedicated HA subnet or host                             |

Each VLAN is isolated at Layer 3. **No implicit routing** exists between them.

---

-|--------|
| Management (VLAN 90) | Proxmox UI, SSH, infrastructure services |
| IoT VLAN | Smart devices (Matter, HomeKit, ESP, Kasa, Sonos, etc.) |
| Trusted / Admin | Admin workstation |
| Infrastructure | Pi-hole + recursive DNS resolver |

Each VLAN is isolated at Layer 3. **No implicit routing** exists between them.

---

## DNS Architecture (Single Source of Truth)

All devices use **Pi-hole** as their *only* DNS server.

Pi-hole runs a **local recursive resolver** on the same host, making it:

* The only system allowed to query root servers
* The only system allowed outbound DNS
* The enforcement point for DNS-based policy

### Firewall Enforcement

* All DNS (TCP/UDP 53) is blocked except:

  * Clients → Pi-hole
  * Pi-hole → Internet (root servers)
* DoH and DoT from IoT networks are explicitly blocked

This guarantees:

* No DNS bypass
* No vendor hardcoded resolvers
* No split-brain resolution

---

## Home Assistant Trust Model

Home Assistant is treated as a **high-trust internal controller**:

* Not directly exposed to the Internet
* Accessed externally only via **Cloudflare Tunnel**
* End-to-end TLS from client → Cloudflare → tunnel → reverse proxy → HA

IoT devices **cannot initiate arbitrary connections** back to HA.

---

## IoT Control & Discovery

### mDNS

mDNS is required for:

* Matter
* HomeKit
* ESP-based devices
* Vendor discovery protocols

Instead of flooding all VLANs:

* HA → IoT mDNS is explicitly allowed
* IoT → HA mDNS is explicitly allowed
* No other multicast is permitted

### TCP / UDP Control

IoT devices are controlled through **explicit port allowlists**:

* Matter
* TP-Link Kasa
* Sonos
* Wiz bulbs
* ESPHome / Tasmota / WLED

Ports are grouped by **TCP vs UDP** and allowed **only from HA**.

---

## Firewall Ruleset

### LAN_IN Rules (Order Matters)

1. Allow Established and Related
2. Allow IoT → Pi-hole DNS
3. Allow HA → IoT HTTP/S (TCP 80, 443)
4. Allow HA → IoT mDNS (UDP 5353)
5. Allow IoT → HA mDNS (UDP 5353)
6. Allow HA → IoT TCP (Matter / vendor-specific ports)
7. Allow HA → IoT UDP (Matter / vendor-specific ports)
8. Block IoT → Public DNS
9. Block IoT → DoH / DoT (Work in Progress)
10. Block All External DNS (except Pi-hole)
11. Block All Inter-VLAN Traffic (Default Deny)

### LAN_LOCAL Rules

* Block gateway ICMP to other VLAN gateways

---

## Maintenance Access Model

Direct device management (e.g. Tasmota, WLED web UI) is **intentionally disabled by default**.

Two rules exist but remain disabled:

* Admin PC → IoT subnet (HTTP)
* Admin PC → IoT subnet (mDNS)

These rules are enabled **only during maintenance windows** and disabled immediately after.

Most firmware updates and configuration changes are performed via Home Assistant.

---

## Security Analysis

### What Is Allowed

* HA can fully control IoT devices
* IoT devices can only talk to Pi-hole and HA (selectively)
* Admin access to IoT is disabled by default

### What Is Prevented

* IoT lateral movement
* DNS exfiltration
* Vendor cloud bypass
* Accidental trust expansion
* Network discovery across VLANs

### Accepted Risk

If Home Assistant is compromised, IoT control is possible.

This is a **conscious trust decision**, mitigated by:

* No direct exposure
* Cloudflare Tunnel
* TLS everywhere
* Minimal inbound paths

---

## Matter, HomeKit, and Vendor Port Appendix

> **Note:** Ports listed here are based on observed behavior and specifications. Always validate in your environment.

### Common Protocols (All Vendors)

| Protocol | Transport | Ports | Notes                                         |
| -------- | --------- | ----- | --------------------------------------------- |
| mDNS     | UDP       | 5353  | Required for discovery (Matter, HomeKit, ESP) |
| HTTP     | TCP       | 80    | Local device APIs, some Matter/Wiz devices    |
| HTTPS    | TCP       | 443   | Secure local APIs                             |

---

### Matter (CSA Specification)

| Purpose        | Transport | Ports |
| -------------- | --------- | ----- |
| Matter Control | TCP       | 5540  |
| Matter UDP     | UDP       | 5540  |
| Discovery      | UDP       | 5353  |

---

### Apple HomeKit

| Purpose           | Transport | Ports                 |
| ----------------- | --------- | --------------------- |
| Pairing & Control | TCP       | 80, 443               |
| Accessory Control | TCP       | 51826–51850 (dynamic) |
| Discovery         | UDP       | 5353                  |

---

### TP-Link Kasa

| Purpose       | Transport | Ports |
| ------------- | --------- | ----- |
| Local Control | TCP       | 9999  |
| Discovery     | UDP       | 9999  |

---

### Sonos

| Purpose   | Transport | Ports      |
| --------- | --------- | ---------- |
| Control   | TCP       | 1400, 1443 |
| Discovery | UDP       | 1900, 5353 |

---

### ESPHome / Tasmota / WLED

| Purpose   | Transport | Ports    |
| --------- | --------- | -------- |
| API / OTA | TCP       | 80, 8266 |
| Discovery | UDP       | 5353     |

---

## Conclusion

This architecture delivers:

* Strong VLAN isolation
* Predictable behavior
* Vendor-agnostic IoT support
* Operational sanity

It is **secure by default**, flexible when needed, and fully auditable.

This is how zero-trust networking principles can be **practically applied** to a real smart home.

---

*If you’re building something similar, adapt the port lists — not the trust model.*
