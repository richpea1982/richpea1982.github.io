---
layout: default
title: Sécurité
nav_order: 6
---

# Sécurité

La sécurité est traitée en profondeur (défense en profondeur) et de manière déclarative.  
Aucun secret n’est stocké en clair dans les dépôts Git.

---

## Gestion des secrets

| Couche | Outil | Contenu |
| -------- | ------- | --------- |
| Pré-vault / bootstrap | **Semaphore** (nœud d’automatisation) | Clés SSH, credentials Proxmox et secrets nécessaires avant le déchiffrement du vault |
| Provisioning | **Ansible Vault** (`vault.yml`) | Jetons K3s, mots de passe MinIO, credentials Semaphore, etc. |
| Runtime (cluster) | **Kubernetes Secrets** | Secrets consommés par les pods (variables d’environnement, certificats internes) |

- Le jeton de cluster K3s est injecté chiffré au bootstrap, puis n’existe plus que comme secret Kubernetes.
- Pas de HashiCorp Vault ni d’ExternalSecrets pour l’instant : choix volontaire de simplicité et d’auditabilité.

---

## Durcissement système

- Authentification SSH **uniquement par clé** (injectée via Cloud-Init / Terraform).
- Comptes de service dédiés sans shell interactif (ex. utilisateur `minio` → `/usr/sbin/nologin`).
- Swap désactivé de façon persistante sur tous les nœuds K3s (prérequis Ansible).
- Mises à jour et paquets de base gérés par les playbooks.

---

## Sécurité réseau et périmétrique

| Couche | Mécanisme | Rôle |
| -------- | ----------- | ------ |
| Périmètre | OPNsense (default drop) + VLANs | Filtrage inter-VLAN strict |
| Ingress public | Cloudflare Tunnels uniquement | Aucun port ouvert en entrée |
| Cluster (L3) | Calico NetworkPolicy | Micro-segmentation des pods |
| Application (L7) | Traefik v3 + CrowdSec | Analyse comportementale et blocage des requêtes malveillantes |

La DMZ (VLAN 40) ne peut initier aucune connexion vers les réseaux internes.

---

## Intégrité de l’état Terraform

- Backend S3 (MinIO) avec **object lock** activé sur le bucket `homelab-tf-state`.
- Protège le state contre les suppressions ou modifications accidentelles / malveillantes.

---

## Points clés

- Secrets pré-vault dans Semaphore, reste dans Ansible Vault, runtime en Kubernetes Secrets.
- Aucun port-forwarding public.
- Isolation forte entre plan de gestion, cluster et DMZ.
- CrowdSec lit les logs Traefik et agit directement au niveau de l’ingress.

---

**Pages liées :**

- [← Architecture réseau](/networking.html)
- [Services →](/services.html)
- [Sauvegarde →](/backup-strategy.html)
