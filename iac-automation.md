---
layout: default
title: IaC et automatisation
nav_order: 3
---

# IaC et automatisation

Toute l’infrastructure est gérée de manière déclarative.  
Deux dépôts Git séparent clairement les responsabilités.

| Dépôt | Rôle |
|-------|------|
| [infra-homelab](https://github.com/richpea1982/infra-homelab) | Provisioning des VMs/LXCs (Terraform) + configuration OS et bootstrap K3s (Ansible) |
| [k3s](https://github.com/richpea1982/k3s) | Manifests applicatifs et charts Helm gérés en GitOps par ArgoCD |

Le README du dépôt `infra-homelab` est la source de vérité.

---

## Chaîne de déploiement

Le workflow se déroule en trois couches :

**1. Provisioning (Terraform)**  

- Crée les VMs et LXC sur Proxmox (module `bpg/proxmox`).  
- Injecte les clés SSH, la configuration réseau (VLAN + IP statique) et active qemu-guest-agent.  
- State stocké sur MinIO (NAS).

**2. Configuration et bootstrap (Ansible)**  

- Durcissement OS, paquets de base, configuration SSH.  
- Déploiement du cluster K3s (3 nœuds, etcd embarqué, kube-vip).  
- Installation d’ArgoCD + injection de la deploy key SSH vers le dépôt `k3s`.  
- Configuration du NAS (ZFS, NFS, MinIO) et des rôles annexes (smartctl exporter, alert router…).

**3. GitOps (ArgoCD)**  

- Une fois le cluster et ArgoCD en place, Ansible n’intervient plus sur les applications.  
- ArgoCD synchronise en continu les manifests du dépôt `k3s` (Traefik, CrowdSec, monitoring, Cloudflare Tunnel, portfolio, Vaultwarden, Velero, etc.).

---

## Orchestration : Semaphore

L’exécution des plans Terraform et des playbooks Ansible est centralisée sur le **nœud d’automatisation** (pve1) via **Semaphore**.

- Semaphore est installé sur ce nœud.
- Il lance les jobs Terraform et Ansible.
- Les clés et secrets nécessaires **avant** l’accès au vault Ansible (clés SSH, credentials Proxmox, etc.) sont stockés et gérés dans Semaphore.
- Une fois le vault accessible, les secrets sensibles restants sont lus depuis `ansible/group_vars/all/vault.yml` (Ansible Vault).

Cette séparation permet de démarrer l’automatisation sans dépendre d’un vault déjà déchiffré.

---

## Séquence de bootstrap (résumé)

```bash
# 1. Provisioning des machines
cd terraform
terraform init
terraform plan
terraform apply

# 2. Configuration du nœud d’automatisation et du NAS
cd ../ansible
ansible-playbook -i hosts/hosts.ini playbooks/deploy_control_node.yml
ansible-playbook -i hosts/hosts.ini nas_setup.yml

# 3. Bootstrap du cluster K3s + ArgoCD
ansible-playbook -i hosts/hosts.ini deploy_k3s.yml

Les détails et points d’attention (token de cluster, deploy key, vérifications post-install) sont documentés dans
ansible/roles/k3s_cluster/K3S-BOOTSTRAP-README.md.

Haute disponibilité de l’API K3s

ÉlémentValeurVIP (kube-vip)10.0.20.20:6443Nœudsk3s-pve2 (10.0.20.21), k3s-pve3 (10.0.20.22), k3s-pve4 (10.0.20.23)
kube-vip gère une adresse virtuelle flottante en mode ARP.
Si le nœud leader tombe, la VIP bascule automatiquement sur un autre nœud du control-plane.

Principes retenus

Séparation stricte entre infrastructure de base (infra-homelab) et charges applicatives (k3s).
Ansible pour le bootstrap uniquement ; GitOps pour tout ce qui suit.
Secrets pré-vault gérés dans Semaphore ; secrets applicatifs dans Ansible Vault.
Plan de gestion (Semaphore, PBS, OPNsense) isolé du cluster de calcul.
Aucune modification manuelle durable : tout repasse par Git.


Statut (août 2026)


ÉtapeÉtatTerraform (VMs / LXC)OpérationnelAnsible (OS + NAS + bootstrap K3s)OpérationnelSemaphore (orchestration)OpérationnelArgoCD + GitOpsEn cours de validation et stabilisationMigration complète des services dans K3sProgressive

Pages liées :

← Vue d’ensemble de l’infrastructure
Réseau → · Sécurité →
Services →
