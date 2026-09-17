---
layout: default
title: Architecture réseau
nav_order: 4
---

# Architecture réseau

Le réseau est construit autour d’une segmentation stricte et d’une politique « default drop ».  
Aucun port n’est ouvert en entrée sur l’IP publique.

---

## Segmentation (VLANs)

| VLAN | Sous-réseau     | Usage                                      |
|------|-----------------|--------------------------------------------|
| 10   | 10.0.10.0/24    | Management (Proxmox, PBS, automation, NAS, OPNsense) |
| 20   | 10.0.20.0/24    | Cluster K3s et services internes           |
| 30   | 10.0.30.0/24    | Média (Jellyfin, etc.)                     |
| 40   | 10.0.40.0/24    | DMZ – VMs web publiques (WordPress)        |
| 50   | 10.0.50.0/24    | Untrusted / lab                            |

---

## Pare-feu (OPNsense)

OPNsense tourne en VM sur **pve1** (hors cluster de calcul).  
Il assure le routage inter-VLAN et applique les règles de filtrage.

### Règles principales

| Source              | Destination       | Politique                                      |
|---------------------|-------------------|------------------------------------------------|
| VLAN 10 (MGMT)      | Tous les VLANs    | Autorisé (administration)                      |
| VLAN 20 (K3s)       | NAS (MinIO)       | Autorisé uniquement ports 9000/9001            |
| VLAN 40 (DMZ)       | Réseau interne    | **Strict drop** (aucun accès sortant vers l’interne) |
| Tous les VLANs      | Internet (WAN)    | Autorisé (NAT) sur ports standards (80, 443, 53…) |

Toute communication non explicitement autorisée est rejetée.

---

## Accès distant

Deux canaux distincts, aucun port-forwarding sur la box ISP.

**1. Services publics**  

- Cloudflare Tunnels uniquement.  
- Le démon `cloudflared` tourne dans le cluster K3s et établit une connexion sortante.  
- Le trafic arrive sur Traefik après inspection Cloudflare (WAF / DDoS).

**2. Accès d’administration**  

- WireGuard (self-hosted).  
- Connexion sortante depuis OPNsense vers un VPS relais (contournement CGNAT).  
- Seuls les flux d’administration authentifiés sont autorisés vers les VLANs 10 et 20.

---

## Isolation supplémentaire

| Couche              | Outil              | Rôle                                              |
|---------------------|--------------------|---------------------------------------------------|
| Réseau (L3/L4)      | OPNsense           | Filtrage inter-VLAN et default drop               |
| Cluster (L3)        | Calico NetworkPolicy | Micro-segmentation des pods                     |
| Application (L7)    | Traefik + CrowdSec | Analyse des requêtes HTTP et blocage comportemental |

---

## Points clés

- Aucun port ouvert en entrée sur l’IP publique.
- La DMZ (VLAN 40) ne peut pas initier de connexions vers l’interne.
- Le plan de gestion (VLAN 10) est le seul autorisé à administrer le reste.
- Cloudflare Tunnels + WireGuard remplacent tout port-forwarding classique.

---

**Pages liées :**

- [← IaC et automatisation](/iac-automation.html)
- [Sécurité →](/security.html)
- [Services →](/services.html)
