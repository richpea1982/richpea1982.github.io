---
layout: default
title: Sécurité
nav_order: 6
---

# Sécurité

Cette page détaille la posture de sécurité, les mécanismes de protection des secrets, ainsi que le durcissement système et réseau implémentés à travers l'ensemble du homelab. La sécurité est pensée en profondeur (*Defense in Depth*) et gérée de manière déclarative.

---

## 1. Gestion des Secrets et Identifiants

La compromission des secrets est l'un des risques majeurs en infrastructure as code. Aucun mot de passe, token ou clé privée n'est stocké en clair dans le dépôt Git.

Le homelab applique une séparation stricte entre les secrets de provisioning et les secrets applicatifs run-time, plutôt qu'un coffre-fort centralisé unique :

* **Ansible Vault (couche provisioning)** : L'intégralité des variables sensibles utilisées lors du déploiement — jetons, mots de passe, clés — est stockée chiffrée en AES256 dans `ansible/group_vars/all/vault.yml`.
* **Sécurisation du Control Plane K3s** : Le jeton d'initialisation et de jonction du cluster (`vault_k3s_cluster_token`) est injecté de manière chiffrée par Ansible au moment du bootstrap, puis n'existe plus qu'en tant que secret Kubernetes natif à l'intérieur du cluster.
* **Sécurisation des Services d'Infrastructure** : Les identifiants de l'interface d'automatisation Semaphore (mots de passe administrateur, clés de chiffrement des cookies et des accès) ainsi que les accès administrateur à MinIO (`vault_minio_root_user` et `vault_minio_root_password`) sont tous gérés via ce même coffre-fort Ansible.
* **Secrets applicatifs (couche runtime K3s)** : Une fois à l'intérieur du cluster, les secrets consommés par les pods (variables d'environnement, certificats internes) restent gérés via l'objet **Kubernetes Secret** natif, sans intégration avec un service de vault externe (pas d'ExternalSecrets, pas de Vault HashiCorp connecté au cluster). Ce choix délibéré préserve un workflow K3s simple, natif et facilement auditable, sans dépendance supplémentaire sur le control plane.

## 2. Durcissement Système et Identités (OS Hardening)

L'accès aux machines physiques et virtuelles est strictement contrôlé dès la phase de provisionnement.

* **Authentification SSH Exclusive par Clé** : L'injection des utilisateurs et des configurations système lors de la création des VMs Proxmox s'effectue via Cloud-Init. Le module Terraform déploie le compte utilisateur `rich` en y associant automatiquement une clé SSH publique, éliminant ainsi le recours aux mots de passe.
* **Comptes de Services Restreints** : Les services exécutés en bare-metal ou hors-conteneur utilisent des comptes dédiés sans privilèges d'accès interactifs. Par exemple, l'utilisateur système `minio` est explicitement configuré avec le shell `/usr/sbin/nologin`.
* **Désactivation du Swap** : Pour garantir la stabilité et la sécurité des environnements Kubernetes, le swap est désactivé dynamiquement et de manière persistante (`/etc/fstab`) sur tous les nœuds via les playbooks de pré-requis Ansible.

## 3. Sécurité Réseau et Périmétrique

L'architecture réseau s'appuie sur une segmentation forte et une analyse comportementale au niveau de la couche applicative.

* **Micro-Segmentation (VLANs)** : Le trafic est cloisonné physiquement et logiquement entre le réseau de gestion (`VLAN 10`), le trafic interne K3s (`VLAN 20`), les services internes (`VLAN 30`) et les applications exposées publiquement (`VLAN 40`).
* **Politiques Réseau Kubernetes (CNI)** : Le cluster K3s utilise **Calico** comme plugin réseau (configuré avec le CIDR `10.42.0.0/16` et une encapsulation VXLAN), permettant l'application de politiques de flux strictes entre les pods.
* **Protection Active (CrowdSec)** : Un moteur de sécurité CrowdSec est déployé via Helm au sein du namespace `kube-system`. Il est spécifiquement configuré pour acquérir et analyser en temps réel les journaux du routeur d'ingress Traefik, bloquant ainsi les adresses IP malveillantes à la frontière du cluster.

## 4. Intégrité des Données d'Infrastructure

La protection de l'état de l'infrastructure (*State*) est critique pour éviter toute corruption ou modification non autorisée.

* **S3 Object Locking** : Le backend Terraform stockant l'état de l'infrastructure est hébergé sur le bucket MinIO `homelab-tf-state`. Ce bucket est provisionné automatiquement avec la fonctionnalité de verrouillage d'objets (`object_lock_enabled: true`) activée. Cela garantit l'intégrité des transactions Terraform et protège le fichier d'état contre les altérations involontaires ou malveillantes.

---

* **[Suivant : Sauvegarde & Reprise →](/backup-strategy.html)**
* **[← Accueil](/index.html)**
