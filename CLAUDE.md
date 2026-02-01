# Project Guide

Homelab Wiki - Dokumentation für die gesamte Homelab-Infrastruktur, Services und Netzwerk-Konzepte. Wird als Wiki.js Wiki betrieben und via Git synchronisiert.

## Struktur

```
homelab_2/
├── home.md                          # Wiki Startseite (Index aller Seiten)
├── infrastructure/                  # Server & Netzwerk Infrastruktur
│   ├── NETWORK_OVERVIEW.md          # Master-Referenz: IPs, Dienste, Topologie
│   ├── VPS.md                       # VPS: Nginx, Headscale, Stream Proxy
│   ├── LINUXVM.md                   # Debian VM (192.168.0.111)
│   └── proxmox-netzwerk-setup.md    # Proxmox Netzwerk-Konfiguration
├── services/                        # Service-Dokumentationen
│   ├── homepage-dashboard-setup.md  # Homepage Dashboard
│   ├── wikijs-setup.md              # Wiki.js + Git Sync
│   ├── stats-api.md                 # Bun Monitoring API
│   ├── paperless-ngx.md             # Dokumentenverwaltung
│   ├── fog_project_setup_dokumentation_debian_12_lxc.md  # FOG/PXE/DHCP
│   ├── drawio-setup.md              # Draw.io auf VPS
│   ├── linkwarden-setup.md          # Bookmark Manager
│   └── pterodactyl-setup.md         # Gameserver Panel + Wings
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
| Proxmox | 192.168.0.101 | - | Virtualisierung |
| Debian VM | 192.168.0.111 | - | Homepage, Wiki.js, Draw.io |
| Tailscale LXC | 192.168.0.112 | 100.64.0.1 | Subnet Router, Exit Node |
| Pterodactyl | 192.168.0.120 | - | Gameserver (Veloren, Xonotic) |

## Architektur

- **VPS** läuft Nginx als HTTP Reverse Proxy (Port 443) und Stream Proxy (Ports 14004, 14005)
- **Headscale** (selbst-gehostetes Tailscale) verbindet VPS mit Heimnetz
- **Tailscale LXC** (100.64.0.1) ist Subnet Router für 192.168.0.0/24
- Gameserver-Traffic: Internet → VPS:14004/14005 → Tailscale → 192.168.0.120
- VPN-only Services (Vaultwarden, SearXNG, Draw.io): nur über Tailscale erreichbar

## Konventionen

- Sprache: Deutsch (mit englischen Fachbegriffen)
- Alle .md Dateien haben Wiki.js YAML Frontmatter (title, description, published, date, tags, editor, dateCreated)
- Tags: bilingual (deutsch + englisch), kommagetrennt
- Dateien referenzieren sich gegenseitig mit relativen Pfaden
- Diagramme als ASCII-Art in Code-Blöcken

## Wichtige Konfigurationsdateien (auf VPS)

- `/etc/nginx/sites-available/default` - HTTP Reverse Proxy
- `/etc/nginx/nginx.conf` - Stream Proxy (Gameserver-Ports)
- `/etc/headscale/config.yaml` - Headscale (in Docker)
- Tailscale braucht `--accept-routes` auf dem VPS für Subnet-Routing
