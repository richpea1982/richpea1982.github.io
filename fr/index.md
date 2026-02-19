---
layout: default
title: Accueil
nav_order: 1
---

<div style="text-align: right">
  <a href="/en/index.html">🇬🇧 English</a>
</div>

# Richard Pearsall
**Ingénierie Cloud | Infrastructure-as-Code | Linux | Automatisation**

[GitHub](https://github.com/richpea1982/homelab-infra) | 
[LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | 
[Email](rpearsall1982@gmail.com)

---

## Ce que j'ai construit

Un homelab fonctionnant sur deux serveurs physiques, conçu et géré 
entièrement sous forme de code avec une intervention manuelle minimale. 
Chaque composant — de la topologie réseau au déploiement applicatif — 
est provisionné par Terraform et configuré par Ansible, avec Semaphore 
comme pipeline CI/CD auto-hébergé.

L'infrastructure héberge des charges de travail réelles : trois sites 
WordPress en production avec du trafic réel, une stack d'observabilité 
complète, une gestion des secrets, et une architecture réseau 
zero-trust avec segmentation VLAN.

---

## Stack technique

| Couche | Technologie |
|--------|-------------|
| Provisionnement | Terraform + provider bpg/proxmox |
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
| proxmox-srv | Hôte de virtualisation | Intel i5-6500, 16 Go RAM |
| docker-srv | Hôte de conteneurs | AMD A10, 16 Go RAM |

Les deux serveurs sont des mini-PC grand public acquis d'occasion. 
La contrainte budgétaire est délibérée : l'objectif est de démontrer 
que des pratiques d'automatisation et de sécurité de qualité 
professionnelle peuvent être appliquées à coût minimal, sans chercher 
à reproduire du matériel d'entreprise.

---

## Philosophie de conception : points de défaillance uniques

Ce homelab comporte plusieurs points de défaillance uniques, 
assumés pleinement. Les charges de travail hébergées — sites personnels, 
médias, outils auto-hébergés — peuvent tolérer des heures ou des jours 
d'indisponibilité sans impact significatif.

La haute disponibilité introduit de la complexité : nœuds 
supplémentaires, gestion du quorum, état distribué, redondance réseau. 
Dans cet environnement, cette complexité dépasserait les bénéfices et 
élargirait la surface d'attaque.

La priorité est donnée à la sécurité et à la correction de 
l'automatisation plutôt qu'à la disponibilité. Un projet AWS 
complémentaire illustrera comment les mêmes patterns IaC s'appliquent 
à une architecture hautement disponible et multi-région.

---

## Ajouts prévus

- **StirlingPDF** — traitement PDF auto-hébergé
- **Jellyfin + stack *arr** — serveur média et acquisition automatisée
- **Immich** — gestion de photos auto-hébergée

---

## Pages du projet

- **[Vue d'ensemble de l'infrastructure](/fr/infrastructure.html)** — 
Architecture physique, topologie réseau, et schémas complets.
- **[IaC & Automatisation](/fr/iac-automation.html)** — Modules 
Terraform, rôles Ansible, pipelines Semaphore, et gestion d'état MinIO.
- **[Architecture réseau](/fr/networking.html)** — Routage VyOS, 
segmentation VLAN, ingress zero-trust Cloudflare, et enforcement CrowdSec.
- **[Services & Charges de travail](/fr/services.html)** — WordPress, 
stack de monitoring, Vaultwarden, et observabilité.
- **[Sécurité](/fr/security.html)** — Modèle de menace, couches de 
défense, gestion des secrets, et isolation VLAN.
- **[Sauvegarde & Reprise](/fr/backup-strategy.html)** — Stratégie 
3-2-1, snapshots Proxmox, PBS, Restic, et politique de rétention.

---

## Note sur la méthodologie

Ce projet a été construit et documenté avec l'assistance de Claude 
(Anthropic). L'IA a été utilisée comme outil collaboratif tout au long 
du projet — pour déboguer des erreurs Terraform, structurer des rôles 
Ansible, et rédiger la documentation. Toutes les décisions 
d'architecture, la conception de l'infrastructure, et la mise en œuvre 
sont les miennes.

Utiliser l'IA comme outil d'ingénierie reflète le fonctionnement des 
équipes infrastructure modernes : la compétence réside dans savoir quoi 
construire, comment évaluer le résultat, et comment l'adapter à ses 
besoins spécifiques.

---

*Ce site est mis à jour régulièrement au fil de l'évolution du homelab.*

---

**[Suivant : Vue d'ensemble de l'infrastructure →](/fr/infrastructure.html)**
