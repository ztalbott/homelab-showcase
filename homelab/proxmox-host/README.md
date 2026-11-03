# Proxmox VE Host (Hypervisor)

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
0 2 * * 0  pct exec 108 -- sh -c 'apt-get update && apt-get -y upgrade'                >> /var/log/kuma-update.log 2>&1

# Sunday 02:30: the host itself, via a script that knows when a reboot is needed
30 2 * * 0 /usr/local/bin/host-update.sh >> /var/log/proxmox-update.log 2>&1
```

The host script upgrades, then reboots if either the standard reboot flag exists or the running kernel isn't the newest one installed. That second check matters on Proxmox (see [lessons learned](#lessons-learned)).

```bash
#!/bin/sh
# /usr/local/bin/host-update.sh
set -e
apt-get update
apt-get -y full-upgrade

running=$(uname -r)
newest=$(ls /boot/vmlinuz-*-pve | sed 's|/boot/vmlinuz-||' | sort -V | tail -n 1)

if [ -f /var/run/reboot-required ] || [ "$running" != "$newest" ]; then
    echo "$(date): rebooting ($running -> $newest)"
    reboot
fi
```

_Sanitized and representative of the live crontab and script._ VMs patch from inside the guest; OPNsense uses its own firmware updater. Changes to host config like this one are committed to a local git repo, so every edit has a history and a backup copy (for example, the old crontab was saved before the switch).

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

**Confirm automation actually fires.** The host's reboot-if-required logic had never been seen triggering on its own; the last kernel reboot had been done by hand. A full review showed why: Proxmox kernel upgrades don't create `/var/run/reboot-required`, so the cron job's condition was never true. The host was still running an older kernel than the one installed, and old kernels were piling up on the root disk. I replaced the one-liner with a script that also compares the running kernel to the newest installed one, rebooted into the current kernel (every container and the firewall VM came back on their own), and purged 21 old kernel images. Root disk usage dropped from 35 GB to 13 GB. The same review caught that the Uptime Kuma container had been left out of the patch cron, so it was 105 packages behind; it's patched and in the schedule now.

**Keep a fallback kernel.** When purging, I kept the running kernel, the one before it, and the newest of the previous series (the metapackage depends on it). A bad kernel update can then be rolled back from the boot menu instead of from a rescue USB.
