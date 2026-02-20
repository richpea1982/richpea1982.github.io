---
layout: default
title: Accueil
nav_order: 1
---

# Richard Pearsall
**Cloud Engineering | Infrastructure-as-Code | Linux | Automatisation**

[GitHub](https://github.com/richpea1982/homelab-infra) | 
[LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | 
[Email](rpearsall1982@gmail.com) | 
[🇬🇧 Version anglaise](/en/)

---

## Ce que j'ai construit

Un homelab fonctionnant sur deux serveurs physiques, conçu et géré en tant que code avec une intervention manuelle minimale. Chaque composant — de la topologie réseau au déploiement des applications — est provisionné par Terraform et configuré par Ansible, avec Semaphore fournissant une pipeline CI/CD auto-hébergée pour l'exécution automatisée.

L'infrastructure exécute des charges de travail réelles : trois sites WordPress en production servant du trafic en direct, une pile complète d'observabilité, la gestion des secrets, et une architecture réseau zero-trust avec segmentation VLAN.

---

## Stack technique

| Couche | Technologie |
|--------|-------------|
| Provisioning | Terraform + provider bpg/proxmox |
| Backend d'état | MinIO (compatible S3) |
| Configuration | Ansible + Semaphore |
| Virtualisation | Proxmox VE |
| Routage | VyOS (VM) |
| Ingress | Cloudflare Tunnel + Traefik |
| Sécurité | CrowdSec + Cloudflare Bouncer |
| Secrets | Ansible Vault |
| Monitoring | Prometheus + Grafana + Loki |
| Charges de travail | WordPress, Vaultwarden, Jellyfin, StirlingPDF, Immich |

---

## Matériel

| Hôte | Rôle | Spécifications |
|------|------|----------------|
| proxmox-srv | Hôte de virtualisation | Intel i5-6500, 16GB RAM |
| docker-srv | Hôte de conteneurs | AMD A10, 16GB RAM |

Les deux serveurs sont des mini-PC grand public achetés d'occasion. Le budget matériel reflète une contrainte délibérée : l'objectif est de démontrer que des pratiques d'automatisation et de sécurité de qualité production peuvent être appliquées à moindre coût, et non de reproduire du matériel d'entreprise.

---

## Philosophie de conception: points de défaillance uniques

Ce homelab comporte plusieurs points de défaillance uniques et n'en fait aucune excuse. Les charges de travail qu'il exécute — sites personnels, médias et outils auto-hébergés — peuvent tolérer des heures ou des jours d'indisponibilité sans impact significatif.

La haute disponibilité introduit de la complexité : nœuds supplémentaires, gestion de quorum, état distribué et redondance réseau. Pour cet environnement, cette complexité l'emporterait sur le bénéfice et créerait une surface d'attaque plus grande.

La priorité ici est la sécurité et la justesse de l'automatisation plutôt que la disponibilité. Un futur projet compagnon sur AWS démontrera comment les mêmes modèles IaC s'étendent à une architecture hautement disponible et multi-région où la HA est réellement justifiée.

---

## Ajouts prévus

- **StirlingPDF** — outils PDF auto-hébergés
- **Jellyfin + *arr stack** — serveur multimédia et acquisition automatisée
- **Immich** — gestion de photos auto-hébergée

---

## Pages du projet

- **[Aperçu de l'infrastructure](/infrastructure.html)** — Disposition physique, topologie réseau et diagrammes d'architecture complets.
- **[IaC et automatisation](/iac-automation.html)** — Modules Terraform, rôles Ansible, pipelines Semaphore et gestion d'état MinIO.
- **[Architecture réseau](/networking.html)** — Routage VyOS, segmentation VLAN, ingress zero-trust Cloudflare et application de CrowdSec.
- **[Services et charges de travail](/services.html)** — WordPress, pile de monitoring, Vaultwarden et observabilité.
- **[Sécurité](/security.html)** — Modèle de menace, couches de défense, gestion des secrets et isolation VLAN.
- **[Sauvegarde et récupération](/backup-strategy.html)** — Stratégie 3-2-1, snapshots Proxmox, PBS, Restic et politique de rétention.

---

## Remarque sur la méthodologie

Ce projet a été construit et documenté avec l'assistance de Claude (Anthropic). L'IA a été utilisée comme outil collaboratif tout au long du processus — aidant à déboguer des erreurs Terraform, structurer des rôles Ansible et rédiger la documentation. Toutes les décisions d'architecture, la conception de l'infrastructure et la mise en œuvre sont de mon fait.

Utiliser l'IA comme outil d'ingénierie reflète la façon dont les équipes d'infrastructure modernes travaillent : la compétence consiste à savoir quoi construire, comment évaluer la sortie et comment l'adapter à vos exigences spécifiques.

---

*Ce site est mis à jour régulièrement au fur et à mesure de l'évolution du homelab.*

---

**[Suivant: Aperçu de l'infrastructure →](/infrastructure.html)**
