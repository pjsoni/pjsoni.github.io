---
title: "Proxmox Network Isolation Series – Part 1: Securing a Single-Node Hypervisor"
date: 2026-01-08
slug: proxmox-network-isolation-series-part-1-securing-a-single-node-hypervisor 
summary: "The first part of the Proxmox Network Isolation Series focuses on securing a single Proxmox hypervisor by removing management traffic from the native VLAN. It covers management isolation, VLAN-aware VM networking, and firewall-enforced boundaries without introducing the additional complexity of clustering or Corosync."
series:
  - "Proxmox Network Isolation Series"
series_order: 1
categories: [Infrastructure, Security]
tags: [Firewall, Homelab, Proxmox, Security, Virtualization, VLAN]
jumbotron:
  meta: true
---

## Introduction

This article is **Part 1 of a multi-part series** focused on hardening Proxmox by removing the hypervisor from the native VLAN and enforcing clear network isolation boundaries.

The series starts with the simplest scenario: a **single-node Proxmox hypervisor**.

A single node does not have the complexity of a cluster, but it still deserves the same basic security discipline. A Proxmox host is not simply another server on the network. It controls virtual machines, provides access to storage, and can have access to backup infrastructure and other sensitive systems.

If the hypervisor is placed on the same network as ordinary user devices, IoT devices, or guest systems, compromising one of those systems can potentially provide a path toward the virtualization host.

The objective of this first part is therefore straightforward:

* Remove Proxmox management traffic from the native VLAN
* Put Proxmox management on a dedicated VLAN
* Keep virtual machine networking separate from the management network
* Use firewall policy to control which systems can reach the Proxmox management interface
* Keep the design simple enough that it can be established safely on a single node

This article focuses exclusively on:

* One Proxmox node
* Management traffic isolation
* VLAN-aware VM networking
* Firewall-enforced security boundaries

> No clustering, Corosync, or migration complexity is introduced in Part 1.

The intention is to establish the network isolation model first. Later parts of the series build on this foundation when additional Proxmox nodes and cluster communication are introduced.

## Threat Model (Why This Matters)

Without network isolation, a single-node Proxmox host is typically exposed to several different types of systems on the network:

* User endpoints
* IoT devices
* Guest or lab networks

These networks have very different trust levels.

A workstation may be reasonably trusted, while an IoT device or guest system may have considerably less trust. If all of these systems can communicate directly with the Proxmox management interface, the hypervisor is exposed to every system that can reach that network.

That creates an unnecessarily large attack surface.

The objective is not to make the Proxmox host invisible to the network. Administrators still need to manage it. Instead, management access should come from a clearly defined and trusted network.

The design therefore establishes three important boundaries:

* The **Proxmox management plane** is reachable only from trusted admin networks
* **VM traffic** is kept separate from the hypervisor management network
* The **native VLAN** is completely removed from the host

This distinction is important because simply creating VLANs does not automatically make the environment secure. The network must also enforce which systems are allowed to communicate with the management network.

## Prerequisites (Non-Negotiable)

Before changing the Proxmox network configuration, the network infrastructure must be capable of supporting the design.

VLANs provide logical separation, but VLANs alone do **not** provide security policy.

Two components are therefore required:

1. A managed switch to provide VLAN segmentation and trunking
2. A stateful firewall/router to control communication between those networks

### Mandatory Components

The environment requires:

* **Managed switch** with VLAN tagging and trunking
* **Stateful firewall/router** capable of:

  * Inter-VLAN routing
  * Explicit allow/deny rules
* Console or out-of-band access to the Proxmox host

The console or out-of-band access is particularly important when changing the network configuration of a remote hypervisor. A network configuration mistake can disconnect the management interface, so having another way to access the host provides a recovery path.

> ⚠️ If your router can route VLANs but cannot enforce firewall rules, this architecture provides **no real security benefit**.

Inter-VLAN routing determines that two VLANs can communicate. The firewall determines whether that communication should actually be permitted.

### Roles and Responsibilities

Each component has a specific responsibility:

* **Switch** → Enforces segmentation
* **Firewall** → Enforces security

The switch determines which VLAN traffic reaches the Proxmox host.

The firewall determines which networks and hosts are allowed to communicate with the Proxmox management network.

Both are required.

## VLAN Design (Single-Node Only)

For a single-node hypervisor, the VLAN model is intentionally minimal.

There are only two categories required for this design:

| VLAN ID | Name        | Purpose                |
| ------: | ----------- | ---------------------- |
|      90 | Management  | Proxmox Web UI, SSH    |
|    200+ | VM Networks | Guest virtual machines |

VLAN 90 is dedicated to the Proxmox management plane.

This is where the Proxmox Web UI and SSH access reside.

VLAN 200 and above are used for virtual machine networks. These networks are intentionally kept separate from the Proxmox management network.

### Design Principles

The design follows three simple rules:

* Proxmox management traffic must live on a **dedicated management VLAN**
* VM networks must never share the management VLAN
* The native (untagged) VLAN must not touch the host

The third point is especially important.

The objective is not simply to assign Proxmox an address on VLAN 90. The objective is to ensure that the physical connection to the host does not provide an additional untagged path to the host.

The resulting model is therefore:

```text
VLAN 90
   |
   +---- Proxmox Management
   |
   +---- Web UI / SSH

VLAN 200+
   |
   +---- Virtual Machines

Native VLAN
   |
   +---- Not connected to the Proxmox host
```

This keeps the management and VM networks separate from the beginning.

## Example Environment

For the examples in this article, the Proxmox host is:

| Hostname | Role         | Management IP |
| -------- | ------------ | ------------- |
| hv1      | Proxmox Node | 192.168.90.11 |

The management address belongs to VLAN 90.

### Logical Topology

The resulting network looks like this:

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
```

The firewall provides routing and policy enforcement.

The managed switch carries the required VLANs to the Proxmox host.

The Proxmox host receives management traffic through VLAN 90, while virtual machines use the VM VLANs.

The native VLAN is intentionally absent from the host connection.

## Step 1: Switch Configuration

The first step is to configure the switch port connected to the Proxmox host as a **trunk**.

The trunk allows the same physical connection to carry multiple VLANs.

For this design:

* Allowed VLANs: `90, 200+`
* Native VLAN: **None** (or unused VLAN)

A conceptual configuration looks like this:

```text
Proxmox Port:
  Tagged VLANs: 90, 200
  Native VLAN: Disabled
```

The important part is that VLAN 90 and the VM VLANs are carried as tagged traffic.

There should not be a useful untagged/native network presented to the Proxmox host.

This ensures the host never sees untagged traffic from the native VLAN.

The physical connection can therefore carry multiple logical networks without placing the Proxmox host directly onto the native network.

## Step 2: Firewall Configuration

The firewall is responsible for controlling access to the management VLAN.

Create a VLAN interface for **VLAN 90 (Management)** and assign it a gateway IP.

For example:

```text
VLAN 90 gateway:
192.168.90.1
```

The Proxmox host will use this network for management:

```text
Proxmox:
192.168.90.11/24
```

The firewall should also have VLAN interfaces for the VM networks as required by the environment.

### Minimum Firewall Rules

The minimum management rules are:

* Allow HTTPS (8006) to VLAN 90 **only** from admin networks or hosts
* Allow SSH only from trusted IPs
* Deny all other inbound access to VLAN 90

The Proxmox Web UI uses TCP port `8006`, so access to that port should be restricted to the systems that actually need to administer Proxmox.

SSH should follow the same principle.

The important point is that putting Proxmox on a management VLAN does not mean every system on that VLAN should automatically have access to it. Firewall policy still determines who can reach the management services.

> If a user VLAN can reach the Proxmox UI, the design has failed.

The management VLAN is therefore not intended to be an unrestricted network. It is the network on which the Proxmox management interface resides, with access controlled by firewall policy.

## Step 3: Proxmox Network Configuration

With the switch and firewall prepared, configure the Proxmox host.

Edit:

```text
/etc/network/interfaces
```

The management configuration is:

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

There are two important details in this configuration.

First, the physical interface `eno1` itself does not receive the management address.

Instead, the management bridge uses:

```text
bridge-ports eno1.90
```

This means the management path is associated with VLAN 90.

Second, the management bridge has the host's IP address and default gateway:

```text
address 192.168.90.11/24
gateway 192.168.90.1
```

This bridge is **management only**.

It is not being used as the bridge for the virtual machines.

That separation is intentional and provides a clear distinction between the network used to administer Proxmox and the networks used by the guests.

## Step 4: VLAN-Aware VM Bridge

Virtual machines need their own network path.

Create a dedicated bridge for virtual machines:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno1
    bridge-vlan-aware yes
    bridge-stp off
    bridge-fd 0
```

Unlike the management bridge, this bridge does not have an IP address assigned to the Proxmox host.

It is a VLAN-aware bridge intended to carry the VM networks.

Attach VM NICs to `vmbr1` and assign VLAN tags, such as:

```text
VLAN 200
```

The important separation is:

```text
vmbr0
  |
  +---- Proxmox management
        VLAN 90

vmbr1
  |
  +---- Virtual machines
        VLAN 200+
```

The Proxmox management interface therefore does not need to share the VM bridge.

This also means that a VM's network connection does not automatically become a connection to the Proxmox management interface.

## Step 5: Validation

Network isolation should always be validated rather than assumed.

From the Proxmox host, check the interfaces:

```bash
ip a
```

Then check the routing table:

```bash
ip route
```

Confirm the following:

* The Proxmox management address is on VLAN 90
* The default route exists only on VLAN 90
* No interface is attached to an untagged VLAN

The default route is particularly important because the management network is the network through which the Proxmox host reaches destinations outside its local subnet.

The VM bridge itself does not need to become another management path for the host.

Then test the design from an untrusted VLAN.

The expected result is:

* Proxmox UI should be unreachable

This test validates the firewall boundary from the perspective that matters: a system that should **not** have access to the Proxmox management plane must not be able to reach it.

## Firewall Rule Examples

The exact firewall implementation depends on the platform, but the policy remains the same.

### pfSense / OPNsense

For **VLAN 90 (Management)**:

* Allow TCP 8006 from admin subnet
* Allow SSH from admin IPs
* Block all other inbound traffic
* Log activities (Optional but important)

The management network should therefore allow the required administrative access while denying access from networks that should not be able to manage the hypervisor.

For **VLAN 99 (Cluster)**:

* Block all routed traffic
* No default gateway preferred

VLAN 99 is not required by the single-node configuration itself, but the firewall example establishes the policy that will be relevant when the architecture is extended later in the series.

For **VLAN 200+ (VMs)**:

* Explicit allow rules only

The VM networks should similarly be controlled through explicit firewall policy rather than being treated as trusted simply because they are internal VLANs.

### Ubiquiti (Conceptual)

For Ubiquiti environments, the same policy can be expressed conceptually as:

* Create LAN-IN rules denying access to VLAN 90
* Permit admin subnet → VLAN 90
* Block VLAN 200+ → VLAN 90/99

The exact rule implementation depends on the firewall platform, but the intended boundary remains consistent:

```text
Trusted Admin Network
        |
        | allowed
        v
     VLAN 90
        |
        X
Untrusted Networks
```

The firewall is what turns the VLAN separation into an enforced security boundary.

## Common Pitfalls (Single Node)

### Leaving the Native VLAN Enabled

One of the easiest mistakes is to configure the required VLANs while leaving a native or untagged VLAN enabled on the Proxmox switch port.

That creates an additional network path to the host.

Even one untagged interface can expose the host.

**Fix:** Remove native VLAN access entirely.

The objective is for the Proxmox host to receive only the VLAN traffic that has been intentionally configured for it.

### Treating VLANs as Security Controls

A VLAN provides segmentation, but segmentation alone does not determine whether two networks are allowed to communicate.

If the firewall routes between VLANs without appropriate access controls, systems on one VLAN may still be able to reach systems on another VLAN.

VLANs without firewall rules are therefore not security boundaries.

**Fix:** Enforce policy at the firewall.

The switch provides the segmentation; the firewall determines which communication is permitted.

### Mixing VM and Management Traffic

Another common mistake is using the same bridge for both Proxmox management and virtual machine networking.

This increases the blast radius if a VM or another system on the VM network becomes compromised.

The management plane should have its own bridge, while virtual machines should use the VLAN-aware VM bridge.

**Fix:** Separate management and VM bridges.

The result is the two-bridge design established in this article:

```text
vmbr0
  |
  +---- Management
        VLAN 90

vmbr1
  |
  +---- VM Networks
        VLAN 200+
```

## What Comes Next

Part 1 establishes the network isolation model on a **single Proxmox node**.

In **Part 2**, the same design is extended to a **fresh three-node Proxmox cluster**.

The next part introduces the additional network requirements that appear when multiple Proxmox nodes need to communicate:

* Dedicated cluster networking
* Corosync isolation
* Quorum-safe design

The important point is that the basic management and VM separation established here does not need to be discarded when the environment grows.

Instead, the cluster architecture builds on the same isolation principles.

## Closing Thoughts

A single-node hypervisor deserves the same security discipline as a cluster.

The fact that a Proxmox installation does not yet have multiple nodes does not make its management interface any less important. The host still controls virtual machines, provides access to storage, and can hold credentials or access to other infrastructure.

By removing the native VLAN from the host, placing management on VLAN 90, separating VM networking onto VLAN 200+, and enforcing access through the firewall, the network has clear and understandable boundaries.

The result is a Proxmox host that is easier to reason about and provides a clean foundation for the cluster architecture introduced in the following parts of this series.
