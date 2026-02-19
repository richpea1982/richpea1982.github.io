---
layout: default
title: Home
nav_order: 1
---

# Richard Pearsall
**Cloud Engineering | Infrastructure-as-Code | Linux | Automation**

[GitHub](https://github.com/richpea1982/homelab-infra) | 
[LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | 
[Email](rpearsall1982@gmail.com) | 
[🇫🇷 Version française](/fr/)

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
demonstrate that production-quality automation and security practices 
can be applied at minimal cost, not to replicate enterprise hardware.

---

## Design philosophy: single points of failure

This homelab has multiple single points of failure and makes no 
apology for it. The workloads it runs — personal websites, media, 
and self-hosted tools — can tolerate hours or days of downtime without 
meaningful impact.

High availability introduces complexity: additional nodes, quorum 
management, distributed state, and network redundancy. For this 
environment that complexity would outweigh the benefit and create 
a larger attack surface.

The priority here is security and automation correctness over 
availability. A future AWS companion project will demonstrate how 
the same IaC patterns scale to a highly available, multi-region 
architecture where HA is genuinely warranted.

---

## Planned additions

- **StirlingPDF** — self-hosted PDF tooling
- **Jellyfin + *arr stack** — media server and automated acquisition
- **Immich** — self-hosted photo management

---

## Project pages

- **[Infrastructure Overview](/infrastructure.html)** — Physical layout,
network topology, and full architecture diagrams.
- **[IaC & Automation](/iac-automation.html)** — Terraform modules,
Ansible roles, Semaphore pipelines, and MinIO state management.
- **[Network Architecture](/networking.html)** — VyOS routing, VLAN
segmentation, Cloudflare zero-trust ingress, and CrowdSec enforcement.
- **[Services & Workloads](/services.html)** — WordPress, monitoring
stack, Vaultwarden, and observability.
- **[Security](/security.html)** — Threat model, defence layers,
secrets management, and VLAN isolation.
- **[Backup & Recovery](/backup-strategy.html)** — 3-2-1 strategy,
Proxmox snapshots, PBS, Restic, and retention policy.

---

## A note on methodology

This project was built and documented with the assistance of Claude 
(Anthropic). AI was used as a collaborative tool throughout — helping 
debug Terraform errors, structure Ansible roles, and draft 
documentation. All architecture decisions, infrastructure design, and 
implementation were my own.

Using AI as an engineering tool reflects how modern infrastructure teams 
work: the skill is knowing what to build, how to evaluate the output, 
and how to adapt it to your specific requirements.

---

*This site is updated regularly as the homelab evolves.*

---

**[Next: Infrastructure Overview →](/infrastructure.html)**
