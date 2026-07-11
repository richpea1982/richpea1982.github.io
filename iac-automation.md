---
layout: default
title: IaC & Automatisation
nav_order: 3
---

# IaC & Automatisation

Cette page décrit la stratégie d'Infrastructure as Code (IaC) et les mécanismes d'automatisation de bout en bout utilisés pour provisionner, configurer et maintenir l'intégralité du homelab. Le dépôt Git `infra-homelab` constitue mon **source unique de vérité**.

---

## Objectif de la page

Fournir une vue opérationnelle, exhaustive et technique des processus IaC. Cela inclut le cycle de vie du backend Terraform, l'architecture des modules, la structure des rôles Ansible, l'orchestration des pipelines CI/CD, les procédures de bootstrap initial et le cycle de déploiement applicatif GitOps basé sur Helm.

---

## Principes directeurs

- **Idempotence Absolue** : Tous les playbooks Ansible, scripts système et modules Terraform sont conçus pour être rejoués à l'infini sans altérer ou dégrader l'état stable de la production.
- **Séparation des Responsabilités (Out-of-Band Control Plane)** : Les moteurs et serveurs d'automatisation (Ansible Control Node, serveurs d'intégration) s'exécutent sur des ressources matérielles dédiées (`pve1`), complètement isolées du pool de calcul général. Une panne globale du cluster Kubernetes ou du stockage distribué n'affecte jamais mes outils de gestion ou de reprise d'activité.

---

## Composants clés & Spécifications techniques

### 1. Gestion d'Infrastructure (Terraform)
- **Fournisseur Principal** : `bpg/proxmox` pour interagir de manière native avec l'API Proxmox VE.
- **Gestion du State (Backend S3)** : 
  - Stockage hébergé sur le point d'accès S3 de mon infrastructure locale : (MinIO s'exécutant sur le NAS Bare-Metal).
  - Sécurisation du State : Chiffrement au repos appliqué par défaut. Le mécanisme de *state locking* utilise l'API S3 native complétée par une table de verrouillage. En cas de blocage intempestif d'un pipeline, la commande `terraform force-unlock <LOCK-ID>` est documentée et restreinte au nœud d'administration.
- **Modularité** : Le code est découpé en sous-modules réutilisables :
  - `modules/proxmox_lxc` : Provisionnement standardisé des conteneurs système (CPU, RAM, VLAN tagging).
  - `modules/proxmox_vm` : Création de machines virtuelles pour les nœuds K3s avec injection de configurations Cloud-Init (clés SSH, réseaux statiques).

### 2. Configuration & Provisionnement Système (Ansible)
- **Gestion de l'Inventaire** : Utilisation d'un inventaire dynamique généré par script (`infra/scripts/inventory_snapshot.sh`) interrogeant l'API Proxmox pour mapper dynamiquement les hôtes selon leur état d'exécution et leur appartenance aux VLANs.
- **Rôles Applicatifs Déployés** :
  - `os_hardening` : Sécurisation de base des distributions Linux (désactivation SSH par mot de passe, configuration du pare-feu local UFW/NFTables, application des correctifs de sécurité).
  - `zfs_provisioning` : Configuration automatisée des pools de disques, application du layout RAID-Z2 (6 x 1 To sur le NAS) et monitoring de l'état de santé du pool via SMARTd.
  - `minio_setup` : Déploiement et sécurisation de l'instance S3 locale, configuration des politiques d'accès IAM et des compartiments (*buckets*) pour Terraform et les sauvegardes applicatives.
  - `automation_node` : Provisionnement de l'interface et du moteur d'automatisation Ansible Semaphore sur `pve1`.

### 3. Orchestration & Cycle Applicatif (K3s & Helm Controller)
- **Architecture GitOps Déclarative** : Le cluster K3s intègre nativement un contrôleur Helm permettant de piloter des déploiements applicatifs via des définitions de ressources personnalisées (*CRDs*) `HelmChart` et `HelmChartConfig`.
- **Workflow CI/CD Typique** :
  1. Modification du code d'une application ou d'une valeur de configuration dans le dépôt Git local.
  2. Déclenchement automatique du pipeline de validation : `helm lint` sur les configurations graphiques + validation syntaxique YAML.
  3. Packaging et publication automatique de la charte vers mon registre d'images privé (ou OCI-registry local).
  4. Mise à jour automatique ou manuelle du manifeste `HelmChart` CRD pointant vers la version exacte figée (*pinned version*) dans Git.
  5. Le contrôleur Helm interne à K3s détecte la modification, réconcilie l'état du cluster, applique les rollbacks automatiques en cas d'échec et journalise l'audit.
- **Sécurisation des Secrets** : Interdiction formelle de stocker des mots de passe, tokens ou clés privées en clair dans les fichiers `values.yaml`. L'injection s'effectue via des mécanismes d'intégration sécurisés comme Mozilla SOPS, Bitnami SealedSecrets ou une stack ExternalSecrets connectée à une instance Vault d'infrastructure.

---

## Procédure de bootstrap de l'infrastructure (Résumé pas-à-pas)

Le déploiement complet à partir de zéro respecte scrupuleusement l'ordonnancement de dépendances suivant :

1. **Étape 1 : Fondations Stockage** – Initialisation physique du NAS Bare-Metal, création du pool ZFS RAID-Z2 et amorçage du service S3 MinIO.
2. **Étape 2 : Initialisation du Backbone** – Installation et configuration de base de l'hyperviseur physique `pve1`. Déploiement initial des machines virtuelles critiques de routage de base (**OPNsense**), de centralisation des sauvegardes (**PBS**) et du conteneur d'orchestration (**Nœud d'automation**).
3. **Étape 3 : Initialisation Terraform** – Exécution de `terraform init` depuis le nœud d'automation pour connecter le state distant sur l'endpoint S3 `10.0.10.15:9000`.
4. **Étape 4 : Provisionnement Compute** – Exécution de `terraform apply` pour instancier les modules réseaux, assigner les VLANs et provisionner les VMs destinées au cluster K3s sur le stockage local des hôtes `pve2`, `pve3` et `pve4`.
5. **Étape 5 : Configuration Système** – Exécution des playbooks Ansible sur les nouvelles instances (OS Hardening, configuration réseau, configuration des variables d'environnement Ceph avec facteur de réplication à 3).
6. **Étape 6 : Amorçage Kubernetes & Ingress** – Bootstrapping automatique du cluster K3s Haute Disponibilité via Ansible. Une fois le control-plane opérationnel, le contrôleur Helm applique automatiquement les chartes d'infrastructure de base stockées dans le dépôt Git (Traefik v3, cert-manager, politiques réseau Calico et bouncers de sécurité CrowdSec).

---

## Stratégie de Récupération d'Urgence (Disaster Recovery)

En cas de perte matérielle totale ou de corruption sévère du cluster de calcul :
1. Reconstruction prioritaire du nœud physique `pve1` et du stockage de fichiers NAS.
2. Restauration des configurations réseau et du state Terraform depuis les sauvegardes distantes ou locales de MinIO.
3. Exécution de Terraform pour recréer instantanément les machines virtuelles à leur état nominal stable.
4. Redéploiement automatique de la pile logicielle complète via les pipelines GitOps. Les données persistantes critiques de production (bases de données, stockages d'applications complexes) sont réinjectées à partir des snapshots immuables gérés par Proxmox Backup Server (PBS) et répliqués de manière asynchrone sur les disques ZFS.

---

## Liens Utiles & Emplacements du Code

- **Dépôt Git Principal** : `https://github.com/richpea1982/infra-homelab`
- **Script d'Inventaire Dynamique** : `infra/scripts/inventory_snapshot.sh`
- **Définition Variables Cluster Stockage** : `infra/vars/ceph.yml` (paramètre cible : `replication: 3`, `min_size: 2`)


* **[Vue d'ensemble de l'infra](/infrastructure.md)** — Layout physique, ségrégation du backbone et inventaire matériel.
* **[Architecture Réseau](/networking.md)** — Règles de routage OPNsense, configurations VLAN et tunneling Zero-Trust.
* **[Services & Applications](/services.md)** — Topologie Kubernetes (K3s), routage d'ingress Traefik et cycle de vie des bases de données
* **[Modèle de Sécurité](/security.md)** — Analyse des menaces, parsing de logs CrowdSec et injection de secrets.
* **[Sauvegarde & Plan de Reprise](/backup-strategy.md)** — Implémentation de la règle 3-2-1, Proxmox Backup Server et réplication ZFS.

---

[← Accueil](/index.md)
