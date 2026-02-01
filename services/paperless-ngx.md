---
title: Paperless-ngx
description: Dokumentenverwaltung mit OCR
published: true
date: 2026-01-25T00:00:00.000Z
tags: service, docker, ocr, dokumentenverwaltung, document-management, proxmox
editor: markdown
dateCreated: 2026-01-25T00:00:00.000Z
---

# Paperless-ngx

Dokumentenverwaltung mit automatischer OCR-Texterkennung.

## Übersicht

| | |
|---|---|
| **URL** | http://192.168.0.115:8000 |
| **Server** | LXC Container (Proxmox) |
| **CT ID** | 104 |
| **IP** | 192.168.0.115 |

## Was macht Paperless?

- Scannt/importiert Dokumente (PDF, Bilder)
- OCR - liest Text aus gescannten Bildern
- Automatische Kategorisierung (Rechnungen, Verträge, etc.)
- Volltextsuche über alle Dokumente
- Tags, Datum, Korrespondenten

## Installation

### 1. LXC Container erstellen

In Proxmox:
- **CT ID:** 104
- **Hostname:** paperless
- **Template:** Debian 13
- **Disk:** 50 GB
- **Cores:** 2-4
- **RAM:** 2048-4096 MB
- **Nesting:** Aktivieren (wichtig für Docker\!)

### 2. Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### 3. Paperless einrichten

```bash
mkdir -p /opt/paperless && cd /opt/paperless
nano docker-compose.yml
```

Docker Compose mit:
- Redis (Message Broker)
- PostgreSQL (Datenbank)
- Paperless-ngx (Webserver)

### 4. Starten

```bash
docker compose up -d
```

## Dokumente hochladen

1. **Web UI** - Drag & Drop im Browser
2. **Consume Ordner** - Dateien in `/opt/paperless/consume/` werden automatisch importiert
3. **Mobile App** - Paperless App für Android/iOS
4. **E-Mail** - Dokumente per Mail senden (braucht Konfiguration)

## Wichtige Pfade

| Pfad | Beschreibung |
|------|--------------|
| `/opt/paperless/` | Installation |
| `/opt/paperless/consume/` | Auto-Import Ordner |
| `/opt/paperless/export/` | Export Ordner |

## Befehle

```bash
# Status prüfen
docker ps

# Logs anzeigen
docker compose logs -f

# Neustart
docker compose restart

# Stoppen
docker compose down

# Update
docker compose pull && docker compose up -d
```

## Einstellungen

- **OCR Sprache:** Deutsch + Englisch
- **Zeitzone:** Europe/Berlin
- **Admin User:** admin

## Tipps

- Ändere das Admin-Passwort nach dem ersten Login
- Erstelle Tags für bessere Organisation (Rechnungen, Verträge, etc.)
- Nutze Korrespondenten für Absender (Telekom, Finanzamt, etc.)
- Regelmäßig Backup des Containers machen
