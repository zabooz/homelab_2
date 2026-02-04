---
title: Netzwerk-Übersicht
description: Master-Referenz für IP-Adressen, Dienste und Netzwerk-Topologie
published: true
date: 2026-02-01T00:00:00.000Z
tags: infrastructure, netzwerk, network, ip-adressen, dns, dhcp, übersicht, overview
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Netzwerk-Übersicht

Master-Referenz für die gesamte Infrastruktur.

**Siehe auch:**
- [IP-Adressen](/en/konzepte/ip-adressen) - IPv4, IPv6, Private/Public
- [Subnetting](/en/konzepte/subnetting) - CIDR, Subnetzmasken
- [Routing](/en/konzepte/routing) - Gateway, Routing-Tabellen

---

## Infrastruktur-Diagramm

```
                              INTERNET
                                  │
                                  │
              ┌───────────────────┴───────────────────┐
              │                                       │
              │         VPS (152.53.111.11)           │
              │         zabooz.duckdns.org            │
              │                                       │
              │    ┌─────────────────────────────┐    │
              │    │ Nginx HTTP (443)            │    │
              │    │  ├── / → Headscale (8090)   │    │
              │    │  ├── /vault/ → Vaultwarden  │    │
              │    │  ├── /searx/ → SearXNG      │    │
              │    │  └── /web → Headscale UI    │    │
              │    └─────────────────────────────┘    │
              │    ┌─────────────────────────────┐    │
              │    │ Nginx Stream (Layer 4)      │    │
              │    │  ├── :14004 → .120:14004    │    │
              │    │  │   (Veloren, TCP)         │    │
              │    │  └── :14005 → .120:14005    │    │
              │    │      (Xonotic, TCP+UDP)     │    │
              │    └──────────────┬──────────────┘    │
              │                   │ via Tailscale     │
              │    Tailscale IP: 100.64.0.5           │
              └───────────────────┬───────────────────┘
                                  │
                    Headscale Mesh VPN (WireGuard)
                         100.64.0.0/10
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       │                          │                          │
       │                          │                          │
┌──────▼──────┐          ┌────────▼────────┐         ┌───────▼───────┐
│   Laptop    │          │  Tailscale LXC  │         │    Handy      │
│ maschinchen │          │   100.64.0.1    │         │  zabooz-phone │
│ 100.64.0.2  │          │  192.168.0.112  │         │  100.64.0.4   │
└─────────────┘          │                 │         └───────────────┘
                         │  Exit-Node +    │
                         │  Subnet Router  │
                         └────────┬────────┘
                                  │
                       ┌──────────▼──────────┐
                       │  HEIMNETZWERK       │
                       │  192.168.0.0/24     │
                       │  Gateway: .1        │
                       │                     │
                       │  ┌───────────────┐  │
                       │  │ Proxmox .101  │  │
                       │  │   homeserver  │  │
                       │  │               │  │
                       │  │ ┌───────────┐ │  │
                       │  │ │Win VM .110│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Deb VM .111│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │TS LXC .112│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │FOG  .113  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │HA   .114  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Paper .115 │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │n8n  .116  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │LW   .119  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Ptero .120 │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │WoW  .121  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Draw .122  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Home .123  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Wiki .124  │ │  │
                       │  │ └───────────┘ │  │
                       │  └───────────────┘  │
                       └─────────────────────┘
```

---

## IP-Adressen Master-Liste

### VPS (Internet)

| Hostname | Public IP | Tailscale IP | Dienste |
|----------|-----------|--------------|---------|
| zaboozMegaFescherSuperServer | 152.53.111.11 | 100.64.0.5 | Headscale, Vaultwarden, SearXNG, Gameserver Proxy |

### Heimnetzwerk (192.168.0.0/24)

| Gerät | IP | MAC | Typ | Funktion |
|-------|-----|-----|-----|----------|
| **Router** | 192.168.0.1 | - | Gateway | Internet-Gateway (DHCP deaktiviert) |
| **Proxmox** | 192.168.0.101 | ec:b1:d7:72:c9:d1 | Host | Virtualisierungs-Host |
| **Windows VM** | 192.168.0.110 | bc:24:11:a9:bf:52 | VM | Windows Server 2025 |
| **Debian VM** | 192.168.0.111 | bc:24:11:1a:24:3d | VM | VPS Stats API |
| **Tailscale LXC** | 192.168.0.112 | bc:24:11:d8:a7:b2 | LXC | VPN Exit-Node/Subnet Router |
| **FOG Server** | 192.168.0.113 | - | LXC | DHCP Server, PXE/Imaging |
| **Home Assistant** | 192.168.0.114 | - | LXC/VM | Smart Home Steuerung |
| **Paperless-ngx** | 192.168.0.115 | - | LXC | Dokumentenverwaltung |
| **Linkwarden** | 192.168.0.119 | - | LXC | Bookmark Manager |
| **n8n** | 192.168.0.116 | - | LXC | Workflow Automation |
| **Pterodactyl** | 192.168.0.120 | - | LXC/VM | Gameserver Panel |
| **AzerothCore** | 192.168.0.121 | - | LXC | WoW Server |
| **Draw.io** | 192.168.0.122 | - | LXC | Diagramm-Editor |
| **Homepage** | 192.168.0.123 | - | LXC | Homepage Dashboard |
| **Wiki.js** | 192.168.0.124 | - | LXC | Wiki + Git Sync |

### Tailscale VPN (100.64.0.0/10)

| Hostname | Tailscale IP | User | Rolle |
|----------|--------------|------|-------|
| tailscale | 100.64.0.1 | zabooz | Subnet Router, Exit-Node |
| maschinchen | 100.64.0.2 | zabooz | Laptop |
| jules | 100.64.0.3 | jules | Client |
| zabooz-phone | 100.64.0.4 | zabooz | Handy |
| zaboozmegafeschersuperserver | 100.64.0.5 | zabooz | VPS |

---

## IP-Bereiche

```
192.168.0.1         - Gateway/Router (kein DHCP!)
192.168.0.100-119   - Statische IPs (Server/Infrastruktur)
192.168.0.120-199   - Reserviert für Erweiterungen
192.168.0.200-250   - FOG DHCP Pool (für alle Clients + PXE Boot)
```

## DHCP

| Parameter | Wert |
|-----------|------|
| **DHCP Server** | FOG (192.168.0.113) |
| **Router DHCP** | Deaktiviert |
| **DHCP Range** | 192.168.0.200 - 192.168.0.250 |
| **Gateway** | 192.168.0.1 |
| **DNS** | 192.168.0.1 (Router) |

> **Wichtig:** Router-DHCP ist deaktiviert. FOG übernimmt DHCP für das gesamte Netzwerk inkl. PXE Boot.

---

## Dienste-Übersicht

### VPS-Dienste

| Dienst | URL | Port (intern) | Zugriff |
|--------|-----|---------------|---------|
| Headscale | https://zabooz.duckdns.org/ | 8090 | Public |
| Headscale UI | https://zabooz.duckdns.org/web/ | 8080 | **VPN only** |
| SearXNG | https://zabooz.duckdns.org/searx/ | 8888 | **VPN only** |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | 8000 | **VPN only** |
| Veloren | 152.53.111.11:14004 | 14004 (→ 192.168.0.120) | Public (TCP) |
| Xonotic | 152.53.111.11:14005 | 14005 (→ 192.168.0.120) | Public (TCP+UDP) |

### Heimnetz-Dienste

| Dienst | URL | Host | Port |
|--------|-----|------|------|
| Proxmox | https://192.168.0.101:8006 | Proxmox | 8006 |
| Homepage | http://192.168.0.123 | Homepage LXC | 80 |
| Wiki.js | http://192.168.0.124 | Wiki.js LXC | 80 |
| Draw.io | http://192.168.0.122 | Draw.io LXC | 80 |
| FOG Project | http://192.168.0.113/fog/management | FOG LXC | 80 |
| Home Assistant | http://192.168.0.114:8123 | HA LXC/VM | 8123 |
| Paperless-ngx | http://192.168.0.115:8000 | Paperless LXC | 8000 |
| Linkwarden | http://192.168.0.119:3000 | Linkwarden LXC | 3000 |
| n8n | http://192.168.0.116:5678 | n8n LXC | 5678 |
| Pterodactyl | http://192.168.0.120 | Pterodactyl LXC/VM | 80 |
| Veloren | 152.53.111.11:14004 | Pterodactyl (via VPS Proxy) | 14004 |
| Xonotic | 152.53.111.11:14005 | Pterodactyl (via VPS Proxy) | 14005 |

### MagicDNS (über Tailscale)

| Name | Auflösung | Zweck |
|------|-----------|-------|
| home.lab | 192.168.0.123 | Homepage Dashboard |
| vps.lab | 100.64.0.5 | VPS via Tailscale |
| zabooz.duckdns.org | 100.64.0.5 | VPS (überschreibt Public DNS für VPN-Clients) |

> **Hinweis:** `zabooz.duckdns.org` wird für VPN-Clients auf die Tailscale IP überschrieben, damit VPN-only Services mit gültigem SSL-Zertifikat erreichbar sind.

---

## Netzwerk-Parameter

| Parameter | Wert |
|-----------|------|
| **Netzwerk** | 192.168.0.0/24 |
| **Subnetzmaske** | 255.255.255.0 |
| **Gateway** | 192.168.0.1 |
| **DNS (primär)** | 192.168.0.1 |
| **DNS (sekundär)** | 8.8.8.8 |
| **Tailscale-Netz** | 100.64.0.0/10 |

---

## Service-Abhängigkeiten

```
┌─────────────────────────────────────────────────┐
│                    Vaultwarden                   │
│            (braucht: Headscale, Nginx)           │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                    Headscale                     │
│         (braucht: Nginx, Let's Encrypt)          │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                 Tailscale LXC                    │
│          (braucht: Headscale, Proxmox)           │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                    Proxmox                       │
│              (braucht: Hardware)                 │
└─────────────────────────────────────────────────┘
```

**Startreihenfolge bei komplettem Neustart:**
1. Proxmox Host
2. Tailscale LXC (102)
3. VPS (Headscale) muss bereits laufen
4. Andere VMs/LXCs

---

## Quick Reference Card

### VPN verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns
```

### Wichtige IPs

```
VPS:         152.53.111.11 / 100.64.0.5
Proxmox:     192.168.0.101
Subnet LXC:  192.168.0.112 / 100.64.0.1
Homepage:    192.168.0.123
```

### Management-URLs

```
Proxmox:     https://192.168.0.101:8006
Vaultwarden: https://zabooz.duckdns.org/vault/
Homepage:    http://home.lab  (über VPN)
```

---

*Letzte Aktualisierung: 1. Februar 2026*
