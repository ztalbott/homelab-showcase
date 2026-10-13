# Uptime Kuma (Monitoring)

Self-hosted uptime monitoring for every service in the lab.

**What this shows:** Monitoring design, choosing the right health signal for each service, and running services as least-privilege users.

| | |
| --- | --- |
| Guest | LXC 108 |
| Resources | 1 vCPU, 512 MB RAM, 8 GB on SSD |
| OS | Debian 13, unprivileged |
| Version | Uptime Kuma 2.5.0 |
| Starts on boot | Yes |

## Setup

Installed from source rather than Docker, to keep it independent of the Docker host it monitors. Representative install steps:

```bash
# prerequisites: Node.js 22, npm, git, build-essential, pm2
useradd --system --create-home --home-dir /opt/uptime-kuma uptimekuma

sudo -u uptimekuma git clone https://github.com/louislam/uptime-kuma.git /opt/uptime-kuma/app
cd /opt/uptime-kuma/app
sudo -u uptimekuma git checkout 2.5.0     # pin to a stable release
sudo -u uptimekuma npm run setup

# run under PM2 as the service user, and persist across reboots
sudo -u uptimekuma pm2 start server/server.js --name uptime-kuma
pm2 startup systemd -u uptimekuma --hp /opt/uptime-kuma
sudo -u uptimekuma pm2 save
```

## Configuration

- Runs as a dedicated, unprivileged `uptimekuma` user, never as root.
- **Firewall:** only the web UI port (3001) is open, and only from the LAN.
- Included in the weekly backup job now that it holds real monitor configuration.

### Monitors

| Target | Check type |
| --- | --- |
| Proxmox host | Ping, plus HTTPS on the web UI |
| Pi-hole | HTTP admin page and DNS |
| Jellyfin | HTTP |
| Portainer | HTTPS |
| Tailscale | Service/TCP (ICMP is blocked by design) |
| Proxmox MCP | TCP port check, not HTTP |

## Lessons learned

**Monitor outside the thing you're monitoring.** Running Kuma in its own container, not inside Docker, means a Docker outage can't take the monitor down with it.

**Pick the check that matches the service.** An HTTP check against the MCP server would always fail, and a ping check against Tailscale would always fail. Each monitor uses whatever signal truly means "healthy" for that service.

**Next:** a backup-failure alert, so a broken backup job can never go unnoticed again (see the [host write-up](../proxmox-host/)).
