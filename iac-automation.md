---
layout: default
title: IaC & Automatisation
nav_order: 3
---

# IaC & Automatisation

Cette page décrit la stratégie d'Infrastructure as Code et les mécanismes d'automatisation utilisés pour provisionner et maintenir le homelab. Le dépôt Git `infra-homelab` constitue la **source unique de vérité**.

**Extrait autoritatif du dépôt :**  
"Cette page détaille la topologie physique et virtuelle du homelab. Le dépôt Git infra-homelab constitue la **source unique de vérité**."

---

## Objectif de la page

Fournir une vue opérationnelle et infra‑spécifique des composants IaC : backend Terraform, modules, playbooks Ansible, pipeline CI, procédure de bootstrap et stratégie de déploiement applicatif via Helm.

---

## Principes directeurs

- **GitOps** : déclarer l'état désiré dans Git ; pipelines CI valident et appliquent.  
- **Idempotence** : tous les rôles/ modules doivent être idempotents.  
- **Séparation des responsabilités** : outils d'automatisation hors du pool de calcul.

---

## Composants clés

**Backend Terraform**  
- Endpoint S3 : `10.0.10.15:9000` (MinIO sur NAS).  
- State locking : backend S3 + mécanisme de lock (documenter procédure de récupération).

**Ansible**  
- Rôles : OS hardening, ZFS provisioning, MinIO, PBS, OPNsense, déploiement Semaphore.  
- Inventaire dynamique : `infra/scripts/inventory_snapshot.sh`.

**Helm + K3s (intégration complète)**  
- **Helm v3** is the package manager for K3s; K3s includes a Helm controller that supports `HelmChart` CRDs for declarative installs.   
- **Recommended pattern** : publish charts (OCI or private chart repo), store `HelmChart` / `HelmRelease` manifests in Git, and let the Helm controller (or Flux/ArgoCD) reconcile the cluster. This provides reproducibility, rollbacks and audit trails.   
- **CI workflow** : `helm lint` → `helm package` → publish (OCI or chart repo) → update `HelmChart` CRD in Git → pipeline reconciles. Pin chart versions and values files in Git.   
- **Secrets** : do not store secrets in plain `values.yaml`. Use SealedSecrets, ExternalSecrets, or Vault integration. 

---

## Procédure de bootstrap (résumé)

1. Provisionner NAS + MinIO.  
2. Provisionner pve1 (OPNsense, PBS, automation node).  
3. `terraform init` (backend `10.0.10.15:9000`).  
4. Appliquer modules réseau / datastores.  
5. Déployer VMs K3s sur `local-lvm`.  
6. Déployer chart infra (Traefik, cert-manager, CrowdSec) via Helm controller / GitOps.

---

## Récupération d'urgence

Restaurer d'abord pve1 et NAS, récupérer l'état Terraform depuis MinIO, reprovisionner hôtes et réappliquer modules. PBS conserve les sauvegardes VM et réplique vers le NAS ZFS.

---

## Emplacements du code

- `https://github.com/richpea1982/infra-homelab`  
- Scripts : `infra/scripts/inventory_snapshot.sh` ; `infra/vars/ceph.yml` (replication: 3).

[← Accueil](/index.md)
