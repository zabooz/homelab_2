# Project Guide

Homelab Wiki - Dokumentation für die gesamte Homelab-Infrastruktur, Services und Netzwerk-Konzepte. Wird als Wiki.js Wiki betrieben und via Git synchronisiert.

## Struktur

```
homelab_2/
├── home.md                          # Wiki Startseite (Index aller Seiten)
├── infrastructure/                  # Server & Netzwerk Infrastruktur
│   ├── NETWORK_OVERVIEW.md          # Master-Referenz: IPs, Dienste, Topologie
│   ├── VPS.md                       # VPS: Nginx, Headscale, Stream Proxy
│   └── proxmox-netzwerk-setup.md    # Proxmox Netzwerk-Konfiguration
├── services/                        # Service-Dokumentationen
│   ├── homepage-dashboard-setup.md  # Homepage Dashboard (192.168.0.123)
│   ├── wikijs-setup.md              # Wiki.js + Git Sync (192.168.0.124)
│   ├── stats-api.md                 # Bun Monitoring API
│   ├── paperless-ngx.md             # Dokumentenverwaltung
│   ├── fog_project_setup_dokumentation_debian_12_lxc.md  # FOG/PXE/DHCP
│   ├── drawio-setup.md              # Draw.io (192.168.0.122)
│   ├── linkwarden-setup.md          # Bookmark Manager
│   ├── homeassistant-setup.md       # Home Assistant Smart Home
│   ├── azerothcore-playerbots-setup.md  # WoW Server (AzerothCore)
│   ├── pterodactyl-setup.md         # Gameserver Panel + Wings
│   ├── n8n-setup.md                 # n8n Workflow Automation (192.168.0.116)
│   ├── changedetection-setup.md     # Website-Monitoring (192.168.0.125)
│   ├── gotify-setup.md              # Push-Benachrichtigungen (192.168.0.126)
│   ├── sterlingpdf-setup.md         # PDF-Tools (192.168.0.131)
│   ├── ntopng-setup.md              # Netzwerk-Monitoring (192.168.0.140)
│   ├── crawler4ai-setup.md          # Web Scraping
│   ├── node-red-setup.md            # Flow-basierte Automation
│   └── immich-setup.md              # Google Photos Alternative (192.168.0.144)
├── workflows/                       # n8n Workflow-Dokumentationen
│   └── vm-image-update.md           # VM Image Update Workflow (FOG + Proxmox)
├── vpn/                             # VPN Dokumentation
│   ├── vpn-infrastructure.md        # Headscale Server Setup
│   ├── vpn-client-guide.md          # Client Verbindungsanleitung
│   └── vpn-troubleshooting.md       # VPN Fehlerbehebung
├── operations/                      # Betrieb & Wartung
│   ├── backup-recovery.md           # Backup Prozeduren
│   ├── disaster-recovery.md         # Notfall-Wiederherstellung
│   └── security-hardening.md        # Sicherheitshärtung
└── konzepte/                        # Netzwerk-Konzepte (LAP Prüfung)
    ├── ip-adressen.md               # IPv4, IPv6
    ├── subnetting.md                # CIDR, Subnetzmasken
    ├── dhcp.md                      # DORA-Prozess
    ├── routing.md                   # Gateway, Routing-Tabellen
    ├── nat.md                       # SNAT, DNAT, MASQUERADE
    ├── ip-forwarding.md             # Linux Router-Funktion
    ├── dns.md                       # DNS Hierarchie, MagicDNS
    ├── iptables.md                  # Firewall, Chains, Tables
    ├── linux-interfaces.md          # Interface-Benennung
    ├── vpn.md                       # WireGuard vs OpenVPN
    ├── ports.md                     # Standard- und Homelab-Ports
    ├── reverse-proxy.md             # HTTP + Stream Proxy, TCP/UDP
    ├── diagnose.md                  # Troubleshooting-Befehle
    └── pruefungsfragen.md           # LAP Q&A
```

## Wichtige Hosts

| Host | IP | Tailscale IP | Funktion |
|------|-----|-------------|----------|
| VPS | 152.53.111.11 | 100.64.0.5 | Nginx, Headscale, Stream Proxy |
| homeserver | 192.168.0.101 | - | Proxmox Node 1 (i5-6500T, 32GB, 2TB SSD) |
| homeserver2 | 192.168.0.102 | - | Proxmox Node 2 (i3-8100, 32GB, NVMe+SSD) |
| Tailscale LXC | 192.168.0.112 | 100.64.0.1 | Subnet Router, Exit Node |
| Draw.io LXC | 192.168.0.122 | - | Diagramm-Editor |
| Homepage LXC | 192.168.0.123 | - | Homepage Dashboard |
| Wiki.js LXC | 192.168.0.124 | - | Wiki.js + Git Sync |
| n8n LXC | 192.168.0.116 | - | Workflow Automation |
| ntopng LXC | 192.168.0.140 | - | Netzwerk-Monitoring |

## Architektur

- **Proxmox Cluster "homelab"** mit 2 Nodes: homeserver (.101) + homeserver2 (.102)
- **VPS** läuft Nginx als HTTP Reverse Proxy (Port 443)
- **Headscale** (selbst-gehostetes Tailscale) verbindet VPS mit Heimnetz
- **Tailscale LXC** (100.64.0.1) ist Subnet Router für 192.168.0.0/24
- **homeserver2** hostet FOG Master-Images (VMs 500-509) für OS-Deployment
- VPN-only Services (Vaultwarden, SearXNG): nur über Tailscale erreichbar

## Konventionen

- Sprache: Deutsch (mit englischen Fachbegriffen)
- Alle .md Dateien haben Wiki.js YAML Frontmatter (title, description, published, date, tags, editor, dateCreated)
- Tags: bilingual (deutsch + englisch), kommagetrennt
- Wiki.js Links müssen absolute Pfade mit `/en/` Locale-Prefix verwenden, ohne `.md` Extension (z.B. `(/en/konzepte/routing)`, NICHT `(routing)` oder `(../konzepte/routing.md)`). Bare names und relative Pfade werden von Wiki.js falsch aufgelöst.
- Diagramme als ASCII-Art in Code-Blöcken

## Wichtige Konfigurationsdateien (auf VPS)

- `/etc/nginx/sites-available/default` - HTTP Reverse Proxy
- `/etc/nginx/nginx.conf` - Stream Proxy (Gameserver-Ports)
- `/etc/headscale/config.yaml` - Headscale (in Docker)
- Tailscale braucht `--accept-routes` auf dem VPS für Subnet-Routing
