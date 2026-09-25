---
title: "Implementing a Zero-Trust IoT Firewall for Home Assistant"
date: 2026-01-13
slug: implementing-zero-trust-iot-firewall-home-assistant
summary: "Practical firewall rules for isolating Home Assistant and IoT devices with centralized DNS, controlled mDNS, explicit device access, and temporary administration."
categories:
  - Infrastructure
tags:
  - iot
jumbotron:
  meta: true
---

## Overview

Once the network trust boundaries are defined, the next challenge is implementing them without breaking the smart home.

This is where IoT networks become interesting.

Matter, HomeKit, Kasa, Sonos, ESPHome, Tasmota, and WLED do not all use the same protocols. Some require multicast discovery, while others expose simple local TCP or UDP services.

The solution is not to create one large "Home Assistant can access IoT" rule.

Instead, I use a default-deny firewall and add access for the protocols actually required by the devices.

This post focuses on that implementation.

---

## Start With Default Deny

The basic policy is:

```text
ALLOW established/related

ALLOW required IoT → DNS
ALLOW required HA → IoT
ALLOW required discovery

DENY IoT → other internal networks
DENY unauthorized DNS
DENY everything else
```

The exact syntax depends on the firewall platform, but the ordering is important.

Established and related traffic should be allowed first, followed by the specific exceptions, and finally the broader deny rules.

---

## DNS Rules

Every IoT device receives Pi-hole as its DNS server.

The firewall permits:

```text
IoT → Pi-hole
TCP/UDP 53
ALLOW
```

but blocks:

```text
IoT → Internet
TCP/UDP 53
DENY
```

and:

```text
IoT → Internet
TCP/UDP 853
DENY
```

The second rule blocks DNS-over-TLS.

This prevents a device from simply ignoring the configured DNS server and sending traditional DNS queries directly to an external resolver.

DNS-over-HTTPS is different because it normally uses TCP 443.

Blocking all DoH without also blocking normal HTTPS would be impractical, so I treat DoH blocking as an additional control rather than assuming that it can be completely eliminated with a simple port rule.

---

## Home Assistant → IoT

Home Assistant gets explicit access to the IoT network.

The first group covers common local web interfaces:

```text
Source:      Home Assistant
Destination: IoT
Protocol:    TCP
Ports:       80,443
Action:      ALLOW
```

This is useful for devices that expose local HTTP/HTTPS APIs.

However, I don't assume every IoT device needs these ports. Where the device inventory is known, I prefer narrower rules.

The same principle applies to UDP.

Only protocols required by the installed integrations are allowed.

---

## mDNS

mDNS is:

```text
UDP 5353
```

and is used by several smart-home ecosystems.

Where required, the firewall permits:

```text
Home Assistant → IoT
UDP 5353
ALLOW
```

and:

```text
IoT → Home Assistant
UDP 5353
ALLOW
```

An mDNS reflector/repeater handles the multicast forwarding between VLANs.

The important part is that this is a discovery exception, not a general IoT-to-Home-Assistant rule.

---

## Matter

Matter commonly uses:

```text
TCP 5540
UDP 5540
UDP 5353
```

The corresponding Home Assistant access can therefore be restricted to:

```text
Home Assistant → IoT
TCP 5540
ALLOW
```

```text
Home Assistant → IoT
UDP 5540
ALLOW
```

with mDNS handled separately.

Matter commissioning and specific controller/device combinations can introduce additional traffic, so I validate the actual traffic in my environment rather than treating a port table as authoritative for every deployment.

---

## TP-Link Kasa

For local Kasa control, a common configuration is:

```text
Home Assistant → IoT
TCP 9999
ALLOW
```

and device discovery may use:

```text
UDP 9999
```

If discovery is required:

```text
Home Assistant → IoT
UDP 9999
ALLOW
```

I keep these rules scoped to the Kasa devices where the firewall platform makes that practical.

---

## Sonos

Sonos uses several local discovery and control mechanisms.

Common ports include:

| Purpose        | Protocol | Port |
| -------------- | -------- | ---: |
| Control        | TCP      | 1400 |
| Secure control | TCP      | 1443 |
| SSDP           | UDP      | 1900 |
| mDNS           | UDP      | 5353 |

Rather than opening these ports to the entire trusted network, I allow them from Home Assistant to the IoT devices that require them.

This keeps Sonos control inside the same trust model as the other IoT devices.

---

## ESPHome, Tasmota, and WLED

These devices commonly expose local web interfaces.

Typical ports include:

| Purpose     | Protocol | Port |
| ----------- | -------- | ---: |
| Web/API     | TCP      |   80 |
| ESPHome OTA | TCP      | 8266 |
| Discovery   | UDP      | 5353 |

For normal operation, Home Assistant receives the required access.

Direct administrative access from the trusted network is not permanently enabled.

---

## Temporary Administration

Sometimes I need to open a device's web interface directly.

Instead of maintaining a permanent rule:

```text
Trusted → IoT → TCP 80/443
```

I keep the rule disabled.

For example:

```text
Admin PC → IoT TCP 80/443
Admin PC → IoT UDP 5353
```

When maintenance is required:

```text
1. Enable rule
2. Configure device
3. Test
4. Disable rule
```

This is particularly useful for firmware updates and direct configuration of devices such as WLED or Tasmota.

The administrative path exists, but it is not part of the normal network state.

---

## Rule Ordering

A simplified `LAN_IN` ruleset looks like this:

| Order | Source   | Destination    | Protocol/Port       | Action |
| ----: | -------- | -------------- | ------------------- | ------ |
|     1 | Any      | Any            | Established/Related | Allow  |
|     2 | IoT      | Pi-hole        | TCP/UDP 53          | Allow  |
|     3 | HA       | IoT            | Required TCP        | Allow  |
|     4 | HA       | IoT            | Required UDP        | Allow  |
|     5 | HA ↔ IoT | —              | UDP 5353            | Allow  |
|     6 | IoT      | Internet       | TCP/UDP 53          | Deny   |
|     7 | IoT      | Internet       | TCP/UDP 853         | Deny   |
|     8 | IoT      | Internal VLANs | Any                 | Deny   |
|     9 | Any      | Any            | Any                 | Deny   |

The exact order and rule categories vary by firewall platform, but the principle is consistent: **specific exceptions must be evaluated before the broad deny policy.**

---

## LAN_LOCAL

Traffic destined for the firewall itself should be handled separately from routed traffic.

For example, I restrict unnecessary access to gateway services and prevent clients from using the router as an unintended path between security zones.

One example is restricting ICMP access between VLAN gateway addresses where it is not needed.

The important distinction is:

```text
LAN_IN
    traffic passing through the firewall

LAN_LOCAL
    traffic destined for the firewall itself
```

Both need an explicit policy.

---

## Adding a Device

When adding a new device, I follow a repeatable process.

### 1. Put it in IoT

The device starts with the normal IoT restrictions.

### 2. Configure the Home Assistant integration

If discovery fails, I don't immediately open the entire VLAN.

### 3. Check firewall logs

Look for denied traffic:

```text
source
destination
protocol
port
```

### 4. Add the smallest required rule

For example:

```text
HA → 192.168.50.42
TCP 9999
ALLOW
```

is preferable to:

```text
HA → 192.168.50.0/24
ALL
ALLOW
```

### 5. Test again

If the integration works, the rule becomes part of the documented device policy.

This process makes the firewall itself useful documentation of how the smart home operates.

---

## Troubleshooting

When an integration stops working, I work through the network path rather than immediately changing firewall rules.

### Check DNS

From the relevant VLAN, verify that the device can resolve through Pi-hole.

### Check discovery

For mDNS-related problems, verify that the device is visible across the expected VLAN boundary.

### Check firewall logs

Look for denied traffic between Home Assistant and the device.

### Check the specific port

Once a blocked connection is identified, test the port directly where appropriate.

For example:

```bash
nc -vz 192.168.50.42 9999
```

or:

```bash
nc -vz 192.168.50.42 80
```

The objective is to determine whether the problem is:

```text
DNS
 ↓
Discovery
 ↓
Routing
 ↓
Firewall
 ↓
Application protocol
```

rather than solving every problem by making the firewall less restrictive.

---

## What Not to Do

Several shortcuts make IoT networks easier to configure but undermine the isolation model.

### Don't allow the entire IoT VLAN

Avoid:

```text
Home Assistant → IoT → ALL
```

unless there is a very specific reason.

### Don't permanently allow administration

Avoid:

```text
Trusted → IoT → ALL
```

just because occasionally you need to configure a device.

### Don't allow arbitrary DNS

Avoid:

```text
IoT → Internet → TCP/UDP 53
```

when Pi-hole is intended to be the network DNS authority.

### Don't treat mDNS as general access

Allowing UDP 5353 does not mean the entire IoT network needs unrestricted access to Home Assistant or other VLANs.

---

## The Result

The resulting firewall is intentionally boring:

```text
IoT
 │
 ├── DNS ──────────────> Pi-hole
 │
 ├── Discovery ────────> mDNS reflector
 │
 └── Control ──────────> Home Assistant
                              │
                              └── specific device ports
```

Everything else is denied unless there is a documented reason to allow it.

That makes the network slightly more work to configure initially, but it also makes the resulting security model much easier to understand.

When a new device needs access, I can answer a concrete question:

> **What exactly does this device need to communicate with?**

rather than:

> **Which part of the network should I trust this device with?**

That difference is what makes the zero-trust model practical for a real smart home.
