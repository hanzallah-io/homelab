# Project 01: Proxmox + Pi-hole + Unbound

> LinkedIn carousel: [Homelab Series, Part 1](../../docs/carousels/part1/Proxmox_Pihole_Homelab_Carousel.docx)

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
Proxmox VE 9.2.2 host running Pi-hole in a Debian 12 LXC (CT 100) with Unbound as the upstream recursive resolver. This is the foundation of the lab: DNS filtering and recursive resolution for all other projects.

## Environment
| Item | Detail |
|---|---|
| Host | Laptop running Proxmox VE 9.2.2 (hostname `hanzallah`) |
| Router | Huawei EG8112H, `192.168.1.1` (`192.168.1.0/24`) |
| Proxmox web UI | `https://192.168.1.50:8006` |
| Pi-hole | Debian 12 LXC (CT 100), `192.168.1.60` |
| Upstream DNS | Unbound at `127.0.0.1#5335` inside the Pi-hole container |

Full address plan: [docs/network.md](../../docs/network.md)

## Phase 1: Proxmox host network
**Problem:** Proxmox installer defaulted to `192.168.100.x` subnet but the LAN is `192.168.1.0/24`.
**Fix:** Changed the host IP to `192.168.1.50/24` and gateway to `192.168.1.1`.

**Host (root):**
```bash
# Edit /etc/network/interfaces or use the Proxmox web UI
# Set static IP 192.168.1.50/24, gateway 192.168.1.1
# DNS: 9.9.9.9, 1.1.1.1 (do NOT use Pi-hole here — circular dependency)
```

📸 SS: Host network config (`images/01-template.png`)

## Phase 2: Pi-hole LXC container
**Problem:** Need a Debian 12 LXC for Pi-hole with a static IP.
**Fix:** Create CT 100 with the debian-12 template.

**Host (root):**
```bash
pveam update
pveam download local debian-12-standard_12.12-1_amd64.tar.zst

pct create 100 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname pihole --cores 1 --memory 1024 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.60/24,gw=192.168.1.1 \
  --nameserver 9.9.9.9 \
  --unprivileged 1 --features nesting=1 --onboot 1
pct start 100
pct enter 100
```

📸 SS: `pct config 100` output (`images/02-pihole-install.png`)

## Phase 3: Pi-hole installation and per-device DNS
**Problem:** The ISP router (Huawei EG8112H) locks DHCP DNS fields, so clients can't be pointed to Pi-hole via DHCP.
**Fix:** Install Pi-hole, then configure each device manually to use `192.168.1.60` as DNS.

**Container (root@pihole):**
```bash
apt update && apt install -y curl
curl -sSL https://install.pi-hole.net | bash
# Web UI at http://192.168.1.60/admin
# Set admin password: pihole -a -p
```

📸 SS: Pi-hole web interface (`images/04-query-log.png`)

**Client devices:** Set DNS to `192.168.1.60` manually (Windows: Settings → Network → Edit DNS; Android: Settings → Network → Private DNS → Off, then set DNS in Wi-Fi config).

📸 SS: Router DHCP DNS fields locked (`images/03-router-dns.png`)

## Phase 4: Unbound as upstream resolver
**Problem:** Pi-hole defaults to upstream providers (Google, Cloudflare). Want recursive resolution with DNSSEC.
**Fix:** Install Unbound inside the Pi-hole container and configure Pi-hole to use it.

**Container (root@pihole):**
```bash
apt update && apt install -y unbound
cat > /etc/unbound/unbound.conf.d/pihole.conf <<'EOF'
server:
    verbosity: 1
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-daemonize: yes
    use-syslog: yes
    root-hints: "/var/lib/unbound/root.hints"
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
    val-permissive-mode: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    hide-identity: yes
    hide-version: yes
    qname-minimisation: yes
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m
    so-sndbuf: 1m
    msg-cache-size: 4m
    rrset-cache-size: 8m
    cache-min-ttl: 0
    cache-max-ttl: 86400
    unwanted-reply-threshold: 10000000
    do-not-query-localhost: no
    local-zone: "10.in-addr.arpa." transparent
    local-zone: "168.192.in-addr.arpa." transparent
EOF
wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache
chown unbound:unbound /var/lib/unbound/root.hints
systemctl enable unbound --now
dig @127.0.0.1 -p 5335 example.com
```

**Pi-hole web UI:** Settings → DNS → Upstream DNS Servers → Uncheck all, add `127.0.0.1#5335`.

📸 SS: Unbound dig test (`images/05-unbound-dig.png`)

## Phase 5: Blocklists
**Problem:** Default Pi-hole blocklists were stale.
**Fix:** Replaced with StevenBlack, HaGeZi Pro, HaGeZi TIF, and OISD.

**Pi-hole web UI:** Settings → Blocklists → Add:
- `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`
- `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/pro.txt`
- `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/tif.txt`
- `https://dbl.oisd.nl/`

Then: Tools → Update Gravity.

📸 SS: Blocklists configured (`images/06-blocklists.png`)

## Phase 6: SERVFAIL outage (DNS/NTP circular dependency)
**Problem:** Pi-hole SERVFAIL on all queries. Root cause: host clock was wrong, so Unbound DNSSEC validation failed. But NTP on the host needed DNS to reach time servers, and DNS needed correct time for DNSSEC — circular dependency.
**Fix:** Corrected host clock manually, then set up chrony on the host with hardcoded IP time servers.

**Host (root):**
```bash
# Fix time manually first
date -s "2026-11-15 14:30:00"
# Install chrony with IP addresses (not hostnames)
apt update && apt install -y chrony
cat > /etc/chrony/chrony.conf <<'EOF'
pool 162.159.200.1 iburst
pool 162.159.200.123 iburst
pool 192.168.1.1 iburst
makestep 1 3
rtcsync
EOF
systemctl restart chronyd
chronyc tracking
```

📸 SS: Fixed state (`images/08-fixed.png`)

## Phase 7: Disable laptop sleep and lid actions
**Problem:** Proxmox host (laptop) sleeps on lid close, killing all containers.
**Fix:** Disable sleep and lid actions in systemd-logind.

**Host (root):**
```bash
cat > /etc/systemd/logind.conf.d/nosleep.conf <<'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
IdleAction=ignore
IdleActionSec=0
EOF
systemctl restart systemd-logind
```

📸 SS: Logind config (`images/09-sleep.png`)

## Phase 8: Tailscale interference
**Problem:** Tailscale on the admin laptop was rerouting LAN traffic through the tailnet, blocking access to the Proxmox web UI.
**Fix:** Disable Tailscale on the admin laptop when managing the lab, or enable "Allow local network access" in the Tailscale client.

📸 SS: Proxmox dashboard accessible (`images/10-dashboard.png`)

## Verification
- `nslookup example.com 192.168.1.60` returns correct IP
- `dig @192.168.1.60 example.com +dnssec` shows AD flag (DNSSEC validated)
- Pi-hole query log shows client queries being filtered
- Unbound resolves recursively (no upstream forwarders)
- Proxmox web UI accessible at `https://192.168.1.50:8006`

## Backup / restore
Container config in `/etc/pve/lxc/100.conf`. Pi-hole config in `/etc/pihole/` inside CT 100. Unbound config in `/etc/unbound/` inside CT 100. Back up with vzdump to NAS (see project 03).

## Key takeaways
- Host must not use Pi-hole as DNS (circular dependency with NTP/DNSSEC)
- ISP router DHCP DNS lock → manual per-device DNS config
- Unbound inside the Pi-hole container avoids an extra LXC
- DNSSEC needs correct time — fix host clock first
- Tailscale on admin machine can break local LAN access

## Still open
- Reboot test: verify everything comes up cleanly after host reboot
- Enterprise apt repos on Proxmox host return 401; disable and use pve-no-subscription (pending)

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Proxmox installer set wrong subnet (192.168.100.x) | Default Proxmox network template | Change to 192.168.1.50/24 in /etc/network/interfaces |
| SERVFAIL on all DNS queries | Wrong host clock → DNSSEC validation failure in Unbound | Fix host clock manually, then configure chrony with IP time servers |
| NTP fails to sync | DNS needed for NTP hostnames, but DNS needs time for DNSSEC | Use IP addresses in chrony.conf, not hostnames |
| Pi-hole container has no internet | Gateway not set correctly in pct create | Ensure --net0 includes gw=192.168.1.1 |
| Laptop sleeps on lid close | systemd-logind default behavior | Create /etc/systemd/logind.conf.d/nosleep.conf with HandleLidSwitch=ignore |
| Tailscale blocks Proxmox web UI access | Tailscale routes LAN traffic through tailnet | Disable Tailscale on admin laptop or enable "Allow local network access" |