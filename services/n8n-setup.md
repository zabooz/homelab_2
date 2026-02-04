---
title: n8n Automation Server
description: Workflow-Automation-Server für automatisierte IT-Prozesse im Homelab
published: true
date: 2026-02-03T00:00:00.000Z
tags: service, automation, docker, n8n, workflow, proxmox, fog, automatisierung
editor: markdown
dateCreated: 2026-02-03T00:00:00.000Z
---

# n8n Automation Server

n8n ist ein selbst-gehosteter Workflow-Automation-Server, der verschiedene IT-Prozesse im Homelab automatisiert. Hauptanwendung: Automatisches Update und Imaging von Master-VMs für [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc) Deployment.

---

## Installation

### System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container 106 (Proxmox) |
| **IP** | 192.168.0.116 |
| **OS** | Debian 13 |
| **Hostname** | n8n-automation |
| **RAM** | 2048 MB |
| **CPU** | 2 Cores |
| **Disk** | 20 GB |

### Docker Installation

```bash
# LXC Container erstellen (Proxmox)
pct create 106 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname n8n-automation \
  --memory 2048 \
  --cores 2 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.116/24,gw=192.168.0.1 \
  --features nesting=1

# Container starten
pct start 106

# In Container einloggen
pct enter 106

# System aktualisieren
apt update && apt upgrade -y

# Docker installieren
curl -fsSL https://get.docker.com | bash

# n8n Docker Compose Setup
mkdir -p /opt/n8n
cd /opt/n8n
```

### Docker Compose Konfiguration

**Datei:** `/opt/n8n/docker-compose.yml`

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=192.168.0.116
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://192.168.0.116:5678/
      - GENERIC_TIMEZONE=Europe/Vienna
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

### Container starten

```bash
cd /opt/n8n
docker compose up -d
```

---

## Zugriff

### Web-Interface

- **URL:** http://192.168.0.116:5678
- **Zugriff:** Lokal im Heimnetz (192.168.0.0/24)
- **VPN-Zugriff:** Über Tailscale VPN möglich

### Erster Login

1. Browser öffnen: `http://192.168.0.116:5678`
2. Account erstellen (beim ersten Start)
3. E-Mail und Passwort festlegen

---

## Konfiguration

### SSH Credentials (Proxmox)

**Name:** PROXMOX HOST

| Parameter | Wert |
|-----------|------|
| **Host** | 192.168.0.101 |
| **Port** | 22 |
| **Username** | root |
| **Auth Type** | SSH Key |
| **Private Key** | SSH Private Key für Proxmox |

**SSH Key erstellen:**

```bash
# Auf n8n Container
ssh-keygen -t ed25519 -f ~/.ssh/n8n_proxmox -N ''

# Public Key auf Proxmox kopieren
ssh-copy-id -i ~/.ssh/n8n_proxmox.pub root@192.168.0.101
```

### FOG API Credentials

**Name:** FOG SERVER

| Parameter | Wert |
|-----------|------|
| **URL** | http://192.168.0.113 |
| **fog-api-token** | (siehe FOG Configuration → API System) |
| **fog-user-token** | (siehe User Management → API Settings) |

---

## Workflows

### Aktive Workflows

| Workflow | Zweck | Status |
|----------|-------|--------|
| **VM Image Update** | Automatisches Update und Capture von Master-VMs | Aktiv |

Details siehe: [VM Image Update Workflow](/en/workflows/vm-image-update)

---

## Integration mit bestehendem System

### Proxmox Integration

n8n nutzt SSH-Zugriff auf [Proxmox Host](/en/infrastructure/proxmox-netzwerk-setup) (192.168.0.101) für:
- VM Start/Stop via Proxmox API
- Command Execution in VMs via `qm guest exec`
- VM Status Abfragen

**Verwendete Befehle:**

```bash
# VM starten
curl -k -X POST https://192.168.0.101:8006/api2/json/nodes/homeserver/qemu/{vmid}/status/start \
  -H "Authorization: PVEAPIToken=root@pam!n8n=..."

# VM herunterfahren
curl -k -X POST https://192.168.0.101:8006/api2/json/nodes/homeserver/qemu/{vmid}/status/shutdown \
  -H "Authorization: PVEAPIToken=root@pam!n8n=..."

# Befehl in VM ausführen
qm guest exec {vmid} -- {command}
```

### FOG Project Integration

n8n erstellt Capture Tasks via [FOG](/en/services/fog_project_setup_dokumentation_debian_12_lxc) API:

```bash
curl -X POST http://192.168.0.113/fog/host/{fogHostId}/task \
  -H "fog-api-token: ..." \
  -H "fog-user-token: ..." \
  -H "Content-Type: application/json" \
  -d '{"taskTypeID": 2, "shutdown": true}'
```

**Task Types:**
- `taskTypeID: 1` = Deploy
- `taskTypeID: 2` = Capture

---

## Netzwerk-Architektur

```
┌─────────────────────────────────────────────────────┐
│                  Heimnetzwerk                       │
│                  192.168.0.0/24                     │
│                                                     │
│  ┌──────────────┐         ┌──────────────┐         │
│  │   Proxmox    │         │     FOG      │         │
│  │ 192.168.0.101│◄────────│192.168.0.113 │         │
│  │              │         │              │         │
│  │  ┌────────┐  │         └──────────────┘         │
│  │  │ VM 500 │  │                                  │
│  │  │ VM 501 │  │         ┌──────────────┐         │
│  │  │ VM ... │  │         │     n8n      │         │
│  │  └────────┘  │◄────────│192.168.0.116 │         │
│  └──────────────┘         └──────────────┘         │
│                                                     │
└─────────────────────────────────────────────────────┘

n8n Workflow Flow:
1. Schedule Trigger (alle 5 Tage, Mitternacht)
2. VM Start (Proxmox API)
3. OS Update (qm guest exec)
4. VM Shutdown (Proxmox API)
5. FOG Capture Task (FOG API)
6. VM PXE Boot → FOG Capture → Shutdown
```

---

## Wartung

### n8n Container Management

```bash
# Status prüfen
docker ps | grep n8n

# Logs anschauen
docker logs n8n --tail 50
docker logs n8n -f  # Follow mode

# Container neu starten
docker restart n8n

# Container stoppen/starten
docker stop n8n
docker start n8n

# Update auf neue Version
cd /opt/n8n
docker compose pull
docker compose up -d
```

### Backup

**Wichtige Daten:**

```
/opt/n8n/                           # Docker Compose Config
/var/lib/docker/volumes/n8n_n8n_data/  # n8n Workflows & Settings
```

**Backup erstellen:**

```bash
# n8n Datenbank exportieren
docker exec n8n n8n export:workflow --all --output=/tmp/workflows.json
docker cp n8n:/tmp/workflows.json /backup/n8n-workflows-$(date +%Y%m%d).json

# Docker Volume Backup
docker run --rm \
  -v n8n_n8n_data:/data \
  -v /backup:/backup \
  alpine tar czf /backup/n8n-data-$(date +%Y%m%d).tar.gz /data
```

**Restore:**

```bash
# Workflows importieren
docker cp /backup/n8n-workflows-YYYYMMDD.json n8n:/tmp/workflows.json
docker exec n8n n8n import:workflow --input=/tmp/workflows.json

# Volume wiederherstellen
docker run --rm \
  -v n8n_n8n_data:/data \
  -v /backup:/backup \
  alpine tar xzf /backup/n8n-data-YYYYMMDD.tar.gz -C /
```

---

## Troubleshooting

### n8n nicht erreichbar

```bash
# Container Status
docker ps

# Wenn gestoppt
docker start n8n

# Port prüfen
ss -tuln | grep 5678

# Firewall (falls vorhanden)
iptables -L -n | grep 5678
```

### Workflows führen nicht aus

```bash
# Executions Log prüfen (Web-UI)
# n8n → Executions → Letzte Execution anklicken

# Container Logs
docker logs n8n --tail 100

# Workflow aktiv?
# Web-UI → Workflow → oben rechts "Active" prüfen
```

### SSH Verbindung zu Proxmox fehlschlägt

```bash
# In n8n Container
docker exec -it n8n sh

# SSH Test
ssh -i /root/.ssh/n8n_proxmox root@192.168.0.101

# Key Permissions prüfen
ls -la /root/.ssh/

# Sollte sein: 600 für private key
chmod 600 /root/.ssh/n8n_proxmox
```

### FOG API gibt Fehler

```bash
# FOG API testen
curl -X GET http://192.168.0.113/fog/host \
  -H "fog-api-token: YOUR_TOKEN" \
  -H "fog-user-token: YOUR_TOKEN"

# Tokens prüfen in FOG Web-UI:
# - Configuration → FOG Settings → API System
# - User Management → API Settings
```

---

## Sicherheit

1. **SSH Keys statt Passwörter** — n8n nutzt Key-basierte Auth für Proxmox
2. **FOG API Tokens rotieren** — Regelmäßig neue Tokens generieren
3. **n8n Zugriff beschränken** — Nur aus lokalem Netz erreichbar, kein Internet-Zugriff nötig
4. **Container Updates** — Regelmäßig `docker compose pull` ausführen
5. **Backups** — Wöchentlich Workflow-Backup, bei Änderungen sofort exportieren

---

## Monitoring

**Web-UI → Executions:**
- Zeigt alle Workflow-Läufe
- Erfolg/Fehler Status
- Execution Time
- Output Daten

---

*Siehe auch: [VM Image Update Workflow](/en/workflows/vm-image-update) · [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc) · [Proxmox Netzwerk](/en/infrastructure/proxmox-netzwerk-setup)*
