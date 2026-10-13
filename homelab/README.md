# Homelab

A single-node Proxmox VE lab that runs my home network's core services. Each service gets its own small LXC container (or a VM where a full OS is required), its own firewall policy, and a slot in the weekly patch and backup cycle.

## Hardware

| Component | Spec |
| --- | --- |
| CPU | Intel Core i7-10700K, 8 cores @ 3.8 GHz |
| RAM | 16 GB (about 10 GB free under normal load) |
| Boot / fast tier | ~100 GB SSD |
| Bulk tier | ~1.4 TB HDD |
| NICs | Onboard Intel I219-LM (host), dual-port Intel 82576 (passed through to OPNsense) |
| Hypervisor | Proxmox VE 9.x |

## Guests

| VMID | Name | Type | vCPU | RAM | Disk | Purpose |
| --- | --- | --- | --- | --- | --- | --- |
| 101 | pihole | LXC | 1 | 256 MB | 2 GB | DNS filtering + DHCP |
| 102 | jellyfin | LXC | 1 | 512 MB | 16 GB | Media server |
| 105 | docker | LXC | 2 | 2 GB | 16 GB | Docker + Portainer |
| 106 | tailscale | LXC | 1 | 256 MB | 4 GB | VPN subnet router |
| 107 | proxmox-mcp | LXC | 1 | 512 MB | 6 GB | MCP server for AI-assisted admin |
| 108 | kuma | LXC | 1 | 512 MB | 8 GB | Uptime Kuma monitoring |
| 103 | downloader | VM | 1 | 3 GB | 32 GB | Desktop VM on an isolated network policy |
| 104 | minttemplate | VM template | 1 | 2 GB | 32 GB | Linux Mint golden image |
| 200 | opnsense | VM | 2 | 4 GB | 20 GB | Firewall/router (staged) |

The LXCs together use under 4 GB of RAM. Containers are the default; VMs are reserved for things that need their own kernel (a desktop OS, or FreeBSD for OPNsense).

## How it's organized

- [proxmox-host](proxmox-host/): storage pools, patching, backups, host hardening
- [pihole](pihole/): DNS and DHCP for the whole network
- [jellyfin](jellyfin/): media server with HDD bind mounts and no internet route
- [docker](docker/): container host for app workloads
- [tailscale](tailscale/): remote access without opening ports
- [proxmox-mcp](proxmox-mcp/): self-hosted MCP server and the Claude skill that drives it
- [uptime-kuma](uptime-kuma/): health checks for every service
- [opnsense](opnsense/): virtualized firewall build with PCIe passthrough

## Roadmap

1. **Finish OPNsense and cut over.** Firewall rules, force all DNS to Pi-hole, then move OPNsense to the gateway address and the current router to access-point mode.
2. **VLANs.** Split servers, trusted clients, IoT, and game servers onto separate segments now that the second NIC is in.
3. **Homarr dashboard** as a single landing page for every service.
4. **Game server hosting** with Pterodactyl on its own isolated VLAN.
5. **Observability:** Grafana + Prometheus, and a Wazuh SIEM.
6. **Identity:** Authentik or Keycloak SSO in front of the web UIs.
7. **Self-hosted Git** with Gitea, and a local LLM experiment with Ollama.
8. **Jellyfin hardware transcoding** with Intel Quick Sync on the CPU's iGPU.
