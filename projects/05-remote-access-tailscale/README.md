# Project 05: Remote access (Tailscale subnet router)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 6.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A Tailscale subnet router that connects outbound, so CGNAT does not matter. Built on WireGuard. Replaced the original WireGuard plan after discovering the ISP uses CGNAT.

## Environment
| Item | Detail |
|---|---|
| Type (VM / LXC / Docker) | LXC (Debian 12, unprivileged) |
| ID / hostname | CT 103 / tailscale |
| IP and ports | 192.168.1.63, Tailscale UDP 41641 (auto) |
| Resources (CPU / RAM / disk) | 1 core, 256 MB RAM, 256 MB swap, 2 GB rootfs |

## The check that killed the WireGuard plan
**Original plan:** WireGuard server at home with UDP 51820 forwarded on the router.
**Check:**
```bash
curl https://api.ipify.org      # the IP the internet sees
```
Compared with the WAN IP on the router's status page. The router's WAN address was in 100.64.0.0/10 (carrier-grade NAT) and differed from the public IP. The ISP NATs many customers behind shared addresses, so no port forward on the router can work.

Side finding: the router's WAN page also showed a native IPv6 /56 prefix, so WireGuard over IPv6 was possible in theory but depends on the phone having IPv6 on mobile data. Tailscale was chosen for reliability.

## Phase 1: container (Proxmox host)
**Problem:** Need a container that can create a TUN interface for Tailscale.
**Fix:** Create CT 103 with TUN device passthrough.

**Host (root):**
```bash
pct create 103 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname tailscale --cores 1 --memory 256 --swap 256 \
  --rootfs local-lvm:2 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.63/24,gw=192.168.1.1 \
  --nameserver 192.168.1.60 \
  --unprivileged 1 --features nesting=1 --onboot 1
pct set 103 --dev0 /dev/net/tun,mode=0666
pct start 103
pct enter 103
```
The TUN device passthrough is what lets Tailscale create its tunnel interface inside an unprivileged container.

## Phase 2: install and enable forwarding (inside CT 103)
**Problem:** Need IP forwarding for subnet routing.
**Fix:** Install Tailscale and enable IPv4/IPv6 forwarding via sysctl.

**Container (root@tailscale):**
```bash
apt update && apt install -y curl
curl -fsSL https://tailscale.com/install.sh | sh
cat > /etc/sysctl.d/99-tailscale.conf <<'EOF'
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOF
sysctl --system | grep forward
```

📸 SS: `sysctl --system | grep forward` output (`images/sysctl-forward.png`)

## Phase 3: advertise the home subnet
**Problem:** Tailscale needs to know which subnet to route.
**Fix:** Advertise 192.168.1.0/24 and accept DNS from the tailnet (disabled here to keep Pi-hole).

**Container (root@tailscale):**
```bash
tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```
Open the printed login URL and sign in to the same Tailscale account as the other devices. `--accept-dns=false` keeps the container on the Pi-hole for DNS.

## Phase 4: Tailscale admin console
**Problem:** The advertised route must be approved; key expiry must be disabled; split DNS for *.lab.
**Fix:** In the Tailscale admin console (login.tailscale.com):
1. Approve the 192.168.1.0/24 route for the machine.
2. Disable key expiry for the subnet router.
3. Add split DNS: nameserver 192.168.1.60, restricted to the domain lab, so *.lab names resolve remotely.

Tailscale masquerades subnet traffic, so the Pi-hole sees it as coming from 192.168.1.63, which it already accepts.

📸 SS: Tailscale admin console showing route approval (`images/tailscale-machines.png`)

## Phase 5: test
**Verification:** From a phone (Pixel 6a) on mobile data with the Tailscale app on, open the Pi-hole admin page at http://192.168.1.60/admin. Confirmed working by the owner. (https://pve.lab additionally needs the CA installed on the phone.)

## Verification
- `tailscale status` shows the subnet router advertising 192.168.1.0/24
- Admin console shows route approved and key expiry disabled
- Split DNS entry for lab domain with nameserver 192.168.1.60
- Phone on mobile data can access http://192.168.1.60/admin

## Backup / restore
Container config in /etc/pve/lxc/103.conf. Tailscale state in /var/lib/tailscale/ inside CT 103.

## Key takeaways
- Check for CGNAT before planning a port forward
- Tailscale subnet router connects outbound — CGNAT does not matter
- Route approval in the admin console is required
- Split DNS for *.lab only, not all DNS
- `--accept-dns=false` prevents Tailscale from overriding local DNS

## Still open
- None currently

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| WireGuard remote access impossible | ISP CGNAT, no inbound connectivity | Tailscale subnet router (outbound only) |
| Tailscale earlier redirected the laptop's traffic through it during the Pi-hole setup (see project 01) | Tailscale DNS and routing overriding the LAN | `--accept-dns=false` on the subnet router; split DNS only for lab |