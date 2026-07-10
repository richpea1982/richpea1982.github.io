---
layout: default
title: Services & Charges de travail
nav_order: 5
---

<div style="text-align: right">
  <a href="/en/services.html">🇬🇧 English</a>
</div>

# Services & Charges de travail

Cette page détaille l'architecture logique des applications, la topologie du cluster Kubernetes (K3s), les mécanismes de routage de couche 7 (Ingress) et le cycle de vie de la persistance des données.

---

## Architecture des Charges de Travail

Le homelab sépare strictement l'exécution des services selon leur état opérationnel (*stateful* vs *stateless*) et leurs besoins d'accès au matériel physique :

1. **Services Hors-Cluster (Proxmox LXC/VM)** : Les applications nécessitant un accès matériel direct de type *GPU Passthrough* ou un couplage fort avec le système de fichiers (ex: transcodage Jellyfin, indexation lourde Photoprism) sont isolées dans des conteneurs LXC dédiés sur `pve2`. Elles s'appuient sur le stockage distribué Ceph pour garantir la résilience des données sous-jacentes[cite: 2].
2. **Services Orchestrés (Cluster K3s)** : Le cluster Kubernetes à 3 nœuds (`k3s-pve2`, `k3s-pve3`, `k3s-pve4`) exécute l'ensemble des microservices, des outils de monitoring et des applications cloud natives.

---

## Topologie Kubernetes (K3s) & Cycle de Vie Appliqué

Le cluster s'appuie sur une distribution K3s hautement disponible dotée d'un plan de contrôle distribué (Etcd embarqué). Les nœuds applicatifs s'exécutent sur les datastores `local-lvm` rapides de chaque hyperviseur, car K3s gère nativement sa propre haute disponibilité et le cycle de vie de ses réplicas.

### 1. Organisation des Espaces de Noms (Namespaces)
Le cluster orchestre les applications à travers des frontières logiques étanches :
- `networking` : Démon Cloudflare Tunnel (`cloudflared`) gérant la connectivité externe sécurisée.
- `kube-system` : Contrôleur d'Ingress Traefik v3 et briques fondamentales du cluster.
- `monitoring` : Pile d'observabilité centralisée regroupant Prometheus, Grafana et Dozzle.
- `tools` : Utilitaires et microservices d'infrastructure internes (ex: BentoPDF).
- `web` : Applications frontales de production (Markdown Portfolio).

---

## Routage de Couche 7 & Gestion de l'Ingress

L'acheminement du trafic HTTP/HTTPS repose entièrement sur **Traefik v3**, configuré de manière déclarative.
