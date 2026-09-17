---
layout: default
title: IaC & automation
nav_order: 3
---

# IaC & automation

The entire infrastructure is managed declaratively.  
Two Git repositories keep responsibilities clearly separated.

| Repository | Role |
|------------|------|
| [infra-homelab](https://github.com/richpea1982/infra-homelab) | VM/LXC provisioning (Terraform) + OS configuration and K3s bootstrap (Ansible) |
| [k3s](https://github.com/richpea1982/k3s) | Application manifests and Helm charts managed by ArgoCD (GitOps) |

The README of the `infra-homelab` repository is the source of truth.

---

## Deployment chain

The workflow has three layers:

**1. Provisioning (Terraform)**  

- Creates VMs and LXCs on Proxmox (`bpg/proxmox` provider).  
- Injects SSH keys, network configuration (VLAN + static IP) and enables qemu-guest-agent.  
- State stored on MinIO (NAS).

**2. Configuration & bootstrap (Ansible)**  

- OS hardening, base packages, SSH configuration.  
- Deploys the K3s cluster (3 nodes, embedded etcd, kube-vip).  
- Installs ArgoCD and injects the SSH deploy key for the `k3s` repository.  
- Configures the NAS (ZFS, NFS, MinIO) and supporting roles (smartctl exporter, alert router…).

**3. GitOps (ArgoCD)**  

- Once the cluster and ArgoCD are up, Ansible no longer manages applications.  
- ArgoCD continuously reconciles the manifests from the `k3s` repository (Traefik, CrowdSec, monitoring, Cloudflare Tunnel, portfolio, Vaultwarden, Velero, etc.).

---

## Orchestration: Semaphore

Terraform plans and Ansible playbooks are executed from the **automation node** (pve1) via **Semaphore**.

- Semaphore is installed on this node.
- It runs the Terraform and Ansible jobs.
- Keys and secrets required **before** the Ansible vault can be unlocked (SSH keys, Proxmox credentials, etc.) are stored and managed inside Semaphore.
- Once the vault is accessible, remaining sensitive values are read from `ansible/group_vars/all/vault.yml` (Ansible Vault).

This split allows automation to start without depending on an already-decrypted vault.

---

## Bootstrap sequence (summary)

```bash
# 1. Provision the machines
cd terraform
terraform init
terraform plan
terraform apply

# 2. Configure the automation node and NAS
cd ../ansible
ansible-playbook -i hosts/hosts.ini playbooks/deploy_control_node.yml
ansible-playbook -i hosts/hosts.ini nas_setup.yml

# 3. Bootstrap the K3s cluster + ArgoCD
ansible-playbook -i hosts/hosts.ini deploy_k3s.yml
