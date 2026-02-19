---
layout: default
title: Architecture réseau
nav_order: 4
---

<div style="text-align: right">
  <a href="/en/networking.html">🇬🇧 English</a>
</div>

# Architecture réseau

**Résumé technique**
Une VM VyOS gère tout le routage interne et le NAT entre quatre VLANs 
isolés. Le trafic externe entre exclusivement via un tunnel Cloudflare — 
aucun port entrant n'est ouvert sur le routeur ISP.

---

## Ingress zero-trust
```mermaid
flowchart LR
    USER[Utilisateur Internet]
    CF[Cloudflare Edge]
    TUN[Tunnel Cloudflare]
    TR[Traefik]
    CS[CrowdSec]
    CB[Cloudflare Bouncer]

    USER -->|HTTPS| CF
    CF -->|Tunnel chiffré| TUN
    TUN --> TR
    TR -->|Analyse des logs| CS
    CS -->|Décisions de blocage| CB
    CB -->|Enforcement edge| CF
```

Aucun port n'est forwardé sur le routeur ISP. L'IP publique du 
homelab n'est jamais exposée. Tout le trafic passe par l'intelligence 
mondiale des menaces de Cloudflare avant d'atteindre le réseau.

---

## Segmentation VLAN

| VLAN | Sous-réseau | Usage |
|------|-------------|-------|
| VLAN10 | 10.10.0.0/24 | Management |
| VLAN20 | 10.20.0.0/24 | LXCs WordPress |
| VLAN30 | 10.30.0.0/24 | Applications |
| VLAN40 | 10.40.0.0/24 | Vaultwarden — isolé |
```mermaid
flowchart TD
    VYOS[VM VyOS]

    subgraph VLAN10["VLAN10 — Management"]
        PVE[proxmox-srv 10.10.0.12]
        DOCK[docker-srv 10.10.0.11]
    end

    subgraph VLAN20["VLAN20 — WordPress"]
        WP1[richweb 10.20.0.10]
        WP2[petitsanglais 10.20.0.11]
        WP3[esperance 10.20.0.12]
        WP4[hantaweb 10.20.0.13]
    end

    subgraph VLAN30["VLAN30 — Applications"]
        APP[Jellyfin / StirlingPDF / Immich]
    end

    subgraph VLAN40["VLAN40 — Vaultwarden"]
        VW[Vaultwarden]
    end

    VYOS --- VLAN10
    VYOS --- VLAN20
    VYOS --- VLAN30
    VYOS --- VLAN40
```

Les sites WordPress sont le vecteur d'attaque le plus probable — 
code PHP public avec plugins tiers. Les placer sur VLAN20 signifie 
qu'un site compromis ne peut pas atteindre l'infrastructure de 
management ou Vaultwarden sans passer par les règles de pare-feu VyOS.

---

## Switch managé

| Port | Mode | Affectation |
|------|------|-------------|
| Port 1 | Uplink | Routeur ISP (non tagué) |
| Port 2 | Trunk | proxmox-srv (tous VLANs tagués) |
| Port 3 | Access | docker-srv (VLAN10 non tagué) |

---

## Améliorations prévues

- Règles de pare-feu inter-VLAN sur VyOS
- Serveur DHCP par VLAN sur VyOS

---

[← IaC & Automatisation](/fr/iac-automation.html) | 
**[Suivant : Services & Charges de travail →](/fr/services.html)**
