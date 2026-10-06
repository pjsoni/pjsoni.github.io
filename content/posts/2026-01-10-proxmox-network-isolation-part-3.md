---
title: "Proxmox Network Isolation Series – Part 3: Migrating an Existing Cluster with Running VMs"
date: 2026-01-10
slug: proxmox-network-isolation-series-part-3-migrating-an-existing-cluster-with-running-vms 
summary: "A practical approach to migrating an existing Proxmox cluster from a mixed or poorly isolated network to a properly segmented VLAN architecture while keeping running workloads in mind. The guide focuses on sequencing, quorum protection, management access, Corosync connectivity, and minimizing the risk of losing cluster communication during the migration."
series:
  - "Proxmox Network Isolation Series"
series_order: 3
categories: [Infrastructure, Security]
tags: [Firewall, Homelab, Proxmox, Security, Virtualization, VLAN]
jumbotron:
  meta: true
---

## Introduction

This article is **Part 3** of the Proxmox Network Isolation Series.

In **Part 1**, we established the network isolation model for a single Proxmox host.

In **Part 2**, we extended that architecture to a fresh three-node cluster, where management, cluster communication, and VM traffic could be separated before the cluster was created.

This part addresses the situation where that clean starting point is no longer available.

We are tackling the **hardest and most dangerous scenario**:

> A **running Proxmox cluster** with **active VMs**, where:
>
> * Hypervisors live on the **native VLAN**
> * Management, cluster, and VM traffic are **mixed**
> * Proxmox has already made **bad interface decisions**
> * Downtime must be **minimized**

Unlike Parts 1 and 2, this is **not a clean build**.

The cluster already exists.

The VMs are already running.

The network configuration is already being used by the Proxmox nodes and their workloads.

This means the objective is not simply to configure the desired end state. The objective is to move from the current state to the new state without breaking the cluster along the way.

This is controlled surgery.

## ⚠️ Read This First: Scope and Risk

This guide assumes:

* A **functional but poorly isolated** Proxmox cluster
* **Quorum is healthy** at the start
* VMs are **running and providing services**

The starting condition matters.

A healthy quorum gives us a stable point from which to begin making changes. If the cluster is already unstable, changing its network configuration adds another failure condition to an already unhealthy environment.

Before starting, understand that this is a live infrastructure change.

This guide does **not** attempt:

* Zero-risk migration (that does not exist)
* Blind automation
* "Just reboot everything" shortcuts

The migration is intentionally broken into controlled phases so that each major change can be validated before moving to the next one.

> If you rush this process, you *will* increase the chance of losing cluster communication.

The most important principle throughout the migration is to avoid changing everything simultaneously.


## Why Existing Clusters Break During VLAN Migration

A fresh cluster is relatively straightforward because its network architecture can be established before the cluster exists.

An existing cluster is different.

The nodes may already have:

* Management addresses
* Default gateways
* Corosync communication paths
* Hostname resolution
* Existing network bridges
* Running VMs

Changing one of these without understanding the others can affect cluster communication.

Most legacy clusters fail for predictable reasons:

| Failure                   | Root Cause                         |
| ------------------------- | ---------------------------------- |
| Nodes join using wrong IP | DNS / `/etc/hosts` ambiguity       |
| Corosync flaps            | Routed or unstable cluster network |
| Random node fencing       | Multiple default gateways          |
| Split brain               | Cluster traffic crossing firewall  |
| Management lockout        | Gateway removed too early          |

### Nodes Join Using the Wrong IP

Cluster communication depends on the addresses associated with the cluster configuration.

If a hostname can resolve to more than one network, it becomes difficult to determine which path the cluster will actually use.

This is why name resolution needs to be made explicit before changing the network.

### Corosync Flaps

Corosync should have a predictable and stable network path.

If cluster traffic is routed through a network that is also carrying other traffic, or if the network itself is unstable, Corosync communication can become unreliable.

The target architecture therefore places Corosync on VLAN 99 and keeps that network separate from management and VM traffic.

### Random Node Fencing

Multiple default gateways can create asymmetric routing.

A packet can leave through one interface while the response follows another path.

For a cluster migration, there should be a clear management route rather than multiple competing default routes.

### Split Brain

Cluster communication must not accidentally cross a firewall or other routed boundary that was not intended for the cluster network.

The target design keeps VLAN 99 as an L2-only cluster network.

### Management Lockout

Removing the old gateway before the new management path has been verified can disconnect administrators from the host.

That is why management access is moved before the old path is removed.

The sequence matters.

> **Management first. Cluster last.**

## Target End State (Recap)

The final architecture returns the existing cluster to the same basic separation established in Parts 1 and 2.

| VLAN | Purpose     | Routing              |
| ---: | ----------- | -------------------- |
|   90 | Management  | Routed + Firewalled  |
|   99 | Cluster     | L2-only (no gateway) |
| 200+ | VM Networks | Routed + Firewalled  |

The intended boundaries are:

* No Proxmox service touches the native VLAN
* Corosync traffic never reaches the firewall
* VM traffic is isolated from the hypervisor

Conceptually:

```text
                         Firewall / Router
                              |
                    +---------+---------+
                    |                   |
                 VLAN 90             VLAN 200+
                 Management             VMs
                    |                   |
                    +-------------------+

                         L2 Network
                            |
                         VLAN 99
                            |
                 hv1 <----> hv2 <----> hv3
                       Corosync
```

The important part is that VLAN 99 does not become another routed network.

Management and VM networks can be routed and controlled through the firewall.

The cluster network remains local to the cluster communication path.

## Migration Strategy Overview

This migration follows **four controlled phases**:

1. **Stabilize and observe** (no changes yet)
2. **Add new VLANs in parallel**
3. **Move management traffic first**
4. **Rebind cluster traffic last**

These phases establish the order of operations.

> **Management first. Cluster last. Never reverse this order.**

The reason is straightforward.

Management access is what allows administrators to observe and correct the environment during the migration.

Corosync is the component that maintains cluster communication and quorum.

Changing the cluster communication path before establishing reliable management access creates an unnecessary risk.

## Phase 0: Pre-flight Validation (Do Not Skip)

Before making any network changes, establish a clear picture of the current cluster.

### 1. Verify Cluster Health

Run:

```bash id="q3i8k6"
pvecm status
```

Confirm:

* Quorum is healthy
* All nodes are visible
* No packet loss warnings

The migration should begin from a stable cluster.

If quorum is already unstable, there is no reliable baseline from which to perform the migration.

If quorum is unstable now, **stop**.

Do not use the network migration to try to solve an existing quorum problem.

### 2. Identify Current Traffic Paths

On each node:

```bash id="1u6l3k"
ip route
ip addr
```

Document:

* Default gateway
* Interface used for cluster traffic
* Any secondary routes

The objective is to understand the current state before changing it.

For example, you need to know which interface currently provides:

```text
Management
    |
    +---- IP address
    +---- Default gateway

Cluster
    |
    +---- Current interface/address
```

You *must* know what Proxmox is using today before changing it.

This is particularly important in a legacy environment where the network configuration may have evolved over time.


## Phase 1: Prepare the Network (No Host Changes Yet)

The network infrastructure should be prepared before changing the Proxmox nodes.

This creates the new network paths first, allowing them to be validated independently before the existing paths are removed.

### Switch Configuration

Configure the Proxmox ports as trunk ports.

* Convert Proxmox ports to **trunk ports**
* Allow VLANs: `90`, `99`, `200+`
* Leave native VLAN temporarily

The native VLAN is intentionally left in place during this stage because the existing cluster is still using it.

Removing it immediately would change the network path before the new management path has been established.

The goal of this phase is therefore:

```text
Existing network
      +
New VLANs
      |
      v
Parallel operation
```

This ensures **no traffic loss** simply because the VLANs are being introduced.

The old path remains available while the new paths are prepared.

### Firewall Preparation

Create the firewall interfaces required for the routed networks:

* Create VLAN interfaces for `90` and `200+`
* **Do NOT create VLAN 99 on the firewall**
* Prepare rules but do not enforce yet

VLAN 90 needs to be routable because administrators need to reach the Proxmox management network.

VM VLANs also require routing and firewall policy according to their intended use.

VLAN 99 is different.

It is the cluster network and should remain local to the cluster.

> VLAN 99 must never be routable.

The firewall therefore should not become part of the Corosync path.


## Phase 2: Fix Name Resolution (Critical)

Name resolution becomes particularly important during a migration because the cluster is moving from one set of network addresses to another.

This is where many migrations fail.

Edit `/etc/hosts` on **every node**:

```text id="xq7y1j"
192.168.90.11 hv1.mgmt hv1
192.168.90.12 hv2.mgmt hv2
192.168.90.13 hv3.mgmt hv3

10.99.0.11 hv1-cluster
10.99.0.12 hv2-cluster
10.99.0.13 hv3-cluster
```

The purpose is to make the two types of addresses explicit.

Management names point to management addresses.

Cluster names point to cluster addresses.

Rules:

* Management names resolve **only** to VLAN 90
* Cluster names resolve **only** to VLAN 99
* Never reuse hostnames across VLANs

This prevents an ambiguous name from becoming an accidental path between networks.

For example:

```text
hv1
 |
 +---- Management name
 |       |
 |       +---- VLAN 90
 |
 +---- Cluster name
         |
         +---- VLAN 99
```

The two functions remain explicitly separated.

This step should be completed before changing Corosync connectivity.

## Phase 3: Add Management VLAN (Parallel Operation)

Once the switch and firewall are ready, establish the new management network.

Do this **one node at a time**.

For each node:

1. Add VLAN 90 interface
2. Assign management IP
3. Move Web UI + SSH access

For example:

```text id="e5f3w7"
auto vmbr0
iface vmbr0 inet static
    address 192.168.90.11/24
    gateway 192.168.90.1
    bridge-ports eno1.90
    bridge-stp off
    bridge-fd 0
```

The example uses `hv1`.

The other nodes use their respective VLAN 90 management addresses.

### Why One Node at a Time?

Changing all nodes simultaneously removes the ability to use the remaining nodes as a reference point.

With one node at a time, you can:

* Make the change
* Verify management connectivity
* Confirm the cluster remains healthy
* Move to the next node

The existing management path remains available until the new path has been confirmed.

### Validation

After changing each node:

* Open Proxmox UI via VLAN 90
* Keep existing access until verified

The important part is not simply seeing the login page.

Verify that the new management path actually works before removing the old one.

Repeat node-by-node.

The migration should therefore progress like:

```text
Node 1
  |
  +---- VLAN 90 configured
  +---- Management verified
        |
        v
Node 2
  |
  +---- VLAN 90 configured
  +---- Management verified
        |
        v
Node 3
  |
  +---- VLAN 90 configured
  +---- Management verified
```

## Phase 4: Remove Native VLAN Dependency

Once **all nodes** are reachable via VLAN 90, the old management path can be removed.

At this point, the new management network has been tested on every node.

Now:

* Remove default gateway from native interface
* Ensure **only one default gateway exists**

The objective is to make VLAN 90 the management path without leaving competing default routes behind.

A single default route provides a predictable path for traffic leaving the Proxmox host.

Multiple gateways can introduce asymmetric routing, where traffic leaves through one interface and returns through another.

This step therefore prevents:

* Asymmetric routing
* Corosync flapping

The important sequence is:

```text
New management path
        |
        v
Verified on all nodes
        |
        v
Old management path removed
        |
        v
Single default gateway
```

Do not reverse that sequence.

## Phase 5: Add Cluster VLAN (No Routing)

With management access established, the dedicated cluster network can now be introduced.

On each node:

```text id="k2r6mj"
auto vmbr99
iface vmbr99 inet static
    address 10.99.0.11/24
    bridge-ports eno1.99
    bridge-stp off
    bridge-fd 0
```

Each node uses its own address on the VLAN 99 network.

For example:

```text id="x6e3r4"
hv1 → 10.99.0.11
hv2 → 10.99.0.12
hv3 → 10.99.0.13
```

⚠️ **No gateway. Ever.**

The cluster interface should not become another routed path.

The network is intended specifically for node-to-node cluster communication.

### Validate Node-to-Node Connectivity

Verify node-to-node ping on VLAN 99.

The purpose of this test is to confirm that the nodes can communicate over the new cluster network before Corosync is moved to it.

At this stage, the desired state is:

```text
VLAN 90
    |
    +---- Management
          Routed

VLAN 99
    |
    +---- hv1
    +---- hv2
    +---- hv3
          L2-only

VLAN 200+
    |
    +---- VM Networks
          Routed
```

The cluster network should not require the firewall to provide connectivity between the nodes.

## Phase 6: Rebind Corosync (The Dangerous Step)

This is the most sensitive part of the migration.

Up to this point, the existing cluster communication path has been left intact while the new network was prepared.

Now the cluster communication path itself needs to be moved to VLAN 99.

The important distinction is that an existing cluster should **not be casually destroyed and recreated** as part of this migration.

The command:

```bash
pvecm expected 1
```

is a quorum-recovery mechanism. Proxmox documents it for situations where a configuration needs to be changed while the cluster is not quorate; it temporarily makes the cluster quorate with an expected vote count of one. It is not a normal first step for rebuilding a healthy cluster.

Likewise, recreating the cluster with:

```bash
pvecm create homelab --bindnet0 10.99.0.0
```

would no longer be a migration of the existing cluster. It would be creating a new cluster configuration.

For an existing cluster, Corosync supports changing its link configuration by adding or changing the `ringX_addr` entries in `corosync.conf`. Proxmox documents this as the mechanism for adding links to a running cluster.

### The Migration Principle

The intended sequence remains the same:

```text
Existing Corosync path
        |
        | keep working
        v
New VLAN 99 path
        |
        | verify
        v
Move Corosync
        |
        v
Remove old cluster dependency
```

The critical point is that the new cluster network must be working before Corosync depends on it.

On the existing cluster, update the Corosync configuration so that the cluster addresses point to the VLAN 99 addresses.

The cluster addresses from this example are:

```text
hv1 → 10.99.0.11
hv2 → 10.99.0.12
hv3 → 10.99.0.13
```

Proxmox's current Corosync documentation uses `ringX_addr` for the node addresses associated with Corosync links. When modifying a running configuration, the corresponding link/address needs to be represented consistently across the cluster.

This is the point where the earlier validation becomes important.

Before changing Corosync:

* VLAN 99 exists
* Every node has a VLAN 99 address
* Nodes can communicate over VLAN 99
* Management access through VLAN 90 works
* The cluster currently has healthy quorum

Only then should the Corosync configuration be changed.

### Why This Step Is Dangerous

Corosync is not simply another application that can tolerate a temporary network interruption.

It is responsible for cluster communication and quorum.

If the nodes stop communicating over the configured cluster network, the cluster can lose quorum.

That is why the migration does not start by changing Corosync.

The earlier phases establish a stable fallback and verify the new paths before this final network dependency is changed.

> The dangerous part is not creating VLAN 99. The dangerous part is making the running cluster depend on VLAN 99 for Corosync.



## Phase 7: Lock Down the Firewall

Once the new paths are working and Corosync has been moved to the dedicated cluster network, the firewall policy can be enforced.

The final firewall policy should reflect the target architecture.

### Management VLAN (90)

* Allow 8006 / 22 from admin IPs
* Deny all other inbound

This preserves administrative access while preventing other networks from reaching the Proxmox management services.

### Cluster VLAN (99)

* No firewall interface
* No routing

The cluster network remains an L2-only communication path.

Corosync should not need to cross the firewall.

### VM VLANs (200+)

* Apply least-privilege rules

VM networks should be treated independently from both the management and cluster networks.

The VM networks may be routed, but access should be controlled by firewall policy.

If VLAN 99 appears on the firewall, **stop and fix it**.

The intended design is:

```text
VLAN 90
   |
   +---- Firewall
          |
          +---- Management access

VLAN 99
   |
   +---- No firewall
   |
   +---- Corosync only

VLAN 200+
   |
   +---- Firewall
          |
          +---- VM network policy
```

This preserves the same boundaries established in Parts 1 and 2.



## Common Failure Modes and Recovery

Even with careful sequencing, a live migration can fail.

The symptoms can usually be mapped back to one of the network boundaries being changed incorrectly.

| Symptom            | Likely Cause              | Recovery                   |
| ------------------ | ------------------------- | -------------------------- |
| Node drops         | Cluster VLAN routed       | Remove gateway immediately |
| Join uses wrong IP | DNS ambiguity             | Fix `/etc/hosts`           |
| Web UI unreachable | Gateway removed too early | Re-add temporarily         |
| Split brain        | Multiple gateways         | Single default route       |

### Node Drops

If a node drops after moving cluster traffic, verify that VLAN 99 is not being routed and that the cluster addresses can communicate directly.

**Likely cause:** Cluster VLAN routed.

**Recovery:** Remove the gateway immediately.

### Join Uses Wrong IP

If cluster communication uses an unexpected address, review `/etc/hosts` and the cluster address configuration.

**Likely cause:** DNS ambiguity.

**Recovery:** Fix `/etc/hosts`.

Management names should resolve to VLAN 90.

Cluster names should resolve to VLAN 99.

### Web UI Becomes Unreachable

If the Proxmox UI disappears after the network change, the new management path may not have been established correctly before the old path was removed.

**Likely cause:** Gateway removed too early.

**Recovery:** Re-add temporarily.

This is one reason the migration keeps the old path available until VLAN 90 has been verified.

### Split Brain

If the cluster develops communication problems because traffic is leaving through competing paths, check the routing configuration.

**Likely cause:** Multiple gateways.

**Recovery:** Single default route.

The management network should provide the predictable routed path, while VLAN 99 remains the dedicated cluster network.



## Final State Verification

The migration is not complete simply because the new interfaces exist.

Verify the actual running state.

Run:

```bash id="f7w2q9"
pvecm status
ip route
```

The cluster status should show the expected nodes and healthy quorum.

The routing table should show the expected management route without competing default gateways.

Confirm:

* Corosync on VLAN 99 only
* Management via VLAN 90
* No native VLAN usage

The final architecture should therefore look like:

```text
                    Proxmox Cluster
                         |
          +--------------+--------------+
          |              |              |
        VLAN 90        VLAN 99        VLAN 200+
       Management      Corosync           VMs
          |              |              |
       Routed          L2-only         Routed
       + FW             No GW           + FW
```

The native VLAN is no longer part of the Proxmox network design.

Management is handled through VLAN 90.

Cluster communication is handled through VLAN 99.

VM workloads remain on VLAN 200+ networks.


## Closing Thoughts

Migrating a live Proxmox cluster is **not configuration work** — it is **change management**.

The technical configuration is only one part of the problem.

The more important part is sequencing the changes so that each dependency is established before the previous dependency is removed.

The rules are simple:

* Observe before changing
* One node at a time
* Management first
* Cluster last

Part 1 showed how to remove a single Proxmox host from the native VLAN.

Part 2 showed how to build the same separation into a fresh three-node cluster from the beginning.

Part 3 addresses the reality that existing environments do not always have the opportunity to start cleanly.

With careful sequencing, an existing cluster can be moved toward the same network boundaries without treating the migration as a single large change.

The end goal is the same architecture established throughout this series:

```text
Management
    ↓
VLAN 90

Cluster / Corosync
    ↓
VLAN 99

Virtual Machines
    ↓
VLAN 200+
```

The difference is that, in an existing environment, getting there requires controlled changes rather than a clean installation.

You now have:

* A secure single node
* A clean new cluster design
* A path for migrating an existing cluster
