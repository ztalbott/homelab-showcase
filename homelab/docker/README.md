# Docker + Portainer

A dedicated LXC that hosts Docker workloads, managed through Portainer's web UI.

**What this shows:** Container platform design (Docker inside LXC), least-exposure firewalling, and catching drift between documentation and reality.

| | |
| --- | --- |
| Guest | LXC 105 |
| Resources | 2 vCPU, 2 GB RAM, 16 GB on SSD |
| Starts on boot | Yes |

## Setup

- Docker Engine installed inside an LXC rather than a full VM, which keeps overhead low while still isolating container workloads from the hypervisor.
- Portainer runs as a container itself and provides the management UI over HTTPS.

## Configuration

- **Firewall:** default deny, with only the Portainer UI (and any app ports deliberately published) allowed from the LAN.
- **No SSH.** Like every container in the lab, administration happens through `pct exec` from the host.
- **Weekly maintenance:** Sunday cron runs `apt upgrade` and `docker system prune` so dangling images and stopped containers don't slowly fill the disk.

## Planned workloads

- **Homarr** dashboard as the single landing page for every service. Its config lives in a database rather than YAML, so it'll be backed up with the container rather than tracked in Git.
- **Pterodactyl** game-server panel, Docker-based, on its own VLAN once OPNsense is live. Game servers need CPU and RAM, not a GPU.

## Lessons learned

**Audit, don't assume.** My docs said SSH was never installed in this container. A review found it listening on port 22. It was stopped, disabled, and masked, and port 22 was verified closed. Since then, every review checks the actual listening ports on every container.

**Schedule cleanup alongside updates.** Docker accumulates old image layers quickly; pruning in the same weekly job as updates keeps disk usage flat without having to think about it.
