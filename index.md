---
layout: default
title: Home
nav_order: 1
---

# Richard Pearsall
**Cloud Engineering | Automation | Linux | Infrastructure-as-Code**

This site documents my homelab: a work‑in‑progress platform where I design, build, and validate reproducible, modular, and scalable infrastructure patterns aligned with current best practices. The current project may not fully achieve every goal yet; it is updated regularly as I iterate and improve.

## Goals (concise)
- Build **reproducible** infrastructure and automation.  
- Keep designs **modular** so components can be replaced or scaled independently.  
- Demonstrate **scalability** and alignment with modern best practices.  
- Validate security‑first networking, IaC, and resilient backups in a small, practical environment.

## Current limitations
- Hardware: **two 4‑core, 16 GB RAM mini‑PCs**.  
- There are multiple single points of failure; this is acceptable for my non‑critical homelab workloads.  
- Services can tolerate downtime of a few hours or days.  
- I plan to add an **AWS project** to demonstrate how the architecture can scale and reduce single points of failure.

---

### Project pages
* **[Edge Networking](edge-networking.md)** — Zero‑trust ingress, Cloudflare Tunnel, Traefik, and layered enforcement.  
* **[Internal Networking](internal-networking.md)** — Host segmentation, Docker networks, Proxmox VLANs, and service isolation.  
* **[Backup Strategy](backup-strategy.md)** — 3‑2‑1 backup approach, Proxmox snapshots, PBS, Restic, and retention policy.

---

This site is updated regularly as the homelab evolves.  

---

### 📬 Contact
[GitHub](https://github.com/richpea1982) | [LinkedIn](www.linkedin.com/in/richard-pearsall-960392388) | [Email](rpearsall1982@gmail.com)
