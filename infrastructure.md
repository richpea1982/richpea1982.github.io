---
layout: default
title: Vue d'ensemble de l'infrastructure
nav_order: 2
---

# Vue d'ensemble de l'infrastructure

Cette page décrit la topologie physique et virtuelle actuelle.  
Le dépôt [infra-homelab](https://github.com/richpea1982/infra-homelab) (README) est la **source de vérité**.

---

## Architecture physique (5 machines)

| Rôle                        | Machine              | Fonction principale                                                                 |
|-----------------------------|----------------------|-------------------------------------------------------------------------------------|
| Plan de gestion             | **pve1**             | OPNsense, Proxmox Backup Server, nœud d’automatisation (Semaphore + Ansible)       |
| Stockage                    | **NAS** (bare-metal) | Debian + ZFS RAID-Z2 (6 × 1 To) + MinIO (S3) + exports NFS                          |
| Calcul + stockage distribué | **pve2, pve3, pve4** | Cluster Proxmox VE + Ceph. Héberge les VMs K3s, WordPress et LXC média              |

Le plan de gestion (pve1) est volontairement isolé du cluster de calcul pour éviter les dépendances circulaires.

---

## Segmentation réseau (VLANs)

| VLAN | Sous-réseau     | Usage                                      |
|------|-----------------|--------------------------------------------|
| 10   | 10.0.10.0/24    | Management (Proxmox, PBS, automation, NAS) |
| 20   | 10.0.20.0/24    | Services internes / K3s                    |
| 30   | 10.0.30.0/24    | Média (Jellyfin, etc.)                     |
| 40   | 10.0.40.0/24    | VMs web publiques (DMZ)                    |
| 50   | 10.0.50.0/24    | Untrusted / lab                            |

- Exposition publique : **uniquement** via Cloudflare Tunnels (aucun port ouvert en entrée).
- Accès d’administration : overlay WireGuard / Tailscale.

---

## Inventaire des charges de travail

### Nœuds K3s (VLAN 20 – stockage local-lvm)

| Nom       | Hôte Proxmox | VMID | IP            | vCPU | RAM  |
|-----------|--------------|------|---------------|------|------|
| k3s-pve2  | pve2         | 2021 | 10.0.20.21/24 | 3    | 7 Go |
| k3s-pve3  | pve3         | 2022 | 10.0.20.22/24 | 3    | 7 Go |
| k3s-pve4  | pve4         | 2023 | 10.0.20.23/24 | 3    | 6 Go |

- Endpoint API (kube-vip) : `10.0.20.20:6443`
- Disques système en local-lvm (pas sur Ceph) pour limiter la latence d’etcd.

### VMs WordPress publiques (VLAN 40 – Ceph)

| Nom            | Hôte Proxmox | VMID | IP            | vCPU | RAM  |
|----------------|--------------|------|---------------|------|------|
| hantaweb       | pve3         | 4011 | 10.0.40.11/24 | 3    | 4 Go |
| petitsanglais  | pve4         | 4012 | 10.0.40.12/24 | 1    | 1 Go |
| hanta-assos    | pve3         | 4013 | 10.0.40.13/24 | 1    | 1 Go |

Stockage sur Ceph pour permettre la migration / redémarrage HA en cas de panne d’un nœud.

### LXC média (VLAN 30)

| Nom      | Hôte Proxmox | VMID | IP            | Notes                                      |
|----------|--------------|------|---------------|--------------------------------------------|
| jellyfin | pve2         | 3010 | 10.0.30.10/24 | Privileged (passthrough GPU), bibliothèque sur NFS (NAS) |

D’autres services média / photo sont en cours de validation.

### Plan de gestion (pve1 + NAS)

- **pve1** : OPNsense, PBS, nœud d’automatisation (VLAN 10)
- **NAS** : ZFS RAID-Z2 + MinIO + exports NFS (jellyfin, photoprism, seafile, backups…)

---

## Choix techniques principaux

| Choix                                      | Raison                                                                 |
|--------------------------------------------|------------------------------------------------------------------------|
| Disques K3s en local-lvm                   | etcd est sensible à la latence ; Ceph introduirait trop de jitter     |
| WordPress et services stateful hors-K3s sur Ceph | Migration et redémarrage automatique possibles en cas de panne matérielle |
| LXC média + NFS                            | Passthrough GPU + grosses bibliothèques hors Ceph                     |
| Plan de gestion isolé                      | Évite le problème de l’œuf et de la poule                             |
| Bootstrap Ansible → GitOps ArgoCD          | Ansible pour le socle, ArgoCD pour tout ce qui suit                   |

---

## Statut (août 2026)

| Composant                     | État                                      |
|-------------------------------|-------------------------------------------|
| Proxmox + Ceph + réseau + NAS | Production quotidienne                    |
| VMs WordPress                 | Production                                |
| Automatisation IaC            | Opérationnelle                            |
| Cluster K3s                   | Bootstrappé – validation HA / GitOps en cours |
| Migration services stateless  | Progressive                               |

---

**Pages liées :**

- [← Accueil](/)
- [IaC et automatisation →](/iac-automation.html)
- [Réseau →](/networking.html) · [Sécurité →](/security.html)
- [Services →](/services.html)
