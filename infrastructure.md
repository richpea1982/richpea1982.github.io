---
layout: default
title: Vue d'ensemble de l'infrastructure
nav_order: 2
---

# Vue d'ensemble de l'infrastructure

Cette page détaille la topologie physique et virtuelle cible du homelab, mettant en évidence les choix d'ingénierie et la planification des composants pour garantir la haute disponibilité. 
Le dépôt Git `infra-homelab` constitue la **source unique de vérité**.

---

## Architecture cible (5 nœuds physiques)

**Backbone de gestion (2 nœuds)** - **pve1 — Nœud de gestion spécialisé** : OPNsense, Proxmox Backup Server (PBS), Nœud d'automation (Terraform, Ansible, Semaphore). Maintenu hors-cluster pour éliminer tout problème d'interdépendance ("œuf et la poule").  
- **NAS Bare-Metal** : Debian + ZFS RAID‑Z2 (6 × 1 TB) ; point de terminaison de stockage objet MinIO S3 (endpoint canonicalisé).

**Couche de calcul & Stockage (3 nœuds)** - **pve2, pve3, pve4** — Hyperviseurs Proxmox VE configurés en cluster HA.
- **Stockage distribué Ceph** : Pool répliqué sur pve2/pve3/pve4 (facteur de réplication = 3, min 2). Dédié exclusivement aux services stateful hors-Kubernetes nécessitant une haute disponibilité (VMs WordPress, LXC Seafile, etc.).
- **K3s** : Cluster K3s distribué gérant sa propre résilience interne pour les microservices conteneurisés.

---

## Proxmox / Service mapping

### Nœud de Gestion Spécialisé (pve1)

| Nom de la VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | ID VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Type &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Vlan &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Datastore &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Usage &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | CPU &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Ram &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `OPNsense` | 110 | VM | 10 | local-lvm | Router / Pare-feu | 3 Cœurs | 4Go |
| `PBS` | 130 | VM | 10 | local-lvm | Serveur de sauvegarde | 2 Cœurs | 4Go |
| `Nœud d'automation` | 1040 | LXC | 10 | local-lvm | Orchestration IaC & Semaphore | 2 Cœurs | 4Go |

### Cluster Compute & Stockage (pve2, pve3, pve4)

Conformément à la convention d'infrastructure, le préfixe de l'ID des machines virtuelles détermine leur appartenance réseau (ex: `2xxx` pour le VLAN 20, `4xxx` pour le VLAN 40).

| Nom de la VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | ID VM &nbsp;&nbsp;&nbsp;&nbsp; | Nœud Proxmox &nbsp;&nbsp;&nbsp;&nbsp; | Type &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Vlan &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Datastore &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Usage &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | CPU &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Ram &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `k3s-pve2` | 2021 | pve2 | VM | 20 | local-lvm | K3s control-plane (Bootstrap) / worker | 3 Cœurs | 6Go |
| `k3s-pve3` | 2022 | pve3 | VM | 20 | local-lvm | K3s control-plane / worker | 3 Cœurs | 6Go |
| `k3s-pve4` | 2023 | pve4 | VM | 20 | local-lvm | K3s control-plane / worker | 3 Cœurs | 5Go |
| `hantaweb` | 4011 | pve3 HA | VM | 40 | ceph-storage | WordPress (WooCommerce public) | 2 Cœurs | 4Go |
| `petitsanglais` | 4012 | pve4 HA | VM | 40 | ceph-storage | WordPress (Site vitrine public) | 1 Cœur | 1Go |
| `Seafile` | 4015 | pve3 HA | LXC | 40 | ceph-storage | Stockage de fichiers cloud privé | 2 Cœurs | 2Go |
| `Jellyfin` | 3010 | pve2 | LXC | 30 | local-lvm | Serveur Multimédia (Data sur NAS) | 3 Cœurs | 6Go |
| `Photoprism` | 3011 | pve2 | LXC | 30 | local-lvm | Indexation Photo (Data sur NAS) | 3 Cœurs | 6Go |

> 💡 **Raisonnement du Layout de Stockage : Ceph vs Local-LVM**
> 
> L'allocation des datastores répond à un arbitrage strict entre latence disque et tolérance aux pannes :
> * **Exclusion de Ceph pour K3s** : Les disques système des nœuds K3s (`2021-2023`) sont positionnés sur le stockage `local-lvm` (SSD locaux). Le moteur de consensus d'etcd est extrêmement sensible à la latence d'écriture synchrone (*fsync*). Utiliser un stockage réseau distribué comme Ceph pour héberger l'etcd introduirait des variations de latence réseau risquant de briser le quorum de l'orchestrateur.
> * **Rétention de Ceph pour les Services Hors-K3s** : À l'inverse, les instances WordPress (`hantaweb`, `petitsanglais`) et le cloud `Seafile` bénéficient pleinement du datastore `ceph-storage`. En cas de panne matérielle brutale d'un hyperviseur, Proxmox VE migre et redémarre instantanément ces VMs sur un nœud sain sans aucune perte de données grâce au stockage partagé sous-jacent.

---

### K3s / Service mapping

| Service / App &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Namespace &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Exposed Via &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Access Domain / URL &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Network Security (Calico Policy) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Storage (PV/PVC) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `CF Tunnel` | `networking` | Cloudflare Daemon | `richard.pearsalls.fr` | Autorisé à communiquer UNIQUEMENT avec le pod Ingress | Aucun |
| `Traefik` | `kube-system` | Réseau Local / WG | *Routage Interne* | Contrôle global du routeur d'ingress | Aucun (Apatride) |
| `Prometheus` | `monitoring` | Ingress Traefik | *Interne Uniquement* | Isolé ; sortie uniquement pour la collecte de métriques | `prometheus-pvc` |
| `Grafana` | `monitoring` | Traefik -> WG Local | `grafana.local.lan` | Restreint aux IPs admins authentifiées | `grafana-pvc` |
| `Dozzle` | `monitoring` | Traefik -> WG Local | `logs.local.lan` | Restreint au namespace de monitoring | Aucun |
| `BenToPDF` | `tools` | Traefik -> WG Local | `pdf.local.lan` | Backend isolé | `bento-pvc` |
| `MD Portfolio` | `web` | CF Tunnel -> Traefik | `richard.pearsalls.fr` | Ingress depuis CF autorisé ; egress refusé | `portfolio-assets` |

---

## Réseau physique et plan de continuité

* **Contraintes Matérielles** : Chaque hôte Proxmox de calcul dispose d'une interface physique réseau unique de 1 Gbps.
* **Stratégie d'Isolation Physique** : Pour assurer les performances du stockage Ceph sans saturer la bande passante utilisateur et d'administration, l'architecture cible prévoit l'ajout d'adaptateurs USB-to-Ethernet Gigabit. Les interfaces internes intégrées (*onboard*) sont dédiées exclusivement à la réplication des blocs Ceph à travers un commutateur non administré isolé. Le trafic de gestion de l'hyperviseur (VLAN 10) et les flux applicatifs des VMs/Pods transitent de manière étanche via les liaisons USB de secours.

---

* **[Suivant : IaC et Automation →](/iac-automation.html)**
* **[← Accueil](/index.html)**
