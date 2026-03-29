---
title: Grafana & InfluxDB
description: Monitoring-Stack mit Grafana Dashboards und InfluxDB Zeitreihen-Datenbank
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, monitoring, grafana, influxdb, docker, dashboards, metriken, metrics
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# Grafana & InfluxDB

Monitoring-Stack bestehend aus Grafana (Dashboards/Visualisierung) und InfluxDB (Zeitreihen-Datenbank). Sammelt und visualisiert Metriken aus dem gesamten Homelab.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 124 |
| **Node** | homeserver |
| **IP** | 192.168.0.190 |
| **OS** | Debian |
| **RAM** | 2 GB |
| **CPU** | 2 Cores |
| **Disk** | 20 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Grafana Web-UI** | http://192.168.0.190 |
| **InfluxDB API** | http://192.168.0.190:8086 |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create 124 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname graphana \
  --memory 2048 \
  --cores 2 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=192.168.0.190/24,gw=192.168.0.1 \
  --features nesting=1

pct start 124
pct enter 124
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Compose Setup

```bash
mkdir -p /opt/grafana && cd /opt/grafana
```

**Datei:** `/opt/grafana/docker-compose.yml`

```yaml
services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "80:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    restart: unless-stopped

volumes:
  grafana_data:
  influxdb_data:
```

### Container starten

```bash
cd /opt/grafana
docker compose up -d
```

---

## Ersteinrichtung

### InfluxDB

1. Browser öffnen: `http://192.168.0.190:8086`
2. Setup-Wizard: Organisation, Bucket und Admin-Token erstellen
3. API-Token kopieren für Grafana-Integration

### Grafana

1. Browser öffnen: `http://192.168.0.190`
2. Login mit `admin` / `admin` (Passwort sofort ändern)
3. Data Source hinzufügen: **InfluxDB**
   - URL: `http://influxdb:8086`
   - Auth: InfluxDB API-Token
   - Organization + Default Bucket eintragen

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep -E "grafana|influxdb"

# Logs anschauen
docker logs grafana --tail 50
docker logs influxdb --tail 50

# Container neu starten
cd /opt/grafana
docker compose restart

# Update
docker compose pull
docker compose up -d
```

### Backup

**Wichtige Daten:**

```
grafana_data Docker Volume     # Grafana Dashboards, Einstellungen
influxdb_data Docker Volume    # InfluxDB Zeitreihen-Daten
```

---

## Troubleshooting

### Grafana nicht erreichbar

```bash
# Container läuft?
docker ps | grep grafana

# Port 80 belegt?
ss -tuln | grep 80
```

### Keine Daten in Dashboards

```bash
# InfluxDB erreichbar?
curl http://localhost:8086/health

# Data Source in Grafana korrekt konfiguriert?
# Settings → Data Sources → InfluxDB testen
```

---

*Siehe auch: [ntopng](/en/services/ntopng-setup) · [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [Homepage Dashboard](/en/services/homepage-dashboard-setup)*
