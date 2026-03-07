---
title: Netzwerk-Übersicht
description: Master-Referenz für IP-Adressen, Dienste und Netzwerk-Topologie
published: true
date: 2026-03-06T00:00:00.000Z
tags: infrastructure, netzwerk, network, ip-adressen, dns, dhcp, übersicht, overview, proxmox, cluster
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

## Proxmox Cluster "homelab"

| | homeserver | homeserver2 |
|---|---|---|
| **IP** | 192.168.0.101 | 192.168.0.102 |
| **CPU** | Intel i5-6500T @ 2.50GHz (4C) | Intel i3-8100 @ 3.60GHz (4C) |
| **RAM** | 32 GB | 32 GB |
| **Storage** | 2TB Samsung 850 PRO (SSD) | 256GB NVMe (System) + 500GB SanDisk SSD (~700GB gesamt) |
| **Boot** | EFI | Legacy BIOS |
| **Proxmox** | pve-manager 9.1.5 | pve-manager 9.1.5 |
| **Kernel** | Linux 6.17.9-1-pve | Linux 6.17.9-1-pve |

**Cluster-Gesamt:** 8 CPUs, 64 GB RAM, ~2.7 TiB Storage

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
              │    Tailscale IP: 100.64.0.5           │
              └───────────────────┬───────────────────┘
                                  │
                    Headscale Mesh VPN (WireGuard)
                         100.64.0.0/10
                                  │
       ┌──────────────────────────┼──────────────────────────┐
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
              ┌───────────────────▼───────────────────┐
              │         HEIMNETZWERK                   │
              │         192.168.0.0/24                  │
              │         Gateway: .1                     │
              │                                        │
              │  ┌──────────────────────────────────┐  │
              │  │ homeserver (.101)                 │  │
              │  │ i5-6500T, 32GB, 2TB SSD          │  │
              │  │                                  │  │
              │  │ LXC: tailscale .112              │  │
              │  │      fog-server .113             │  │
              │  │      paperless .115              │  │
              │  │      linkwarden .119             │  │
              │  │      draw.io .122                │  │
              │  │      homepage .123               │  │
              │  │      wikiJs .124                 │  │
              │  │      changedetection .125        │  │
              │  │      gotify .126                 │  │
              │  │      sterlingPdf .131            │  │
              │  │      adguard .137                │  │
              │  │      immich .144                 │  │
              │  │ VM:  webserver .128              │  │
              │  └──────────────────────────────────┘  │
              │                                        │
              │  ┌──────────────────────────────────┐  │
              │  │ homeserver2 (.102)                │  │
              │  │ i3-8100, 32GB, NVMe+SSD          │  │
              │  │                                  │  │
              │  │ LXC: n8n .116                    │  │
              │  │      ntopng .140                 │  │
              │  │ VM:  FOG Master-Images (500-509) │  │
              │  └──────────────────────────────────┘  │
              └────────────────────────────────────────┘
```

---

## IP-Adressen Master-Liste

### VPS (Internet)

| Hostname | Public IP | Tailscale IP | Dienste |
|----------|-----------|--------------|---------|
| zaboozMegaFescherSuperServer | 152.53.111.11 | 100.64.0.5 | Headscale, Vaultwarden, SearXNG, Matrix, BookStack |

### homeserver (192.168.0.101)

| VMID | Gerät | IP | Typ | Funktion |
|------|-------|-----|-----|----------|
| - | **homeserver** | 192.168.0.101 | Host | Proxmox Node 1 |
| 102 | **[Tailscale](/en/vpn/vpn-infrastructure)** | 192.168.0.112 | LXC | VPN Exit-Node/Subnet Router (100.64.0.1) |
| 103 | **[FOG Server](/en/services/fog_project_setup_dokumentation_debian_12_lxc)** | 192.168.0.113 | LXC | DHCP Server, PXE/Imaging |
| 104 | **[Paperless-ngx](/en/services/paperless-ngx)** | 192.168.0.115 | LXC | Dokumentenverwaltung |
| 105 | **[Linkwarden](/en/services/linkwarden-setup)** | 192.168.0.119 | LXC | Bookmark Manager |
| 107 | **[Draw.io](/en/services/drawio-setup)** | 192.168.0.122 | LXC | Diagramm-Editor |
| 108 | **[Homepage](/en/services/homepage-dashboard-setup)** | 192.168.0.123 | LXC | Homepage Dashboard |
| 109 | **[Wiki.js](/en/services/wikijs-setup)** | 192.168.0.124 | LXC | Wiki + Git Sync |
| 110 | **[Changedetection](/en/services/changedetection-setup)** | 192.168.0.125 | LXC | Website-Monitoring |
| 111 | **[Gotify](/en/services/gotify-setup)** | 192.168.0.126 | LXC | Push-Benachrichtigungen |
| 114 | **[Stirling PDF](/en/services/sterlingpdf-setup)** | 192.168.0.131 | LXC | PDF-Tools |
| 116 | **[Immich](/en/services/immich-setup)** | 192.168.0.144 | LXC | Google Photos Alternative |
| - | **[AdGuard Home](/en/services/adguard-setup)** | 192.168.0.137 | LXC | DNS-Werbeblocker |
| 302 | **Webserver** | 192.168.0.128 | VM | Nginx Webserver |
| 100 | **Debian** | - | VM | Gestoppt |
| 101 | **Win2025** | - | VM | Windows Server 2025, gestoppt |
| 300 | **[Pterodactyl](/en/services/pterodactyl-setup)** | - | VM | Gameserver Panel, gestoppt |
| 305 | **[Home Assistant](/en/services/homeassistant-setup)** | - | VM | Smart Home, gestoppt |
| 800 | **Win11-Client** | - | VM | Windows 11 Client, gestoppt |

### homeserver2 (192.168.0.102)

| VMID | Gerät | IP | Typ | Funktion |
|------|-------|-----|-----|----------|
| - | **homeserver2** | 192.168.0.102 | Host | Proxmox Node 2 |
| 106 | **[n8n](/en/services/n8n-setup)** | 192.168.0.116 | LXC | Workflow Automation |
| 115 | **[ntopng](/en/services/ntopng-setup)** | 192.168.0.140 | LXC | Netzwerk-Monitoring |
| 112 | **[Crawler4AI](/en/services/crawler4ai-setup)** | - | LXC | Web Scraping, gestoppt |
| 113 | **[Node-RED](/en/services/node-red-setup)** | - | LXC | Flow Automation, gestoppt |
| 301 | **[WoW Server](/en/services/azerothcore-playerbots-setup)** | - | LXC | AzerothCore, gestoppt |
| 500-509 | **Master-Images** | - | VM | FOG OS-Images (CachyOS, Ubuntu, Fedora, Mint, Debian, Win11, Parrot, NixOS) |

### Clients & Workstations

| Gerät | IP | Tailscale IP | OS | Funktion |
|-------|-----|-------------|-----|----------|
| **[Halo Strixx](/en/infrastructure/halo-strixx)** | 192.168.0.103 | 100.64.0.9 | CachyOS | AI Workstation (Strix Halo) |
| **maschinchen** | DHCP | 100.64.0.2 | - | Laptop |
| **zabooz-phone** | DHCP | 100.64.0.4 | - | Handy |

### Tailscale VPN (100.64.0.0/10)

| Hostname | Tailscale IP | User | Rolle |
|----------|--------------|------|-------|
| tailscale | 100.64.0.1 | zabooz | Subnet Router, Exit-Node |
| maschinchen | 100.64.0.2 | zabooz | Laptop |
| jules | 100.64.0.3 | jules | Client |
| zabooz-phone | 100.64.0.4 | zabooz | Handy |
| zaboozmegafeschersuperserver | 100.64.0.5 | zabooz | VPS |
| halo-strixx | 100.64.0.9 | zabooz | AI Workstation |

---

## IP-Bereiche

```
192.168.0.1         - Gateway/Router (kein DHCP!)
192.168.0.101-102   - Proxmox Cluster Nodes
192.168.0.110-140   - Statische IPs (VMs/LXCs)
192.168.0.200-250   - FOG DHCP Pool (für alle Clients + PXE Boot)
```

## DHCP

| Parameter | Wert |
|-----------|------|
| **DHCP Server** | FOG (192.168.0.113) |
| **Router DHCP** | Deaktiviert |
| **DHCP Range** | 192.168.0.200 - 192.168.0.250 |
| **Gateway** | 192.168.0.1 |
| **DNS** | 192.168.0.137 (AdGuard Home) |

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
| Matrix/Element | https://zabooz.duckdns.org/chat/ | 8010 | **VPN only** |
| BookStack | https://zabooz.duckdns.org/wiki/ | 9000 | Public |

### Heimnetz-Dienste

| Dienst | URL | Host | Node |
|--------|-----|------|------|
| Proxmox (hs1) | https://192.168.0.101:8006 | homeserver | homeserver |
| Proxmox (hs2) | https://192.168.0.102:8006 | homeserver2 | homeserver2 |
| Homepage | http://192.168.0.123 | Homepage LXC | homeserver |
| Wiki.js | http://192.168.0.124 | Wiki.js LXC | homeserver |
| Draw.io | http://192.168.0.122 | Draw.io LXC | homeserver |
| FOG Project | http://192.168.0.113/fog/management | FOG LXC | homeserver |
| Paperless-ngx | http://192.168.0.115:8000 | Paperless LXC | homeserver |
| Linkwarden | http://192.168.0.119:3000 | Linkwarden LXC | homeserver |
| Changedetection | http://192.168.0.125:5000 | Changedetection LXC | homeserver |
| Gotify | http://192.168.0.126:80 | Gotify LXC | homeserver |
| Stirling PDF | http://192.168.0.131:8080 | SterlingPdf LXC | homeserver |
| Webserver | http://192.168.0.128 | Webserver VM | homeserver |
| Immich | http://192.168.0.144:2283 | Immich LXC | homeserver |
| AdGuard Home | http://192.168.0.137 | AdGuard LXC | homeserver |
| n8n | http://192.168.0.116 | n8n LXC | homeserver2 |
| ntopng | http://192.168.0.140:3000 | ntopng LXC | homeserver2 |

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
| **DNS (primär)** | 192.168.0.137 (AdGuard Home) |
| **DNS (sekundär)** | 192.168.0.1 |
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
│              Proxmox Cluster                     │
│     homeserver (.101) + homeserver2 (.102)        │
│              (braucht: Hardware)                 │
└─────────────────────────────────────────────────┘
```

**Startreihenfolge bei komplettem Neustart:**
1. Proxmox Cluster (beide Nodes)
2. Tailscale LXC (102) auf homeserver
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
VPS:          152.53.111.11 / 100.64.0.5
homeserver:   192.168.0.101
homeserver2:  192.168.0.102
Subnet LXC:  192.168.0.112 / 100.64.0.1
Homepage:     192.168.0.123
```

### Management-URLs

```
Proxmox hs1: https://192.168.0.101:8006
Proxmox hs2: https://192.168.0.102:8006
Vaultwarden: https://zabooz.duckdns.org/vault/
Homepage:    http://home.lab  (über VPN)
```

---

*Letzte Aktualisierung: 7. März 2026*
