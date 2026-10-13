# Pi-hole (DNS + DHCP)

Network-wide DNS filtering and the DHCP server for the whole LAN, running in a tiny LXC.

**What this shows:** DNS and DHCP design, IP address planning, and network-wide policy enforcement without per-device setup.

| | |
| --- | --- |
| Guest | LXC 101 |
| Resources | 1 vCPU, 256 MB RAM, 2 GB on SSD |
| Starts on boot | Yes |

## Setup

- Debian-based unprivileged LXC with a static address outside the DHCP pool.
- Pi-hole installed with its standard installer.
- DHCP turned **off** on the router and **on** in Pi-hole, so every client automatically gets Pi-hole as its DNS server without per-device configuration.

## Configuration

- **DHCP pool:** the top of the subnet (`.200` to `.250`). Servers use static addresses in the `.100` range, so a lease can never collide with a service.
- **Upstream DNS:** Pi-hole is the only resolver on the network. When OPNsense takes over as gateway, its built-in resolver stays disabled and it hands DNS to Pi-hole, so there is one place to look at queries and blocklists.
- **Firewall:** DNS and the admin UI are allowed from the LAN only.
- **Updates:** patched every Sunday by host cron, running both `apt upgrade` and `pihole -up`.

## Lessons learned

**Putting DHCP and DNS in the same place simplifies everything.** With DHCP on the router, every device needed a manual DNS setting or a router feature that might not exist. Moving DHCP into Pi-hole made filtering automatic for anything that joins the network.

**Plan the address map before you need it.** Splitting the subnet into gateway (`.1`), host (`.100`), servers (`.101` and up, matching VMID), and a DHCP pool at the top made later work such as firewall aliases and the OPNsense cutover much easier to reason about.

**DNS enforcement belongs on the firewall too.** Clients that hard-code their own DNS servers bypass Pi-hole. A force-DNS-to-Pi-hole rule on OPNsense is part of the cutover plan.
