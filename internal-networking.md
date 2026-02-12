# Internal Networking & Service Isolation

This section provides a high‑level overview of how internal services are segmented across the homelab. The goal is to ensure that applications remain isolated, backend components are never exposed, and each environment is separated by clear network boundaries. This structure reduces the blast radius of any compromise and keeps critical services protected.

---

## Network Segmentation Overview

All traffic entering the homelab is routed through Traefik, which acts as the single ingress point. From there, services are divided across two physical hosts—Docker‑SRV and Proxmox‑SRV—each with their own internal isolation model. Docker services are separated into frontend and backend networks, while Proxmox workloads are segmented by VLANs. This ensures that only intended interfaces are reachable and that sensitive systems remain isolated.

```mermaid
flowchart LR

    %% ============================
    %% Entry Point Layer
    %% ============================
    subgraph Entry["Ingress Layer"]
        Traefik[Traefik Reverse Proxy]
    end

    %% ============================
    %% Docker Host
    %% ============================
    subgraph Docker_SRV["Docker-SRV (Container Host)"]

        %% Frontend Network
        subgraph Docker_Frontend["Frontend Network"]
            Grafana_UI[Grafana UI]
            Prometheus_UI[Prometheus UI]
        end

        %% Backend Network
        subgraph Docker_Backend["Backend Network (Isolated)"]
            Prometheus_Core[Prometheus Core]
            Node_Exporter[Node Exporter]
            cAdvisor[cAdvisor]
            Loki[Loki]
            Promtail[Promtail]
        end

    end

    %% ============================
    %% Proxmox Host
    %% ============================
    subgraph Proxmox_SRV["Proxmox-SRV (Virtualization Host)"]

        subgraph VLAN10["VLAN 10 - WordPress Sites"]
            WP1[Site-1 Nginx]
            WP2[Site-2 Nginx]
        end

        subgraph VLAN20["VLAN 20 - Apps"]
            Jellyfin[Jellyfin]
            StirlingPDF[StirlingPDF]
        end

        subgraph VLAN30["VLAN 30 - Vaultwarden (Isolated)"]
            Vaultwarden[Vaultwarden]
        end

    end

    %% ============================
    %% Traffic Flow
    %% ============================
    Traefik -->|Frontend Net| Grafana_UI
    Traefik -->|Frontend Net| Prometheus_UI

    Grafana_UI -->|Metrics Queries| Prometheus_Core
    Prometheus_UI -->|Metrics Queries| Prometheus_Core

    Prometheus_Core --> Node_Exporter
    Prometheus_Core --> cAdvisor
    Promtail --> Loki

    %% ============================
    %% Proxmox Routing
    %% ============================
    Traefik -->|VLAN 10| WP1
    Traefik -->|VLAN 10| WP2

    Traefik -->|VLAN 20| Jellyfin
    Traefik -->|VLAN 20| StirlingPDF

    Traefik -->|VLAN 30| Vaultwarden
```

---

## Docker Service Isolation

Docker‑SRV uses two internal networks to separate public‑facing interfaces from backend components. Only frontend services are reachable through Traefik, while backend systems such as exporters, log processors, and monitoring cores remain fully isolated. This ensures that internal data pipelines and system‑level metrics are never exposed externally.

---

## Proxmox VLAN Segmentation

Proxmox‑SRV uses VLANs to isolate workloads by function. WordPress sites, general applications, and Vaultwarden each operate on their own dedicated VLAN. This prevents lateral movement between services and ensures that sensitive applications remain isolated from less trusted workloads.

---

## Controlled Access Through Traefik

All internal services—whether running on Docker or Proxmox—are accessed exclusively through Traefik. This enforces a consistent security boundary and ensures that only approved interfaces are reachable. Backend components and isolated VLANs remain inaccessible unless explicitly routed.

