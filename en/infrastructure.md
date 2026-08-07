---
layout: default
title: Infrastructure overview
nav_order: 2
---

# Infrastructure overview

This page describes the current physical and virtual topology.  
The [infra-homelab](https://github.com/richpea1982/infra-homelab) repository (README) is the **source of truth**.

---

## Physical architecture (5 machines)

| Role                        | Machine              | Main function                                                              |
|-----------------------------|----------------------|----------------------------------------------------------------------------|
| Management plane            | **pve1**             | OPNsense, Proxmox Backup Server, automation node (Semaphore + Ansible)    |
| Storage                     | **NAS** (bare metal) | Debian + ZFS RAID-Z2 (6 × 1 TB) + MinIO (S3) + NFS exports                |
| Compute + distributed storage | **pve2, pve3, pve4** | Proxmox VE cluster + Ceph. Hosts K3s VMs, WordPress VMs and media LXCs  |

The management plane (pve1) is deliberately kept outside the compute cluster to avoid circular dependencies.

---

## Network segmentation (VLANs)

| VLAN | Subnet          | Purpose                                      |
|------|-----------------|----------------------------------------------|
| 10   | 10.0.10.0/24    | Management (Proxmox, PBS, automation, NAS)   |
| 20   | 10.0.20.0/24    | Internal services / K3s                      |
| 30   | 10.0.30.0/24    | Media (Jellyfin, etc.)                       |
| 40   | 10.0.40.0/24    | Public-facing web VMs (DMZ)                  |
| 50   | 10.0.50.0/24    | Untrusted / lab                              |

- Public exposure: **only** via Cloudflare Tunnels (no inbound ports open).
- Admin access: WireGuard / Tailscale overlay.

---

## Workload inventory

### K3s nodes (VLAN 20 – local-lvm storage)

| Name      | Proxmox host | VMID | IP            | vCPU | RAM  |
|-----------|--------------|------|---------------|------|------|
| k3s-pve2  | pve2         | 2021 | 10.0.20.21/24 | 3    | 7 GB |
| k3s-pve3  | pve3         | 2022 | 10.0.20.22/24 | 3    | 7 GB |
| k3s-pve4  | pve4         | 2023 | 10.0.20.23/24 | 3    | 6 GB |

- API endpoint (kube-vip): `10.0.20.20:6443`
- System disks on local-lvm (not Ceph) to keep etcd latency low.

### Public WordPress VMs (VLAN 40 – Ceph)

| Name           | Proxmox host | VMID | IP            | vCPU | RAM  |
|----------------|--------------|------|---------------|------|------|
| hantaweb       | pve3         | 4011 | 10.0.40.11/24 | 3    | 4 GB |
| petitsanglais  | pve4         | 4012 | 10.0.40.12/24 | 1    | 1 GB |
| hanta-assos    | pve3         | 4013 | 10.0.40.13/24 | 1    | 1 GB |

Ceph storage allows HA migration / restart if a node fails.

### Media LXC (VLAN 30)

| Name     | Proxmox host | VMID | IP            | Notes                                          |
|----------|--------------|------|---------------|------------------------------------------------|
| jellyfin | pve2         | 3010 | 10.0.30.10/24 | Privileged (GPU passthrough), library on NFS  |

Additional media / photo services are still being validated.

### Management (pve1 + NAS)

- **pve1**: OPNsense, PBS, automation node (VLAN 10)
- **NAS**: ZFS RAID-Z2 + MinIO + NFS exports (jellyfin, photoprism, seafile, backups…)

---

## Main design choices

| Choice                                   | Reason                                                              |
|------------------------------------------|---------------------------------------------------------------------|
| K3s system disks on local-lvm            | etcd is latency-sensitive; Ceph would introduce too much jitter    |
| WordPress & non-K3s stateful workloads on Ceph | Enables migration and HA restart on hardware failure          |
| Media LXCs + NFS                         | GPU passthrough + large libraries kept off Ceph                    |
| Isolated management plane                | Avoids chicken-and-egg problems                                    |
| Ansible bootstrap → ArgoCD GitOps        | Ansible for the foundation, ArgoCD for everything afterwards       |

---

## Status (August 2026)

| Component                     | Status                                      |
|-------------------------------|---------------------------------------------|
| Proxmox + Ceph + network + NAS| Daily production                            |
| WordPress VMs                 | Production                                  |
| IaC automation                | Operational                                 |
| K3s cluster                   | Bootstrapped – HA / GitOps validation ongoing |
| Stateless service migration   | Progressive                                 |

---

**Related pages:**

- [← Home](/en/)
- [IaC & automation →](/en/iac-automation.html)
- [Networking →](/en/networking.html) · [Security →](/en/security.html)
- [Services →](/en/services.html)
