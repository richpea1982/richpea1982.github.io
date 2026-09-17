---
layout: default
title: Home
nav_order: 1
---

## From hospitality to systems administration

After roughly twenty years in hospitality and then plumbing, I moved into systems and networks.  
I am currently studying for a **TSSR (Technicien Supérieur Systèmes et Réseaux)** at L’IDEM Le Soler (expected completion January 2027).

My homelab is the daily practice ground where I work with Infrastructure as Code, virtualisation, Kubernetes and security under real hardware constraints.

---

## Homelab evolution

The current architecture was built step by step:

- **Year 1** — First contact with Linux on an old laptop (dual-boot, CLI).
- **Year 2** — First self-hosted WordPress site on Google Cloud free tier.
- **Year 4** — Move to local hardware: mini-PC running Proxmox, Cloudflare Tunnel, early containers and basic monitoring.
- **Year 5** — Second machine added, Traefik, internal services (Seafile, Jellyfin…). Full rebuild with **Terraform + Ansible**.
- **Year 5.5** — Network segmentation (managed switch, VLANs).
- **Year 6** — Consolidation: 3-node Proxmox + Ceph cluster, dedicated ZFS NAS, start of the HA K3s cluster.

---

## Current architecture (summary)

The environment is split into two clearly separated planes:

**1. Management plane (outside the compute cluster)**  

- **pve1**: OPNsense (firewall/routing), Proxmox Backup Server, automation node (Semaphore + Ansible).  
- **NAS (bare metal)**: Debian + ZFS RAID-Z2 (6 × 1 TB) + MinIO (S3) + NFS exports.

**2. Compute plane (Proxmox + Ceph cluster)**  

- **pve2, pve3, pve4**: Proxmox VE cluster with Ceph storage. Hosts the K3s nodes, public WordPress VMs and media LXCs.

**Networking**  

- VLAN segmentation (management, internal, media, DMZ, untrusted).  
- Public exposure only via **Cloudflare Tunnels** (no inbound ports open).  
- Admin access via WireGuard / Tailscale overlay.

**Kubernetes**  

- 3-node K3s cluster (embedded etcd + kube-vip).  
- Base bootstrap handled by Ansible.  
- Application deployments managed afterwards with ArgoCD (GitOps).

Full details (nodes, IPs, resources, design decisions) live in the [infra-homelab](https://github.com/richpea1982/infra-homelab) repository — the README is the source of truth.

---

## Current status (August 2026)

| Component                        | Status                                      |
|----------------------------------|---------------------------------------------|
| Proxmox + Ceph + network + NAS   | Daily production use                        |
| Public WordPress VMs             | Production                                  |
| Automation (Terraform/Ansible/Semaphore) | Operational                          |
| K3s cluster                      | Bootstrapped — HA and GitOps validation in progress |
| Migration of stateless services  | Progressive                                 |

---

## Working method

- Everything is versioned in Git. Manual changes are avoided.
- The management plane (firewall, backups, automation) is isolated from the compute plane to prevent circular dependencies.
- Technical choices are driven by real hardware constraints (disk latency, limited RAM, older CPUs).
- I use AI as a helper for repetitive configuration, but I always review and understand every block before deploying it.

---

**Next pages:**

- [Infrastructure overview →](/en/infrastructure.html)
- [IaC & automation →](/en/iac-automation.html)
- [Networking →](/en/networking.html) / [Security →](/en/security.html)
- [Services →](/en/services.html)
