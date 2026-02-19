---
layout: default
title: Home
nav_order: 1
---

<div style="text-align: right">
  <a href="/fr/index.html">🇫🇷 Français</a>
</div>

# Richard Pearsall
**Cloud Engineering | Infrastructure-as-Code | Linux | Automation**

[GitHub](https://github.com/richpea1982/homelab-infra) | 
[LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | 
[Email](rpearsall1982@gmail.com)

---

## What I built

A homelab running on two physical servers, designed and managed as code 
with minimal manual intervention. Every component — from network topology 
to application deployment — is provisioned by Terraform and configured 
by Ansible, with Semaphore providing a self-hosted CI/CD pipeline for 
automated execution.

The infrastructure runs real workloads: three production WordPress sites 
serving live traffic, a full observability stack, secrets management, 
and a zero-trust network architecture with VLAN segmentation.

---

## Tech stack

| Layer | Technology |
|-------|------------|
| Provisioning | Terraform + bpg/proxmox provider |
| State backend | MinIO (S3-compatible) |
| Configuration | Ansible + Semaphore |
| Virtualisation | Proxmox VE |
| Routing | VyOS (VM) |
| Ingress | Cloudflare Tunnel + Traefik |
| Security | CrowdSec + Cloudflare Bouncer |
| Secrets | Ansible Vault |
| Monitoring | Prometheus + Grafana + Loki |
| Workloads | WordPress, Vaultwarden, Jellyfin, StirlingPDF, Immich |

---

## Hardware

| Host | Role | Specs |
|------|------|-------|
| proxmox-srv | Virtualisation host | Intel i5-6500, 16GB RAM |
| docker-srv | Container host | AMD A10, 16GB RAM |

Both servers are consumer-grade mini-PCs acquired second-hand. The 
hardware budget reflects a deliberate constraint: the goal is to 
demonstrate that product
