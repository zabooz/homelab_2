---
title: Stats API
description: Bun API für System-Monitoring in Homepage
published: true
---

# Stats API für Homepage

Kleine Bun-APIs die System-Metriken (CPU, RAM, Disk) an das Homepage Dashboard senden.

## Server

| Server | IP | Port | Metriken |
|--------|-----|------|----------|
| Proxmox | 192.168.0.101 | 4000 | Load, RAM, Root-Disk, LVM Storage |
| VPS | 100.64.0.5 | 4000 | Load, RAM, Disk |

## Wie funktioniert es?

1. **Bun** läuft als HTTP-Server auf Port 4000
2. Liest Systemdaten aus `/proc/loadavg`, `/proc/meminfo`, `df`, `lvs`
3. Gibt JSON zurück mit allen Metriken
4. **Homepage** holt die Daten per `customapi` Widget

## Beispiel Response

```json
{
  "load1": 0.66,
  "ram": 44.7,
  "rootDisplay": "19GB / 94GB (22%)",
  "dataDisplay": "146GB / 1754GB (8%)"
}
```

## Installation

### 1. Bun installieren

```bash
apt install -y unzip
curl -fsSL https://bun.sh/install | bash
```

### 2. API Skript erstellen

```bash
mkdir -p /opt/stats-api
nano /opt/stats-api/index.ts
```

Das Skript liest Systemdaten und gibt sie als JSON zurück.

### 3. Systemd Service

```bash
nano /etc/systemd/system/stats-api.service
```

```ini
[Unit]
Description=Stats API for Homepage
After=network.target

[Service]
ExecStart=/root/.bun/bin/bun run /opt/stats-api/index.ts
Restart=always

[Install]
WantedBy=multi-user.target
```

### 4. Aktivieren

```bash
systemctl daemon-reload
systemctl enable stats-api
systemctl start stats-api
```

## Homepage Widget

```yaml
- Proxmox Storage:
    widget:
      type: customapi
      url: http://192.168.0.101:4000/stats
      mappings:
        - field: dataDisplay
          label: VM Storage
        - field: rootDisplay
          label: System
```

## Befehle

```bash
# Status
systemctl status stats-api

# Testen
curl http://localhost:4000/stats

# Logs
journalctl -u stats-api -f
```

## Dateien

| Server | Pfad | Ausführung |
|--------|------|------------|
| Proxmox | `/opt/stats-api/index.ts` | systemd service |
| VPS | `/home/zabooz/vps-stats-api/index.ts` | systemd service |
