---
layout: default
title: Security
nav_order: 6
---

# Security

Security is handled with a defence-in-depth approach and managed declaratively.  
No secrets are stored in clear text in the Git repositories.

---

## Secrets management

| Layer | Tool | Content |
|-------|------|---------|
| Pre-vault / bootstrap | **Semaphore** (automation node) | SSH keys, Proxmox credentials and secrets needed before the vault can be unlocked |
| Provisioning | **Ansible Vault** (`vault.yml`) | K3s tokens, MinIO passwords, Semaphore credentials, etc. |
| Runtime (cluster) | **Kubernetes Secrets** | Secrets consumed by pods (environment variables, internal certificates) |

- The K3s cluster token is injected encrypted at bootstrap and afterwards exists only as a Kubernetes secret.
- No HashiCorp Vault or ExternalSecrets for now — a deliberate choice for simplicity and auditability.

---

## System hardening

- SSH authentication **by key only** (injected via Cloud-Init / Terraform).
- Dedicated service accounts without interactive shells (e.g. `minio` user → `/usr/sbin/nologin`).
- Swap permanently disabled on all K3s nodes (Ansible prerequisite).
- Base packages and updates managed by playbooks.

---

## Network and perimeter security

| Layer | Mechanism | Role |
|-------|-----------|------|
| Perimeter | OPNsense (default drop) + VLANs | Strict inter-VLAN filtering |
| Public ingress | Cloudflare Tunnels only | No inbound ports open |
| Cluster (L3) | Calico NetworkPolicy | Pod micro-segmentation |
| Application (L7) | Traefik v3 + CrowdSec | Behavioural analysis and blocking of malicious requests |

The DMZ (VLAN 40) cannot initiate any connections toward internal networks.

---

## Terraform state integrity

- S3 backend (MinIO) with **object lock** enabled on the `homelab-tf-state` bucket.
- Protects the state against accidental or malicious deletion/modification.

---

## Key points

- Pre-vault secrets in Semaphore, remaining secrets in Ansible Vault, runtime secrets as Kubernetes Secrets.
- No public port-forwarding.
- Strong isolation between management plane, cluster and DMZ.
- CrowdSec reads Traefik logs and acts directly at the ingress layer.

---

**Related pages:**

- [← Networking](/en/networking.html)
- [Services →](/en/services.html)
- [Backup →](/en/backup-strategy.html)
