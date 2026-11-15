# Changelog

## Unreleased
- Repo created with project 01 documentation, helper scripts and LinkedIn carousel

## Project 01: Proxmox + Pi-hole + Unbound
- Installed Proxmox VE 9.2.2 and corrected the network (installer default `192.168.100.x` to `192.168.1.50/24`)
- Deployed Pi-hole in a Debian 12 LXC (CT 100, `192.168.1.60`)
- Configured per-device DNS because the ISP router locks DHCP DNS fields
- Added Unbound as the upstream resolver (`127.0.0.1#5335`)
- Replaced stale blocklists with StevenBlack, HaGeZi Pro, HaGeZi TIF and OISD
- Fixed a SERVFAIL outage caused by a wrong host clock and a DNS/NTP circular dependency
- Disabled laptop sleep and lid actions on the Proxmox host
- Found that Tailscale was rerouting laptop traffic and blocking access to the Proxmox web UI
- Documented the Pi-hole web interface and query filtering workflow