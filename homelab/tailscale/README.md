# Tailscale (VPN)

Remote access to the whole lab without opening a single inbound port on the router.

**What this shows:** Secure remote access with zero open inbound ports, and service isolation.

| | |
| --- | --- |
| Guest | LXC 106 |
| Resources | 1 vCPU, 256 MB RAM, 4 GB on HDD |
| Starts on boot | Yes |

## Setup

- Tailscale runs in its own small LXC configured as a **subnet router**. It advertises the home LAN to my tailnet, so remote devices can reach every service by its normal LAN address.
- Because Tailscale builds outbound WireGuard tunnels, there is no port forwarding and nothing exposed on the public IP.

## Configuration

Typical subnet-router setup inside the container:

```bash
# allow the container to forward traffic for the LAN
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf

# advertise the home subnet (approved once in the Tailscale admin console)
tailscale up --advertise-routes=<home-lan-cidr>
```

- An unprivileged LXC needs access to `/dev/net/tun` for Tailscale to create its interface, set in the container config.
- **Firewall:** ICMP from the LAN is dropped by design, so a failed ping to this container is expected, not a fault.

## Lessons learned

**Put the VPN on its own guest.** Running the subnet router as a dedicated container means it can be rebuilt, patched, or restarted without touching any other service, and its firewall policy can be scoped tightly.

**Document what "broken" looks like.** Pings failing to this container looked like an outage until I recorded that the firewall drops them intentionally. Now health checks use the right signal (service status and TCP checks) instead.

**Next:** Tailscale Funnel to give remote AI clients a controlled path to the [MCP server](../proxmox-mcp/).
