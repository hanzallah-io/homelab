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
| 02 | Phone screen streaming (Docker) | Planned |
| 03 | Storage server / NAS (Samba) | Planned |
| 04 | Backups (Proxmox Backup Server or vzdump) | Planned |
| 05 | Jellyfin media server | Planned |
| 06 | Reverse proxy (Nginx Proxy Manager / Traefik) | Idea |
| 07 | Monitoring (Uptime Kuma, Grafana + Prometheus) | Idea |
| 08 | Remote access (WireGuard / Tailscale) | Idea |

More ideas: Nextcloud, Immich, Paperless-ngx, Syncthing, Second Pi-hole, Portainer, Homepage, Gitea, Home Assistant.

Security tooling (Wazuh, Suricata, etc.) lives in my separate [SOC Blue Team Lab](https://github.com/hanzallah-io/soc-blue-team-lab) repo.

## Repo layout

```
homelab/
├── README.md                  <- you are here
├── CHANGELOG.md               <- dated log of every change
├── docs/
│   ├── network.md             <- IP / VMID allocation table
│   └── PROJECT_TEMPLATE.md    <- copy this for each new project
└── projects/
    └── 01-proxmox-pihole-unbound/
        ├── README.md          <- full write-up (problems + fixes)
        ├── configs/           <- sanitized config files
        ├── scripts/           <- helper and health-check scripts
        ├── images/            <- screenshots
        └── docs/              <- LinkedIn carousel PDF
```

## Adding a new project

1. Copy `docs/PROJECT_TEMPLATE.md` into `projects/NN-project-name/README.md`
2. Add `configs/`, `scripts/` and `images/` as needed
3. Add the project to the table above and to `docs/network.md`
4. Log it in `CHANGELOG.md`

## Security

No passwords, API keys, tokens or public IPs are committed. Screenshots are blurred before upload. See `.gitignore`.

## About

Hanzallah | BS Data Science, UET Lahore | CCNA | Aspiring SOC Analyst
Also see my [SOC Blue Team Lab](https://github.com/hanzallah-io/soc-blue-team-lab).
LinkedIn: [https://www.linkedin.com/in/hanzallah-io]

Licensed under MIT.