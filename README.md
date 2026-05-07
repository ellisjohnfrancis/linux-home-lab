# Flowshift Media Production Infrastructure

A production-grade self-hosted infrastructure running on a dedicated Ubuntu home server and gateway node, connected to a fleet of VPS nodes across multiple cloud providers via a zero-trust Tailscale VPN mesh. Designed, built, and operated independently to support business operations, AI automation pipelines, and internal tooling.

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Home Network                       │
│                                                      │
│  ┌─────────────────────────┐  ┌───────────────────┐  │
│  │   Home Server (Hermes)  │  │  Gateway Machine  │  │
│  │   Ubuntu Linux          │  │  Intel vPro (AMT) │  │
│  │   24 Docker Containers  │  │  dnsmasq DNS      │  │
│  │                         │  │  Tailscale node   │  │
│  │  Applications:          │  └───────────────────┘  │
│  │  LobeHub AI             │                          │
│  │  Immich Photos          │                          │
│  │  Outline Wiki           │                          │
│  │  Matrix Synapse         │                          │
│  │  Element Web            │                          │
│  │  Frigate NVR            │                          │
│  │  Filestash              │                          │
│  │  Silverbullet           │                          │
│  │                         │                          │
│  │  Infrastructure:        │                          │
│  │  Caddy reverse proxy    │                          │
│  │  PostgreSQL (x3)        │                          │
│  │  Redis (x3)             │                          │
│  │  BorgBackup             │                          │
│  │  Syncthing              │                          │
│  │  Ollama / llama.cpp     │                          │
│  └─────────────────────────┘                          │
└──────────────────────────────────────────────────────┘
                        │
                Tailscale VPN Mesh
                (default-deny ACLs)
                        │
    ┌───────────────────┼──────────────────────┐
    │                   │                      │
┌─────────┐      ┌────────────┐         ┌──────────┐     ┌─────────┐
│  Apps   │      │ RemoteVPS  │         │Flowshift │     │   OVH   │
│   DO    │      │Oracle Cloud│         │  Media   │     │         │
│         │      │            │         │    DO    │     │Scraping │
│ n8n     │      │Prometheus  │         │  Caddy   │     │Lead Gen │
│ NocoDB  │      │Grafana     │         │  Fail2ban│     │Staging  │
│Invoice  │      │TimescaleDB │         │          │     │         │
│ Ninja   │      │RustDesk    │         │          │     │         │
│PostgreSQL      │SearXNG     │         │          │     │         │
└─────────┘      └────────────┘         └──────────┘     └─────────┘
```

## Stack

### Home Server

| Service | Purpose | Stack |
|---------|---------|-------|
| Matrix Synapse | Self-hosted E2EE messaging | Docker |
| Element Web | Matrix web client | Docker |
| Mautrix-GMessages | Google Messages bridge | Docker |
| Immich | Self-hosted photo backup | Docker |
| LobeHub | Self-hosted AI chat | Docker |
| Outline | Team wiki | Docker |
| Frigate | NVR / camera system | Docker |
| Filestash | Web file manager | Docker |
| Copilot API | LLM proxy server | Docker |
| Ollama | Local model serving (network-accessible) | Docker |
| llama.cpp | High-performance local inference | Docker |
| Caddy | Reverse proxy + TLS | systemd |
| Cloudflared | Zero-trust tunnel | systemd |
| Tailscale | VPN mesh | systemd |
| Samba | File sharing (SMB) | systemd |
| SFTP | Encrypted file transfer | systemd |
| Syncthing | File sync | systemd |
| BorgBackup | Automated backups | cron |

### Gateway Machine

| Service | Purpose |
|---------|---------|
| dnsmasq | Internal DNS resolution |
| Tailscale | VPN mesh node |
| Intel vPro (AMT) | Out-of-band remote management — OS-independent reboot and access |

### VPS Fleet

| Host | Provider | Services |
|------|----------|----------|
| apps | DigitalOcean | n8n, NocoDB, Invoice Ninja, PostgreSQL, Adminer |
| remotevps | Oracle Cloud | Prometheus, Grafana, TimescaleDB, SearXNG, RustDesk |
| flowshiftmedia | DigitalOcean | Caddy (public reverse proxy), Fail2ban |
| ovh | OVH | Crawl4AI, lead scraping/enrichment, staging environment |

## Networking

* **Zero-trust VPN mesh** via Tailscale with default-deny ACLs — all inter-node traffic is least-privilege
* **All SSH access** via Tailscale, no public SSH ports open on any node
* **Public services** reverse proxied from the Flowshift Media VPS over Tailscale to internal nodes, no internal services exposed directly
* **Reverse proxy** via Caddy with automatic Let's Encrypt TLS on all public services
* **Wildcard SSL** via Cloudflare ACME DNS challenge enables HTTPS on internal VPN-only services without public exposure
* **DNS** via dnsmasq on gateway machine for internal resolution
* **Cloudflare Tunnel** for selected public services without port exposure
* **Firewall** via UFW with iptables for advanced rules (e.g. WireGuard forwarding)

## Automation & AI

* **100+ n8n workflows** (1,229+ nodes, 50+ integrations) for lead generation, CRM pipeline automation, LLM-powered email outreach with inbox rotation, and multi-channel Slack routing
* **Idempotent workflow design** — all workflows filter against PostgreSQL state to prevent duplicate processing and API credit waste
* **Local LLM inference** via Ollama and llama.cpp, network-accessible to all Tailscale nodes
* **MCP servers** for SERP search, web scraping/crawling, PostgreSQL querying, and a custom-built YouTube transcription MCP (yt-dlp + AssemblyAI)
* **AI agents** in n8n with tool/function calling for lead research and Apollo API enrichment

## Backup Strategy

* **Tool**: BorgBackup
* **Schedule**: Daily via cron
* **Scope**: Databases and critical service data
* **Off-site**: VPS database backups write to home server; media backed up to off-site storage
* **Verification**: Post-backup integrity check via `borg check`
* **Monitoring**: Backup logs via rsyslog with Slack alerting on failure
* **Tested**: Backup restored in production after n8n Docker volume overwrite — full recovery with zero data loss

## Monitoring

* Prometheus scraping metrics across nodes
* Grafana dashboards for service health, disk, memory, and network
* TimescaleDB for long-term time-series retention (used for high-frequency crypto spot price data at 30-second intervals)
* Slack alerting for workflow errors, backup failures, and data integrity gaps
* systemd journald for per-service logging

## Key Highlights

* 6-node distributed infrastructure: 1 physical server, 1 gateway machine, 4 VPS instances across 3 cloud providers
* 24 Docker containers managed via Docker Compose across multiple stacks
* Zero-trust network: default-deny Tailscale ACLs, no open management ports, SSH over VPN only
* All services behind TLS with automated cert renewal including internal VPN-only services via wildcard cert
* Intel vPro out-of-band management on home server provides OS-independent remote reboot
* Local LLM inference accessible across the entire VPN mesh
* Custom MCP server for YouTube transcription (yt-dlp + AssemblyAI)
* Production-tested backup recovery, restored from BorgBackup after live data loss incident

---

*This infrastructure is actively maintained and expanded.*
