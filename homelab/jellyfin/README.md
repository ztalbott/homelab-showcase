# Jellyfin (Media server)

Self-hosted media server, deliberately kept off the internet.

**What this shows:** Deliberate network isolation, storage separation between apps and data, and making a security tradeoff on purpose.

| | |
| --- | --- |
| Guest | LXC 102 |
| Resources | 1 vCPU, 512 MB RAM, 16 GB on SSD |
| Media | HDD bind mount |
| Starts on boot | Yes |

## Setup

- Jellyfin runs in an LXC whose root disk lives on the SSD tier: only the application and its metadata.
- The media library lives on the HDD and is bind-mounted into the container, so movies and shows never consume SSD space.

## Configuration

Bind mount from the host into the container (from `/etc/pve/lxc/102.conf`):

```ini
mp0: /mnt/hdd_storage/,mp=/media
```

Library layout on the HDD:

```text
/mnt/hdd_storage/media/
├── movies/
├── tv/
└── downloads/
```

- **LAN-only by design.** The container's network interface has no default gateway. It can serve clients on the LAN but cannot reach the internet at all.
- **Firewall:** only the Jellyfin web port is open, and only from the LAN.

## Lessons learned

**Isolation is a tradeoff you should make on purpose.** Without a gateway, Jellyfin can't fetch posters and metadata online, and `apt` updates inside it don't work. I briefly added a gateway to get metadata, then removed it the same day: I'd rather have a media server that cannot talk to the internet. The missing gateway is documented as intentional so nobody (including an AI assistant) "fixes" it.

**Separate app storage from data storage.** Keeping the library on a bind mount means the container itself stays small, backs up quickly, and can be rebuilt without touching the media.

**Next:** Intel Quick Sync hardware transcoding through the CPU's integrated GPU. No GPU purchase needed.
