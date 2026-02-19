---
layout: default
title: Vue d'ensemble de l'infrastructure
nav_order: 2
---

<div style="text-align: right">
  <a href="/en/infrastructure.html">🇬🇧 English</a>
</div>

# Vue d'ensemble de l'infrastructure

**Résumé technique**
Deux serveurs physiques connectés via un switch managé, avec une VM 
VyOS gérant tout le routage interne et la segmentation VLAN. Toute 
l'infrastructure est provisionnée par Terraform et configurée par 
Ansible avec une intervention manuelle minimale.

---

## Architecture physique

| Équipement | Rôle | Connexion |
|------------|------|-----------|
| Routeur ISP | Accès internet + WiFi | — |
| TP-Link TL-SG108E | Switch managé | Uplink vers routeur ISP |
| proxmox-srv | Hôte de virtualisation | Port trunk (tous VLANs) |
| docker-srv | Hôte de conteneurs | Port access (VLAN10) |
```mermaid
flowchart LR
    ISP[Routeur ISP\n192.168.1.1]
    SW[Switch Managé\nTP-Link TL-SG108E]
    PVE[proxmox-srv\nIntel i5-6500]
    DOCK[docker-srv\nAMD A10]

    ISP -->|Uplink| SW
    SW -->|Trunk - tous VLANs| PVE
    SW -->|Access - VLAN10| DOCK
```

---

## Topologie réseau

Le trafic des quatre VLANs est routé par une VM VyOS fonctionnant 
sur proxmox-srv. VyOS gère le routage inter-VLAN et le NAT, en 
présentant une interface WAN unique au routeur ISP.

| VLAN | Sous-réseau | Usage |
|------|-------------|-------|
| VLAN10 | 10.10.0.0/24 | Management — proxmox-srv, docker-srv |
| VLAN20 | 10.20.0.0/24 | WordPress — conteneurs LXC |
| VLAN30 | 10.30.0.0/24 | Applications — Jellyfin, StirlingPDF, Immich |
| VLAN40 | 10.40.0.0/24 | Vaultwarden — gestionnaire de secrets isolé |
```mermaid
flowchart LR
    ISP[Routeur ISP\n192.168.1.1]

    subgraph VyOS["VM VyOS"]
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

    subgraph VLAN30["VLAN30 - Applications"]
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

## Flux de trafic

Tout le trafic externe entre par Cloudflare — aucun port entrant 
n'est ouvert sur le routeur ISP. Le tunnel Cloudflare tourne sur 
docker-srv et transmet le trafic à Traefik, qui route vers le service 
approprié par nom de domaine.
```mermaid
flowchart LR
    Internet[Utilisateur Internet]
    CF[Cloudflare Edge]
    TUN[Tunnel Cloudflare\ndocker-srv]
    TR[Traefik\ndocker-srv]
    WP[LXC WordPress\nVLAN20]
    SVC[Autres services\nVLAN30 / Docker]

    Internet -->|HTTPS| CF
    CF -->|Tunnel chiffré| TUN
    TUN --> TR
    TR -->|par domaine| WP
    TR -->|par domaine| SVC
```

---

## Résumé de la stack complète

| Couche | Technologie | Emplacement |
|--------|-------------|-------------|
| Réseau physique | TP-Link TL-SG108E | Matériel |
| Routage & NAT | VyOS | VM sur proxmox-srv |
| Virtualisation | Proxmox VE | proxmox-srv |
| Conteneurs | Docker | docker-srv |
| Charges LXC | Debian 13 LXCs | proxmox-srv |
| Provisionnement | Terraform | docker-srv (CLI) |
| Configuration | Ansible + Semaphore | docker-srv |
| Backend d'état | MinIO | docker-srv |
| Ingress | Cloudflare Tunnel + Traefik | docker-srv |
| Sécurité | CrowdSec + Bouncer | docker-srv |
| Monitoring | Prometheus + Grafana + Loki | docker-srv |
| Secrets | Ansible Vault | Dépôt Git (chiffré) |

---

*Cette vue d'ensemble reflète l'état actuel du homelab.
Les pages suivantes couvrent chaque couche en détail.*

---

[← Accueil](/fr/index.html) | **[Suivant : IaC & Automatisation →](/fr/iac-automation.html)**
