---
layout: default
title: Vue d'ensemble de l'infrastructure
nav_order: 2
lang: fr
---

# Vue d'ensemble de l'infrastructure

Cette page présente la topologie, les choix d'architecture et leurs arbitrages. Les adresses, les identifiants et les noms d'hôtes internes sont volontairement omis.

## Topologie physique

| Rôle | Description |
| --- | --- |
| Plan de gestion | Une machine dédiée : pare-feu et routage (OPNsense), sauvegardes (Proxmox Backup Server), automatisation (Semaphore, Ansible). Elle est isolée du cluster de calcul. |
| Stockage | NAS dédié sous Debian : ZFS en RAID-Z2 (six disques d'un téraoctet), stockage objet compatible S3 (MinIO), partages NFS. |
| Calcul et stockage distribué | Trois hyperviseurs Proxmox en cluster avec Ceph. Ils hébergent les VMs K3s, les VMs WordPress et les conteneurs LXC média. |

Le plan de gestion est isolé du cluster de calcul pour éviter les dépendances circulaires : si le cluster de calcul est indisponible, l'automatisation et les sauvegardes restent accessibles.

## Segmentation réseau

| Segment | Usage |
| --- | --- |
| Gestion | Hyperviseurs, sauvegardes, automatisation, NAS. |
| Services et K3s | Nœuds Kubernetes et services internes. |
| Média | Services multimédia, bibliothèques stockées sur NFS. |
| DMZ | Machines virtuelles publiques (WordPress). |
| Laboratoire | Expérimentations et services non fiables. |

La publication publique passe par des tunnels Cloudflare sortants : aucun service n'est exposé par ouverture de port entrant. L'accès d'administration passe par un réseau privé virtuel (WireGuard ou Tailscale).

## Charges de travail

| Type | Nombre | Emplacement | Note |
| --- | --- | --- | --- |
| Nœuds K3s | 3 | Cluster Proxmox, disques locaux | etcd embarqué et adresse API virtuelle. Les disques système restent locaux pour limiter la latence d'etcd. |
| VMs WordPress | 3 | Stockage Ceph | Le stockage Ceph permet la migration entre hyperviseurs et le redémarrage automatique. |
| Conteneurs LXC média | 1 | Cluster Proxmox, données sur NFS | Accès GPU direct et bibliothèques volumineuses laissées hors de Ceph. |

## Choix techniques et arbitrages

| Choix | Raison |
| --- | --- |
| Disques système K3s sur stockage local | etcd est sensible à la latence ; Ceph introduirait trop de variabilité. |
| VMs stateful sur Ceph | Migration et redémarrage automatiques en cas de panne d'un hyperviseur. |
| Conteneurs média avec données sur NFS | Passthrough GPU et grandes bibliothèques hors Ceph. |
| Plan de gestion isolé | Évite le cercle vicieux où il faut le cluster pour réparer le cluster. |
| Ansible pour le socle, ArgoCD ensuite | Séparation nette entre provisionnement et charges applicatives. |

## Limites assumées

La redondance du plan de contrôle K3s ne rend pas toutes les charges de travail redondantes. Les services stateful restent dépendants de Ceph et de sauvegardes ; une restauration complète a été observée une fois, mais certains cheminements manuels restent à automatiser. Ces limites sont documentées dans la page consacrée à la reprise après sinistre.

## Pages liées

- [← Accueil](/)
- [IaC et automatisation →](/iac-automation.html)
- [Reprise après sinistre →](/portfolio-tests-and-verification.html)
- [Projet Groupemen →](/groupemen.html)
