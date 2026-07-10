---
layout: default
title: Home
nav_order: 1
---

# Richard Pearsall
**Cloud Engineering | Infrastructure-as-Code | Linux | Network & Security**

[GitHub](https://github.com/richpea1982/infra-homelab) | [LinkedIn](https://www.linkedin.com/in/richard-pearsall-960392388) | [Email](mailto:rpearsall1982@gmail.com) | [🇫🇷 Version Française](/fr/)

---

## 🚀 My Journey: From Hospitality to Systems Engineering

After spending 20 years developing a strong work ethic in the restaurant industry and plumbing, I executed a professional career transition into IT. Currently undergoing **TSSR (Technicien Supérieur Systèmes et Réseaux) training at L’IDEM Le Soler**, I treat my homelab as a rigorous, production-grade staging environment to back up my technical skills with concrete implementations.

### 📈 The Evolution of My Homelab (Overcoming Hardware Constraints)

My current 5-machine architecture didn't appear overnight. It evolved through hands-on troubleshooting, changing requirements, and scaling hardware step-by-step:

* **Year 1 (The Spark):** Given an old laptop. Started experimenting with Linux CLI and dual-booting. This gave me a sense of complete control over computing and ignited my passion for IT.
* **Year 2 (First Production Challenge):** Built a self-hosted WordPress site for my wife’s business on the Google Cloud Platform (GCP) free tier. This was my first experience managing remote Linux instances.
* **Year 4 (Moving Local):** Acquired a mini-PC to self-host my wife's business site locally under **Proxmox VE**. Implemented a **Cloudflare Tunnel** for ingress, isolated workloads in LXCs, and deployed a basic monitoring stack and Vaultwarden.
* **Year 5 (The Automation Shift):** Added a second physical machine running bare-metal Docker. Moved the entry point, monitoring, and added Traefik for internal routing alongside apps like Seafile, Jellyfin, and Immich. **Milestone:** Rebuilt the entire lab from scratch using **Terraform and Ansible** (IaC).
* **Year 5.5 (Network Segmentation):** Introduced a manageable switch and a **VyOS** virtual appliance to implement strict inter-VLAN routing and isolation.
* **Year 6 (Consolidation & Cluster Build):** Acquired an extra laptop and an old desktop PC. To optimize my physical resource utilization, I phased out VyOS and replaced it with **OPNsense** virtualized inside Proxmox. This allowed me to split my environment into a highly resilient management backbone and a high-availability compute cluster.

---

## 🧠 Engineering Philosophy & Learning Method

> "Design a rock-solid infrastructure backbone before building the cluster on top."

* **GitOps & Total Automation:** I enforce a strict "no manual changes" rule. From provisioning hypervisor VMs to declaring firewall rules, everything is committed to Git and executed via code.
* **Pragmatic Defensively Isolated Control Planes:** A cluster is only as stable as the infrastructure supporting it. I deliberately isolated my core network routing (OPNsense), backups (PBS), and automation engines onto a dedicated machine completely outside the cluster pool[cite: 5]. This ensures I never suffer from a "chicken-and-egg" loop where a cluster failure brings down my deployment tools or core network access[cite: 5].
* **Generative AI Posture:** I actively use GenAI as a pair-programming partner to speed up tedious configuration syntaxes (YAML, HCL). However, I reverse-engineer every block: I never deploy automated configurations unless I can manually troubleshoot, secure, and explain the underlying system mechanics (Linux kernel flags, CNI routing tables).

---

## 🛠️ Current Production Architecture - 5 Nodes (In active developement)

My environment is divided into a production compute tier running directly on top of a highly resilient management backbone:

### 1. The Core Backbone (2 Independent Nodes)
* **Infrastructure Core Node (pve1):** Deliberately left outside the main compute cluster[cite: 5]. It runs a virtualized **OPNsense** firewall acting as the routing entry point for the entire network, a **Proxmox Backup Server (PBS)** instance, and my **automation control node** for provisioning and configuring the environment[cite: 5].
* **Storage Server (Bare-Metal NAS):** A dedicated bare-metal desktop PC acting as a high-capacity NAS, provisioned with a **6-drive ZFS RAID-Z2 array** (6 x 1TB HDDs)[cite: 1]. This layout provides a 2-disk fault tolerance layout for 4TB of usable data protection[cite: 1].

### 2. The Compute Tier (3-Node Hyperconverged Cluster)
* **Proxmox VE + Ceph Cluster:** Composed of 3 physical hypervisor nodes running an embedded **Ceph distributed storage pool** for stateful components[cite: 4]. High Availability (HA) groups ensure that my wife's public e-commerce sites (`hantaweb` WordPress + WooCommerce) live-migrate automatically in the event of host failure[cite: 4].
* **Containerized Microservices:** A highly available, 3-node **K3s Kubernetes cluster** running embedded Etcd multi-master replication to host stateless apps and internal platforms[cite: 3, 4].

### 3. Network & Security Architecture
* **Zero-Trust Entry Point:** Inbound connections are handled via Cloudflare Tunnels with no open external ports[cite: 4]. 
* **Micro-Segmentation:** Traffic is strictly segregated into dedicated management, DMZ, and internal VLANs[cite: 5]. Within the Kubernetes cluster, **Calico CNI** network policies enforce strict pod isolation, while a **Traefik v3** ingress controller routes traffic through local **CrowdSec** bouncers for layer-7 HTTP protection[cite: 2, 4].

---

## 📁 Deep Dive Into The Projects

* **[Infrastructure Overview](/en/infrastructure.html)** — Physical layout, backbone segregation, and hardware inventory[cite: 5].
* **[IaC & Automation](/en/iac-automation.html)** — Terraform modules, Ansible roles, and bootstrap orchestration pipelines[cite: 2, 4].
* **[Network Architecture](/en/networking.html)** — OPNsense routing rules, VLAN configurations, and Zero-Trust tunneling[cite: 4, 5].
* **[Services & Applications](/en/services.html)** — Kubernetes (K3s) layout, Traefik ingress routing, and database lifecycles[cite: 2, 4].
* **[Security Model](/en/security.html)** — Threat modeling, CrowdSec log parsing, and secret injection[cite: 4].
* **[Backup & Disaster Recovery](/en/backup-strategy.html)** — 3-2-1 rule implementation, Proxmox Backup Server, and ZFS replication[cite: 1, 5].

---

**[Next: Infrastructure Overview →](/en/infrastructure.html)**
