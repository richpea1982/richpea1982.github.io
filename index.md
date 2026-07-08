---
layout: default
title: Accueil
nav_order: 1
---

# Richard Pearsall
**Ingénierie Cloud | Infrastructure-as-Code | Linux | Automatisation**

[GitHub](https://github.com/richpea1982/infra-homelab) | [LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | [Email](mailto:rpearsall1982@gmail.com) | [🇬🇧 English Version](/en/)

---

## Vision du Projet & Objectifs

Ce homelab n'est pas une simple infrastructure de commodité : c'est un environnement de test conçu selon les standards de l'ingénierie cloud moderne. Mon objectif est de démontrer qu'avec une rigueur méthodologique, des contraintes matérielles strictes peuvent être surmontées pour déployer une infrastructure sécurisée, entièrement reproductible et gérée exclusivement par le code (IaC).

Ici, aucune modification n'est faite "à la main". Tout élément, depuis les règles de routage réseau jusqu'à l'approvisionnement des machines et la gestion des configurations, est déclaré dans un dépôt Git et orchestré via un pipeline automatisé.

---

## Philosophie Architecturale & Choix d'Ingénierie

### 1. L'Évolution : De la Complexité Physique à la Consolidation Virtuelle
À l'origine, cette infrastructure reposait sur une topologie à 3 machines physiques distinctes, séparant strictement les hyperviseurs et s'appuyant sur un pare-feu physique OPNsense. 

Dans une démarche d'optimisation énergétique et de rationalisation des ressources (alignée sur un budget réaliste), j'ai fait le choix de faire évoluer cette architecture. La topologie actuelle consolide le calcul sur deux nœuds principaux et virtualise l'intelligence réseau au sein d'une instance de routage d'entreprise dédiée (**VyOS**). Ce choix démontre une compétence clé : savoir migrer et condenser une architecture tout en préservant une isolation réseau stricte par VLANs.

### 2. Le Compromis Réaliste : L'acceptation des SPOF (Points Uniques de Défaillance)
Une architecture professionnelle se définit par sa capacité à répondre à un besoin précis sans sur-ingénierie. Les services hébergés ici (sites Web actifs, outils de productivité, supervision) tolèrent un objectif de temps de récupération (RTO) de quelques heures. 

Ajouter de la Haute Disponibilité (HA) à ce stade introduirait une complexité logicielle disproportionnée (gestion de quorum, stockage distribué type Ceph, réplication synchrone de bases de données) qui consommerait les ressources matérielles au détriment des performances applicatives. La priorité a donc été donnée à la **rigueur de l'automatisation**, à la **sécurité Zero-Trust** et à une **stratégie de sauvegarde (3-2-1)** ultra-robuste. *La redondance et la haute disponibilité multi-zones feront l'objet d'un projet compagnon natif sur AWS.*

---

## Stack Technique Fondamentale

| Couche | Technologies de référence | Rôle Métier |
|-------|------------|-------------|
| **Provisioning** | Terraform (bpg/proxmox) | Déclaration immuable des VMs et conteneurs LXC |
| **Backend d'état** | MinIO (S3-compatible) | Centralisation et persistance des états Terraform |
| **Configuration** | Ansible & Semaphore | Gestion des états système et déploiements applicatifs CI/CD |
| **Virtualisation** | Proxmox VE | Hyperviseur de type 1 pour la gestion des ressources |
| **Routage & Sécurité** | VyOS (VM) | Routage inter-VLAN, NAT et pare-feu interne |
| **Ingress & Proxy** | Cloudflare Tunnel + Traefik v3 | Entrée Zero-Trust sans ports ouverts, routage par nom de domaine |
| **Observabilité** | Prometheus + Grafana + Loki | Collecte de métriques centralisée et agrégation de logs |

---

## Pages du Projet

* **[Vue d'ensemble de l'infra](/fr/infrastructure.html)** — Layout physique, topologie réseau et schémas d'architecture.
* **[IaC & Automatisation](/fr/iac-automation.html)** — Modules Terraform, rôles Ansible et pipelines Semaphore.
* **[Architecture Réseau](/fr/networking.html)** — Routage VyOS, segmentation VLAN et tunnel Cloudflare.
* **[Services & Applications](/fr/services.html)** — Déploiement de WordPress et de la stack de supervision.
* **[Sécurité](/fr/security.html)** — Modèle de menace, intégration CrowdSec et gestion des secrets.
* **[Sauvegarde & Reprise](/fr/backup-strategy.html)** — Stratégie 3-2-1, Proxmox Backup Server et Restic.

---

## Note sur la méthodologie & Parcours d'apprentissage

Ce projet sert de laboratoire personnel dans le cadre de ma formation système et réseau. Afin d'accélérer l'écriture des syntaxes de configuration, de valider la structure de mes rôles Ansible et de résoudre des erreurs complexes de déploiement Terraform, j'utilise activement les technologies d'Intelligence Artificielle générative comme outils de programmation pair à pair (*pair-programming*).

L'usage de l'IA a agi comme un multiplicateur de forces pour concevoir rapidement cette architecture globale. Ayant automatisé une grande partie de ces implémentations, je poursuis un travail quotidien d'ingénierie inverse (*reverse-engineering*) pour maîtriser les couches les plus profondes du système (mécanismes du noyau Linux, protocoles de routage avancés, sécurisation des flux). Je valorise l'honnêteté et la transparence technique au-dessus de tout : ce homelab est le reflet de mes ambitions, de ma capacité à piloter des architectures modernes, et le moteur principal de ma montée en compétences.

---

*Ce site est mis à jour régulièrement au fil de l'évolution du homelab.*

---

**[Suivant : Vue d'ensemble de l'infrastructure →](/fr/infrastructure.html)**
