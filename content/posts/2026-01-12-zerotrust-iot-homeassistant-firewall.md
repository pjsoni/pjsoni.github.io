---
title: "Zero-Trust IoT & Home Assistant Firewall Architecture"
date: 2026-01-12
slug: zero-trust-iot-home-assistant-firewall-architecture
summary: "A practical, vendor-agnostic zero-trust network design for Home Assistant and IoT devices using VLAN isolation, centralized DNS, explicit firewall rules, and controlled mDNS and device access."
categories:
  - Infrastructure
tags:
  - iot
jumbotron:
  meta: true
---

## Overview

Smart-home devices are convenient, but they are also some of the least trusted devices on a home network. Many have limited security controls, communicate with vendor cloud services, use proprietary discovery protocols, or expose local APIs.

Rather than putting those devices on the same network as trusted computers and infrastructure, I built a segmented network around a simple principle:

> **IoT devices should be able to do only what they need to do, and nothing else.**

The design uses VLAN isolation, Pi-hole as the network's DNS authority, explicit inter-VLAN firewall rules, and Home Assistant as the primary controller for IoT devices.

The goal isn't to make every device perfectly secure. The goal is to limit what a compromised or poorly designed IoT device can reach.

## Network Layout

The network is divided into separate security zones:

| VLAN            | Purpose                                                    |
| --------------- | ---------------------------------------------------------- |
| Management      | Proxmox, infrastructure administration, SSH                |
| Trusted / Admin | Workstations and trusted client devices                    |
| IoT             | Matter, HomeKit, Kasa, Sonos, ESPHome, Tasmota, WLED, etc. |
| Infrastructure  | Pi-hole and recursive DNS                                  |
| Home Assistant  | Dedicated HA host/subnet                                   |

Inter-VLAN routing is disabled by default.

The important distinction is that VLAN membership is not considered a trust relationship. A device being on the same physical network does not give it access to another VLAN.

The firewall becomes the policy enforcement point.


## Trust Model

The network has three important trust levels:

```text
Trusted/Admin
      │
      │ administrative access
      ▼
Infrastructure ─────── Home Assistant
      ▲                       │
      │ DNS                   │ device control
      │                       ▼
      └──────────────────── IoT
```

### Trusted / Admin

These are machines I control directly and use for administration.

They can reach infrastructure services, but access to IoT devices is intentionally restricted.

### Infrastructure

Pi-hole and the recursive resolver live here.

IoT devices are allowed to reach Pi-hole for DNS, but they are not allowed to use arbitrary DNS servers.

### Home Assistant

Home Assistant is the IoT control plane.

It can initiate the connections required to discover and control devices on the IoT network.

### IoT

IoT devices are the least trusted systems.

They should not be able to initiate arbitrary connections to other internal networks.


## DNS as a Security Control

Every network client receives Pi-hole as its DNS server.

The basic flow is:

```text
IoT device
    │
    │ TCP/UDP 53
    ▼
  Pi-hole
    │
    ▼
Recursive resolver
    │
    ▼
Internet DNS
```

Pi-hole therefore becomes more than an ad blocker. It is also the network's DNS enforcement point.

### Firewall policy

Clients are allowed to send DNS queries only to Pi-hole:

```text
Client VLAN → Pi-hole      TCP/UDP 53     ALLOW
Client VLAN → Internet     TCP/UDP 53     DENY
IoT VLAN    → other DNS    TCP/UDP 53     DENY
```

The recursive resolver is the only system that is permitted to perform external DNS resolution.

I also block common encrypted DNS protocols from the IoT network where practical:

```text
IoT → Internet UDP 853      DENY
IoT → Internet TCP 853      DENY
IoT → known DoH endpoints   DENY
```

Blocking DoH comprehensively is more difficult than blocking traditional DNS because HTTPS is also used for normal web traffic. IP-based rules can help, but they are not a complete solution.

The important control is therefore:

**All conventional DNS must go through Pi-hole.**


## Home Assistant

Home Assistant is not exposed directly to the Internet.

External access is provided through a Cloudflare Tunnel:

```text
Internet
   │
   ▼
Cloudflare
   │
   │ encrypted tunnel
   ▼
Reverse Proxy
   │
   ▼
Home Assistant
```

The IoT network does not receive general access to Home Assistant.

Instead, Home Assistant initiates connections to IoT devices using explicitly permitted protocols.

This produces a useful trust boundary:

```text
IoT ──────X──────> Trusted/Admin
IoT ──────X──────> Management
IoT ──────X──────> Infrastructure services
IoT ──────> Pi-hole
IoT ──────> selected HA services
HA  ──────> IoT control
```


## mDNS Is the Exception

Strict VLAN isolation creates an immediate problem with smart-home discovery.

Protocols such as Matter and HomeKit depend heavily on multicast DNS.

Instead of allowing multicast traffic broadly, I use an mDNS reflector/repeater between the networks that actually need discovery.

The policy is:

```text
Home Assistant ↔ IoT
        UDP 5353
```

Other VLANs do not receive general mDNS access.

This is an important distinction:

> **Discovery is permitted; general network access is not.**

Allowing mDNS between VLANs does not mean allowing unrestricted IP communication between those VLANs.


## Firewall Rules

The exact syntax depends on the firewall platform, but the ordering is important.

My `LAN_IN` policy follows this model.

### 1. Established and related traffic

```text
ESTABLISHED / RELATED → ALLOW
```

This permits return traffic for connections that were already allowed.

### 2. IoT → Pi-hole DNS

```text
Source:      IoT
Destination: Pi-hole
Protocol:    TCP/UDP
Port:        53
Action:      ALLOW
```

### 3. Home Assistant → IoT HTTP/HTTPS

```text
Source:      Home Assistant
Destination: IoT
Protocol:    TCP
Ports:       80,443
Action:      ALLOW
```

Not every device needs both ports. In a production ruleset, I prefer separate rules when the device inventory allows it.

### 4. mDNS

```text
Source:      Home Assistant
Destination: IoT
Protocol:    UDP
Port:        5353
Action:      ALLOW
```

and, where discovery requires it:

```text
Source:      IoT
Destination: Home Assistant
Protocol:    UDP
Port:        5353
Action:      ALLOW
```

### 5. Matter and vendor-specific control

Only the protocols actually required by the installed devices are allowed.

For example:

```text
Home Assistant → IoT
TCP 5540
UDP 5540
```

Additional device-specific ports are added only when required.

### 6. Block external DNS

```text
IoT → Internet
TCP/UDP 53
DENY
```

### 7. Block DoT

```text
IoT → Internet
TCP/UDP 853
DENY
```

### 8. Default deny

Everything else between VLANs is denied.

The resulting model is:

```text
ALLOW established/related

ALLOW IoT → Pi-hole :53
ALLOW HA  → IoT      required control ports
ALLOW HA  ↔ IoT      UDP 5353 where required

DENY  IoT → Internet DNS
DENY  IoT → Internet DoT
DENY  IoT → other internal VLANs

DENY  everything else
```

Rule ordering matters. The explicit allows must occur before the broader deny rules.


## Device-Specific Access

One of the lessons from running this network is that a generic "IoT allow rule" quickly becomes too permissive.

Instead, I maintain a list of protocols actually required by the devices.

### Matter

Matter uses both IP and multicast discovery.

Typical local traffic includes:

| Purpose   | Protocol | Port |
| --------- | -------- | ---: |
| Matter    | TCP      | 5540 |
| Matter    | UDP      | 5540 |
| Discovery | UDP      | 5353 |

Matter deployments can have additional requirements depending on the controller and commissioning architecture, so I validate traffic captures rather than blindly copying a port list.

### TP-Link Kasa

For local Kasa devices:

| Purpose   | Protocol | Port |
| --------- | -------- | ---: |
| Control   | TCP      | 9999 |
| Discovery | UDP      | 9999 |

The rule is restricted to the Home Assistant → IoT direction where possible.

### Sonos

Sonos uses several local discovery and control mechanisms.

Common traffic includes:

| Purpose        | Protocol | Port |
| -------------- | -------- | ---: |
| Control        | TCP      | 1400 |
| Secure control | TCP      | 1443 |
| SSDP discovery | UDP      | 1900 |
| mDNS           | UDP      | 5353 |

Again, the exact requirements depend on the integration and device behavior.

### ESPHome / Tasmota / WLED

Common local access includes:

| Purpose     | Protocol | Port |
| ----------- | -------- | ---: |
| Web/API     | TCP      |   80 |
| ESPHome OTA | TCP      | 8266 |
| Discovery   | UDP      | 5353 |

I don't automatically expose these ports to the administration VLAN.

Instead, Home Assistant gets the normal operating access and temporary administrative access is enabled when I need to work directly on a device.



## Administrative Access Is Temporary

This is one of the more useful operational decisions in the design.

IoT devices frequently expose management interfaces such as:

```text
http://device-ip/
```

It would be easy to permanently allow:

```text
Admin VLAN → IoT VLAN → TCP 80/443
```

but there is little reason to keep that path open all the time.

Instead, the firewall contains administrative rules that are normally disabled:

```text
Admin → IoT TCP 80/443     DISABLED
Admin → IoT mDNS           DISABLED
```

When I need to configure a device:

1. Enable the required firewall rule.
2. Perform the maintenance.
3. Disable the rule.

This makes the normal network state more restrictive without making maintenance impossible.

For devices supported by Home Assistant, routine control and configuration can often be performed through the controller instead of exposing the device's management interface to the trusted network.

---

## LAN_LOCAL Rules

The firewall also controls traffic destined for the gateway itself.

For example, I restrict unnecessary gateway-to-VLAN communication and prevent clients from using the router as an unintended path to other network services.

One example is blocking unnecessary ICMP access between VLAN gateway addresses.

The exact rules depend on the firewall platform, but the principle is the same:

> Control traffic destined for the firewall separately from traffic being routed through the firewall.

---

## Troubleshooting the Firewall

A zero-trust ruleset is only useful if it is possible to determine why something was blocked.

When adding a new device, I don't start by opening an entire VLAN.

Instead, I:

1. Put the device in the IoT VLAN.
2. Start with the default-deny policy.
3. Configure Home Assistant.
4. Monitor firewall logs.
5. Identify the required destination, protocol, and port.
6. Add the smallest rule necessary.
7. Test the integration again.

For example, if a Kasa device needs local control:

```text
Home Assistant
       │
       │ TCP 9999
       ▼
 Kasa device
```

I add that specific path rather than:

```text
Home Assistant → entire IoT VLAN → ALL
```

This makes the resulting firewall easier to understand months later.

---

## What This Actually Protects Against

The segmentation provides useful containment.

An IoT device that becomes compromised does not automatically gain access to:

* Proxmox management
* SSH services
* trusted workstations
* infrastructure management interfaces
* arbitrary DNS servers
* other IoT devices

Its network access is limited to explicitly permitted destinations.

That does **not** make the device trustworthy or prevent all data exfiltration. An IoT device can still communicate through any destination and protocol that the firewall intentionally allows, particularly HTTPS.

The objective is therefore containment rather than perfect isolation.

---

## Accepted Trust

Home Assistant is intentionally given significant access to the IoT network.

If Home Assistant itself were compromised, an attacker could potentially use those permitted paths to control IoT devices.

That is an accepted part of this architecture.

The mitigation is to reduce the number of ways Home Assistant can itself be compromised:

* no direct Internet exposure
* controlled external access
* TLS
* limited administrative access
* segmented infrastructure
* explicit firewall rules

The trust model is therefore not:

> "Home Assistant is perfectly secure."

It is:

> "Home Assistant is sufficiently trusted to control IoT devices, while the IoT devices themselves are not trusted with the rest of the network."

---

## Practical Policy

The final policy is intentionally simple:

```text
                    ┌──────────────┐
                    │ Admin /      │
                    │ Trusted      │
                    └──────┬───────┘
                           │
                     restricted
                           │
                           ▼
┌──────────────┐    ┌──────────────┐
│ Infrastructure│◄───│ Home          │
│ Pi-hole/DNS   │    │ Assistant     │
└──────┬───────┘    └──────┬────────┘
       ▲                    │
       │ DNS                │ control
       │                    │
       └──────────┬─────────┘
                  ▼
             ┌─────────┐
             │   IoT   │
             └─────────┘
```

The IoT network gets:

* DNS through Pi-hole
* required Home Assistant control
* required discovery
* Internet access only where necessary
* no general access to trusted or management networks

Everything else is denied.

## Conclusion

The most important part of this design is not the individual port numbers. Those will change as devices and integrations change.

The durable part is the trust model:

1. **Put IoT devices in their own VLAN.**
2. **Deny inter-VLAN traffic by default.**
3. **Make Pi-hole the DNS enforcement point.**
4. **Allow Home Assistant only the control protocols it actually needs.**
5. **Treat mDNS as a narrowly scoped discovery mechanism rather than a reason to open the VLANs.**
6. **Keep administrative access disabled until it is needed.**
7. **Add device-specific firewall rules instead of broad IoT exceptions.**
8. **Use firewall logs and packet captures to validate requirements rather than guessing.**

The result is a network where adding a new smart-home device does not mean automatically trusting it.

The port lists will evolve. The vendors will change. The integrations will change.

The trust boundaries don't have to.
