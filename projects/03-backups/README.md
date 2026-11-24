# Project 03: Backups (vzdump to the NAS)

> LinkedIn carousel: [Homelab Series, Part 2](../../docs/carousels/part2/homelab-carousel-part2.docx). This project is on slide 4.

![status](https://img.shields.io/badge/status-documented-blue)

## Overview
A folder on the NAS volume is registered as Proxmox backup storage. It is root-owned, so the Samba user can read it over SMB but cannot modify or delete backups. This protects against mistakes and bad updates.

## Environment
| Item | Detail |
|---|---|
| Type (VM / LXC / Docker) | Proxmox backup storage (directory) on host |
| ID / hostname | Storage ID: nas-backup, path /srv/nas/pve-backups |
| IP and ports | N/A (local storage) |
| Resources (CPU / RAM / disk) | Uses NAS volume (project 02), ~233 GB available |

## Phase 1: backup storage (Proxmox host)
**Problem:** Need a backup target that lives on the NAS volume.
**Fix:** Create the backup directory and register it as Proxmox storage with retention policy.

**Host (root):**
```bash
mkdir -p /srv/nas/pve-backups
pvesm add dir nas-backup --path /srv/nas/pve-backups --content backup \
  --prune-backups keep-daily=3,keep-weekly=2
pvesm status
```

📸 SS: `pvesm status` showing nas-backup active (`images/pvesm-status.png`)

## Phase 2: manual backup (no downtime)
**Problem:** Need to back up running containers without downtime.
**Fix:** Use vzdump snapshot mode with zstd compression. The NAS bind mount (mp0) is excluded automatically, so only each container's own root disk is saved, not the NAS data.

**Host (root):**
```bash
vzdump 100 101 --storage nas-backup --mode snapshot --compress zstd
ls -lh /srv/nas/pve-backups/dump/
```

📸 SS: Backup files in /srv/nas/pve-backups/dump/ (`images/vzdump-output.png`)

## Phase 3: weekly schedule
**Problem:** Need automated weekly backups.
**Fix:** Create a scheduled vzdump job via pvesh.

**Host (root):**
```bash
pvesh create /cluster/backup --storage nas-backup --vmid 100,101 \
  --schedule "sun 02:00" --mode snapshot --compress zstd \
  --prune-backups keep-daily=3,keep-weekly=2 --enabled 1
cat /etc/pve/jobs.cfg
```

Confirmed output of `pvesm status` (nas-backup active) and `/etc/pve/jobs.cfg` (see configs/jobs.cfg):
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

## Phase 4: restore test (verified)
**Problem:** A backup is not proven until it is restored.
**Fix:** Restore a container to a new VMID, verify it appears, then destroy it. Do not start the restored container: it has the same IP as the live Pi-hole.

**Host (root):**
```bash
ls /srv/nas/pve-backups/dump/*lxc-100*.tar.zst
pct restore 199 /srv/nas/pve-backups/dump/vzdump-lxc-100-2026_11_20-12_07_00.tar.zst --storage local-lvm
pct list              # CT 199 should be listed as stopped
pct config 199 | grep -E "hostname|net0|rootfs"
pct destroy 199 --purge
```

**Status:** Restore test confirmed — CT 199 restored successfully, appeared in `pct list` as stopped with correct hostname (pihole), network (192.168.1.60/24), and rootfs (local-lvm:vm-199-disk-0, size=8G). Destroyed cleanly with `pct destroy 199 --purge`.

## Verification
- `pvesm status` shows nas-backup as active
- `ls /srv/nas/pve-backups/dump/` shows .tar.zst files for CT 100 and CT 101
- `/etc/pve/jobs.cfg` contains the scheduled job
- Restore test: CT 199 restored successfully, appeared in `pct list` as stopped with correct config (hostname pihole, net0 192.168.1.60/24, rootfs local-lvm:vm-199-disk-0). Destroyed cleanly with `pct destroy 199 --purge`.

## Backup / restore
Backup storage is at /srv/nas/pve-backups on the host. To back up the backup storage itself, copy /srv/nas/pve-backups to an external drive (planned).

## Key takeaways
- Snapshot-mode backups run without container downtime
- Retention policy (keep-daily=3, keep-weekly=2) prevents unlimited growth
- The NAS bind mount is automatically excluded — only container root disks are saved
- A backup is not proven until it is restored

## Still open
- Backups are on the same SSD as Proxmox — they protect against mistakes, not a dead disk
- Planned: copy /srv/nas/pve-backups to an external drive

## Problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `pvesm status and cat ...` returned "400 too many arguments" | The word "and" was typed between two commands | Run each command on its own line |
| `-bash: archive-name-from-ls: No such file or directory` | A placeholder was pasted literally and the shell treated < as a redirect | Replace placeholders with the real filename (e.g., `vzdump-lxc-100-<timestamp>.tar.zst`) |