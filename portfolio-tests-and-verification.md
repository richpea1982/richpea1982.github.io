---
layout: default
title: Reprise après sinistre
nav_order: 4
lang: fr
---

# Reprise après sinistre : reconstruction du cluster et restauration etcd

Une architecture documentée et un scénario de reprise réellement exécuté sont deux choses distinctes. Cette page relate le test d'infrastructure le plus complet mené sur ce homelab : la reconstruction des nœuds K3s associée à une restauration etcd depuis un snapshot S3. Il a été déclenché par un incident Terraform, pas par un exercice planifié.

## Ce qui s'est passé

Un `terraform apply` destiné à cibler un seul nœud a en réalité détruit et recréé les trois VMs du cluster. La cause racine n'est pas confirmée : l'hypothèse retenue est un problème d'échappement shell avec l'option `-target` appliquée à un module indexé par `for_each`. HashiCorp documente `-target` comme un outil de dernier recours, pas comme un mécanisme de ciblage sûr. C'est la leçon la plus durable de cet épisode, davantage que l'erreur de commande elle-même.

L'incident a transformé une maintenance ordinaire en test de reprise après sinistre non planifié : reconstruire les nœuds avec Terraform et restaurer l'état du cluster depuis le snapshot etcd le plus récent, stocké sur un espace objet compatible S3.

## Parcours de récupération

1. **Reconstruction des nœuds** — les trois VMs ont été recréées à partir du module Terraform existant.
2. **Restauration etcd** — le premier nœud a été restauré depuis le snapshot le plus récent avec le mécanisme natif de k3s ; les deux autres ont rejoint le cluster restauré.
3. **Friction après restauration** — un snapshot etcd restaure l'état des objets Kubernetes, mais pas l'état des nœuds. Les machines recréées n'avaient pas tous les paquets système nécessaires, et le stockage Ceph conservait des verrous obsolètes liés à l'identité des anciens nœuds. Ces deux points ont dû être traités avant que les charges dépendantes, notamment un volume NFS de bibliothèque photo, ne redeviennent saines.
4. **Automatisation partielle** — le rôle Ansible existant a correctement configuré l'hôte, mais il ne disposait pas de chemin de code pour le cas « restauration depuis un snapshot etcd ». Cette étape et le nettoyage des verrous ont été réalisés manuellement. Il s'agit désormais d'un chantier identifié et cadré, pas d'une lacune dissimulée.
5. **Vérification** — contrôle de l'état des pods du plan de contrôle et des principales applications après récupération. Certains contrôles, comme l'état complet des nœuds, d'ArgoCD ou des verrous Ceph, n'ont pas été formalisés dans un compte rendu exhaustif ; ils font partie des améliorations identifiées.

## Pourquoi cela compte

Avoir une sauvegarde correctement configurée et disposer d'une sauvegarde réellement restaurable en conditions réelles sont deux affirmations différentes. Une vérification antérieure avait confirmé que les snapshots etcd étaient produits correctement ; cet incident est la première fois qu'une restauration complète a été menée jusqu'à un cluster fonctionnel. La preuve est d'autant plus solide qu'elle n'était pas planifiée.

## Limites assumées

Ce test couvre la reconstruction des nœuds et la restauration de l'état Kubernetes. Il ne démontre pas la récupération de tous les volumes Ceph dans tous les scénarios, ni l'absence de verrous résiduels après une panne multiple. Le cas d'une partition réseau avec plusieurs nœuds hors service reste à tester.

## Suite identifiée

- Ajouter au rôle Ansible de bootstrap un chemin dédié à la restauration depuis un snapshot.
- Automatiser le nettoyage des verrous de stockage obsolètes.
- Documenter un exercice de reprise reproductible, cette fois planifié, avec une liste de vérification complète.

## Pages liées

- [← IaC et automatisation](/iac-automation.html)
- [Accueil](/)
