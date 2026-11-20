---
layout: default
title: IaC et automatisation
nav_order: 3
lang: fr
---

# IaC et automatisation

L'infrastructure est gérée de manière déclarative. Le code est réparti entre deux dépôts privés, avec une responsabilité distincte ; leurs noms et leurs liens ne sont pas publiés ici.

| Dépôt privé | Responsabilité |
| --- | --- |
| Infrastructure | Provisionnement des VMs et LXC avec Terraform, configuration système et bootstrap K3s avec Ansible. |
| Charges applicatives | Manifests Kubernetes et charts Helm synchronisés par ArgoCD. |

## Chaîne de déploiement

### 1. Provisionnement avec Terraform

- Création des VMs et LXC sur Proxmox via le provider dédié.
- Injection des clés SSH, configuration réseau (VLAN et adresses statiques), activation de l'agent invité.
- State conservé sur un stockage objet compatible S3 hébergé sur le NAS.
- Les renommages de ressources passent par des blocs `moved` afin d'éviter la recréation accidentelle de machines.

### 2. Configuration et bootstrap avec Ansible

- Durcissement système, paquets de base, configuration SSH.
- Déploiement du cluster K3s : trois nœuds, etcd embarqué, adresse API virtuelle.
- Installation d'ArgoCD et mise en place du suivi GitOps.
- Configuration du NAS (ZFS, NFS, stockage objet) et des rôles annexes : supervision SMART, routeur d'alertes.
- Une option fait échouer un playbook lorsqu'aucun hôte ne correspond au motif fourni, pour éviter qu'une faute de frappe ne soit silencieusement ignorée.

### 3. GitOps avec ArgoCD

- Une fois le socle en place, Ansible n'intervient plus sur les applications.
- ArgoCD synchronise en continu les manifests : routage, sécurité, supervision, sauvegarde, portfolio et services internes.
- Les versions de charts Helm sont épinglées pour rendre les changements prévisibles.

## Orchestration avec Semaphore

Les plans Terraform et les playbooks Ansible sont lancés depuis un nœud d'automatisation isolé via Semaphore. Les secrets nécessaires avant l'ouverture du coffre Ansible — clés SSH et accès à l'hyperviseur — sont gérés dans Semaphore ; les secrets applicatifs restent chiffrés dans Ansible Vault. Cette séparation permet de démarrer l'automatisation sans dépendre d'un coffre déjà déchiffré.

## Redondance de l'API Kubernetes

Les trois nœuds forment le plan de contrôle avec etcd embarqué. Une adresse IP virtuelle, portée par kube-vip en mode ARP, donne accès à l'API Kubernetes : si le nœud qui la porte tombe, l'adresse bascule sur un autre nœud du plan de contrôle. Cette redondance ne couvre pas automatiquement les charges de travail, qui dépendent du stockage et des sauvegardes.

## Principes retenus

- Séparation stricte entre infrastructure de base et charges applicatives.
- Ansible pour le bootstrap uniquement, GitOps pour tout ce qui suit.
- Secrets pré-coffre gérés par Semaphore, secrets applicatifs chiffrés dans Ansible Vault.
- Plan de gestion isolé du cluster de calcul.
- Aucune modification manuelle durable : tout repasse par Git.

## État d'avancement (novembre 2026)

| Étape | État |
| --- | --- |
| Terraform (VMs et LXC) | Opérationnel |
| Ansible (système, NAS, bootstrap K3s) | Opérationnel |
| Semaphore (orchestration) | Opérationnel |
| ArgoCD et GitOps | En place, généralisation en cours |
| Migration complète des services vers K3s | Progressive |

## Pages liées

- [← Vue d'ensemble de l'infrastructure](/infrastructure.html)
- [Accueil](/)
- [Reprise après sinistre →](/portfolio-tests-and-verification.html)
- [Projet Groupemen →](/groupemen.html)
