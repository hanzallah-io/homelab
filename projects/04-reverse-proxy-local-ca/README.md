# Project 04: Reverse proxy + private CA

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 5.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
An nginx reverse proxy with a private CA (mkcert) that terminates TLS for *.lab domains. The browser asks Pi-hole for pve.lab; a local DNS record returns the proxy's IP. nginx on CT 102 terminates TLS with a wildcard certificate for *.lab, issued by the private CA. nginx proxies to the Pi-hole admin page and the Proxmox web UI (with WebSocket headers so the console works).

## Environment
| Item | Detail |
|---|---|
| Type (VM / LXC / Docker) | LXC (Debian 12, unprivileged) |
| ID / hostname | CT 102 / proxy |
| IP and ports | 192.168.1.62, 80, 443 |
| Resources (CPU / RAM / disk) | 1 core, 512 MB RAM, 256 MB swap, 4 GB rootfs |

## Phase 1: container (Proxmox host)
**Problem:** Need a dedicated container for the reverse proxy.
**Fix:** Create CT 102 with the debian-12 template.

**Host (root):**
```bash
pct create 102 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname proxy --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.62/24,gw=192.168.1.1 \
  --nameserver 192.168.1.60 \
  --unprivileged 1 --features nesting=1 --onboot 1
pct start 102
pct enter 102
```

## Phase 2: nginx and the CA (inside CT 102)
**Problem:** Need TLS certificates trusted by local devices.
**Fix:** Install nginx and mkcert, create and install a local CA, generate a wildcard certificate for *.lab.

**Container (root@proxy):**
```bash
apt update && apt install -y nginx mkcert libnss3-tools
mkcert -install
mkdir -p /etc/nginx/certs
mkcert -cert-file /etc/nginx/certs/lab.pem -key-file /etc/nginx/certs/lab-key.pem "*.lab" lab
```

## Phase 3: nginx config
**Problem:** Need to terminate TLS and proxy to backend services with WebSocket support for Proxmox.
**Fix:** Configure nginx with SSL certificates, redirect HTTP to HTTPS, and proxy to Pi-hole and Proxmox.

**Container (root@proxy):** Save as /etc/nginx/conf.d/lab.conf (see configs/lab.conf):
```nginx
ssl_certificate     /etc/nginx/certs/lab.pem;
ssl_certificate_key /etc/nginx/certs/lab-key.pem;

server {
  listen 80 default_server;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name pihole.lab;
  location / {
    proxy_pass http://192.168.1.60;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

server {
  listen 443 ssl;
  server_name pve.lab;
  client_max_body_size 0;
  location / {
    proxy_pass https://192.168.1.50:8006;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_buffering off;
  }
}
```
```bash
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

📸 SS: `nginx -t` output and `systemctl status nginx` (`images/nginx-config.png`)

## Phase 4: local DNS
**Problem:** Local clients need to resolve *.lab to the proxy IP.
**Fix:** Add local DNS records in Pi-hole pointing pihole.lab and pve.lab to 192.168.1.62.

**Pi-hole web UI:** Settings → Local DNS Records:
- pihole.lab → 192.168.1.62
- pve.lab → 192.168.1.62

**Verify:**
```bash
nslookup pihole.lab 192.168.1.60
```

## Phase 5: trust the CA on Windows
**Problem:** Windows needs to trust the mkcert root CA.
**Fix:** Copy the public root CA from the container to the NAS, import it via certutil, then remove the temporary copy.

**Host (root):**
```bash
# Proxmox host
pct pull 102 /root/.local/share/mkcert/rootCA.pem /srv/nas/rootCA.pem
```

```cmd
:: Windows, Command Prompt as Admin
net use \\192.168.1.61\nas /user:nas *
certutil -addstore -f "ROOT" \\192.168.1.61\nas\rootCA.pem
net use \\192.168.1.61\nas /delete
```

```bash
# Proxmox host: remove the temporary copy
rm /srv/nas/rootCA.pem
```

Firefox keeps its own certificate store, so import rootCA.pem there manually.

## Phase 6: test
**Verification:** https://pihole.lab/admin and https://pve.lab load with a padlock. `tail -f /var/log/nginx/access.log` in CT 102 shows requests.

📸 SS: Browser showing padlock on pve.lab (`images/pve-lab-https.png`)

## Verification
- `nslookup pve.lab 192.168.1.60` returns 192.168.1.62
- `nginx -t` passes
- https://pihole.lab/admin loads with a valid certificate
- https://pve.lab loads with a valid certificate and WebSocket console works
- Windows certutil imports rootCA.pem without error

## Backup / restore
Container config in /etc/pve/lxc/102.conf. Certificates in /etc/nginx/certs/ inside CT 102. CA root at /root/.local/share/mkcert/ inside CT 102.

## Key takeaways
- Local split DNS (Pi-hole) resolves *.lab to the proxy
- mkcert creates a locally trusted CA without internet
- TLS termination at the proxy simplifies backend configuration
- WebSocket proxying requires specific headers (Upgrade, Connection)
- Windows prefers router's IPv6 DNS — disable IPv6 or set IPv4 DNS explicitly

## Still open
- None currently

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `nslookup pve.lab` returned NXDOMAIN on the Proxmox host | Host's DNS is deliberately 9.9.9.9 (fix from project 01) and does not know .lab | Query the Pi-hole directly: `nslookup pve.lab 192.168.1.60`. Do not point the host at the Pi-hole (circular dependency) |
| Windows refused to save DNS settings | An IP address was entered in the DNS-over-HTTPS template field | Set DNS from PowerShell instead of the settings page |
| `nslookup` on Windows still asked the router (fe80::1) | Windows prefers the router's IPv6 DNS over the IPv4 Pi-hole | `Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses 192.168.1.60` and `Disable-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip6` (applied and confirmed working) |
| `certutil` failed with ERROR_SMB_GUEST_LOGON_BLOCKED (0x800704f8) | Elevated prompt does not share Explorer's saved credentials, so Windows tried a blocked guest login | `net use \\192.168.1.61\nas /user:nas *` first, then `certutil` |