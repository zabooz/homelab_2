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
| **IP** | 192.168.0.137 |
| **OS** | Debian |
| **RAM** | 256 MB |
| **CPU** | 1 Core |
| **Disk** | 4 GB |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create <CTID> local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname adguard \
  --memory 256 \
  --cores 1 \
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