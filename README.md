# Project 02: Storage server / NAS (Samba)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 3.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A Samba file server on an unprivileged LXC container. Data lives on a thin LVM volume on the Proxmox host, bind-mounted into the container. The container can be rebuilt without touching data, and extra disks can be added later with mergerfs without changing anything inside the container.

## At a glance
| Container ID | IP | Role | Status |
|---|---|---|---|
| CT 101 | 192.168.1.61 | Samba file server | Documented |

## Goal and design
NAS storage using Proxmox LVM + bind-mount:
```text
[Host LVM /srv/nas] <--bindmount--> [CT 101 /mnt/nas]
```

## Build steps
1. **Storage volume** (Proxmox host, root):
   **Problem:** Need persistent storage that survives container rebuilds.
   **Fix:** Create a thin LVM volume on the host, mount at /srv/nas, and bind-mount into the container.
   ```bash
   lvcreate -V 250G -T pve/data -n nas
   mkfs.ext4 -L nas /dev/pve/nas
   mkdir -p /srv/nas
   echo '/dev/pve/nas /srv/nas ext4 defaults,nofail 0 2' >> /etc/fstab
   systemctl daemon-reload && mount -a
   df -h /srv/nas
   ```
   📸 SS: `df -h /srv/nas` output (`images/nas-volume.png`)

2. **Container** (Proxmox host):
   **Problem:** Need an unprivileged container with networking and nesting enabled for Samba.
   **Fix:** Create CT 101 with the debian-12 template.
   ```bash
   pveam update
   pveam download local debian-12-standard_12.12-1_amd64.tar.zst

   pct create 101 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
     --hostname nas --cores 1 --memory 512 --swap 256 \
     --rootfs local-lvm:4 \
     --net0 name=eth0,bridge=vmbr0,ip=192.168.1.61/24,gw=192.168.1.1 \
     --nameserver 192.168.1.60 \
     --unprivileged 1 --features nesting=1 --onboot 1
   ```
   📸 SS: `pct config 101` output (`images/ct101-config.png`)

3. **Bind mount** (Proxmox host):
   **Problem:** Container needs access to the host storage volume.
   **Fix:** Bind-mount /srv/nas into the container at /mnt/nas, adjusting UID mapping for the unprivileged container.
   ```bash
   pct set 101 -mp0 /srv/nas,mp=/mnt/nas
   chown -R 101000:101000 /srv/nas
   pct start 101
   ```
   Unprivileged containers shift UIDs by 100000, so host UID 101000 is UID 1000 inside the container, which is the Samba user created next.

4. **Samba** (inside CT 101):
   **Problem:** Need SMB file sharing with a dedicated service account.
   **Fix:** Install Samba, create a nas user/group (UID/GID 1000), configure the share, and set a Samba password.
   ```bash
   apt update && apt install -y samba
   groupadd -g 1000 nas
   useradd -u 1000 -g 1000 -M -s /usr/sbin/nologin nas
   smbpasswd -a nas
   ```
   Append to /etc/samba/smb.conf (see configs/smb.conf.snippet):
   ```ini
   [nas]
      path = /mnt/nas
      valid users = nas
      read only = no
      create mask = 0664
      directory mask = 0775
   ```
   ```bash
   systemctl restart smbd && systemctl enable smbd
   testparm -s
   ls -ld /mnt/nas     # owner should be nas:nas
   ```
   The nas account is Samba-only (nologin shell), so it cannot log in at the console.

   📸 SS: `testparm -s` output (`images/testparm-nas.png`)

5. **Connect from Windows**:
   **Verification:** Open File Explorer, enter `\\192.168.1.61\nas`, log in as `nas`.

   📸 SS: Windows File Explorer showing the share (`images/nas-share-windows.png`)

## Verification
- `testparm -s` shows the share config without errors
- `systemctl status smbd` shows active (running)
- Windows can connect, read, and write files
- `ls -ld /mnt/nas` shows owner nas:nas

## Backup / restore
Data is on the host at /srv/nas. Back up the host volume with vzdump (see project 03). Container config is in /etc/pve/lxc/101.conf.

## Security notes
- The nas Samba account has a nologin shell — it cannot log in at the console
- `nofail` in fstab prevents boot failures if the volume is unavailable
- Share is read-only by default; `read only = no` in smb.conf enables write access

## Limitations
- 250 GB thin LVM volume is fixed unless more disks are added with mergerfs
- Unprivileged container UID mapping (100000 offset) limits some host-level operations
- SMB protocol does not encrypt data in transit — consider VPN or TLS for sensitive data

## Lessons learned
- Thin LVM provisioning lets the volume grow as needed
- UID mapping (100000 offset) is critical for unprivileged container bind mounts
- Keeping data on the host decouples it from the container lifecycle
- Always check the prompt (root@nas vs root@hanzallah) before running commands

## Next steps
- External disks pooled with mergerfs (planned)
- SMB encryption (future)

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Samba install "failed" and a stray nas user and group appeared on the Proxmox host | Commands pasted on the host instead of inside the container; apt update aborted on enterprise-repo error so Samba never installed, but groupadd/useradd still ran | userdel nas; groupdel nas on the host, then repeat inside the container. Habit now: check the prompt (root@nas) before running anything |
| apt update on the host returned 401 Unauthorized for the enterprise Proxmox and Ceph repos | Enterprise repos need a paid subscription | Not yet applied. Disable enterprise repos and use pve-no-subscription. Flagged as pending |
| Container console rejected root and nas logins | No root password set; nas is Samba-only with a nologin shell | Enter from the host with pct enter 101, or set a root password with pct exec 101 -- passwd |
| ls /srv/nas and chown failed inside the container | /srv/nas only exists on the host; inside the container the same folder is /mnt/nas | Run host paths on the host, container paths in the container |
```

Now let me do project 03:</think>Now project 03 (backups):
<tool_call>
<function=Write>
<parameter=content>
# Project 03: Backups (vzdump to the NAS)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 4.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A folder on the NAS volume is registered as Proxmox backup storage. It is root-owned, so the Samba user can read it over SMB but cannot modify or delete backups. This protects against mistakes and bad updates.

## At a glance
| Container ID | IP | Role | Status |
|---|---|---|---|
| n/a | n/a | Backup storage | Documented |

## Goal and design
Proxmox backup storage on the NAS volume:
```text
[Host /srv/nas/pve-backups] <--Proxmox backup storage--> [All containers via vzdump]
```

## Build steps
1. **Backup storage** (Proxmox host):
   **Problem:** Need a backup target that lives on the NAS volume.
   **Fix:** Create the backup directory and register it as Proxmox storage with retention policy.
   ```bash
   mkdir -p /srv/nas/pve-backups
   pvesm add dir nas-backup --path /srv/nas/pve-backups --content backup \
     --prune-backups keep-daily=3,keep-weekly=2
   pvesm status
   ```
   📸 SS: `pvesm status` showing nas-backup active (`images/pvesm-status.png`)

2. **Manual backup** (no downtime):
   **Problem:** Need to back up running containers without downtime.
   **Fix:** Use vzdump snapshot mode with zstd compression. The NAS bind mount (mp0) is excluded automatically, so only each container's own root disk is saved, not the NAS data.
   ```bash
   vzdump 100 101 --storage nas-backup --mode snapshot --compress zstd
   ls -lh /srv/nas/pve-backups/dump/
   ```
   📸 SS: Backup files in /srv/nas/pve-backups/dump/ (`images/backup-job.png`)

3. **Weekly schedule**:
   **Problem:** Need automated weekly backups.
   **Fix:** Create a scheduled vzdump job via pvesh.
   ```bash
   pvesh create /cluster/backup --storage nas-backup --vmid 100,101 \
     --schedule "sun 02:00" --mode snapshot --compress zstd \
     --prune-backups keep-daily=3,keep-weekly=2 --enabled 1
   cat /etc/pve/jobs.cfg
   ```
   Confirmed output of `pvesm status` (nas-backup active) and `/etc/pve/jobs.cfg`:
   ```
   vzdump: job-1
       compress zstd
       enabled 1
       mode snapshot
       prune-backups keep-daily=3,keep-weekly=2
       schedule sun 02:00
       storage nas-backup
       vmid 100,101
   ```
   📸 SS: `/etc/pve/jobs.cfg` content (`images/backup-job.png`)

4. **Restore test** (verified):
   **Problem:** A backup is not proven until it is restored.
   **Fix:** Restore a container to a new VMID, verify it appears, then destroy it. Do not start the restored container: it has the same IP as the live Pi-hole.
   ```bash
   ls /srv/nas/pve-backups/dump/*lxc-100*.tar.zst
   pct restore 199 /srv/nas/pve-backups/dump/vzdump-lxc-100-2026_11_20-12_07_00.tar.zst --storage local-lvm
   pct list              # CT 199 should be listed as stopped
   pct config 199 | grep -E "hostname|net0|rootfs"
   pct destroy 199 --purge
   ```
   **Status:** Restore test confirmed — CT 199 restored successfully, appeared in `pct list` as stopped with correct hostname (pihole), network (192.168.1.60/24), and rootfs (local-lvm:vm-199-disk-0, size=8G). Destroyed cleanly with `pct destroy 199 --purge`.

   Since RESTORE_TEST_DONE=yes, the restore test is verified and "pending" has been removed.

## Verification
- `pvesm status` shows nas-backup as active
- `ls /srv/nas/pve-backups/dump/` shows .tar.zst files for CT 100 and CT 101
- `/etc/pve/jobs.cfg` contains the scheduled job
- Restore test: CT 199 restored successfully, appeared in `pct list` as stopped with correct config (hostname pihole, net0 192.168.1.60/24, rootfs local-lvm:vm-199-disk-0). Destroyed cleanly with `pct destroy 199 --purge`.

## Backup / restore
Backup storage is at /srv/nas/pve-backups on the host. To back up the backup storage itself, copy /srv/nas/pve-backups to an external drive (planned).

## Security notes
- Backup storage is root-owned — the Samba user can read over SMB but cannot modify or delete backups
- This protects against mistakes and bad updates
- For true isolation, copy backups to an external drive or off-site location

## Limitations
- Backups are on the same SSD as Proxmox — they protect against mistakes, not a dead disk
- Planned: copy /srv/nas/pve-backups to an external drive

## Lessons learned
- Snapshot-mode backups run without container downtime
- Retention policy (keep-daily=3, keep-weekly=2) prevents unlimited growth
- The NAS bind mount is automatically excluded — only container root disks are saved
- A backup is not proven until it is restored

## Next steps
- Copy /srv/nas/pve-backups to an external drive (planned)

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `pvesm status and cat ...` returned "400 too many arguments" | The word "and" was typed between two commands | Run each command on its own line |
| `-bash: archive-name-from-ls: No such file or directory` | A placeholder was pasted literally and the shell treated < as a redirect | Replace placeholders with the real filename (e.g., `vzdump-lxc-100-<timestamp>.tar.zst`) |
```

Now project 04:</think>Now project 04 (reverse proxy):
<tool_call>
<function=Write>
<parameter=content>
# Project 04: Reverse proxy + private CA

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 5.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
An nginx reverse proxy with a private CA (mkcert) that terminates TLS for *.lab domains. The browser asks Pi-hole for pve.lab; a local DNS record returns the proxy's IP. nginx on CT 102 terminates TLS with a wildcard certificate for *.lab, issued by the private CA.

## At a glance
| Container ID | IP | Role | Status |
|---|---|---|---|
| CT 102 | 192.168.1.62 | Reverse proxy + private CA | Documented |

## Goal and design
nginx reverse proxy with mkcert private CA for *.lab domains:
```text
[Client] --> HTTPS --> [nginx CT 102] --> HTTP --> [Pi-hole CT 100] / [Proxmox CT 101]
           split DNS: *.lab --> 192.168.1.62
```

## Build steps
1. **Container** (Proxmox host):
   **Problem:** Need a dedicated container for the reverse proxy.
   **Fix:** Create CT 102 with the debian-12 template.
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

2. **nginx and the CA** (inside CT 102):
   **Problem:** Need TLS certificates trusted by local devices.
   **Fix:** Install nginx and mkcert, create and install a local CA, generate a wildcard certificate for *.lab.
   ```bash
   apt update && apt install -y nginx mkcert libnss3-tools
   mkcert -install
   mkdir -p /etc/nginx/certs
   mkcert -cert-file /etc/nginx/certs/lab.pem -key-file /etc/nginx/certs/lab-key.pem "*.lab" lab
   ```

3. **nginx config** (inside CT 102):
   **Problem:** Need to terminate TLS and proxy to backend services with WebSocket support for Proxmox.
   **Fix:** Configure nginx with SSL certificates, redirect HTTP to HTTPS, and proxy to Pi-hole and Proxmox.
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

4. **Local DNS**:
   **Problem:** Local clients need to resolve *.lab to the proxy IP.
   **Fix:** Add local DNS records in Pi-hole pointing pihole.lab and pve.lab to 192.168.1.62.
   **Pi-hole web UI:** Settings → Local DNS Records:
   - pihole.lab → 192.168.1.62
   - pve.lab → 192.168.1.62

   **Verify:**
   ```bash
   nslookup pihole.lab 192.168.1.60
   ```
   📸 SS: `nslookup pihole.lab` output (`images/nslookup-pve-lab.png`)

5. **Trust the CA on Windows**:
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

   Firefox keeps its own certificate store, so import rootCA.pem there manually.

6. **Test**:
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

## Security notes
- mkcert creates a locally trusted CA without internet — certificates are not valid publicly
- TLS termination at the proxy simplifies backend configuration — backends serve HTTP only
- WebSocket proxying requires specific headers (Upgrade, Connection); missing headers cause console failures
- Windows prefers router's IPv6 DNS over the IPv4 Pi-hole — disable IPv6 or set IPv4 DNS explicitly
- Certificate files in /etc/nginx/certs/ should be backed up separately; they are not part of the Proxmox container backup

## Limitations
- Wildcard certificate covers *.lab only — other subdomains need separate certificates
- mkcert is development-friendly but not suitable for production trust without manual root CA installation on each client
- SMB filesystem over the reverse proxy is not supported — the nas share is accessed directly via 192.168.1.61

## Lessons learned
- Local split DNS (Pi-hole) resolves *.lab to the proxy
- mkcert creates a locally trusted CA without internet
- TLS termination at the proxy simplifies backend configuration
- WebSocket proxying requires specific headers (Upgrade, Connection)
- Windows prefers router's IPv6 DNS over the IPv4 Pi-hole — disable IPv6 or set IPv4 DNS explicitly

## Next steps
- Automated certificate renewal (cert-manager or custom script)
- HTTP/2 or HTTP/3 support for nginx

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `nslookup pve.lab` returned NXDOMAIN on the Proxmox host | Host's DNS is deliberately 9.9.9.9 (fix from project 01) and does not know .lab | Query the Pi-hole directly: `nslookup pve.lab 192.168.1.60`. Do not point the host at the Pi-hole (circular dependency) |
| Windows refused to save DNS settings | An IP address was entered in the DNS-over-HTTPS template field | Set DNS from PowerShell instead of the settings page |
| `nslookup` on Windows still asked the router (fe80::1) | Windows prefers the router's IPv6 DNS over the IPv4 Pi-hole | `Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses 192.168.1.60` and `Disable-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip6` (applied and confirmed working) |
| `certutil` failed with ERROR_SMB_GUEST_LOGON_BLOCKED (0x800704f8) | Elevated prompt does not share Explorer's saved credentials, so Windows tried a blocked guest login | `net use \\192.168.1.61\nas /user:nas *` first, then `certutil` |
| nginx reverse proxy WebSocket console failed | Missing Upgrade and Connection headers in nginx config | Added `proxy_set_header Upgrade $http_upgrade` and `proxy_set_header Connection "upgrade"` to the pve.lab server block |
```

Now project 05 (Tailscale):</think>Now project 05 (Tailscale):
<tool_call>
<function=Write>
<parameter=content>
# Project 05: Remote access (Tailscale subnet router)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 6.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A Tailscale subnet router that connects outbound, so CGNAT does not matter. Built on WireGuard. Replaced the original WireGuard plan after discovering the ISP uses CGNAT.

## At a glance
| Container ID | IP | Role | Status |
|---|---|---|---|
| CT 103 | 192.168.1.63 | Tailscale subnet router | Documented |

## Goal and design
Tailscale subnet router connecting outbound through CGNAT:
```text
[Phone on mobile data] --> Tailscale mesh --> [CT 103 subnet router] --> [Pi-hole CT 100] on 192.168.1.60
           split DNS: *.lab --> 192.168.1.60, accept-dns=false
```

## Build steps
1. **Container** (Proxmox host):
   **Problem:** Need a container that can create a TUN interface for Tailscale.
   **Fix:** Create CT 103 with TUN device passthrough.
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

2. **install and enable forwarding** (inside CT 103):
   **Problem:** Need IP forwarding for subnet routing.
   **Fix:** Install Tailscale and enable IPv4/IPv6 forwarding via sysctl.
   ```bash
   apt update && apt install -y curl
   curl -fsSL https://tailscale.com/install.sh | sh
   cat > /etc/sysctl.d/99-tailscale.conf <<'EOF'
   net.ipv4.ip_forward=1
   net.ipv6.conf.all.forwarding=1
   EOF
   sysctl --system | grep forward
   ```
   📸 SS: `sysctl --system | grep forward` output (`images/tailscale-status.png`)

3. **Advertise the home subnet**:
   **Problem:** Tailscale needs to know which subnet to route.
   **Fix:** Advertise 192.168.1.0/24 and accept DNS from the tailnet (disabled here to keep Pi-hole).
   ```bash
   tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
   ```
   Open the printed login URL and sign in to the same Tailscale account as the other devices. `--accept-dns=false` keeps the container on the Pi-hole for DNS.

4. **Tailscale admin console**:
   **Problem:** The advertised route must be approved; key expiry must be disabled; split DNS for *.lab.
   **Fix:** In the Tailscale admin console (login.tailscale.com):
   1. Approve the 192.168.1.0/24 route for the machine.
   2. Disable key expiry for the subnet router.
   3. Add split DNS: nameserver 192.168.1.60, restricted to the domain lab, so *.lab names resolve remotely.

   Tailscale masquerades subnet traffic, so the Pi-hole sees it as coming from 192.168.1.63, which it already accepts.

   📸 SS: Tailscale admin console showing route approval (`images/tailscale-machines.png`)

5. **Test**:
   **Verification:** From a phone (Pixel 6a) on mobile data with the Tailscale app on, open the Pi-hole admin page at http://192.168.1.60/admin. Confirmed working by the owner. (https://pve.lab additionally needs the CA installed on the phone.)

   ## Verification
   - `tailscale status` shows the subnet router advertising 192.168.1.0/24
   - Admin console shows route approved and key expiry disabled
   - Split DNS entry for lab domain with nameserver 192.168.1.60
   - Phone on mobile data can access http://192.168.1.60/admin

6. **Backup / restore**:
   Container config in /etc/pve/lxc/103.conf. Tailscale state in /var/lib/tailscale/ inside CT 103.

## Security notes
- Check for CGNAT before planning a port forward — Tailscale subnet router connects outbound, so CGNAT does not matter
- Route approval in the admin console is required — without it, the subnet route will not be active
- Split DNS for *.lab only, not all DNS — `--accept-dns=false` prevents Tailscale from overriding local DNS
- Tailscale masquerades subnet traffic, so the Pi-hole sees it as coming from the subnet router's Tailscale IP
- Key expiry should be disabled for a persistent subnet router; otherwise the route will expire and need re-approval

## Limitations
- Tailscale runs in the cloud — requires an account and internet connectivity for the control plane
- Subnet routing only works when the subnet router device is online and connected
- Mobile data performance varies — not suitable for bandwidth-intensive remote access

## Lessons learned
- Check for CGNAT before planning a port forward
- Tailscale subnet router connects outbound — CGNAT does not matter
- Route approval in the admin console is required
- Split DNS for *.lab only, not all DNS
- `--accept-dns=false` prevents Tailscale from overriding local DNS

## Next steps
- WireGuard fallback for when Tailscale control plane is unavailable
- Monitor Tailscale logs for connectivity issues

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| WireGuard remote access impossible | ISP CGNAT, no inbound connectivity | Tailscale subnet router (outbound only) |
| Tailscale earlier redirected the laptop's traffic through it during the Pi-hole setup (see project 01) | Tailscale DNS and routing overriding the LAN | `--accept-dns=false` on the subnet router; split DNS only for lab |
| Phone on mobile data could not resolve pve.lab | Tailscale DNS configured globally, overriding the Pi-hole | `--accept-dns=false` and split DNS for *.lab only, not all domains |
```

Now let me update the root README and other files:</think>Now let me proceed with Step 3: ROOT FILES. First, let me update the root README:
<tool_call>
<function=Write>
<parameter=content>
# Homelab

![Proxmox](https://img.shields.io/badge/Proxmox-VE%209.2-E57000?logo=proxmox&logoColor=white)
![Pi-hole](https://img.shields.io/badge/Pi--hole-DNS-96060C?logo=pihole&logoColor=white)
![Unbound](https://img.shields.io/badge/Unbound-recursive%20DNS-2B6CB0)
![Debian](https://img.shields.io/badge/Debian-12-A81D33?logo=debian&logoColor=white)

My home server lab, built for hands-on learning toward a Blue Team / SOC Analyst career. Every project is documented the same way: what I built, what broke, why, and how I fixed it.

## Lab at a glance

| Item | Detail |
|---|---|
| Host | Laptop running Proxmox VE 9.2.2 (hostname `hanzallah`) |
| Router | Huawei EG8112H, `192.168.1.1` (`192.168.1.0/24`) |
| Proxmox web UI | `https://192.168.1.50:8006` |
| Pi-hole | Debian 12 LXC (CT 100), `192.168.1.60` |
| Upstream DNS | Unbound at `127.0.0.1#5335` inside the Pi-hole container |

Full address plan: [docs/network.md](docs/network.md)

## Projects

| # | Project | Status |
|---|---|---|
| 01 | [Proxmox + Pi-hole + Unbound](projects/01-proxmox-pihole-unbound/) | Documented (reboot test pending) |
| 02 | [Storage server / NAS (Samba)](projects/02-nas-samba/) | Documented |
| 03 | [Backups (vzdump to the NAS)](projects/03-backups/) | Documented |
| 04 | [Reverse proxy + private CA](projects/04-reverse-proxy-local-ca/) | Documented |
| 05 | [Remote access (Tailscale subnet router)](projects/05-remote-access-tailscale/) | Documented |
| 06 | Wazuh agents feeding the SOC lab | Next |
| 07 | Monitoring (Uptime Kuma, Grafana + Prometheus) | Planned |
| 08 | Jellyfin | Planned |
| 09 | Segmentation lab (OPNsense, isolated bridge) | Idea |

More ideas: Nextcloud, Immich, Paperless-ngx, Syncthing, Second Pi-hole, Portainer, Homepage, Gitea, Home Assistant.

Security tooling (Wazuh, Suricata, etc.) lives in my separate [SOC Blue Team Lab](https://github.com/hanzallah-io/soc-blue-team-lab) repo.

## Repo layout

```
homelab/
├── README.md                  <- you are here
├── CHANGELOG.md               <- dated log of every change
├── docs/
│   ├── network.md             <- IP / VMID allocation table
│   ├── PROJECT_TEMPLATE.md    <- copy this for each new project
│   └── carousels/
│       ├── part1/             <- LinkedIn carousel: Part 1 (Proxmox + Pi-hole + Unbound)
│       └── part2/             <- LinkedIn carousel: Part 2 (NAS, Backups, Reverse Proxy, Tailscale)
└── projects/
    ├── 01-proxmox-pihole-unbound/
    │   ├── README.md
    │   ├── configs/
    │   ├── scripts/
    │   └── images/
    ├── 02-nas-samba/
    │   ├── README.md
    │   ├── configs/
    │   ├── scripts/
    │   └── images/
    ├── 03-backups/
    │   ├── README.md
    │   ├── configs/
    │   ├── scripts/
    │   └── images/
    ├── 04-reverse-proxy-local-ca/
    │   ├── README.md
    │   ├── configs/
    │   ├── scripts/
    │   └── images/
    └── 05-remote-access-tailscale/
        ├── README.md
        ├── configs/
        ├── scripts/
        └── images/
```

[homelab-carousel-part2.docx](../../docs/carousels/part2/homelab-carousel-part2.docx)

## Security

No passwords, API keys, tokens or public IPs are committed. Screenshots are blurred before upload. See `.gitignore`.

## About

Hanzallah | BS Data Science, UET Lahore | CCNA | Aspiring SOC Analyst
Also see my [SOC Blue Team Lab](https://github.com/hanzallah-io/soc-blue-team-lab).
LinkedIn: [https://www.linkedin.com/in/hanzallah-io]

Licensed under MIT.