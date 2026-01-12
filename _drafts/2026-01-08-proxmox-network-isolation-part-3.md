---

layout: post
title: "Proxmox Network Isolation Series – Part 3: Migrating an Existing Cluster with Running VMs"
date: 2026-01-08
categories: [homelab, proxmox, security, networking]
tags: [Proxmox, VLAN, Migration, Cluster, Corosync, Firewall, Security]
-----------------------------------------------------------------------

## Introduction

This article is **Part 3** of the Proxmox Network Isolation Series.

In this part, we tackle the **hardest and most dangerous scenario**:

> A **running Proxmox cluster** with **active VMs**, where:
>
> * Hypervisors live on the **native VLAN**
> * Management, cluster, and VM traffic are **mixed**
> * Proxmox has already made **bad interface decisions**
> * Downtime must be **minimized**

Unlike Parts 1 and 2, this is **not a clean build**. This is controlled surgery.

---

## ⚠️ Read This First: Scope and Risk

This guide assumes:

* A **functional but poorly isolated** Proxmox cluster
* **Quorum is healthy** at the start
* VMs are **running and providing services**

This guide does **not** attempt:

* Zero-risk migration (that does not exist)
* Blind automation
* "Just reboot everything" shortcuts

> If you rush this process, you *will* lose cluster communication.

---

## Why Existing Clusters Break During VLAN Migration

Most legacy clusters fail for predictable reasons:

| Failure                   | Root Cause                         |
| ------------------------- | ---------------------------------- |
| Nodes join using wrong IP | DNS / `/etc/hosts` ambiguity       |
| Corosync flaps            | Routed or unstable cluster network |
| Random node fencing       | Multiple default gateways          |
| Split brain               | Cluster traffic crossing firewall  |
| Management lockout        | Gateway removed too early          |

Proxmox is **not wrong** — it simply uses the *first valid path it sees*.

---

## Target End State (Recap)

By the end of this migration:

| VLAN | Purpose     | Routing              |
| ---: | ----------- | -------------------- |
|   90 | Management  | Routed + Firewalled  |
|   99 | Cluster     | L2-only (no gateway) |
| 200+ | VM Networks | Routed + Firewalled  |

* No Proxmox service touches the native VLAN
* Corosync traffic never reaches the firewall
* VM traffic is isolated from the hypervisor

---

## Migration Strategy Overview

This migration follows **four controlled phases**:

1. **Stabilize and observe** (no changes yet)
2. **Add new VLANs in parallel**
3. **Move management traffic first**
4. **Rebind cluster traffic last**

> Management first. Cluster last. Never reverse this order.

---

## Phase 0: Pre-flight Validation (Do Not Skip)

### 1. Verify Cluster Health

```bash
pvecm status
```

Confirm:

* Quorum is healthy
* All nodes are visible
* No packet loss warnings

If quorum is unstable now, **stop**.

---

### 2. Identify Current Traffic Paths

On each node:

```bash
ip route
ip addr
```

Document:

* Default gateway
* Interface used for cluster traffic
* Any secondary routes

You *must* know what Proxmox is using today before changing it.

---

## Phase 1: Prepare the Network (No Host Changes Yet)

### Switch Configuration

* Convert Proxmox ports to **trunk ports**
* Allow VLANs: `90`, `99`, `200+`
* Leave native VLAN temporarily

This ensures **no traffic loss** when VLANs are added.

---

### Firewall Preparation

* Create VLAN interfaces for `90` and `200+`
* **Do NOT create VLAN 99 on the firewall**
* Prepare rules but do not enforce yet

> VLAN 99 must never be routable.

---

## Phase 2: Fix Name Resolution (Critical)

This is where most migrations fail.

Edit `/etc/hosts` on **every node**:

```text
192.168.90.11 hv1.mgmt hv1
192.168.90.12 hv2.mgmt hv2
192.168.90.13 hv3.mgmt hv3

10.99.0.11 hv1-cluster
10.99.0.12 hv2-cluster
10.99.0.13 hv3-cluster
```

Rules:

* Management names resolve **only** to VLAN 90
* Cluster names resolve **only** to VLAN 99
* Never reuse hostnames across VLANs

---

## Phase 3: Add Management VLAN (Parallel Operation)

On **one node at a time**:

1. Add VLAN 90 interface
2. Assign management IP
3. Move Web UI + SSH access

Example:

```text
auto vmbr0
iface vmbr0 inet static
    address 192.168.90.11/24
    gateway 192.168.90.1
    bridge-ports eno1.90
    bridge-stp off
    bridge-fd 0
```

### Validation

* Open Proxmox UI via VLAN 90
* Keep existing access until verified

Repeat node-by-node.

---

## Phase 4: Remove Native VLAN Dependency

Once **all nodes** are reachable via VLAN 90:

* Remove default gateway from native interface
* Ensure **only one default gateway exists**

This step prevents:

* Asymmetric routing
* Corosync flapping

---

## Phase 5: Add Cluster VLAN (No Routing)

On each node:

```text
auto vmbr99
iface vmbr99 inet static
    address 10.99.0.11/24
    bridge-ports eno1.99
    bridge-stp off
    bridge-fd 0
```

⚠️ No gateway. Ever.

Verify node-to-node ping on VLAN 99.

---

## Phase 6: Rebind Corosync (The Dangerous Step)

On **one node only**:

```bash
pvecm expected 1
```

Then recreate the cluster configuration:

```bash
pvecm create homelab --bindnet0 10.99.0.0
```

Rejoin remaining nodes **one at a time**:

```bash
pvecm add 10.99.0.11
```

Restore quorum:

```bash
pvecm expected 3
```

---

## Phase 7: Lock Down the Firewall

Now — and only now — enforce rules.

### Management VLAN (90)

* Allow 8006 / 22 from admin IPs
* Deny all other inbound

### Cluster VLAN (99)

* No firewall interface
* No routing

### VM VLANs (200+)

* Apply least-privilege rules

If VLAN 99 appears on the firewall, **stop and fix it**.

---

## Common Failure Modes and Recovery

| Symptom            | Likely Cause              | Recovery                   |
| ------------------ | ------------------------- | -------------------------- |
| Node drops         | Cluster VLAN routed       | Remove gateway immediately |
| Join uses wrong IP | DNS ambiguity             | Fix `/etc/hosts`           |
| Web UI unreachable | Gateway removed too early | Re-add temporarily         |
| Split brain        | Multiple gateways         | Single default route       |

---

## Final State Verification

```bash
pvecm status
ip route
```

Confirm:

* Corosync on VLAN 99 only
* Management via VLAN 90
* No native VLAN usage

---

## Closing Thoughts

Migrating a live Proxmox cluster is **not configuration work** — it is **change management**.

The rules are simple:

* Observe before changing
* One node at a time
* Management first
* Cluster last

Do this correctly, and you transform a fragile environment into a **predictable, hardened platform**.

---

**Series Complete** 🚀

You now have:

* A secure single node
* A clean new cluster design
* A safe path for legacy migrations
