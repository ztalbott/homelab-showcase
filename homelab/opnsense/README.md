# OPNsense Firewall VM

A virtualized OPNsense firewall with a dedicated, passed-through NIC, built in a sandbox and staged to replace the consumer router as the network's gateway.

| | |
| --- | --- |
| Guest | VM 200 |
| Resources | 2 vCPU (host CPU type), 4 GB RAM, 20 GB on SSD |
| Machine type | q35 |
| NICs | Dual-port Intel 82576, both ports passed through |
| Version | OPNsense 26.7 |
| Status | Installed and configured, awaiting cutover |

## Setup

OPNsense is FreeBSD-based, so it has to be a VM, not an LXC.

1. **Install a second NIC.** A dual-port Intel 82576 card; the onboard NIC stays with the Proxmox host.
2. **Enable IOMMU.** VT-d was already on in the BIOS. Added the kernel flags and confirmed the kernel reported IOMMU enabled:
   ```bash
   # /etc/default/grub
   GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
   update-grub && reboot
   dmesg | grep -e DMAR -e IOMMU
   ```
   Both NIC ports landed in their own clean IOMMU groups, so passthrough needed no ACS workarounds.
3. **Pass both ports to the VM** (bound to `vfio-pci`):
   ```ini
   # /etc/pve/qemu-server/200.conf (excerpt)
   machine: q35
   cpu: host
   hostpci0: 02:00.0
   hostpci1: 02:00.1
   ```
4. **Install OPNsense** from the ISO, then detach the ISO. Interfaces: `igb0` = WAN, `igb1` = LAN.

## Configuration

- **Staged, not live.** LAN sits on a temporary staging address and the WAN cable stays unplugged until cutover. The live network was never touched during the build.
- **DNS:** Pi-hole is the DNS server, the built-in Unbound resolver is disabled, and WAN-supplied DNS is not allowed to override it.
- **DHCP:** disabled on OPNsense's LAN. Pi-hole keeps DHCP.
- **WAN:** DHCP from the ISP, block private and bogon networks, IPv6 off for now.
- **Brute-force protection:** OPNsense's native lockout table is active by default and blocks repeated failed logins to the web UI and SSH. Staying current on updates matters for it.

### Cutover plan

1. Finish firewall rules, including a LAN rule forcing all DNS to Pi-hole.
2. Verify routing on an isolated bench switch with a laptop.
3. Move the current router to access-point mode and give OPNsense the gateway address.
4. Connect WAN, check for firmware updates, and clone the old router's WAN MAC if the ISP doesn't issue a lease.

The cutover is the only disruptive step, so it happens with console access at a low-stakes time.

## Lessons learned

**A leftover virtual NIC can break a passthrough build.** After passthrough, the VM still had its original placeholder virtio NIC bridged onto the same subnet. Two interfaces on one subnet caused ARP and routing confusion: ping worked for a moment, then dropped to "Destination Host Unreachable." Removing the stray NIC and rebooting made it 100% stable.

**Test authentication changes before you commit to them.** While setting up TOTP two-factor login, I switched the admin authentication server to the new TOTP server without testing it first with OPNsense's built-in tester. The GUI stopped prompting for a code and every login failed. I recovered with OPNsense's official non-destructive procedure: booted the install ISO and used the installer's "Password Reset / Recover Installation" option (a full reinstall would have wiped everything). The config came through completely intact. The rule I follow now: **always validate in System > Access > Tester before switching auth servers.**

**Read the live system, not just the docs.** I initially thought OPNsense needed an add-on for brute-force protection. Inspecting the auto-generated rules showed native lockout was already active. Add-ons like CrowdSec are an upgrade, not a baseline requirement.

**Open question:** keep the firewall as a VM (cheap, but the network depends on the hypervisor being up) or move it to dedicated hardware later.
