---
layout: default
title: Services & workloads
nav_order: 5
---

# Services & workloads

Applications are placed according to their nature (stateful / stateless), resource needs and the most suitable storage type.

---

## Outside the cluster (Proxmox VMs / LXCs)

### WordPress VMs (VLAN 40 – Ceph)

| Name           | Host | VMID | Resources      | Role                       |
|----------------|------|------|----------------|----------------------------|
| hantaweb       | pve3 | 4011 | 3 vCPU / 4 GB  | WooCommerce e-commerce     |
| petitsanglais  | pve4 | 4012 | 1 vCPU / 1 GB  | Showcase site              |
| hanta-assos    | pve3 | 4013 | 1 vCPU / 1 GB  | Association site           |

Ceph storage allows HA migration / restart.

### Media LXC (VLAN 30 – local-lvm + NFS)

| Name     | Host | VMID | Notes                                            |
|----------|------|------|--------------------------------------------------|
| jellyfin | pve2 | 3010 | Privileged (GPU passthrough), library on NFS    |

Additional media / photo services are still being validated.

---

## Inside the K3s cluster (VLAN 20)

The 3-node cluster runs containerised services via ArgoCD (GitOps).

| Service           | Namespace   | Exposure                | Access                     | Storage        |
|-------------------|-------------|-------------------------|----------------------------|----------------|
| Cloudflare Tunnel | networking  | Outbound only           | —                          | None           |
| Traefik           | kube-system | Ingress                 | Internal / public routing  | None           |
| Prometheus        | monitoring  | Traefik (internal)      | Admin only                 | Local PVC      |
| Grafana           | monitoring  | Traefik → WG            | grafana.local.lan          | Local PVC      |
| Portfolio         | web         | CF Tunnel → Traefik     | Public domain              | PVC / assets   |
| Vaultwarden       | —           | Traefik / WG            | Internal                   | PVC            |
| Velero            | —           | —                       | Cluster backups            | S3 (MinIO)     |

Calico policies restrict pod-to-pod communication (e.g. the Cloudflare tunnel may only talk to Traefik).

---

## Routing

**Public traffic**  
Client → Cloudflare (WAF) → cloudflared tunnel → Traefik → service (e.g. portfolio)

**Administrative traffic**  
Client → WireGuard → Traefik → internal services (Grafana, etc.)

No ports are opened inbound on the public IP.

---

## Storage strategy

| Data type                     | Location                 | Reason                                       |
|-------------------------------|--------------------------|----------------------------------------------|
| K3s system disks / etcd       | local-lvm                | Minimal latency for etcd                     |
| WordPress VMs                 | Ceph                     | HA / migration on node failure               |
| Media libraries               | NAS (ZFS) via NFS        | Large volume + dual disk failure tolerance   |
| Terraform state + backups     | MinIO (S3) on NAS        | Object storage + object lock                 |
| K3s application PVCs          | local-lvm (per node)     | Simplicity and performance                   |

---

## Status (August 2026)

| Category                         | Status                          |
|----------------------------------|---------------------------------|
| WordPress VMs                    | Production                      |
| Jellyfin                         | Production                      |
| Monitoring stack (Prometheus/Grafana) | Operational                |
| Portfolio (inside K3s)           | Deployed / stabilising          |
| Other GitOps services            | Progressive migration           |

---

**Related pages:**

- [← Security](/en/security.html)
- [Backup →](/en/backup-strategy.html)
- [Lessons learned →](/en/lessons-learned.html)
