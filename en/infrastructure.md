---
layout: default
title: Infrastructure Overview
nav_order: 2
---

<div style="text-align: right">
  <a href="/fr/infrastructure.html">🇫🇷 Français</a>
</div>

# Infrastructure Overview

**Technical summary**
Two physical servers connected via a managed switch, with a VyOS router 
VM handling all internal routing and VLAN segmentation. All 
infrastructure is provisioned by Terraform and configured by Ansible 
with minimal manual intervention.

---

## Physical layout

| Device | Role | Connection |
|--------|------|------------|
| ISP Router | Internet uplink + WiFi | — |
| TP-Link TL-SG108E | Managed switch | Uplink to ISP router |
| proxmox-srv | Virtualisation host | Trunk port (all VLANs) |
| docker-srv | Container host | Access port (VLAN10) |
```mermaid
flowchart LR
    ISP[ISP Router\n192.168.1.1]
    SW[Managed Switch\nTP-Link TL-SG108E]
    PVE[proxmox-srv\nIntel i5-6500]
    DOCK[docker-srv\nAMD A10]

    ISP -->|Uplink| SW
    SW -->|Trunk - all VLANs| PVE
    SW -->|Access - VLAN10| DOCK
```

---

## Network topology

Traffic from all four VLANs is routed by a VyOS VM running on 
proxmox-srv. VyOS handles inter-VLAN routing and NAT, presenting 
a single WAN interface to the ISP router.

| VLAN | Subnet | Purpose |
|------|--------|---------|
| VLAN10 | 10.10.0.0/24 | Management — proxmox-srv, docker-srv |
| VLAN20 | 10.20.0.0/24 | WordPress — LXC containers |
| VLAN30 | 10.30.0.0/24 | Apps — Jellyfin, StirlingPDF, Immich |
| VLAN40 | 10.40.0.0/24 | Vaultwarden — isolated secrets manager |
```mermaid
flowchart LR
    ISP[ISP Router\n192.168.1.1]

    subgraph VyOS["VyOS Router VM"]
        WAN[eth0 - WAN\n192.168.1.2]
        V10[eth1.10\n10.10.0.1]
        V20[eth1.20\n10.20.0.1]
        V30[eth1.30\n10.30.0.1]
        V40[eth1.40\n10.40.0.1]
    end

    subgraph VLAN10["VLAN10 - Management"]
        PVE[proxmox-srv\n10.10.0.12]
        DOCK[docker-srv\n10.10.0.11]
    end

    subgraph VLAN20["VLAN20 - WordPress"]
        WP1[richweb\n10.20.0.10]
        WP2[petitsanglais\n10.20.0.11]
        WP3[esperance\n10.20.0.12]
        WP4[hantaweb\n10.20.0.13]
    end

    subgraph VLAN30["VLAN30 - Apps"]
        APP1[Jellyfin]
        APP2[StirlingPDF]
        APP3[Immich]
    end

    subgraph VLAN40["VLAN40 - Vaultwarden"]
        VW[Vaultwarden]
    end

    ISP --- WAN
    V10 --- VLAN10
    V20 --- VLAN20
    V30 --- VLAN30
    V40 --- VLAN40
```

---

## Traffic flow

All external traffic enters through Cloudflare — no inbound ports 
are open on the ISP router. The Cloudflare tunnel runs on docker-srv 
and forwards traffic to Traefik, which routes to the appropriate 
service by domain name.
```mermaid
flowchart LR
    Internet[Internet User]
    CF[Cloudflare Edge]
    TUN[Cloudflare Tunnel\ndocker-srv]
    TR[Traefik\ndocker-srv]
    WP[WordPress LXC\nVLAN20]
    SVC[Other Services\nVLAN30 / Docker]

    Internet -->|HTTPS| CF
    CF -->|Encrypted tunnel| TUN
    TUN --> TR
    TR -->|by domain| WP
    TR -->|by domain| SVC
```

---

## Full stack summary

| Layer | Technology | Where it runs |
|-------|------------|---------------|
| Physical network | TP-Link TL-SG108E | Hardware |
| Routing & NAT | VyOS | VM on proxmox-srv |
| Virtualisation | Proxmox VE | proxmox-srv |
| Containers | Docker | docker-srv |
| LXC workloads | Debian 13 LXCs | proxmox-srv |
| Provisioning | Terraform | docker-srv (CLI) |
| Configuration | Ansible + Semaphore | docker-srv |
| State backend | MinIO | docker-srv |
| Ingress | Cloudflare Tunnel + Traefik | docker-srv |
| Security | CrowdSec + Bouncer | docker-srv |
| Monitoring | Prometheus + Grafana + Loki | docker-srv |
| Secrets | Ansible Vault | Git repo (encrypted) |

---

*This overview reflects the current state of the homelab. 
Pages below cover each layer in detail.*

---

[← Home](/en/index.html) | **[Next: IaC & Automation →](/en/iac-automation.html)**
