# Project 02: Storage server / NAS (Samba)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 3.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A Samba file server on an unprivileged LXC container. Data lives on a thin LVM volume on the Proxmox host, bind-mounted into the container. The container can be rebuilt without touching data, and extra disks can be added later with mergerfs without changing anything inside the container.

## Environment
| Item | Detail |
|---|---|
| Type (VM / LXC / Docker) | LXC (Debian 12, unprivileged) |
| ID / hostname | CT 101 / nas |
| IP and ports | 192.168.1.61, 445 (SMB) |
| Resources (CPU / RAM / disk) | 1 core, 512 MB RAM, 256 MB swap, 4 GB rootfs |

## Phase 1: storage volume (Proxmox host, root)
**Problem:** Need persistent storage that survives container rebuilds.
**Fix:** Create a thin LVM volume on the host, mount at /srv/nas, and bind-mount into the container.

**Host (root):**
```bash
lvcreate -V 250G -T pve/data -n nas
mkfs.ext4 -L nas /dev/pve/nas
mkdir -p /srv/nas
echo '/dev/pve/nas /srv/nas ext4 defaults,nofail 0 2' >> /etc/fstab
systemctl daemon-reload && mount -a
df -h /srv/nas
```
`nofail` stops a mount problem from blocking boot. Result: about 233 GB available on /srv/nas.

📸 SS: `df -h /srv/nas` output (`images/storage-volume.png`)

## Phase 2: container (Proxmox host)
**Problem:** Need an unprivileged container with networking and nesting enabled for Samba.
**Fix:** Create CT 101 with the debian-12 template.

**Host (root):**
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

📸 SS: `pct config 101` output (`images/container-config.png`)

## Phase 3: bind mount (Proxmox host)
**Problem:** Container needs access to the host storage volume.
**Fix:** Bind-mount /srv/nas into the container at /mnt/nas, adjusting UID mapping for the unprivileged container.

**Host (root):**
```bash
pct set 101 -mp0 /srv/nas,mp=/mnt/nas
chown -R 101000:101000 /srv/nas
pct start 101
```
Unprivileged containers shift UIDs by 100000, so host UID 101000 is UID 1000 inside the container, which is the Samba user created next.

## Phase 4: Samba (inside CT 101)
**Problem:** Need SMB file sharing with a dedicated service account.
**Fix:** Install Samba, create a nas user/group (UID/GID 1000), configure the share, and set a Samba password.

**Container (root@nas):**
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

📸 SS: `testparm -s` output (`images/samba-config.png`)

## Phase 5: connect from Windows
**Verification:** Open File Explorer, enter `\\192.168.1.61\nas`, log in as `nas`.

📸 SS: Windows File Explorer showing the share (`images/nas-share-windows.png`)

## Verification
- `testparm -s` shows the share config without errors
- `systemctl status smbd` shows active (running)
- Windows can connect, read, and write files
- `ls -ld /mnt/nas` shows owner nas:nas

## Backup / restore
Data is on the host at /srv/nas. Back up the host volume with vzdump (see project 03). Container config is in /etc/pve/lxc/101.conf.

## Key takeaways
- Thin LVM provisioning lets the volume grow as needed
- UID mapping (100000 offset) is critical for unprivileged container bind mounts
- Keeping data on the host decouples it from the container lifecycle
- Always check the prompt (root@nas vs root@hanzallah) before running commands

## Still open
- External disks pooled with mergerfs (planned)
- Enterprise apt repos on Proxmox host return 401; disable and use pve-no-subscription (pending)

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Samba install "failed" and a stray nas user and group appeared on the Proxmox host | Commands pasted on the host instead of inside the container; apt update aborted on enterprise-repo error so Samba never installed, but groupadd/useradd still ran | userdel nas; groupdel nas on the host, then repeat inside the container. Habit now: check the prompt (root@nas) before running anything |
| apt update on the host returned 401 Unauthorized for the enterprise Proxmox and Ceph repos | Enterprise repos need a paid subscription | Not yet applied. Disable enterprise repos and use pve-no-subscription. Flagged as pending |
| Container console rejected root and nas logins | No root password set; nas is Samba-only with a nologin shell | Enter from the host with pct enter 101, or set a root password with pct exec 101 -- passwd |
| ls /srv/nas and chown failed inside the container | /srv/nas only exists on the host; inside the container the same folder is /mnt/nas | Run host paths on the host, container paths in the container |