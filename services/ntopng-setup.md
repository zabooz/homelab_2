---
title: ntopng Setup
description: Netzwerk-Traffic-Monitoring und -Analyse
published: true
date: 2026-03-07T00:00:00.000Z
tags: services, monitoring, ntopng, netzwerk, network, traffic, docker
editor: markdown
dateCreated: 2026-03-06T00:00:00.000Z
---

# ntopng

Netzwerk-Traffic-Monitoring-Tool mit Echtzeit-Analyse des Netzwerkverkehrs.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container 115 (Proxmox) |
| **IP** | 192.168.0.140 |
| **Node** | homeserver2 |
| **OS** | Debian |
| **Web UI** | http://192.168.0.140 |
| **Edition** | Community |

---

## Docker Compose

**Datei:** `/opt/ntopng/compose.yml`

```yaml
services:
  redis:
    image: redis:alpine
    restart: always
    network_mode: host
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --save 60 1

  ntopng:
    image: ntop/ntopng:latest
    restart: always
    network_mode: host
    depends_on:
      - redis
    cap_add:
      - NET_ADMIN
      - NET_RAW
    command:
      - --redis
      - 127.0.0.1:6379
      - --interface
      - eth0
      - --local-networks
      - 192.168.0.0/24
      - --community
      - --dont-change-user
      - -w
      - "80"
    volumes:
      - ntopng_data:/var/lib/ntopng

volumes:
  redis_data:
  ntopng_data:
```

**Wichtig:**
- **Redis separat** mit `network_mode: host` und persistentem Volume — sonst gehen Einstellungen und Passwort bei Restart verloren
- **`--community`** Flag verhindert Pro-Trial-Timer der alle Einstellungen resettet
- **`--appendonly yes --save 60 1`** — Redis speichert alle 60 Sekunden auf Disk

---

## Konfiguration

### Ersteinrichtung

1. Browser: `http://192.168.0.140`
2. Login: `admin` / `admin`
3. Passwort ändern

### Empfohlene Einstellungen

| Einstellung | Wert | Pfad |
|-------------|------|------|
| **Host Identifier** | MAC Address | Settings → Preferences → Local Broadcast Domain Hosts Identifier |
| **DHCP Range** | 192.168.0.200-250 | Interfaces → eth0 → DHCP Range |
| **Erlaubter DHCP Server** | 192.168.0.113 (FOG) | Policies → Network Configuration |
| **Erlaubter DNS Server** | 192.168.0.137 (AdGuard) | Policies → Network Configuration |

### Host Identifier auf MAC Address

Da das Netzwerk DHCP nutzt (IPs können sich ändern), ist MAC-Adresse als Identifier besser. ntopng erkennt Geräte anhand ihrer MAC statt der IP.

### Erlaubte Server (Security Alerts)

Unter **Policies → Network Configuration** die erlaubten DHCP/DNS Server eintragen. ntopng alertet dann wenn ein unbekannter DHCP- oder DNS-Server im Netzwerk auftaucht (Rogue DHCP Detection).

---

## Monitoring-Einschränkungen

ntopng läuft in einer LXC mit virtueller Bridge und sieht daher:

- Traffic **von/zu** LXCs und VMs auf Proxmox
- Traffic der **durch den Router** geht (Internet, DNS)
- **Unidirektionaler Traffic** — ntopng sieht teilweise nur eine Richtung

**Nicht sichtbar:**
- Direkter WLAN-zu-WLAN Traffic der nicht über den Router läuft

**Tailscale-Hinweis:** Wenn Tailscale mit Exit Node aktiv ist, wird aller Traffic über den Subnet Router (192.168.0.112) geroutet. ntopng sieht dann nur .112 als Quelle statt den eigentlichen Client.

Für vollständiges Monitoring wäre Port Mirroring am Switch nötig.

---

## Wartung

```bash
# Status
cd /opt/ntopng
docker compose ps

# Logs
docker logs ntopng-ntopng-1 --tail 50

# Neustart
docker compose restart

# Update
docker compose pull
docker compose down && docker compose up -d
```

---

## Troubleshooting

### Passwort wird nicht gespeichert

Redis muss persistent laufen. Prüfen:

```bash
# Redis antwortet?
redis-cli ping

# Redis hat Daten auf Disk?
docker exec ntopng-redis-1 ls -la /data/
```

Wenn `dump.rdb` und `appendonlydir` vorhanden sind, ist Redis persistent.

### Pro-Trial resettet Einstellungen

`--community` Flag in der compose.yml setzen und Container neu starten.

---

*Siehe auch: [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [AdGuard Home](/en/services/adguard-setup) · [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc)*