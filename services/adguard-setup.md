---
title: AdGuard Home
description: Netzwerkweiter Werbeblocker und DNS-Server für das Homelab
published: true
date: 2026-03-07T00:00:00.000Z
tags: service, dns, adguard, docker, werbeblocker, adblock, netzwerk, network
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# AdGuard Home

AdGuard Home ist ein netzwerkweiter Werbeblocker und DNS-Server. Alle Geräte im Netzwerk werden über [FOG DHCP](/en/services/fog_project_setup_dokumentation_debian_12_lxc) auf AdGuard als DNS-Server verwiesen.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 117 |
| **Node** | homeserver |
| **IP** | 192.168.0.137 |
| **OS** | Debian |
| **RAM** | 512 MB |
| **CPU** | 2 Cores |
| **Disk** | 4 GB |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create <CTID> local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname adguard \
  --memory 512 \
  --cores 2 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.137/24,gw=192.168.0.1 \
  --features nesting=1

pct start <CTID>
pct enter <CTID>
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Compose Setup

```bash
mkdir -p /opt/adguard && cd /opt/adguard
```

**Datei:** `/opt/adguard/docker-compose.yml`

```yaml
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3000:3000/tcp"
      - "80:80/tcp"
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```

### Container starten

```bash
cd /opt/adguard
docker compose up -d
```

---

## Ersteinrichtung

1. Browser öffnen: `http://192.168.0.137:3000`
2. Setup-Wizard durchgehen
3. Admin-Account erstellen
4. Nach Abschluss ist die Web-UI unter `http://192.168.0.137` erreichbar (Port 80)

---

## DNS Integration mit FOG DHCP

AdGuard wird als DNS-Server über [FOG DHCP](/en/services/fog_project_setup_dokumentation_debian_12_lxc) an alle Clients verteilt.

**Auf FOG Server** (192.168.0.113) in `/etc/dhcp/dhcpd.conf`:

```conf
option domain-name-servers 192.168.0.137;
```

Danach DHCP neu starten:

```bash
systemctl restart isc-dhcp-server
```

---

## Netzwerk-Architektur

```
┌──────────────────────────────────────────────────┐
│              Heimnetzwerk 192.168.0.0/24         │
│                                                  │
│  ┌──────────────┐       ┌──────────────┐        │
│  │    Router     │       │   AdGuard    │        │
│  │ 192.168.0.1  │       │192.168.0.137 │        │
│  │  Gateway      │       │  DNS Server  │        │
│  └──────────────┘       └──────┬───────┘        │
│                                │ DNS             │
│  ┌──────────────┐              │                 │
│  │  FOG Server  │──────────────┘                 │
│  │192.168.0.113 │  DHCP verteilt                 │
│  │ DHCP + PXE   │  AdGuard als DNS               │
│  └──────────────┘                                │
│                                                  │
│  ┌──────────────┐                                │
│  │   Clients    │ bekommen DNS=192.168.0.137     │
│  │  via DHCP    │ per DHCP von FOG               │
│  └──────────────┘                                │
└──────────────────────────────────────────────────┘
```

---

## Custom DNS Rewrites (`.lab` Domain)

Alle Homelab-Services sind über `.lab`-Domains erreichbar. Die DNS Rewrites werden in der AdGuard Web-UI unter **Filter → DNS-Rewrites** konfiguriert.

| Domain | IP / Ziel | Service |
|--------|-----------|---------|
| wiki.lab | 192.168.0.124 | Wiki.js |
| fog.lab | 192.168.0.113 | FOG Server |
| n8n.lab | 192.168.0.116 | n8n Automation |
| homepage.lab | 192.168.0.123 | Homepage Dashboard |
| ntopng.lab | 192.168.0.140 | ntopng Monitoring |
| adguard.lab | 192.168.0.137 | AdGuard Home |
| immich.lab | 192.168.0.144 | Immich |
| dolibarr.lab | 192.168.0.149 | Dolibarr ERP |
| pdf.lab | 192.168.0.131 | Stirling PDF |
| paperless.lab | 192.168.0.115 | Paperless-ngx |
| links.lab | 192.168.0.119 | Linkwarden |
| drawio.lab | 192.168.0.122 | Draw.io |
| detect.lab | 192.168.0.125 | Changedetection |
| gotify.lab | 192.168.0.126 | Gotify |
| scanner.lab | 192.168.0.148 | ScanServJS |
| collabora.lab | 192.168.0.150 | Collabora Online |
| homeassistant.lab | 192.168.0.114 | Home Assistant |

Für jede Domain gibt es auch einen `www.`-CNAME-Eintrag (z.B. `www.wiki.lab → wiki.lab`).

| portainer.lab | 192.168.0.146 | Portainer |

> **Veralteter Eintrag:** `shared.lab` (.142) verweist auf einen nicht mehr existierenden Service (ehemals Samba) und kann entfernt werden.

> **Fehlend:** OpenCloud (.104), PBS (.180) und Grafana (.190) haben noch keine `.lab`-Domains. Diese können bei Bedarf hinzugefügt werden.

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep adguard

# Logs anschauen
docker logs adguard --tail 50
docker logs adguard -f

# Container neu starten
docker restart adguard

# Update auf neue Version
cd /opt/adguard
docker compose pull
docker compose up -d
```

### Backup

**Wichtige Daten:**

```
/opt/adguard/conf/    # AdGuard Konfiguration
/opt/adguard/work/    # Query Logs, Statistiken
```

---

## Troubleshooting

### AdGuard nicht erreichbar

```bash
# Container läuft?
docker ps | grep adguard

# Port 53 belegt?
ss -tuln | grep 53

# systemd-resolved blockiert Port 53?
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

### DNS funktioniert nicht

```bash
# DNS-Auflösung testen (vom Client)
nslookup google.com 192.168.0.137

# Vom AdGuard LXC selbst
dig @127.0.0.1 google.com
```

---

*Siehe auch: [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc) · [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW)*