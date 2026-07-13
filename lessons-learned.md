---
layout: default
title: Rétrospective & Leçons Apprises
nav_order: 5
---

# Rétrospective Technique & Leçons Apprises

Cette page fait office de journal d'ingénierie. Elle regroupe les anomalies réelles, les conflits de dépendances et les erreurs de configuration rencontrés lors du développement et du déploiement de ce homelab, ainsi que les correctifs appliqués. 

Documenter ces résolutions démontre une compréhension approfondie du comportement interne des outils (Terraform, Ansible, Kubernetes) face aux contraintes réelles du matériel et des systèmes d'exploitation.

---

## 🚀 Résolution des Anomalies (Session Log)

### 1. Incompatibilité Microarchitecturale des Conteneurs (SIGILL Faults)
* **Symptôme** : Plusieurs images de conteneurs modernes crashaient instantanément au démarrage avec une erreur de type `SIGILL` (Illegal Instruction) sur les nœuds K3s.
* **Cause** : Par défaut, Terraform provisionnait les VMs Proxmox avec un type de processeur virtuel standard `kvm64`. Or, les chaînes de compilation récentes (notamment la bibliothèque `glibc`) intègrent une base de référence exigeant l'architecture CPU `x86-64-v2`. Le processeur `kvm64` masque les instructions modernes requises.
* **Correctif** : Modification du module Terraform personnalisé pour forcer le mode passthrough : `cpu.type = "host"`. Les conteneurs exploitent désormais directement le jeu d'instructions natif des processeurs physiques sous-jacents (Intel i5, AMD A10, i7).

### 2. Rupture de Compatibilité des CRDs Calico (Moteur de Validation CEL)
* **Symptôme** : L'application des manifests du CNI Calico version `3.32.0` échouait lors de la création des Custom Resource Definitions (CRDs).
* **Cause** : Les règles de validation internes des CRDs de Calico 3.32 exploitent la fonction CEL (Common Expression Language) `isCIDR()`. Cette fonction nécessite une API Kubernetes en version `1.31` minimum. Le cluster K3s était initialement figé sur la version `1.30.2`.
* **Correctif** : Alignement et mise à niveau de la variable globale de version K3s vers la version stable `v1.31.5+k3s1` dans les `group_vars` d'Ansible, résolvant instantanément le conflit syntaxique.

### 3. Masquage de Variables Réseau (VLAN Threading)
* **Symptôme** : Les nœuds du cluster K3s se voyaient attribuer des adresses IP hors-contexte et se retrouvaient positionnés par défaut sur le réseau natif non segmenté de Proxmox.
* **Cause** : Bien que la variable `vlan_id` ait été correctement déclarée dans le fichier `.auto.tfvars` et typée dans les déclarations de variables globales de Terraform, elle n'était pas passée au bloc de ressources final du module de création de VM.
* **Correctif** : Correction de la liaison au sein du code Terraform en insérant explicitement l'argument `vlan_id = var.vlan_id` dans la déclaration de l'interface réseau du module.

### 4. Conflit d'Environnement Python sous Debian (PEP 668)
* **Symptôme** : Le playbook Ansible de bootstrap échouait sur le nœud d'automation lors de la tentative d'installation du paquet python `kubernetes-client` via `pip`.
* **Cause** : Protection stricte introduite par la directive PEP 668 sous Debian 12 (Externally Managed Environment). Le système bloque l'usage de `pip install` global pour éviter d'écraser les paquets gérés par le gestionnaire système `apt` (notamment `PyYAML`).
* **Correctif** : Isolation du processus en configurant Ansible pour exécuter ses tâches Kubernetes au sein d'un environnement virtuel dédié (`virtualenv`) ou en s'appuyant de préférence sur les paquets packagés par la distribution (`python3-kubernetes`).

### 5. Obsolescence de Chart Helm (CrowdSec Lifecycle)
* **Symptôme** : La synchronisation ArgoCD du pod de sécurité CrowdSec restait bloquée à l'état *Degraded* ou renvoyait une erreur de récupération d'artefact HTTP 404.
* **Cause** : Le manifest GitOps pointait vers une version historique du chart Helm (`0.9.3`) qui avait été purgée et archivée définitivement des dépôts amonts du fournisseur.
* **Correctif** : Ajustement du fichier de déclaration GitOps `gitops/apps/crowdsec.yaml` pour cibler la révision stable `0.24.0` du chart, restaurant immédiatement la boucle de réconciliation applicative.

---

## 🔁 Optimisations d'Automatisation et Idempotence

Au-delà des corrections de bugs bloquants, des refactorisations ont été menées pour fiabiliser la robustesse des scripts de déploiement :

* **Vérification d'État vs Présence de Fichiers** : Le playbook Ansible d'initialisation de K3s utilisait un test basé sur la présence de fichiers locaux (`stat`) pour déterminer si le cluster était déjà initialisé. Ce mécanisme masquait les installations partielles ou corrompues. Il a été remplacé par une inspection dynamique de l'état du service systemd (`systemctl is-active --quiet k3s`).
* **Instabilité des Clés d'Hôtes SSH** : Chaque reconstruction de VM via Terraform générait une nouvelle paire de clés d'hôte SSH, provoquant des blocages de sécurité lors des exécutions Ansible suivantes (*Host key verification failed*). Intégration d'une tâche idempotente de nettoyage automatique et de rafraîchissement des clés via `ssh-keyscan` dans la phase préliminaire du playbook.
* **Gestion des Secrets Vault Interrompue** : Erreur d'intégration de la variable `vault_k3s_cluster_token`. Le rôle Ansible avait été mis à jour pour exiger un jeton de sécurité fort partagé pour l'etcd, mais le coffre chiffré Ansible Vault associé n'avait pas été provisionné avec cette nouvelle clé. La variable a été canonicalisée et documentée dans le fichier `vault.yml`.

---

## 📈 Enseignements Clés pour les Projets Futurs

1. **La Frontière des Outils** : Bien délimiter où s'arrête un orchestrateur et où commence le suivant est vital. Confier le déploiement applicatif à ArgoCD et la configuration OS à Ansible (plutôt que de surcharger Ansible avec des modules Helm complexes) simplifie radicalement le débogage.
2. **Ne Jamais Faire Confiance aux Paramètres par Défaut** : Les profils de virtualisation génériques (comme le CPU `kvm64` ou l'absence de tag VLAN) créent des comportements imprévisibles en couches hautes. Valider les configurations matérielles et réseau au plus bas niveau possible avant de monter l'orchestrateur.

---

* **[← Retour à l'Accueil](/index.html)**
---
