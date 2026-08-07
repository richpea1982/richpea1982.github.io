---
layout: default
title: Services & Charges de travail
nav_order: 5
---

# Services & Charges de travail

Les applications sont réparties selon leur nature (stateful / stateless), leurs besoins en ressources et le type de stockage le plus adapté.

---

## Hors cluster (Proxmox VMs / LXC)

### VMs WordPress (VLAN 40 – Ceph)

| Nom            | Hôte   | VMID | Ressources     | Rôle                          |
|----------------|--------|------|----------------|-------------------------------|
| hantaweb       | pve3   | 4011 | 3 vCPU / 4 Go  | E-commerce WooCommerce        |
| petitsanglais  | pve4   | 4012 | 1 vCPU / 1 Go  | Site vitrine                  |
| hanta-assos    | pve3   | 4013 | 1 vCPU / 1 Go  | Site association              |

Stockage sur Ceph pour permettre la migration / redémarrage HA.

### LXC média (VLAN 30 – local-lvm + NFS)

| Nom      | Hôte | VMID | Notes                                              |
|----------|------|------|----------------------------------------------------|
| jellyfin | pve2 | 3010 | Privileged (GPU passthrough), bibliothèque sur NFS |

D’autres services média / photo sont en cours de validation.

---

## Dans le cluster K3s (VLAN 20)

Le cluster (3 nœuds) gère les services conteneurisés via ArgoCD (GitOps).

| Service          | Namespace   | Exposition              | Accès                      | Stockage          |
|------------------|-------------|-------------------------|----------------------------|-------------------|
| Cloudflare Tunnel| networking  | Sortant uniquement      | —                          | Aucun             |
| Traefik          | kube-system | Ingress                 | Routage interne / public   | Aucun             |
| Prometheus       | monitoring  | Traefik (interne)       | Admin uniquement           | PVC local         |
| Grafana          | monitoring  | Traefik → WG            | grafana.local.lan          | PVC local         |
| Portfolio        | web         | CF Tunnel → Traefik     | Domaine public             | PVC / assets      |
| Vaultwarden      | —           | Traefik / WG            | Interne                    | PVC               |
| Velero           | —           | —                       | Sauvegardes cluster        | S3 (MinIO)        |

Les politiques Calico limitent les communications entre pods (ex. : le tunnel Cloudflare ne peut parler qu’à Traefik).

---

## Routage

**Trafic public**  
Client → Cloudflare (WAF) → Tunnel cloudflared → Traefik → service (ex. portfolio)

**Trafic d’administration**  
Client → WireGuard → Traefik → services internes (Grafana, etc.)

Aucun port n’est ouvert en entrée sur l’IP publique.

---

## Stratégie de stockage

| Type de données              | Emplacement              | Raison                                      |
|------------------------------|--------------------------|---------------------------------------------|
| Disques système K3s / etcd   | local-lvm                | Latence minimale pour etcd                  |
| VMs WordPress                | Ceph                     | HA / migration en cas de panne d’un nœud    |
| Bibliothèques média          | NAS (ZFS) via NFS        | Volume important + double tolérance disque  |
| State Terraform + backups    | MinIO (S3) sur NAS       | Object storage + object lock                |
| PVC applicatifs K3s          | local-lvm (par nœud)     | Simplicité et performance                   |

---

## Statut (août 2026)

| Catégorie                    | État                          |
|------------------------------|-------------------------------|
| VMs WordPress                | Production                    |
| Jellyfin                     | Production                    |
| Stack monitoring (Prometheus/Grafana) | Opérationnelle          |
| Portfolio (dans K3s)         | Déployé / en stabilisation    |
| Autres services GitOps       | Migration progressive         |

---

**Pages liées :**

- [← Sécurité](/security.html)
- [Sauvegarde →](/backup-strategy.html)
- [Rétrospective →](/lessons-learned.html)
