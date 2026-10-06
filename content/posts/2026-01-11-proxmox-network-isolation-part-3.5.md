---
title: "Proxmox Network Isolation Series – Part 3.5: Recovering From Broken Quorum Without Losing VMs"
date: 2026-01-11
slug: proxmox-network-isolation-series-part-3-5-recovering-from-broken-quorum-without-losing-vms
summary: "What to do when a Proxmox cluster has unstable quorum but the VMs and their data still need to be preserved. This guide takes a controlled approach: freeze cluster operations, establish temporary single-node quorum, safely remove Corosync, validate VM and storage integrity, and convert the hosts to standalone nodes before rebuilding with a proper network design."
series:
  - "Proxmox Network Isolation Series"
series_order: 3.5
categories: [Infrastructure, Security]
tags: [Firewall, Homelab, Proxmox, Security, Virtualization, VLAN]
jumbotron:
  meta: true
---

## Introduction

This article is a **direct follow-up to Part 3** of the Proxmox Network Isolation Series.

In Part 3, we established an important rule:

> **If quorum is unstable, stop.**

That rule exists because network migration and Corosync changes are fundamentally different when the cluster itself cannot maintain a stable view of its members.

This article covers what to do when you reach that situation.

The scenario is:

* A Proxmox cluster already exists
* Quorum is **broken, unstable, or flapping**
* The network design is mixed, incorrect, or difficult to reason about
* VMs are **running and their data must be preserved**
* Continuing the network migration is no longer safe

The objective here is **not to repair the cluster in place**.

Instead, the objective is to regain administrative control, deliberately remove the broken cluster dependency, and return the hypervisors to a known state.

The desired outcome is:

> **Preserve the VMs and their storage, dismantle the broken cluster, and rebuild the cluster later using the hardened network design from Parts 1 and 2.**

This is a recovery procedure, not a normal cluster-management procedure.

## What This Article Does — and Does Not — Do

### This Article Does

This guide provides a controlled path to:

* Preserve existing VM configurations and disks
* Stop normal cluster operations while the environment is unstable
* Temporarily make a node quorate when required for administrative recovery
* Remove Corosync and the existing cluster configuration
* Validate that VM and storage configuration remains available
* Convert the affected hypervisors into standalone Proxmox nodes
* Prepare the environment for a clean network redesign and cluster rebuild

### This Article Does NOT

This procedure does not attempt to:

* Guarantee zero downtime
* Repair every possible Corosync failure
* Automatically recover a damaged storage configuration
* Hide the risks involved in dismantling a cluster
* Continue changing VLANs while quorum is unstable

A broken cluster is already an unsafe operating condition.

The goal is therefore **controlled disassembly**, not increasingly complicated attempts to keep an unhealthy cluster alive.

> When the cluster cannot provide a reliable control plane, reducing complexity is often safer than adding more changes.

## Why Broken Quorum Must Be Addressed First

Proxmox uses quorum to maintain a consistent cluster state. When quorum is lost, cluster state changes are restricted because allowing independent nodes to modify shared cluster state can create conflicting views of the environment.

That becomes particularly dangerous during network migration.

For example, if you simultaneously:

* Change management VLANs
* Change default gateways
* Change Corosync addresses
* Modify firewall rules
* Restart cluster services

you can no longer easily determine whether a failure is caused by:

* the network,
* Corosync,
* quorum,
* name resolution,
* firewall policy,
* or the cluster configuration itself.

That is why the recovery process starts by **stopping the change process**.

If quorum is unstable:

* **Do not continue the VLAN migration**
* **Do not rebind Corosync**
* **Do not remove gateways**
* **Do not migrate VMs between nodes**
* **Do not make several cluster changes simultaneously**

First regain control of the environment.

## High-Level Recovery Strategy

The recovery follows five broad stages:

1. **Freeze the environment**
2. **Establish temporary administrative control**
3. **Verify VM and storage integrity**
4. **Remove the broken cluster configuration**
5. **Operate the hosts independently**

Only after these steps are complete should network redesign and cluster rebuilding begin.

The important distinction is that this process separates **recovery** from **redesign**.

We are not trying to solve the original network problem while the cluster is already unstable.

## Phase 0: Freeze the Environment

Before changing anything, stop normal cluster operations.

The objective is to prevent the environment from changing while you are trying to recover it.

Avoid:

* VM migrations
* HA-driven movement
* Scheduled backups that depend on cluster state
* Cluster configuration changes
* Network changes

If HA is enabled, stop the HA services while performing the recovery:

```bash
systemctl stop pve-ha-lrm
systemctl stop pve-ha-crm
```

The exact state of HA should be checked before proceeding.

The important point is that **HA should not be making additional decisions while the cluster is being dismantled**.

### Why Freeze First?

A recovery operation is much easier to reason about when the environment is static.

You want to know:

> "This VM was running here before I started."

rather than:

> "This VM was here, then HA moved it, then Corosync lost quorum, then the network changed."

The first situation is recoverable and understandable.

The second quickly becomes difficult to reconstruct.


## Phase 1: Identify the Most Stable Node

Before forcing quorum or removing cluster configuration, identify the node that gives you the best chance of maintaining administrative control.

Prefer the node that:

* Has reliable console access
* Can consistently access its local storage
* Has the most important or largest number of currently running VMs
* Has stable local networking
* Is currently responding normally to Proxmox commands

This node becomes the temporary **administrative authority** for the recovery process.

That does not mean it becomes the permanent cluster leader.

It simply gives you one known-good place from which to verify the state of the environment.

## Phase 2: Temporarily Establish Quorum

If the cluster has lost quorum and you need to make cluster configuration changes, Proxmox provides:

```bash
pvecm expected 1
```

This temporarily changes the expected vote count to one.

The important distinction is that this command **does not repair Corosync** and does not make the underlying network healthy.

It simply changes the quorum expectation so that the node can become quorate for administrative recovery.

Proxmox documents this as a workaround for situations where a cluster has lost quorum and you need to perform administrative operations.

> ⚠️ **This is a temporary emergency state, not a cluster repair.**

Do not interpret successful execution as proof that the cluster is healthy.

The network may still be broken.

Corosync may still be unstable.

Other nodes may still have conflicting views of cluster state.

The purpose of this step is simply to regain enough control to safely dismantle the existing cluster configuration.

## Phase 3: Verify VM and Storage Integrity

Before removing cluster configuration, make sure the node actually has access to the workloads and storage you intend to preserve.

Run:

```bash
qm list
pvesm status
```

Confirm that:

* Expected VMs are visible
* Running VMs are still running
* VM disks are accessible
* Storage is online
* Storage is not unexpectedly read-only
* Local VM configuration is available

If storage is unavailable, **stop here**.

Do not proceed with cluster dismantling while you are still trying to determine whether the underlying VM data is accessible.

### Pay Particular Attention to Shared Storage

This is one of the most important considerations when converting cluster nodes into standalone systems.

A Proxmox cluster may have shared storage configured between nodes.

After the cluster is dismantled, those nodes no longer have the same cluster coordination and locking semantics.

Proxmox specifically warns that a node separated from a cluster can still have access to shared storage, and that the same shared storage should not simply be reused across separate clusters. Storage locking does not work across cluster boundaries and VMID conflicts can occur.

Therefore:

> **Do not assume that shared storage is automatically safe just because the VM disks are intact.**

Before rebuilding the cluster, shared storage must be deliberately reviewed and separated appropriately.

For this recovery procedure, the priority is simply to establish:

**the VM configuration exists and the underlying storage is accessible.**

## Phase 4: Remove Corosync and the Cluster Configuration

This is the point where we deliberately stop treating the affected node as part of the existing cluster.

Proxmox's documented procedure for separating a node without reinstalling is more deliberate than simply deleting Corosync files. It first stops the cluster services, runs `pmxcfs` in local mode, removes the cluster configuration, and then starts the cluster filesystem again normally.

Perform this **one node at a time**.

### Step 4.1: Stop Cluster Services

On the node being separated:

```bash
systemctl stop pve-cluster
systemctl stop corosync
```

Stopping `pve-cluster` is important because `/etc/pve` is backed by the Proxmox Cluster File System (`pmxcfs`).

### Step 4.2: Start the Cluster Filesystem in Local Mode

Run:

```bash
pmxcfs -l
```

The `-l` option starts `pmxcfs` in local mode.

This gives the node local access to its Proxmox configuration without requiring an operational Corosync cluster.

At this point, you are deliberately transitioning from:

```text
Clustered configuration
        ↓
Local configuration
```


### Step 4.3: Remove Corosync Configuration

Remove the cluster Corosync configuration:

```bash
rm /etc/pve/corosync.conf
rm -r /etc/corosync/*
```

This removes the configuration that defines the node's participation in the existing Corosync cluster.

Be extremely careful with paths at this stage.

The objective is to remove **cluster configuration**, not VM storage.

### Step 4.4: Restart the Cluster Filesystem

Stop the temporary local `pmxcfs` process:

```bash
killall pmxcfs
```

Then start the normal Proxmox cluster filesystem service:

```bash
systemctl start pve-cluster
```

At this point, the node should operate using its local Proxmox configuration rather than the previous Corosync cluster.

Proxmox documents this sequence specifically for separating a node from a cluster without reinstalling it.


### What About `/var/lib/corosync`?

The original recovery procedure included:

```bash
rm -rf /var/lib/corosync/*
```

Do not treat removal of that directory as the primary mechanism for separating the node.

The important configuration that must be removed is the cluster configuration under:

```text
/etc/pve/corosync.conf
/etc/corosync/
```

The documented separation procedure does not require blindly deleting arbitrary Corosync state before the node has been cleanly moved into local `pmxcfs` mode.

As with any recovery procedure, avoid deleting additional files simply because they appear related to Corosync.


## Phase 5: Restart Proxmox Services

Once the node has been separated, restart the normal Proxmox services as necessary:

```bash
systemctl restart pvedaemon
systemctl restart pveproxy
```

The management interface should now represent the node as a standalone Proxmox system.

You should no longer be managing the node through the old cluster configuration.

The important distinction is:

```text
Before:

        Broken Cluster
        ┌──────┬──────┬──────┐
       hv1    hv2    hv3
        └──────┴──────┴──────┘
              Corosync

After:

        Standalone       Standalone       Standalone
        ┌──────┐         ┌──────┐         ┌──────┐
        │ hv1  │         │ hv2  │         │ hv3  │
        └──────┘         └──────┘         └──────┘
```

The VM data is not being migrated as part of this operation.

The cluster control plane is being removed.

## Phase 6: Validate Standalone Operation

After the node has been separated, verify its state.

Run:

```bash
pvecm status
```

The node should no longer report itself as a member of the old cluster.

Then verify the workloads:

```bash
qm list
```

And verify storage:

```bash
pvesm status
```

Confirm that:

* VMs are visible
* VM configuration is present
* VM disks are accessible
* Storage is available
* VMs can be started and stopped normally
* The Web UI operates normally
* The host no longer depends on the broken Corosync configuration

Only after the first node is confirmed to be stable should you proceed to the next node.

## Phase 7: Repeat Carefully on the Remaining Nodes

The same separation process can then be performed on the remaining hosts.

The key word is:

> **One at a time.**

Do not dismantle every node simultaneously.

For each host:

1. Establish console or reliable administrative access
2. Verify its VM and storage state
3. Stop cluster services
4. Start `pmxcfs` in local mode
5. Remove the Corosync configuration
6. Restart `pve-cluster`
7. Restart the management services as necessary
8. Verify VMs and storage
9. Confirm the node operates independently

This gives you a checkpoint after every node.

If something unexpected happens, you still have the other hosts available for comparison and troubleshooting.

## Why This Works — and Why the VMs Survive

There is an important distinction between **cluster coordination** and **VM data**.

Proxmox uses `pmxcfs` as its cluster filesystem, and VM configuration is stored within the Proxmox configuration hierarchy.

The cluster configuration and Corosync provide coordination between nodes.

The VM's virtual disk, however, is stored on the configured storage.

Therefore, removing Corosync does not inherently mean deleting the VM's disk.

The recovery process is intentionally separating:

```text
Cluster coordination
        ↓
Corosync
pmxcfs cluster state
HA coordination
```

from:

```text
Workload data
        ↓
VM configuration
VM disks
Storage
```

That separation is what makes this recovery strategy possible.

However, this should **not** be interpreted as "cluster removal can never affect a VM."

Shared storage, storage configuration, VMID conflicts, and HA state still need to be considered. Proxmox explicitly warns about shared-storage issues when separating a node from a cluster.

The correct mindset is:

> **Removing Corosync does not delete VM data, but you still have to validate the configuration and storage around those VMs.**


## Common Mistakes During Quorum Recovery

### Treating `pvecm expected 1` as a Quorum Fix

This is one of the easiest mistakes to make.

```bash
pvecm expected 1
```

does not repair:

* Corosync networking
* VLAN configuration
* Firewall policy
* Name resolution
* Multiple gateways
* Packet loss

It only changes the expected vote count.

Use it as a temporary administrative measure, not as a permanent configuration.


### Removing Corosync Before Establishing Administrative Control

If the cluster is already non-quorate, immediately deleting cluster configuration without understanding the current state can make recovery harder.

First establish:

* Which node you are working on
* Which VMs are running there
* Which storage is available
* Whether you have reliable console access
* Whether you need temporary quorum to make the required configuration changes

Then dismantle the cluster deliberately.


### Attempting Network Migration During Recovery

This is the biggest temptation.

You may be thinking:

> "The problem is the network, so let's fix the VLANs while we are here."

Do not combine the two operations.

A broken cluster plus a network migration creates too many variables.

Instead:

```text
Stabilize
    ↓
Dismantle
    ↓
Validate
    ↓
Redesign
    ↓
Rebuild
```

This is much easier to reason about than:

```text
Broken quorum
    +
VLAN migration
    +
Corosync changes
    +
Firewall changes
    =
Unknown state
```

### Leaving HA Running

HA depends on cluster state and quorum.

If the objective is to dismantle the cluster, HA should not continue making workload-management decisions during that process.

Disable or stop HA-related activity before beginning the separation procedure.


### Ignoring Shared Storage

This is particularly dangerous after the cluster has been dismantled.

A storage target that was previously shared by all cluster nodes may still be accessible from multiple independent nodes.

Once those nodes are no longer members of the same cluster, the assumptions that previously coordinated access no longer apply.

Proxmox explicitly warns that the same shared storage should not be accessed by separate clusters because storage locking does not work across the cluster boundary.

Therefore, shared storage must be reviewed before rebuilding the cluster.


## Why Not Just "Fix Quorum?"

This is a fair question.

If the underlying problem is a bad Corosync configuration, why not simply repair it?

Sometimes that is exactly the right answer.

For example, if:

* The VLAN architecture is already correct
* Cluster traffic is properly isolated
* Gateways are unambiguous
* Name resolution is deterministic
* Corosync has a well-defined network path
* The failure can be reproduced and understood

then repairing the existing cluster can make sense.

The problem is when quorum failure is only one symptom of a larger network problem.


### The Core Problem

A cluster may be unstable because of:

* Mixed or ambiguous VLAN usage
* Multiple default gateways
* Routed cluster traffic
* DNS or `/etc/hosts` ambiguity
* Incorrect Corosync addresses
* Unstable network paths

In that situation, fixing the quorum symptom without correcting the network foundation leaves the underlying problem in place.

The result can be temporary stability followed by another failure when the same condition occurs again.


### Why In-Place Repair Can Be Risky

Attempting several changes simultaneously on an unstable cluster can produce failures such as:

* Loss of cluster communication
* Unexpected HA behavior
* Conflicting cluster state
* Loss of administrative access
* Repeated quorum transitions

The particularly dangerous part is that a change can appear successful initially and fail later when another network event occurs.

That makes troubleshooting much harder.

## The Safer Philosophy: Control First, Optimization Later

This series deliberately follows a conservative sequence.

### Regain Control

If necessary, temporarily establish quorum so administrative recovery can proceed.

### Remove Undefined Cluster Behavior

If the existing Corosync environment cannot be trusted, remove the broken cluster dependency rather than continuing to build on it.

### Validate the Workloads

Make sure the VMs, configurations, and storage are intact before introducing another major change.

### Redesign the Network

Apply the principles established in the earlier parts:

* Dedicated management VLAN
* Dedicated cluster VLAN
* Isolated VM networks
* No unnecessary native VLAN dependency
* Firewall-enforced management boundaries

### Rebuild the Cluster

Once the individual hosts are stable and the network is correct, rebuild the cluster using the clean architecture from Part 2.

This approach may take longer than attempting to repair everything in place.

The advantage is that each stage has a clear state and a clear purpose.

## When Does It Make Sense to Repair Quorum Directly?

Direct quorum repair can be reasonable when the cluster's underlying network architecture is already sound.

You should be able to establish that:

* Cluster traffic is isolated
* Network paths are stable
* Gateways are unambiguous
* Name resolution is deterministic
* Corosync addresses are correct
* The failure is understood well enough to correct it safely

If those conditions are not true, continuing to repair quorum may simply preserve a fragile design.

In that situation, returning the hosts to a known standalone state provides a cleaner starting point.


## Transition to the Secure Rebuild

At the end of this recovery process, the environment should look fundamentally different from where we started.

Instead of:

```text
Broken Cluster
      │
      ├── Corosync
      ├── Unstable quorum
      ├── Mixed networking
      └── Running workloads
```

you should have:

```text
Standalone Node       Standalone Node       Standalone Node
      │                     │                     │
      ├── VMs              ├── VMs              ├── VMs
      └── Storage          └── Storage          └── Storage
```

The important achievement is not that the cluster exists.

It is that **the workloads are back under administrative control without requiring the broken cluster to remain operational**.

From here, the network can be redesigned without simultaneously trying to rescue Corosync.


## What Comes Next

Once the hosts are operating independently:

### Apply Part 1

Use the single-node design to establish:

* Dedicated management networking
* VLAN-aware VM networking
* Firewall-enforced management access
* Removal of unnecessary native VLAN dependency

### Then Apply Part 2

Once the individual nodes have the correct network foundation, rebuild the three-node cluster using:

* Dedicated management VLAN
* Dedicated cluster VLAN
* Isolated VM networks
* Controlled Corosync communication

### Part 3

If you encounter another existing cluster that is still healthy enough to migrate, Part 3 covers the controlled migration process without first dismantling the cluster.



## Final Thoughts

A broken cluster is not necessarily a data-loss event.

The most important distinction is between **cluster state** and **workload data**.

When the cluster control plane becomes unreliable, the safest response is not necessarily to keep adding changes until quorum returns.

Sometimes the safer approach is to:

* Stop
* Freeze the environment
* Regain administrative control
* Verify the VMs and storage
* Remove the broken cluster configuration
* Validate each standalone node
* Redesign the network
* Rebuild the cluster cleanly

The goal is not to preserve the broken cluster at all costs.

The goal is to preserve the **workloads**, regain control of the infrastructure, and create a clean foundation for the next cluster.

> **Stabilize first. Dismantle deliberately. Rebuild correctly.**

This completes the recovery path for the network-isolation series.
