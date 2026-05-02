# Linux Home Lab — Production Infrastructure

A production-grade self-hosted infrastructure running on a dedicated Ubuntu home server, connected to a fleet of VPS nodes across multiple cloud providers via a zero-trust Tailscale VPN mesh.

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│           Home Server (Hermes)              │
│           Ubuntu Linux                      │
│           24 Docker Containers              │
├─────────────────┬───────────────────────────┤
│   Applications  │   Infrastructure          │
│─────────────────│───────────────────────────│
│  LobeHub AI     │  Caddy reverse proxy      │
│  Immich Photos  │  PostgreSQL (x3)          │
│  Outline Wiki   │  Redis (x3)               │
│  Matrix Synapse │  MariaDB                  │
│  Element Web    │  Prometheus               │
│  Frigate NVR    │  Grafana                  │
│  Filestash      │  BorgBackup               │
│  Silverbullet   │  Syncthing                │
└─────────────────┴───────────────────────────┘
                    │
            Tailscale VPN Mesh
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───────┐     ┌──────────┐    ┌──────────┐
│  VPS  │     │   VPS    │    │   VPS    │
│  Apps │     │  RemoteVPS│   │  FlowShift│
│  DO   │     │  Oracle  │    │   DO     │
│n8n    │     │Prometheus│    │  Caddy   │
│NocoDB │     │Grafana   │    │  Fail2ban│
│Invoice│     │CouchDB   │    │          │
│Ninja  │     │RustDesk  │    │          │
└───────┘     └──────────┘    └──────────┘
```

## Stack

### Home Server

| Service | Purpose | Stack |
|---------|---------|-------|
| Matrix Synapse | Self-hosted E2EE messaging | Docker |
| Element Web | Matrix web client | Docker |
| Mautrix-GMessages | Google Messages bridge | Docker |
| Immich  | Self-hosted photo backup | Docker |
| LobeHub | Self-hosted AI chat | Docker |
| Outline | Team wiki | Docker |
| Frigate | NVR / camera system | Docker |
| Filestash | Web file manager | Docker |
| Copilot API | LLM proxy server | Docker |
| Caddy   | Reverse proxy + TLS | systemd |
| Cloudflared | Zero-trust tunnel | systemd |
| Tailscale | VPN mesh | systemd |
| Samba   | File sharing (SMB) | systemd |
| Syncthing | File sync | systemd |
| BorgBackup | Automated backups | cron  |

### VPS Fleet

| Host | Provider | Services |
|------|----------|----------|
| apps | DigitalOcean | n8n, NocoDB, Invoice Ninja, PostgreSQL, Adminer |
| remotevps | Oracle Cloud (ARM) | Prometheus, Grafana, TimescaleDB, CouchDB, SearXNG, RustDesk |
| flowshiftmedia | DigitalOcean | Caddy (public reverse proxy), Fail2ban |
| ovh  | OVH      | Copilot API, Crawl4AI |

## Networking

* **Zero-trust VPN mesh** via Tailscale ACLs, all inter-node communication stays private
* **Reverse proxy** via Caddy with automatic Let's Encrypt TLS on all public services
* **DNS** via dnsmasq for internal resolution and claudflare Acme challange with wildcard
* **Cloudflare Tunnel** for selected public services without port exposure

## Backup Strategy

* **Tool**: BorgBackup
* **Schedule**: Daily via cron
* **Verification**: Post-backup integrity check via `borg check`
* **Monitoring**: Backup log tailed and reviewed via rsyslog

## Monitoring

* Prometheus scraping metrics from all nodes
* Grafana dashboards for service health, disk, memory, and network
* TimescaleDB for long-term time-series retention
* systemd journal for per-service logging

## Key Highlights

* 24 Docker containers managed via Docker Compose across multiple stacks
* 5 VPS nodes across 3 cloud providers (DigitalOcean, OVH, Oracle Cloud)
* All services behind TLS with automated cert renewal
* End-to-end encrypted self-hosted messaging (Matrix/Element)
* Custom Android app with on-device speech-to-text (Parakeet v3 model, 2.3GB)


---

*This infrastructure is actively maintained and expanded.*
