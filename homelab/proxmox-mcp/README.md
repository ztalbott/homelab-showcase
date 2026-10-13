# Proxmox MCP Server

A self-hosted [Model Context Protocol](https://modelcontextprotocol.io) server that lets Claude manage the lab: list and configure guests, take snapshots and backups, edit firewall rules, run commands, and keep a running memory of the lab's state.

| | |
| --- | --- |
| Guest | LXC 107 |
| Resources | 1 vCPU, 512 MB RAM, 6 GB on SSD |
| Transport | Streamable HTTP |
| Service | systemd unit, starts on boot |

## Setup

- A Python MCP server installed under `/opt` in its own container, talking to the Proxmox API with a dedicated API token and to the host over SSH with its own key.
- It runs as a systemd service, so it is always available and doesn't depend on any desktop PC being on.
- Clients authenticate with a bearer token. Credentials live outside the repo and outside the docs.

## Configuration

Representative systemd unit (sanitized):

```ini
[Unit]
Description=Proxmox MCP server
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/proxmox-homelab
EnvironmentFile=/etc/proxmox-mcp/env
ExecStart=/opt/proxmox-homelab/.venv/bin/proxmox-mcp --transport streamable-http
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- **Firewall:** the MCP port is open to the LAN only.
- **Health checks:** systemd status plus a TCP port check. A plain HTTP GET against an MCP endpoint hangs, because the server expects a full JSON-RPC handshake.

## The Claude skill on top

Day-to-day administration goes through a custom Claude skill, `proxmox-homelab`. It triggers on requests like "check my server", "update pihole", or "add a firewall rule" and encodes the lab's rules:

1. **Load state first.** Every session starts by reading the lab's memory file so the assistant knows the current configuration before touching anything.
2. **Follow house conventions.** Default-deny firewalls, `pct exec` instead of SSH for containers, VMID matches IP octet.
3. **Leave a record.** After any meaningful change, update the memory file and regenerate the network documentation, including a dated change log.
4. **Respect intentional choices.** Things like Jellyfin's missing gateway are documented as deliberate so the assistant doesn't undo them.

## Lessons learned

**Unbounded logging will eat your context window.** The memory file had grown to about 187 KB and roughly 1,500 lines, about 95% of it auto-logged command output, and it was overflowing the model's context. I traced it to the server logging *every* `run_ssh_command` call (reads included) with no size cap. I patched the server to stop auto-logging those calls and to cap the automatic change log at 40 entries with pruning, then condensed the memory to a ~100-line current-state snapshot. Real state changes (create, destroy, config, firewall) are still logged. Memory now stays clean without manual cleanup.

**Health checks have to match the protocol.** `curl` against the endpoint looked like an outage. It wasn't: MCP isn't a request/response web page. The monitor for this service is a TCP check, not HTTP.

**Treat the AI like any other admin.** It gets its own credentials, a written runbook (the skill), and an audit trail (the change log). The same practices that make human operators safe make an assistant safe.
