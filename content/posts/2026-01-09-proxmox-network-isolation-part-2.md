---
title: "Proxmox Network Isolation Series – Part 2: Fresh Three-Node Cluster"
date: 2026-01-09
slug: proxmox-network-isolation-series-part-2-fresh-three-node-cluster
summary: "The second part of the Proxmox Network Isolation Series covers building a new three-node cluster with network isolation designed in from the beginning. It introduces dedicated management, cluster, and VM VLANs and explains the boundaries that should exist between Proxmox management traffic, Corosync communication, and guest workloads."
series:
  - "Proxmox Network Isolation Series"
series_order: 2
categories: [Infrastructure, Security]
tags: [Firewall, Homelab, Proxmox, Security, Virtualization, VLAN]
jumbotron:
  meta: true
---

## Scope and Assumptions

In [Part 1](/posts/proxmox-network-isolation-series-part-1-securing-a-single-node-hypervisor/), we established the basic network isolation model for a single Proxmox node:

* Management traffic is placed on a dedicated management VLAN
* VM traffic is separated from the management network
* The native VLAN is removed from the Proxmox host
* Firewall policy controls access to the management plane

This article builds **directly on that design**.

The difference is that we now have multiple Proxmox nodes that need to communicate with one another. A cluster introduces another type of traffic that did not exist in the single-node scenario: **cluster communication through Corosync**.

That traffic should not be mixed with management traffic or guest workloads.

The objective is therefore to extend the isolation model from Part 1 rather than redesign it.

**Assumptions:**

* Three *fresh* Proxmox nodes
* No existing VMs
* No existing cluster
* Managed switch + stateful firewall already in place

Because the nodes are fresh, the network architecture can be established before the cluster is created.

This distinction is important.

If a cluster already exists, changing the network underneath it introduces additional considerations because the existing cluster and its VMs may already depend on the current network configuration.

If your cluster already exists or hosts VMs, **do not follow this guide**.

That scenario is covered in **Part 3**, where the existing environment and its dependencies are handled separately.

## VLAN Architecture (Expanded)

With three nodes, the network now has three distinct responsibilities:

| VLAN | Name       | Purpose                 |
| ---: | ---------- | ----------------------- |
|   90 | Management | Web UI, SSH             |
|   99 | Cluster    | Corosync (L2 preferred) |
| 200+ | VM         | Guest workloads         |

Each VLAN has a clearly defined role.

### VLAN 90 — Management

VLAN 90 continues the design established in Part 1.

It carries Proxmox management traffic such as:

* Web UI
* SSH

Management traffic needs to remain reachable from the appropriate administrative networks, but it should also remain subject to firewall policy.

The management network is therefore intentionally different from the cluster network.

### VLAN 99 — Cluster

VLAN 99 is dedicated to **Corosync**.

Unlike the management network, this network exists for communication between the Proxmox nodes themselves.

The purpose of separating it is to prevent Corosync communication from competing with user, VM, or other management traffic.

Corosync communication is particularly important to the operation of a Proxmox cluster, so giving it a dedicated network makes the traffic boundary explicit.

For this architecture, **L2 is preferred** for the cluster network.

### VLAN 200+ — VM

VLAN 200 and above are reserved for guest workloads.

These networks are separate from both:

* Proxmox management
* Cluster communication

VM traffic should therefore not become another path into either the management or cluster networks.

### Key Rules

The three networks have different purposes, and those purposes should remain separate.

* Corosync **must not traverse user or VM networks**
* Management traffic must remain routable and firewalled
* VM traffic must never touch VLAN 90 or 99

These rules define the basic security and traffic boundaries of the cluster.

Conceptually:

```text
VLAN 90
   |
   +---- Proxmox Management
         Web UI
         SSH

VLAN 99
   |
   +---- Corosync
         hv1 <----> hv2 <----> hv3

VLAN 200+
   |
   +---- Guest Workloads
```

The important point is that these are three different traffic classes even though they may ultimately travel across the same physical switching infrastructure.

## Logical Topology

The physical infrastructure can carry the different VLANs while keeping their logical purposes separate.

![Cluster Topology](/assets/proxmox-part2-topology.svg)

The topology established in Part 1 is therefore extended for multiple nodes.

Each Proxmox host needs access to the management VLAN and the cluster VLAN, while the VM networks are carried separately for guest workloads.

The result is a consistent architecture across all three nodes rather than having each node use a different network arrangement.

## Host Addressing

Each node receives an address on both VLAN 90 and VLAN 99.

| Host | Mgmt (VLAN 90) | Cluster (VLAN 99) |
| ---- | -------------- | ----------------- |
| hv1  | 192.168.90.11  | 192.168.99.11     |
| hv2  | 192.168.90.12  | 192.168.99.12     |
| hv3  | 192.168.90.13  | 192.168.99.13     |

This makes the purpose of each address explicit.

For example:

```text
hv1
  Management → 192.168.90.11
  Cluster    → 192.168.99.11

hv2
  Management → 192.168.90.12
  Cluster    → 192.168.99.12

hv3
  Management → 192.168.90.13
  Cluster    → 192.168.99.13
```

The management addresses are used for administrative access.

The cluster addresses provide the dedicated network for Corosync communication.

Keeping these addresses in separate subnets makes it immediately apparent which network is being used for which purpose.

## Switch Configuration

The switch configuration follows directly from the VLAN architecture.

Each Proxmox port should carry the VLANs required by the host:

* Tagged VLANs: `90, 99, 200+`
* Native VLAN: **None**

The Proxmox connection is therefore a trunk carrying the required tagged networks.

The important difference from a conventional access-port configuration is that there is no native VLAN being presented to the host.

This ensures deterministic traffic separation.

The host can therefore distinguish:

```text
VLAN 90  → Management
VLAN 99  → Cluster
VLAN 200+ → VM networks
```

rather than receiving an additional untagged network whose purpose is ambiguous.

Because all three nodes use the same basic switch configuration, the cluster has a consistent network foundation across its members.

## Firewall Responsibilities

The switch provides VLAN segmentation, but segmentation alone does not determine which networks are allowed to communicate.

That remains the responsibility of the firewall.

### Critical Requirement

If your firewall cannot enforce **inter-VLAN policy**, this architecture collapses.

A "VLAN-capable" router without firewall rules provides **zero isolation**.

This distinction is fundamental to the design.

For example, having VLAN 90 and VLAN 200 as separate networks does not by itself mean that a device on VLAN 200 cannot communicate with the Proxmox management interface.

If the router simply routes the traffic between the VLANs, communication can still occur.

The firewall therefore needs to enforce the intended boundaries.

The resulting model is:

```text
              Managed Switch
                    |
        +-----------+-----------+
        |           |           |
      VLAN 90     VLAN 99     VLAN 200+
        |           |           |
    Management   Corosync       VMs
        |           |
        +----- Firewall policy
```

Management remains routable because administrators need to reach it.

Cluster traffic remains isolated from user and VM networks.

VM traffic remains separate from both management and cluster communication.

## Proxmox Network Configuration (All Nodes)

The network configuration is applied consistently across all three nodes.

The addresses shown below use `hv1` as the example. The corresponding addresses for `hv2` and `hv3` come from the host-addressing table above.

### Management Bridge

The management bridge carries the Proxmox management address on VLAN 90:

```text id="5qj7sk"
auto vmbr0
iface vmbr0 inet static
    address 192.168.90.11/24
    gateway 192.168.90.1
    bridge-ports eno1.90
```

The important part is that the management network is explicitly associated with VLAN 90:

```text id="1p6r4n"
bridge-ports eno1.90
```

The management address is:

```text id="f2h5yz"
192.168.90.11/24
```

and the gateway is on the same management network:

```text id="k9s2q8"
192.168.90.1
```

This follows the management design established in Part 1.

The other nodes use their respective management addresses:

```text
hv1 → 192.168.90.11
hv2 → 192.168.90.12
hv3 → 192.168.90.13
```

### Cluster Interface

The cluster network gets its own interface:

```text id="7p6f0q"
auto vmbr99
iface vmbr99 inet static
    address 192.168.99.11/24
    bridge-ports eno1.99
```

Again, the example uses `hv1`.

The other nodes use:

```text
hv2 → 192.168.99.12
hv3 → 192.168.99.13
```

The interface is explicitly associated with VLAN 99:

```text id="y4q5jw"
bridge-ports eno1.99
```

This provides a dedicated path for the cluster network rather than using the management address for Corosync.

> VLAN 99 should ideally be **non-routed**.

The purpose of this network is communication between the Proxmox nodes. Keeping it non-routed reinforces that boundary and prevents it from becoming another general-purpose network.

The resulting network arrangement on each host is therefore:

```text
Proxmox Node
    |
    +---- VLAN 90
    |       |
    |       +---- Management
    |
    +---- VLAN 99
    |       |
    |       +---- Cluster / Corosync
    |
    +---- VLAN 200+
            |
            +---- VM Networks
```

The three traffic classes remain distinct all the way from the switch to the Proxmox configuration.

## Cluster Creation

Once the network configuration is in place on the fresh nodes, the Proxmox cluster can be created.

The cluster is created from `hv1`:

```bash id="3m9j0p"
pvecm create secure-cluster --bindnet0 192.168.99.0
```

The cluster is therefore associated with the dedicated cluster network rather than the management network.

The remaining nodes are then joined to the cluster using their **VLAN 99 addresses only**.

The addressing established earlier makes the intended communication path explicit:

```text
192.168.99.11 → hv1
192.168.99.12 → hv2
192.168.99.13 → hv3
```

The management addresses on VLAN 90 remain available for administrative access, while the cluster communication uses VLAN 99.

This separation is the central change introduced in Part 2.

Instead of having management and cluster communication share the same network, each has its own defined network boundary.

## Threat–Mitigation Matrix

The architecture can be summarized by looking at the threats it addresses.

| Threat                 | Mitigation                  |
| ---------------------- | --------------------------- |
| User device compromise | Mgmt VLAN isolation         |
| VM breakout            | Separate VM VLANs           |
| Corosync instability   | Dedicated cluster VLAN      |
| Lateral movement       | Firewall policy enforcement |
| Mis-tagged traffic     | No native VLAN              |

### User Device Compromise

User devices should not have unrestricted access to the Proxmox management plane.

Management VLAN isolation reduces the exposure of the Proxmox interfaces to those networks.

### VM Breakout

VMs operate on separate VLANs rather than sharing VLAN 90 or VLAN 99.

This keeps guest workloads separate from the Proxmox management and cluster networks.

### Corosync Instability

Corosync receives its own dedicated VLAN.

The intention is to keep cluster communication away from user and VM traffic that could otherwise interfere with the cluster network.

### Lateral Movement

Firewall policy provides the enforcement mechanism between the networks.

The VLAN structure establishes the segmentation, while firewall rules control which traffic is actually allowed between those segments.

### Mis-Tagged Traffic

The switch configuration does not provide a native VLAN to the Proxmox ports.

This removes the untagged network from the host connection and makes the expected VLAN assignment deterministic.

## Common Pitfalls

The architecture is straightforward, but several configuration mistakes can undermine the intended separation.

### Using Management IPs for Corosync

The management network and cluster network have different purposes.

Using the management addresses for Corosync defeats the separation established by VLAN 99.

The cluster should use the dedicated VLAN 99 addresses:

```text
hv1 → 192.168.99.11
hv2 → 192.168.99.12
hv3 → 192.168.99.13
```

### Allowing VLAN 99 to Route

VLAN 99 is intended to provide the cluster communication network.

It should ideally remain non-routed.

Allowing it to become a general routed network weakens the intended boundary around cluster communication.

### Forgetting to Firewall East–West Traffic

Network isolation is not complete simply because different VLANs have been created.

Traffic between networks still needs to be controlled by the firewall.

In particular, access from VM or other untrusted networks toward the management and cluster networks should not be implicitly allowed.

The firewall remains the enforcement point for the boundaries established by the VLAN architecture.

## What’s Next

Part 2 has covered the cleanest way to introduce network isolation when starting with a completely fresh three-node Proxmox cluster.

The important advantage of this scenario is that the network architecture can be established **before** the cluster exists and before VMs depend on the existing configuration.

But many real-world environments do not have that luxury.

You may already have:

* An existing cluster
* Live VMs
* A native VLAN dependency
* Existing network configuration that cannot simply be replaced

That is the scenario addressed by **Part 3**.

Part 3 covers the hardest scenario:

* Existing cluster
* Live VMs
* Native VLAN dependency
* Zero-downtime migration strategy

The objective there is no longer simply to design the network correctly from scratch. The challenge is to move an existing environment toward the isolated architecture while accounting for the systems that are already running.

