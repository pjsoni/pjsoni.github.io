---

layout: post
title: "Proxmox Network Isolation Series – Part 2: Fresh Three-Node Cluster"
date: 2026-01-08
categories: [homelab, proxmox, security, networking]
tags: [Proxmox, VLAN, Cluster, Corosync, Firewall]
--------------------------------------------------

## Scope and Assumptions

This article builds **directly on Part 1**. Nothing introduced here replaces earlier guidance.

**Assumptions:**

* Three *fresh* Proxmox nodes
* No existing VMs
* No existing cluster
* Managed switch + stateful firewall already in place

If your cluster already exists or hosts VMs, **do not follow this guide**. That scenario is covered in **Part 3**.

---

## VLAN Architecture (Expanded)

| VLAN | Name       | Purpose                 |
| ---: | ---------- | ----------------------- |
|   90 | Management | Web UI, SSH             |
|   99 | Cluster    | Corosync (L2 preferred) |
| 200+ | VM         | Guest workloads         |

### Key Rules

* Corosync **must not traverse user or VM networks**
* Management traffic must remain routable and firewalled
* VM traffic must never touch VLAN 90 or 99

---

## Logical Topology

![Cluster Topology](/assets/proxmox-part2-topology.svg)

---

## Host Addressing

| Host | Mgmt (VLAN 90) | Cluster (VLAN 99) |
| ---- | -------------- | ----------------- |
| hv1  | 192.168.90.11  | 192.168.99.11     |
| hv2  | 192.168.90.12  | 192.168.99.12     |
| hv3  | 192.168.90.13  | 192.168.99.13     |

---

## Switch Configuration

Each Proxmox port:

* Tagged VLANs: `90, 99, 200+`
* Native VLAN: **None**

This ensures deterministic traffic separation.

---

## Firewall Responsibilities

### Critical Requirement

If your firewall cannot enforce **inter-VLAN policy**, this architecture collapses.

A "VLAN-capable" router without firewall rules provides **zero isolation**.

---

## Proxmox Network Configuration (All Nodes)

### Management Bridge

```text
auto vmbr0
iface vmbr0 inet static
    address 192.168.90.11/24
    gateway 192.168.90.1
    bridge-ports eno1.90
```

### Cluster Interface

```text
auto vmbr99
iface vmbr99 inet static
    address 192.168.99.11/24
    bridge-ports eno1.99
```

> VLAN 99 should ideally be **non-routed**.

---

## Cluster Creation

From `hv1`:

```bash
pvecm create secure-cluster --bindnet0 192.168.99.0
```

Join `hv2` and `hv3` using their **VLAN 99 addresses only**.

---

## Threat–Mitigation Matrix

| Threat                 | Mitigation                  |
| ---------------------- | --------------------------- |
| User device compromise | Mgmt VLAN isolation         |
| VM breakout            | Separate VM VLANs           |
| Corosync instability   | Dedicated cluster VLAN      |
| Lateral movement       | Firewall policy enforcement |
| Mis-tagged traffic     | No native VLAN              |

---

## Common Pitfalls

* Using management IPs for Corosync
* Allowing VLAN 99 to route
* Forgetting to firewall east–west traffic

---

![Diagram](/assets/images/proxmox-part2-topology.svg)

---

## What’s Next

**Part 3** covers the hardest scenario:

* Existing cluster
* Live VMs
* Native VLAN dependency
* Zero-downtime migration strategy

---

**End of Part 2**
