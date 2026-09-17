---
layout: default
title: Accueil
nav_order: 1
---

## Mon parcours : de la restauration à l’administration système

Après une vingtaine d’années dans la restauration puis la plomberie, j’ai engagé une reconversion vers les systèmes et réseaux.  
Je suis actuellement en formation **TSSR (Technicien Supérieur Systèmes et Réseaux)** à L’IDEM Le Soler (fin prévue janvier 2027).

Mon homelab me sert de terrain d’entraînement quotidien. J’y pratique l’Infrastructure as Code, la virtualisation, Kubernetes et la sécurisation d’infrastructures sous contraintes matérielles réelles.

---

## Évolution du homelab

L’architecture actuelle n’est pas née d’un coup. Elle s’est construite progressivement :

- **Année 1** — Premier contact avec Linux sur un vieux PC portable (dual-boot, CLI).
- **Année 2** — Premier site WordPress auto-hébergé sur le free tier de Google Cloud Platform.
- **Année 4** — Passage au local : mini-PC sous Proxmox, tunnel Cloudflare, premiers conteneurs et monitoring basique.
- **Année 5** — Ajout d’une deuxième machine, Traefik, services internes (Seafile, Jellyfin…). Reconstruction complète de l’infrastructure via **Terraform + Ansible**.
- **Année 5.5** — Segmentation réseau (switch administrable, VLANs).
- **Année 6** — Consolidation : cluster Proxmox + Ceph à 3 nœuds, NAS dédié en ZFS, démarrage du cluster K3s en haute disponibilité.

---

## Architecture actuelle (résumé)

L’environnement est divisé en deux plans clairement séparés :

**1. Plan de gestion (hors cluster de calcul)**  

- **pve1** : OPNsense (pare-feu / routage), Proxmox Backup Server, nœud d’automatisation (Semaphore + Ansible).  
- **NAS bare-metal** : Debian + ZFS RAID-Z2 (6 × 1 To) + MinIO (stockage objet S3).

**2. Plan de calcul (cluster Proxmox + Ceph)**  

- **pve2, pve3, pve4** : hyperviseurs Proxmox en cluster avec stockage Ceph.  
- Hébergent les VMs WordPress publiques, les nœuds K3s et les LXC média.

**Réseau**  

- Segmentation en VLANs (management, interne, média, DMZ, untrusted).  
- Exposition publique uniquement via **Cloudflare Tunnels** (aucun port ouvert en entrée).  
- Accès d’administration via overlay WireGuard / Tailscale.

**Kubernetes**  

- Cluster K3s à 3 nœuds (etcd embarqué + kube-vip).  
- Bootstrap et configuration de base gérés par Ansible.  
- Déploiements applicatifs gérés ensuite en GitOps via ArgoCD.

Le détail des nœuds, adresses IP, ressources et choix techniques se trouve dans le dépôt [infra-homelab](https://github.com/richpea1982/infra-homelab) (README = source de vérité).

---

## Statut actuel (août 2026)

| Composant                        | État                                      |
|----------------------------------|-------------------------------------------|
| Proxmox + Ceph + réseau + NAS    | En production quotidienne                 |
| VMs WordPress publiques          | En production                             |
| Automatisation (Terraform/Ansible/Semaphore) | Opérationnelle                     |
| Cluster K3s                      | Bootstrappé, validation HA et GitOps en cours |
| Migration des services stateless | Progressive                               |

L’architecture décrite ici et dans le dépôt Git correspond à l’état cible et au code actuel. Certaines parties (notamment la finalisation du GitOps et le déplacement de tous les services dans K3s) sont encore en cours de consolidation.

---

## Méthode de travail

- Tout est versionné dans Git. Les modifications manuelles sont évitées.
- Le plan de gestion (pare-feu, sauvegardes, automatisation) est isolé du plan de calcul pour éviter les dépendances circulaires.
- Les choix techniques sont guidés par les contraintes réelles du matériel (latence disque, RAM limitée, vieux CPU).
- J’utilise l’IA comme aide à la rédaction de code répétitif, mais je valide et comprends chaque bloc avant de le déployer.

---

**Pages suivantes :**

- [Vue d’ensemble de l’infrastructure →](/infrastructure.html)
- [IaC et automatisation →](/iac-automation.html)
- [Réseau et sécurité →](/networking.html) / [Sécurité →](/security.html)
- [Services →](/services.html)
- [Rétrospective et leçons apprises →](/lessons-learned.html)
