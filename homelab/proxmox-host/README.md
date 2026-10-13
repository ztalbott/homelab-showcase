# Proxmox VE Host

The bare-metal hypervisor everything else runs on. It owns storage, the network bridge, the patch schedule, and backups.

**What this shows:** Virtualization, storage tiering, patch management, and backup and recovery, including finding and fixing a backup failure.

## Setup

- Proxmox VE 9.x on an Intel i7-10700K workstation with 16 GB RAM.
- One SSD for the OS and fast guest disks, one large HDD for media and backups.
- The onboard NIC carries the `vmbr0` bridge that every guest attaches to. A second, dual-port NIC is reserved for passthrough to the [OPNsense VM](../opnsense/).
- Each guest gets a VMID that matches its last IP octet, so `101` is always at `.101`.

## Configuration

### Storage tiers

| Pool | Type | Used for |
| --- | --- | --- |
| `SSD-Fast` | lvmthin | Container root disks that need speed |
| `HDD-VMs` | lvmthin | VM disks and larger container disks |
| `local` | dir | ISOs and LXC templates |
| `HDD-Bulk` | dir | Backup archives and bulk files |
| `hdd_storage` | host path | Media library, bind-mounted into Jellyfin |

Apps and OS images stay on SSD; anything big and sequential (movies, backups) stays on HDD.

### Weekly patching (cron on the host)

```cron
# Sunday 02:00: every service container, in parallel
0 2 * * 0  pct exec 101 -- sh -c 'apt-get update && apt-get -y upgrade && pihole -up'  >> /var/log/pihole-update.log 2>&1
0 2 * * 0  pct exec 102 -- sh -c 'apt-get update && apt-get -y upgrade'                >> /var/log/jellyfin-update.log 2>&1
0 2 * * 0  pct exec 105 -- sh -c 'apt-get update && apt-get -y upgrade && docker system prune -f' >> /var/log/docker-update.log 2>&1
0 2 * * 0  pct exec 106 -- sh -c 'apt-get update && apt-get -y upgrade'                >> /var/log/tailscale-update.log 2>&1
0 2 * * 0  pct exec 107 -- sh -c 'apt-get update && apt-get -y upgrade'                >> /var/log/mcp-update.log 2>&1

# Sunday 02:30: the host itself, rebooting only if a kernel update asks for it
30 2 * * 0 apt-get update && apt-get -y full-upgrade && [ -f /var/run/reboot-required ] && reboot >> /var/log/proxmox-update.log 2>&1
```

_Sanitized and representative of the live crontab._

### Weekly backups (vzdump)

| Setting | Value |
| --- | --- |
| Schedule | Sunday 01:00, before the patch window |
| Mode | Snapshot, zstd compression |
| Target | `HDD-Bulk` |
| Guests | Every LXC and VM |
| Retention | 7 daily, 4 weekly, 3 monthly |

Running backups an hour before patching means there is always a fresh restore point if an update breaks something.

### Access hardening

- SSH to the host is key-only (`PasswordAuthentication no`).
- SSH is stopped, disabled, and masked inside every container. Containers are managed with `pct exec` from the host instead.
- Each guest runs its own Proxmox firewall policy: default deny inbound, with only its service port opened to the LAN.
- Cluster-level security groups encapsulate reusable policies, such as an "allow only DNS and gateway" group paired with a "drop LAN" group for guests that should reach the internet but not the rest of the network.

## Lessons learned

**Backups that fail silently aren't backups.** An audit found the weekly backup job pointed at a storage name that didn't exist, so every scheduled run for about three months had failed. The failure emails went unnoticed. I repointed the job, expanded it from four guests to all of them (several had never been backed up), and ran immediate one-off backups of every guest the same day. Next step is an alert that I can't miss, probably via [Uptime Kuma](../uptime-kuma/).

**Know what a storage type can hold.** A directory-type pool can store ISOs, templates, and backups, but not a container's root filesystem. Container disks have to go on an lvmthin (or similar block) pool. That's why `HDD-Bulk` never shows up as an option when creating a container.

**Verify the docs against the machine.** A routine review found SSH listening on one container even though the docs said it had never been installed. It was masked and verified closed, and every review since checks all containers rather than trusting the record.

**Confirm automation actually fires.** The host's reboot-if-required logic hadn't been observed triggering on its own; the last kernel reboot was done by hand. Automation is only "done" once I've seen it work.
