---
layout: default
title: Accueil
nav_order: 1
lang: fr
---

# Richard Pearsall — administration systèmes et réseaux

Après une vingtaine d'années dans la restauration puis la plomberie, j'ai engagé une reconversion vers les systèmes et réseaux. Je suis actuellement en formation **TSSR (Technicien Supérieur Systèmes et Réseaux)** à L'IDEM Le Soler, avec une fin prévue en janvier 2027, et je suis disponible à partir de février 2027.

Mon homelab me sert de terrain d'entraînement quotidien. J'y pratique l'Infrastructure as Code, la virtualisation, Kubernetes et la sécurisation d'infrastructures sous contraintes matérielles réelles. Ce site présente les choix d'architecture, la chaîne d'automatisation et un retour d'expérience sur une reprise après sinistre.

> Les dépôts contenant le code d'infrastructure restent privés. Les adresses IP, les noms d'hôtes et les identifiants internes sont volontairement absents de cette présentation.

## Mon parcours

- **Étape 1** — Premier contact avec Linux sur un vieux portable (dual-boot, ligne de commande).
- **Étape 2** — Premier site WordPress auto-hébergé sur l'offre gratuite de Google Cloud Platform.
- **Étape 3** — Retour au local : mini-PC sous Proxmox, tunnel Cloudflare, premiers conteneurs et supervision.
- **Étape 4** — Deuxième machine, Traefik, services internes, reconstruction complète via Terraform et Ansible.
- **Étape 5** — Segmentation réseau par VLAN et ajout d'un switch administrable.
- **Étape 6** — Consolidation : cluster Proxmox + Ceph à trois nœuds, NAS dédié en ZFS, cluster K3s à trois nœuds.

## Architecture en résumé

### Plan de gestion, isolé du cluster de calcul

- Pare-feu et routage (OPNsense), sauvegardes (Proxmox Backup Server), automatisation (Semaphore et Ansible) sur une machine dédiée.
- NAS bare-metal : Debian, ZFS en RAID-Z2 (six disques d'un téraoctet), stockage objet compatible S3 et partages NFS.

### Plan de calcul

- Trois hyperviseurs Proxmox en cluster avec stockage Ceph.
- VMs WordPress publiques, nœuds du cluster K3s et conteneurs LXC média.

### Réseau et exposition

- Segmentation en VLAN : gestion, services, média, DMZ et environnement de test.
- Publication via des tunnels Cloudflare sortants ; accès d'administration par un VPN WireGuard ou Tailscale.

### Kubernetes

- Cluster K3s à trois nœuds avec etcd embarqué et une adresse IP virtuelle pour l'API.
- Bootstrap et configuration du socle par Ansible, puis déploiements applicatifs en GitOps via ArgoCD.
- La redondance concerne le plan de contrôle ; les charges de travail ne sont pas toutes répliquées et reposent encore partiellement sur la sauvegarde et la restauration.

## État d'avancement (novembre 2026)

- Proxmox, Ceph, NAS et segmentation réseau : en usage quotidien.
- Provisioning Terraform, configuration Ansible et orchestration Semaphore : opérationnels.
- Cluster K3s : plan de contrôle redondé, GitOps en place pour les applications principales.
- Généralisation du GitOps et migration des derniers services vers K3s : en cours.

## Méthode de travail

- Tout est versionné dans Git ; les modifications manuelles durables sont évitées.
- Le plan de gestion est isolé du plan de calcul pour éviter les dépendances circulaires.
- Les choix techniques sont guidés par les contraintes réelles du matériel : latence disque, mémoire limitée, anciens processeurs.
- J'utilise l'IA comme aide à la rédaction de code répétitif, mais je valide et comprends chaque bloc avant de le déployer.

## Pour aller plus loin

- [Vue d'ensemble de l'infrastructure](/infrastructure.html)
- [IaC et automatisation](/iac-automation.html)
- [Reprise après sinistre : reconstruction du cluster et restauration etcd](/portfolio-tests-and-verification.html)
- [Projet Groupemen : environnement Windows de pré-production](/groupemen.html)
