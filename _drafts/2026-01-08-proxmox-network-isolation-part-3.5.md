---

layout: post
title: "Proxmox Network Isolation Series – Part 3.5: Recovering From Broken Quorum Without Losing VMs"
date: 2026-01-08
categories: [homelab, proxmox, security, networking, recovery]
tags: [Proxmox, Quorum, Corosync, Recovery, VLAN, Migration, Security]
----------------------------------------------------------------------

## Introduction

This article is a **direct follow-up to Part 3**.

In Part 3, we stated clearly:

> **If quorum is unstable, stop.**

This article explains **what to do next** when you cannot proceed safely.

We focus on a common real-world scenario:

* A Proxmox cluster exists
* Quorum is **broken or flapping**
* Network design is mixed or incorrect
* VMs are **running and must be preserved**

The goal is **not** to fix quorum immediately.

The goal is to:

> **Safely dismantle the broken cluster while keeping all VMs intact**,
> then rebuild the cluster later using a hardened VLAN design.

---

## What This Article Does (and Does Not) Do

### This Article Does

* Preserve all existing VMs and disks
* Break a failing cluster **deliberately and cleanly**
* Convert each hypervisor into a **standalone node**
* Remove Corosync safely
* Prepare hosts for secure re-clustering

### This Article Does NOT

* Attempt live quorum repair on unstable networks
* Hide risk behind automation
* Guarantee zero downtime

> A broken cluster is already unsafe. Controlled disassembly is safer.

---

## Why Broken Quorum Must Be Addressed First

Operating on an unstable cluster is dangerous because:

* Proxmox fencing decisions become unpredictable
* VM ownership may change unexpectedly
* Network changes can trigger split brain

If quorum is not stable:

* **Do not migrate networking**
* **Do not rebind Corosync**
* **Do not remove gateways**

First, regain **administrative control**.

---

## High-Level Recovery Strategy

We will:

1. Freeze the environment
2. Force single-node quorum temporarily
3. Safely remove Corosync
4. Verify VMs remain accessible
5. Convert hosts to standalone mode

Only *after* this process should VLAN segmentation begin.

---

## Phase 0: Freeze the Environment

### Stop All Cluster Operations

* Disable HA (if enabled)
* Stop scheduled backups
* Avoid VM migrations

On each node:

```bash
systemctl stop pve-ha-lrm
systemctl stop pve-ha-crm
```

---

## Phase 1: Identify the Most Stable Node

Choose the node that:

* Has the **most running VMs**
* Has reliable console access
* Can reach its own disks consistently

This node will temporarily become the **cluster authority**.

---

## Phase 2: Force Quorum (Temporary Emergency State)

On the selected node **only**:

```bash
pvecm expected 1
```

This tells Proxmox:

> "Assume you are alone. Do not fence."

⚠️ This is a **temporary survival measure**, not a fix.

---

## Phase 3: Verify VM Integrity

Confirm that:

* All VMs appear in the Web UI
* VM disks are accessible
* Storage is not read-only

```bash
qm list
pvesm status
```

If disks are unavailable, **stop here**.

---

## Phase 4: Cleanly Remove Corosync

On **each node**, one at a time:

```bash
systemctl stop corosync
systemctl disable corosync
```

Then remove cluster configuration:

```bash
rm -rf /etc/corosync/*
rm -rf /var/lib/corosync/*
```

On the authoritative node:

```bash
rm -rf /etc/pve/corosync.conf
```

This breaks cluster membership **without touching VM data**.

---

## Phase 5: Restart Proxmox Services

On each node:

```bash
systemctl restart pve-cluster
systemctl restart pvedaemon
systemctl restart pveproxy
```

The Web UI should now show:

* No cluster
* Local node only

---

## Phase 6: Validate Standalone Operation

On each host:

```bash
pvecm status
```

Expected output:

* Not a member of any cluster

Confirm:

* VMs start and stop normally
* Storage behaves correctly

At this point, **each hypervisor is independent**.

---

## Phase 7: Why This Works (And Why VMs Survive)

Key facts:

* VM configs live in `/etc/pve/nodes/<node>/qemu-server/`
* VM disks are not stored in Corosync
* Corosync controls **coordination**, not storage

Removing Corosync does **not** delete VMs.

---

## Common Mistakes During Quorum Recovery

### 1. Removing Corosync Before Forcing Quorum

This can cause:

* Services to stop responding
* UI lockups

**Always force quorum first.**

---

### 2. Attempting Network Migration Mid-Recovery

Changing VLANs during quorum instability increases risk.

**Stabilize → dismantle → redesign → rebuild.**

---

### 3. Leaving HA Enabled

HA assumes quorum.

Disable it before proceeding.

---

## Transition to Secure Rebuild

You now have:

* Independent Proxmox nodes
* All VMs intact
* No cluster dependencies

From here, you can safely:

* Apply **Part 1** to each node
* Introduce VLAN isolation cleanly
* Proceed to **Part 2** to rebuild a secure cluster

---

## Why Not Just “Fix Quorum?”

This is a fair question, and it deserves a direct answer.

In theory, quorum issues *can* sometimes be fixed by:

* Adjusting Corosync bind addresses
* Tuning timeouts
* Restarting services
* Reordering interfaces

In practice, **that approach is unsafe when the network itself is wrong**.

### The Core Problem

Quorum failures are rarely the *root cause*.

They are a **symptom** of deeper issues such as:

* Mixed or ambiguous VLAN usage
* Multiple default gateways
* Routed cluster traffic
* DNS or `/etc/hosts` ambiguity
* Proxmox binding to the "first working interface"

Trying to fix quorum without fixing these fundamentals is like:

> Re-aligning wheels on a car with a bent frame.

You may get temporary stability, but the system remains fragile.

---

### Why Repairing Quorum In-Place Is Risky

Attempting live quorum repair in a misdesigned network can cause:

* Sudden fencing events
* VM ownership confusion
* Split brain during partial fixes
* Cascading failures when a "fix" partially works

Worse, these failures often happen **minutes or hours later**, not immediately.

---

### The Safer Philosophy: Control First, Optimization Later

This series deliberately chooses a conservative approach:

1. **Regain control** (force quorum or dismantle)
2. **Remove undefined behavior** (Corosync on a bad network)
3. **Redesign networking correctly** (VLAN isolation + firewall enforcement)
4. **Rebuild the cluster cleanly**

This may feel slower — but it is:

* More predictable
* Easier to reason about
* Far less likely to cause data loss

---

### When *Does* It Make Sense to Fix Quorum Directly?

Direct quorum repair can be reasonable **only if**:

* Network design is already correct
* Cluster traffic is isolated
* Gateways are unambiguous
* Name resolution is deterministic
* You can reproduce the failure reliably

If those conditions are not true, fixing quorum treats the symptom — not the disease.

---

## Final Thoughts

A broken cluster is not a failure — it is a **signal**.

By intentionally dismantling it:

* You regain control
* You eliminate undefined behavior
* You create space for a correct design

Only after control is restored does security work begin.

---

**Next Steps**

* Apply Part 1 to each standalone node
* Rebuild using Part 2
* If needed, return to Part 3 for migration guidance

This completes the **full lifecycle**: from broken quorum → hardened cluster.
