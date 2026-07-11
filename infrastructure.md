---
layout: default
title: Vue d'ensemble de l'infrastructure
nav_order: 2
---

# Vue d'ensemble de l'infrastructure

Cette page détaille la topologie physique et virtuelle du homelab. Le dépôt Git infra-homelab constitue la **source unique de vérité**.

---

## Architecture cible (5 nœuds physiques)

**Backbone de gestion (2 nœuds)**  
- **pve1 — Nœud de gestion spécialisé** : OPNsense, Proxmox Backup Server (PBS), Nœud d'automation; Terraform, Ansible, Semaphore.  
- **NAS Bare-Metal** : Debian + ZFS RAID‑Z2 (6 × 1 TB) ; MinIO S3 (endpoint canonicalisé).

**Couche de calcul (3 nœuds)**  
- **pve2, pve3, pve4** — hyperviseurs Proxmox.  
- **Stockage distribué** : Ceph répliqué sur pve2/pve3/pve4 (replication factor = 3 min 2).  
- **K3s** : cluster K3s distribué ; K3s gère sa propre HA interne pour les services qu'il orchestre.

---
**Proxmox / Service mapping**

- **Pve1**

| Nom de la VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | ID VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Type &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Vlan &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Datastore &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Usage &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | CPU &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Ram &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :---| :--- | :--- | :--- | :--- | :--- | :--- | :--- 
| `OPNsense` | 110 | VM | 10 | local-lvm | Router / Pare-feu | 3 Cœurs | 4Go |
| `PBS` | 130 | VM | 10 | local-lvm | Serveur de sauvegard | 2 Cœurs | 4Go |
| `Nœud d'automation` | 1040 | LXC | 10 | local-lvm | Automation | 2 Cœurs | 4Go |


- **Cluster pve2 pve3 pve4**

| Nom de la VM &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | ID VM &nbsp;&nbsp;&nbsp;&nbsp; | Nœud Proxmox &nbsp;&nbsp;&nbsp;&nbsp; | Type &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Vlan &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Datastore &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Usage &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | CPU &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Ram &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :---| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `k3s-pve2` | 1021 | pve2 | VM | 20 | local-lvm | K3s control-plane / worker | 3 Cœurs | 6Go |
| `k3s-pve3` | 1022 | pve3 | VM | 20 | local-lvm | K3s control-plane / worker | 3 Cœurs | 6Go |
| `k3s-pve4` | 1023 | pve4 | VM | 20 | local-lvm | K3s control-plane / worker | 3 Cœurs | 5Go |
| `hantaweb` | 4011 | pve3 HA | VM | 40 | ceph-storage | WordPress (WooCommerce) | 2 Cœurs | 4Go |
| `petitsanglais` | 4012 | pve4 HA | VM | 40 | ceph-storage | WordPress (site vitrine) | 1 Cœur | 1Go |
| `Jellyfin` | 2010 | pve2 | LXC | 30 | local-lvm | Media-server | 3 Cœurs | 6Go |
| `Photoprism` | 2011 | pve2 | LXC | 30 | local-lvm | Photo-db | 3 Cœurs | 6Go |

Remarque : Bien que Jellyfin et Photoprism soient installés sur le stockage local-lvm pour leurs conteneurs, l’intégralité de leurs données et médias est hébergée sur le NAS en RAID‑Z2.

**K3s / Service mapping**

| Service / App &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Namespace &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Exposed Via &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Access Domain / URL &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Network Security (Calico Policy) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Storage (PV/PVC) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `CF Tunnel` | `networking` | Cloudflare Daemon | `richard.pearsalls.fr` | Autorisé à communiquer UNIQUEMENT avec le pod Portfolio | Aucun |
| `Traefik` | `kube-system` | Réseau Local / WG | *Routage Interne* | Contrôle global du routeur d'ingress | Aucun (Apatride) |
| `Prometheus` | `monitoring` | Ingress Traefik | *Interne Uniquement* | Isolé ; sortie uniquement pour la collecte de métriques | `prometheus-pvc` |
| `Grafana` | `monitoring` | Traefik -> WG Local | `grafana.local.lan` | Restreint aux IPs admins authentifiées | `grafana-pvc` |
| `Dozzle` | `monitoring` | Traefik -> WG Local | `logs.local.lan` | Restreint au namespace de monitoring | Aucun |
| `BenToPDF` | `tools` | Traefik -> WG Local | `pdf.local.lan` | Backend isolé | `bento-pvc` |
| `MD Portfolio` | `web` | CF Tunnel -> Traefik | `richard.pearsalls.fr` | Ingress depuis CF autorisé ; egress refusé | `portfolio-assets` |

---

## Réseau physique et plan de continuité

Contraintes : chaque hôte Proxmox (pve2/pve3/pve4) dispose d'une seule interface 1 Gbps.  
Plan de mitigation : ajouter un adaptateur USB‑Ethernet si nécessaire ; dédier les ports onboard au trafic Ceph via un switch non‑géré et utiliser les adaptateurs USB pour le trafic management/VM.

---

* **[IaC & Automatisation](/iac-automation.html)** — Modules Terraform, rôles Ansible et pipelines d'orchestration de déploiement.
* **[Architecture Réseau](/networking.html)** — Règles de routage OPNsense, configurations VLAN et tunneling Zero-Trust.
* **[Services & Applications](/services.html)** — Topologie Kubernetes (K3s), routage d'ingress Traefik et cycle de vie des bases de données
* **[Modèle de Sécurité](/security.html)** — Analyse des menaces, parsing de logs CrowdSec et injection de secrets.
* **[Sauvegarde & Plan de Reprise](/backup-strategy.html)** — Implémentation de la règle 3-2-1, Proxmox Backup Server et réplication ZFS.

---

[← Accueil](/index.html)
