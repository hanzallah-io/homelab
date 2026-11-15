# Network and ID allocation

Update this file whenever a service is added.

| Host | Type | ID | IP | Ports | Purpose |
|---|---|---|---|---|---|
| Router (Huawei EG8112H) | Hardware | - | 192.168.1.1 | 80/443 | Gateway, DHCP (DNS fields locked by ISP) |
| `hanzallah` | Proxmox VE 9.2.2 host | - | 192.168.1.50 | 8006 | Hypervisor |
| `pihole` | LXC (Debian 12) | 100 | 192.168.1.60 | 53, 80 | DNS filtering |
| Unbound | Service inside `pihole` | 100 | 127.0.0.1 | 5335 | Recursive resolver |
| [next service] | | | | | |

## DNS flow

Client -> Pi-hole (filter + log) -> Unbound (recursive, DNSSEC) -> root servers

## Rules learned the hard way

- The Proxmox host must not use the Pi-hole as its own DNS (circular dependency). Host resolvers: `9.9.9.9`, `1.1.1.1`.
- Tailscale on the admin laptop can reroute LAN traffic. Turn it off (or allow local network access) when managing the lab.
- The host runs `chrony` for time sync; LXC containers use the host clock.