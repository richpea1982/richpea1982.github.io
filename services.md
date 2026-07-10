---
layout: default
title: Services & Charges de travail
nav_order: 5
---

<div style="text-align: right">
  <a href="/en/services.html">🇬🇧 English</a>
</div>

# Services & Charges de travail

Cette page détaille l'architecture logique des applications, la topologie du cluster Kubernetes (K3s), les mécanismes de routage de couche 7 (Ingress) ainsi que la gestion de la persistance et du stockage des données du homelab.

L'ensemble des déploiements décrits ici est orchestré de manière déclarative via notre dépôt Git, constituant la source unique de vérité.

---

## Architecture des Charges de Travail

Le homelab sépare strictement l'exécution des services selon leur état opérationnel (*stateful* vs *stateless*), leurs besoins en ressources et leur adhérence avec le stockage physique :

### 1. Services Hors-Cluster (Proxmox LXC/VM)
Les applications gourmandes en calcul ou nécessitant un accès de stockage massif non-cloud-native sont isolées en dehors de Kubernetes afin de maximiser les performances :
- **`Jellyfin` (LXC 2010, pve2, VLAN 30)** : Serveur multimédia bénéficiant d'un accès direct aux ressources de calcul de `pve2`. Son conteneur est stocké sur Ceph pour la résilience du système.
- **`Photoprism` (LXC 2011, pve2, VLAN 30)** : Base de données et indexation de photos, installée de manière similaire sur `pve2`.

> 💡 **Règle de découplage du stockage** : Bien que les conteneurs système de Jellyfin et Photoprism résident sur le pool résilient `ceph-storage`, l'intégralité de leurs données applicatives, fichiers sources et médias bruts est hébergée de manière centralisée sur le **NAS Bare-Metal**.

- **VMs de Production WordPress (VLAN 40)** : Deux instances critiques s'exécutent en Haute Disponibilité (HA) grâce au stockage distribué Ceph[cite: 3] :
  - **`hantaweb` (VM 4011, pve3 HA)** : Instance e-commerce WooCommerce de production (2 Cœurs CPU, 4 Go RAM).
  - **`petitsanglais` (VM 4012, pve4 HA)** : Site vitrine de production (1 Cœur CPU, 1 Go RAM).

### 2. Services Orchestrés (Cluster K3s)
Le cluster Kubernetes hautement disponible s'étend sur trois nœuds virtuels dédiés (`k3s-pve2`, `k3s-pve3`, `k3s-pve4`) situés sur le **VLAN 20**[cite: 3]. K3s gère sa propre haute disponibilité interne pour l'ensemble des microservices et outils qu'il héberge.

---

## Cartographie des Espaces de Noms K3s (Namespaces)

L'organisation interne du cluster s'articule autour de frontières logiques étanches sécurisées par des politiques réseau strictes (Calico) :

| Service / App | Namespace | Exposé via | Domaine / URL d'accès | Sécurité Réseau (Calico Policy) | Stockage (PV/PVC) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`CF Tunnel`** | `networking` | Cloudflare Daemon | `richard.pearsalls.fr` | Autorisé à communiquer **UNIQUEMENT** avec le pod Portfolio | Aucun |
| **`Traefik`** | `kube-system` | Réseau Local / WG | *Routage Interne* | Contrôle global du routeur d'ingress | Aucun (Apatride) |
| **`Prometheus`** | `monitoring` | Ingress Traefik | *Interne Uniquement* | Isolé ; sortie uniquement pour la collecte de métriques | `prometheus-pvc` |
| **`Grafana`** | `monitoring` | Traefik -> WG Local | `grafana.local.lan` | Restreint aux IPs admins authentifiées | `grafana-pvc` |
| **`Dozzle`** | `monitoring` | Traefik -> WG Local | `logs.local.lan` | Restreint au namespace de monitoring | Aucun |
| **`BenToPDF`** | `tools` | Traefik -> WG Local | `pdf.local.lan` | Backend isolé | `bento-pvc` |
| **`MD Portfolio`** | `web` | CF Tunnel -> Traefik | `richard.pearsalls.fr` | Ingress depuis CF autorisé ; egress refusé | `portfolio-assets` |

---

## Routage de Couche 7 & Flux Réseau

L'aiguillage du trafic vers le cluster est segmenté selon la sensibilité des applications :
