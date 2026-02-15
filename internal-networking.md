# Internal Networking & Service Isolation
**[github](https://github.com/richpea1982/homelab-infra)**
**Technical summary**  
Services are segmented across two physical hosts—Docker‑SRV and Proxmox‑SRV—using frontend/backend Docker networks and VLANs on Proxmox. Traefik remains the single ingress point, ensuring consistent access control and preventing direct exposure of backend components.

## At a glance
- **Traefik as single ingress** for both Docker and Proxmox workloads.  
- **Docker segmentation:** Frontend network for public UIs; backend network for exporters, logging, and monitoring.  
- **Proxmox segmentation:** VLANs by function (WordPress, Apps, Vaultwarden) to limit lateral movement.  
- **Backend isolation:** Monitoring and system services are internal-only and not reachable from the public internet.
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
## Diagram explanation
1. **Traefik receives inbound requests.** It routes only to frontend networks or VLANs that are explicitly allowed.  
2. **Docker‑SRV Frontend network.** Hosts UIs such as Grafana and Prometheus UI that Traefik can reach.  
3. **Docker‑SRV Backend network.** Hosts Prometheus core, node exporters, cAdvisor, Loki, and Promtail; these are isolated from Traefik and the public internet.  
4. **Proxmox‑SRV VLANs.** VLAN 10 for WordPress sites; VLAN 20 for general apps (Jellyfin, StirlingPDF); VLAN 30 for Vaultwarden (isolated).  
5. **Controlled access.** Frontend UIs query backend services over internal networks only; Traefik never exposes backend endpoints directly.

## Operational notes
- **Network policy:** Enforce strict firewall rules between VLANs and Docker networks; allow only necessary ports and protocols.  
- **Least privilege routing:** Only allow Traefik to reach frontend containers; use mTLS or service accounts for backend communication where feasible.  
- **Secrets handling:** Keep Vaultwarden on an isolated VLAN; never store backup keys on public networks.  
- **Testing:** Periodically verify backend services are unreachable from the public internet and validate Traefik routing rules.

## Benefits summary
- Reduces blast radius by isolating services by function.  
- Keeps monitoring and logging systems internal and protected.  
- Provides a clear, auditable ingress and access path.

**[Next page](https://richpea1982.github.io/backup-strategy.html)** | **[Home](https://richpea1982.github.io/index.html)** | **[github](https://github.com/richpea1982/homelab-infra)**
