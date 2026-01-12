---
layout: post
title: "Proxmox Network Isolation Series – Part 1: Securing a Single-Node Hypervisor"
date: 2026-01-08
categories: [homelab, proxmox, security, networking]
tags: [Proxmox, VLAN, Homelab, Virtualization, Security, Firewall]
---

## Introduction

This article is **Part 1 of a multi-part series** focused on hardening Proxmox by removing it from the native VLAN and enforcing strict network isolation.

We intentionally start with the **simplest and lowest-risk scenario**: a **single-node Proxmox hypervisor**.

Even without clustering, a Proxmox host is a **high-value target**. It controls virtual machines, storage access, and often backup credentials. Exposing it on the same network as user devices significantly increases risk.

This article focuses exclusively on:

- One Proxmox node
- Management traffic isolation
- VLAN-aware VM networking
- Firewall-enforced security boundaries

> No clustering, Corosync, or migration complexity is introduced in Part 1.

---

## Threat Model (Why This Matters)

Without isolation, a single-node Proxmox host is typically exposed to:

- User endpoints
- IoT devices
- Guest or lab networks

If any of those systems are compromised, the hypervisor itself becomes reachable.

Our goal is to ensure:

- The **Proxmox management plane** is reachable only from trusted admin networks
- **VM traffic** is isolated from the hypervisor
- The **native VLAN is completely removed** from the host

---

## Prerequisites (Non-Negotiable)

VLANs alone do **not** provide security. They only provide separation.

### Mandatory Components

- **Managed switch** with VLAN tagging and trunking
- **Stateful firewall/router** capable of:
  - Inter-VLAN routing
  - Explicit allow/deny rules
- Console or out-of-band access to the Proxmox host

> ⚠️ If your router can route VLANs but cannot enforce firewall rules, this architecture provides **no real security benefit**.

### Roles and Responsibilities

- **Switch** → Enforces segmentation
- **Firewall** → Enforces security

Both are required.

---

## VLAN Design (Single-Node Only)

For a single-node hypervisor, the VLAN model is intentionally minimal.

| VLAN ID | Name        | Purpose |
|-------:|-------------|---------|
| 90     | Management  | Proxmox Web UI, SSH |
| 200+   | VM Networks | Guest virtual machines |

### Design Principles

- Proxmox management traffic must live on a **dedicated management VLAN**
- VM networks must never share the management VLAN
- The native (untagged) VLAN must not touch the host

---

## Example Environment

| Hostname | Role         | Management IP |
|---------|--------------|---------------|
| hv1     | Proxmox Node | 192.168.90.11 |

### Logical Topology

```text
                ┌──────────────────────┐
                │   Firewall / Router  │
                │  (Routing + Policy)  │
                └─────────┬────────────┘
                          │
                ┌─────────┴─────────┐
                │   Managed Switch  │
                │ (VLAN Trunk Port) │
                └─────────┬─────────┘
                          │
                        hv1
        ┌──────────────────────────────────┐
        │ VLAN 90  → Proxmox Management     │
        │ VLAN 200 → Virtual Machines       │
        │ No Native VLAN                    │
        └──────────────────────────────────┘
````

---

## Step 1: Switch Configuration

Configure the switch port connected to the Proxmox host as a **trunk**.

* Allowed VLANs: `90, 200+`
* Native VLAN: **None** (or unused VLAN)

Conceptual example:

```text
Proxmox Port:
  Tagged VLANs: 90, 200
  Native VLAN: Disabled
```

This ensures the host never sees untagged traffic.

---

## Step 2: Firewall Configuration

On the firewall:

* Create a VLAN interface for **VLAN 90 (Management)**
* Assign a gateway IP (e.g. `192.168.90.1`)
* Create VLAN interfaces for VM networks as needed

### Minimum Firewall Rules

* Allow HTTPS (8006) to VLAN 90 **only** from admin networks or hosts
* Allow SSH only from trusted IPs
* Deny all other inbound access to VLAN 90

> If a user VLAN can reach the Proxmox UI, the design has failed.

---

## Step 3: Proxmox Network Configuration

Edit `/etc/network/interfaces`:

```text
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.90.11/24
    gateway 192.168.90.1
    bridge-ports eno1.90
    bridge-stp off
    bridge-fd 0
```

This bridge is **management only**.

---

## Step 4: VLAN-Aware VM Bridge

Create a dedicated bridge for virtual machines:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno1
    bridge-vlan-aware yes
    bridge-stp off
    bridge-fd 0
```

Attach VM NICs to `vmbr1` and assign VLAN tags (e.g. `200`).

---

## Step 5: Validation

From the Proxmox host:

```bash
ip a
ip route
```

Confirm:

* Default route exists only on VLAN 90
* No interface is attached to an untagged VLAN

From an untrusted VLAN:

* Proxmox UI should be unreachable

---
## Firewall Rule Examples

### pfSense / OPNsense

**VLAN 90 (Management)**

* Allow TCP 8006 from admin subnet
* Allow SSH from admin IPs
* Block all other inbound traffic
* Log activities (Optional but important)

**VLAN 99 (Cluster)**

* Block all routed traffic
* No default gateway preferred

**VLAN 200+ (VMs)**

* Explicit allow rules only

---

### Ubiquiti (Conceptual)

* Create LAN-IN rules denying access to VLAN 90
* Permit admin subnet → VLAN 90
* Block VLAN 200+ → VLAN 90/99

---

## Common Pitfalls (Single Node)

### Leaving the Native VLAN Enabled

Even one untagged interface can expose the host.

**Fix:** Remove native VLAN access entirely.

---

### Treating VLANs as Security Controls

VLANs without firewall rules are not security boundaries.

**Fix:** Enforce policy at the firewall.

---

### Mixing VM and Management Traffic

Sharing a bridge increases blast radius.

**Fix:** Separate management and VM bridges.

---

## What Comes Next

In **Part 2**, we extend this exact design to a **fresh three-node Proxmox cluster**, introducing:

* Dedicated cluster networking
* Corosync isolation
* Quorum-safe design

No rework required.

---

## Closing Thoughts

A single-node hypervisor deserves the same security discipline as a cluster.

When the native VLAN is removed, management is isolated, and firewall policy is enforced, Proxmox becomes **predictable, secure, and safe to grow**.

Happy virtualizing 🚀

