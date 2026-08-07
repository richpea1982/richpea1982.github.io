
---

### `en/networking.md`

```markdown
---
layout: default
title: Networking
nav_order: 4
---

# Networking

The network is built around strict segmentation and a default-drop policy.  
No ports are opened inbound on the public IP.

---

## Segmentation (VLANs)

| VLAN | Subnet          | Purpose                                      |
|------|-----------------|----------------------------------------------|
| 10   | 10.0.10.0/24    | Management (Proxmox, PBS, automation, NAS, OPNsense) |
| 20   | 10.0.20.0/24    | K3s cluster and internal services            |
| 30   | 10.0.30.0/24    | Media (Jellyfin, etc.)                       |
| 40   | 10.0.40.0/24    | DMZ – public web VMs (WordPress)             |
| 50   | 10.0.50.0/24    | Untrusted / lab                              |

---

## Firewall (OPNsense)

OPNsense runs as a VM on **pve1** (outside the compute cluster).  
It handles inter-VLAN routing and filtering rules.

### Main rules

| Source              | Destination       | Policy                                           |
|---------------------|-------------------|--------------------------------------------------|
| VLAN 10 (MGMT)      | All VLANs         | Allowed (administration)                         |
| VLAN 20 (K3s)       | NAS (MinIO)       | Allowed only on ports 9000/9001                  |
| VLAN 40 (DMZ)       | Internal networks | **Strict drop** (no outbound access to internal) |
| All VLANs           | Internet (WAN)    | Allowed (NAT) on standard ports (80, 443, 53…)  |

Any traffic not explicitly permitted is dropped.

---

## Remote access

Two separate channels; no port-forwarding on the ISP router.

**1. Public services**  
- Cloudflare Tunnels only.  
- The `cloudflared` daemon runs inside the K3s cluster and opens an outbound connection.  
- Traffic reaches Traefik after Cloudflare inspection (WAF / DDoS).

**2. Administrative access**  
- WireGuard (self-hosted).  
- Outbound connection from OPNsense to a relay VPS (bypasses CGNAT).  
- Only authenticated admin traffic is allowed toward VLANs 10 and 20.

---

## Additional isolation

| Layer               | Tool                 | Role                                              |
|---------------------|----------------------|---------------------------------------------------|
| Network (L3/L4)     | OPNsense             | Inter-VLAN filtering and default drop             |
| Cluster (L3)        | Calico NetworkPolicy | Pod micro-segmentation                            |
| Application (L7)    | Traefik + CrowdSec   | Behavioural analysis and blocking of malicious requests |

---

## Key points

- No inbound ports open on the public IP.
- The DMZ (VLAN 40) cannot initiate connections toward internal networks.
- The management plane (VLAN 10) is the only zone allowed to administer the rest.
- Cloudflare Tunnels + WireGuard replace classic port-forwarding.

---

**Related pages:**

- [← IaC & automation](/en/iac-automation.html)
- [Security →](/en/security.html)
- [Services →](/en/services.html)
