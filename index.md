---
layout: default
title: Accueil
nav_order: 1
---

## 🚀 Mon Parcours : De la Restauration à l'Ingénierie Système

Après 20 ans d'expérience terrain dans la restauration et la plomberie, j'ai opéré une reconversion professionnelle vers l'informatique. Actuellement en formation **TSSR (Technicien Supérieur Systèmes et Réseaux) à l'IDEM Le Soler**, j'utilise mon homelab comme un environnement de staging rigoureux et de niveau production pour valider mes compétences techniques par des implémentations concrètes.

### 📈 L'Évolution de mon Homelab (L'école de la contrainte matérielle)

Mon architecture actuelle à 5 machines n'est pas née du jour au lendemain. Elle a évolué étape par étape, au fil des résolutions de pannes, de l'évolution des besoins et de la mise à l'échelle du matériel :

* **An 1 (Le Déclic) :** Récupération d'un vieux PC portable. Début des expérimentations avec la CLI Linux et le dual-boot. C'est là que j'ai découvert le contrôle total du système et que ma passion pour l'IT est née.
* **An 2 (Le Premier Défi en Production) :** Déploiement d'un site WordPress auto-hébergé pour l'entreprise de ma femme sur le free tier de Google Cloud Platform (GCP). Ma première expérience de gestion d'instances Linux distantes.
* **An 4 (Le Passage au Local) :** Achat d'un mini-PC pour héberger le site de ma femme localement sous **Proxmox VE**. Mise en place d'un **Tunnel Cloudflare** pour les flux entrants, isolation des charges de travail dans des LXC, et déploiement d'une stack de monitoring basique et de Vaultwarden.
* **An 5 (Le Virage de l'Automatisation) :** Ajout d'une deuxième machine physique sous Docker bare-metal. Migration du point d'entrée, du monitoring, et intégration de Traefik pour le routage interne aux côtés d'applications comme Seafile, Jellyfin et Immich. **Étape majeure :** Reconstruction complète du lab à partir de zéro via **Terraform et Ansible** (IaC).
* **An 5.5 (La Segmentation Réseau) :** Intégration d'un switch administrable et d'une appliance virtuelle **VyOS** pour implémenter un routage inter-VLAN et une isolation stricts.
* **An 6 (Consolidation & Construction du Cluster) :** Récupération d'un ordinateur portable supplémentaire et d'un ancien PC fixe. Pour optimiser l'utilisation de mes ressources matérielles, j'ai abandonné VyOS au profit d'un pare-feu **OPNsense** virtualisé dans Proxmox. Cela m'a permis de scinder mon environnement entre un backbone de gestion hautement résilient et un cluster de calcul en haute disponibilité.

---

## 🧠 Philosophie d'Ingénierie & Méthode d'Apprentissage

> "Concevoir une infrastructure de base solide comme un roc avant de construire un cluster par-dessus."

* **GitOps & Automatisation Totale :** J'applique une règle stricte : "aucune modification manuelle". Du provisionnement des VMs de l'hyperviseur à la déclaration des règles de pare-feu, tout est poussé sur Git et exécuté via le code.
* **Isolation Défensive et Pragmatique du Control Plane :** Un cluster n'est stable que si l'infrastructure qui le soutient l'est aussi. J'ai délibérément isolé mon routage réseau cœur (OPNsense), mes sauvegardes (PBS) et mes moteurs d'automatisation sur une machine dédiée, totalement en dehors du cluster. Cela me protège du problème de "l'œuf et la poule", où une panne de cluster ferait s'effondrer mes outils de déploiement ou mes accès réseau principaux.
* **Posture face à l'IA Générative :** J'utilise activement l'IA comme un partenaire de *pair-programming* pour accélérer l'écriture des syntaxes de configuration répétitives (YAML, HCL). Cependant, je pratique l'**ingénierie inverse** sur chaque bloc : je ne déploie jamais de configurations automatisées sans être capable de dépanner, sécuriser et expliquer manuellement les mécanismes système sous-jacents (paramètres du noyau Linux, tables de routage CNI).

---

## 🛠️ Architecture de Production Actuelle - 5 Nœuds (En développement actif)

Mon environnement est divisé en une couche de calcul de production qui s'exécute directement au-dessus d'un backbone de gestion hautement résilient :

### 1. Le Cœur de l'Infrastructure (2 Nœuds Indépendants)
* **Nœud Cœur Infra (pve1) :** Délibérément maintenu hors du cluster de calcul principal. Il héberge un pare-feu virtualisé **OPNsense** (point d'entrée de routage de tout le réseau), une instance **Proxmox Backup Server (PBS)**, et mon **nœud de contrôle d'automatisation** qui provisionne et configure l'environnement.
* **Serveur de Stockage (NAS Bare-Metal) :** Un PC fixe dédié faisant office de NAS haute capacité, équipé d'un **pool ZFS RAID-Z2 de 6 disques (6 x 1 To)**. Cette configuration offre une tolérance de panne de 2 disques pour 4 To de stockage utile hautement protégé.

### 2. La Couche de Calcul (Cluster Hyperconvergé à 3 Nœuds)
* **Cluster Proxmox VE + Ceph :** Composé de 3 nœuds hyperviseurs physiques exécutant un **pool de stockage distribué Ceph** pour les composants stateful. Cette stack soutient deux sites WordPress en production : `petitsanglais`, un petit site que j'ai construit et que je maintiens moi-même, et `hantaweb`, une boutique e-commerce WooCommerce complète que j'héberge et administre au niveau de l'infrastructure (la conception et le développement applicatif étant gérés par le propriétaire du site).
* **Microservices Conteneurisés :** Un cluster **Kubernetes K3s à 3 nœuds** avec Etcd embarqué, actuellement en cours de finalisation. Je valide actuellement le bootstrapping des nœuds du control-plane en HA et les comportements d'auto-guérison (*self-healing*) avant d'y migrer mes applications stateless et mes plateformes internes.

### 3. Architecture Réseau & Sécurité
* **Point d'Entrée Zero-Trust :** Les connexions entrantes sont gérées via des Tunnels Cloudflare, sans aucun port ouvert sur l'extérieur.
* **Micro-Segmentation :** Le trafic est strictement segmenté en VLANs dédiés (Management, DMZ, Production Interne). Au sein du cluster Kubernetes, les politiques réseau de **Calico CNI** imposent une isolation stricte des pods, tandis qu'un contrôleur d'ingress **Traefik v3** route le trafic à travers des bouncers **CrowdSec** locaux pour une protection HTTP à la couche 7.

---

## 🎯 Statut du Déploiement & Méthodologie

L'architecture présentée sur ce site représente l'**état final cible désiré**. Chaque étape du déploiement a été entièrement pensée, mûrie et planifiée en amont avant l'écriture du moindre bloc de code. L'état réel et actuel de la configuration de mon homelab est directement traçable dans mon [dépôt Git infra-homelab](https://github.com/richpea1982/infra-homelab). Cette approche rigoureuse standardise l'ensemble de mon documentation ; cette précision méthodologique s'applique par défaut à toutes les sections suivantes.

---

## 📁 Zoom sur les Projets

* **[Vue d'ensemble de l'infra](/infrastructure.md)** — Layout physique, ségrégation du backbone et inventaire matériel.
* **[IaC & Automatisation](/iac-automation.md)** — Modules Terraform, rôles Ansible et pipelines d'orchestration de déploiement.
* **[Architecture Réseau](/networking.md)** — Règles de routage OPNsense, configurations VLAN et tunneling Zero-Trust.
* **[Services & Applications](/services.md)** — Topologie Kubernetes (K3s), routage d'ingress Traefik et cycle de vie des bases de données.
* **[Modèle de Sécurité](/security.md)** — Analyse des menaces, parsing de logs CrowdSec et injection de secrets.
* **[Sauvegarde & Plan de Reprise](/backup-strategy.md)** — Implémentation de la règle 3-2-1, Proxmox Backup Server et réplication ZFS.

---

**[Suivant : Vue d'ensemble de l'infrastructure →](/fr/infrastructure.html)**
