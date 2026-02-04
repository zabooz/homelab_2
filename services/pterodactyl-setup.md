---
title: Pterodactyl Setup
description: Pterodactyl Panel & Wings Gameserver auf LXC/VM
published: true
date: 2026-02-01T15:00:00.000Z
tags: service, gameserver, spieleserver, pterodactyl, wings, docker, proxmox
editor: markdown
dateCreated: 2026-02-01T15:00:00.000Z
---

# Pterodactyl Setup

**Host:** gameserver (LXC/VM auf Proxmox)
**IP:** 192.168.0.120
**Panel URL:** http://192.168.0.120
**Wings API:** http://192.168.0.120:8080
**SFTP:** 192.168.0.120:2022
**Zugriff:** Lokal / VPN / Public (Gameserver-Ports via VPS)

**Siehe auch:**
- [VPS](/en/infrastructure/VPS) - Stream Proxy Konfiguration für öffentlichen Zugang
- [Reverse Proxy](/en/konzepte/reverse-proxy) - Layer 4 vs Layer 7 Proxy
- [Netzwerk Übersicht](/en/infrastructure/NETWORK_OVERVIEW) - IP-Adressen

---

## Übersicht

Pterodactyl ist ein Open-Source Gameserver-Management-Panel. Es besteht aus zwei Komponenten:

- **Panel** - Web-UI zur Verwaltung (PHP/Laravel)
- **Wings** - Daemon der die Gameserver als Docker-Container ausführt

Beide laufen auf demselben Host (192.168.0.120).

## Architektur

```
┌─────────────────────────────────────────────────┐
│  gameserver (192.168.0.120)                     │
│                                                 │
│  ┌──────────────────────┐  ┌──────────────────┐ │
│  │ Panel (Nginx/PHP)    │  │ Wings (:8080)    │ │
│  │ http://:80            │  │ /usr/local/bin/  │ │
│  │ /var/www/pterodactyl/ │  │                  │ │
│  └──────────────────────┘  └────────┬─────────┘ │
│                                     │           │
│                            Docker (pterodactyl_nw)
│                            172.18.0.0/16        │
│                                     │           │
│                   ┌─────────────────┼────────┐  │
│                   │                 │        │  │
│              ┌────▼─────┐    ┌──────▼─────┐  │  │
│              │ Veloren   │    │ Xonotic    │  │  │
│              │ :14004    │    │ :14005     │  │  │
│              └──────────┘    └────────────┘  │  │
│                                              │  │
└──────────────────────────────────────────────┘  │
                                                  │
```

## Zugang zu Gameservern

Spieler können sich auf zwei Wegen verbinden:

| Weg | Adresse | Wer |
|-----|---------|-----|
| **Lokal / VPN** | 192.168.0.120:14004 / :14005 | Im Heimnetz oder über Tailscale |
| **Internet** | 152.53.111.11:14004 / :14005 | Spieler von außen (kein VPN nötig) |

Der öffentliche Zugang läuft über einen Nginx Stream Proxy auf dem VPS, der den Traffic via Tailscale-Tunnel weiterleitet. Siehe [VPS Dokumentation](/en/infrastructure/VPS).

## Installation

Pterodactyl wurde **manuell** installiert (kein Docker für Panel/Wings).

### Panel

Installiert unter `/var/www/pterodactyl/` mit:
- PHP + Nginx als Webserver
- Laravel-basierte Anwendung
- Queue Worker als systemd Service (`pteroq.service`)

### Wings

Installiert unter `/usr/local/bin/wings` mit:
- Konfiguration: `/etc/pterodactyl/config.yml`
- systemd Service: `wings.service`
- Docker-Netzwerk: `pterodactyl_nw` (172.18.0.0/16)
- Daten: `/var/lib/pterodactyl/volumes`
- Logs: `/var/log/pterodactyl`
- Timezone: Europe/Vienna

## Laufende Gameserver

| Server | Port | Protokoll | Beschreibung |
|--------|------|-----------|--------------|
| Veloren | 14004 | TCP | Open-Source Multiplayer RPG |
| Xonotic | 14005 | TCP + UDP | Open-Source Arena Shooter (Quake-Engine) |

## Wichtige Dienste

```bash
# Panel Queue Worker
systemctl status pteroq

# Wings Daemon
systemctl status wings
```

## Konfigurations-Dateien

| Datei | Zweck |
|-------|-------|
| `/var/www/pterodactyl/` | Panel Installation |
| `/etc/pterodactyl/config.yml` | Wings Konfiguration |
| `/var/lib/pterodactyl/volumes` | Gameserver-Daten |
| `/var/lib/pterodactyl/backups` | Gameserver-Backups |
| `/var/log/pterodactyl` | Wings Logs |

## Wings Konfiguration (Auszug)

```yaml
# /etc/pterodactyl/config.yml
api:
  host: 0.0.0.0
  port: 8080

system:
  root_directory: /var/lib/pterodactyl
  data: /var/lib/pterodactyl/volumes
  username: pterodactyl
  timezone: Europe/Vienna
  sftp:
    bind_port: 2022
  crash_detection:
    enabled: true

docker:
  network:
    name: pterodactyl_nw
    driver: bridge
    interfaces:
      v4:
        subnet: 172.18.0.0/16
        gateway: 172.18.0.1

remote: http://192.168.0.120
```

## Wartungs-Befehle

```bash
# Wings Status
systemctl status wings

# Queue Worker Status
systemctl status pteroq

# Wings neustarten
systemctl restart wings

# Queue Worker neustarten
systemctl restart pteroq

# Wings Logs
journalctl -u wings --tail 50

# Laufende Gameserver-Container anzeigen
docker ps

# Gameserver Logs (Container-Name aus docker ps)
docker logs <container-name> --tail 50
```

## Fehlerbehebung

### Panel nicht erreichbar

```bash
# Nginx prüfen
systemctl status nginx
nginx -t

# PHP prüfen
systemctl status php*-fpm
```

### Gameserver startet nicht

```bash
# Wings Logs prüfen
journalctl -u wings -f

# Docker-Netzwerk prüfen
docker network ls | grep pterodactyl

# Container-Status
docker ps -a
```

### Spieler können von außen nicht verbinden

1. VPS Stream Proxy prüfen: `ss -tlnp | grep -E '14004|14005'` (auf VPS)
2. Tailscale Subnet Router online? `tailscale status` (auf VPS)
3. Direkter Test vom VPS: `nc -zv 192.168.0.120 14004`

## Sicherheit

- **Panel**: Nur im Heimnetz / über VPN erreichbar (kein öffentlicher Zugang)
- **Wings API**: Nur lokal (kein SSL, nicht öffentlich exponiert)
- **Gameserver-Ports**: Öffentlich erreichbar via VPS Stream Proxy
- **SFTP**: Port 2022, nur im Heimnetz

---

*Letzte Aktualisierung: 1. Februar 2026*
