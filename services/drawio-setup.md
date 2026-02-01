---
title: Draw.io Setup
description: Self-hosted Draw.io Diagramm-Editor auf dem VPS
published: true
date: 2026-01-25T23:30:00.000Z
tags:
editor: markdown
dateCreated: 2026-01-25T23:30:00.000Z
---

# Draw.io Setup

**Host:** VPS (zaboozMegaFescherSuperServer)
**URL:** https://zabooz.duckdns.org/draw/
**Zugriff:** Nur VPN (Tailscale)
**Port (intern):** 127.0.0.1:8081

**Siehe auch:**
- [VPS](../infrastructure/VPS.md) - VPS Konfiguration & Services
- [Reverse Proxy](../konzepte/reverse-proxy.md) - Forward vs. Reverse Proxy

---

## Übersicht

Draw.io (diagrams.net) ist ein selbst-gehosteter Diagramm-Editor für Flowcharts, Netzwerkdiagramme, UML und mehr. Diese Installation läuft als Docker-Container auf dem VPS und ist nur über VPN erreichbar.

## Architektur

```
┌──────────────┐         ┌─────────────────────────────────┐
│ VPN Client   │         │  VPS                            │
│ (Tailscale)  │─────────│                                 │
│ 100.64.0.x   │  HTTPS  │  Nginx (:443)                   │
└──────────────┘         │    │                            │
                         │    ├─ /draw/ ──► Draw.io (:8081)│
                         │    │            (VPN only)      │
                         │    └─ ...                       │
                         └─────────────────────────────────┘
```

## Installation

### 1. Projekt-Verzeichnis erstellen

```bash
mkdir -p ~/draw.io
cd ~/draw.io
```

### 2. Docker Compose Konfiguration

Erstelle `docker-compose.yml`:

```yaml
services:
  drawio:
    image: jgraph/drawio
    container_name: drawio
    restart: unless-stopped
    ports:
      - 127.0.0.1:8081:8080
      - 127.0.0.1:8443:8443
    environment:
      PUBLIC_DNS: domain
      ORGANISATION_UNIT: unit
      ORGANISATION: org
      CITY: city
      STATE: state
      COUNTRY_CODE: country
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080 || exit 1"]
      interval: 1m30s
      timeout: 10s
      retries: 5
      start_period: 10s
```

### 3. Container starten

```bash
docker compose up -d
```

### 4. Nginx Konfiguration

In `/etc/nginx/sites-available/default` hinzufügen:

```nginx
# Redirect /draw to /draw/
location = /draw {
    return 301 /draw/;
}

location /draw/ {
    allow 100.64.0.0/10;
    deny all;

    proxy_pass http://127.0.0.1:8081;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 128M;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
}
```

### 5. Nginx neu laden

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## Verwendung

1. VPN-Verbindung herstellen (Tailscale)
2. Browser öffnen: https://zabooz.duckdns.org/draw/
3. Diagramm erstellen und als `.drawio`, `.png`, `.svg` oder `.pdf` exportieren

### Speicheroptionen

Draw.io unterstützt verschiedene Speicherorte:
- **Device** - Lokaler Download
- **Browser** - Im Browser-Cache
- **Google Drive** - Mit Google-Konto
- **OneDrive** - Mit Microsoft-Konto
- **GitHub** - Direkt in Repository speichern

## Wartungs-Befehle

```bash
# Container Status prüfen
docker ps | grep drawio

# Logs anzeigen
docker logs drawio --tail 50

# Container neustarten
docker restart drawio

# Container aktualisieren
cd ~/draw.io
docker compose pull
docker compose up -d
```

## Fehlerbehebung

### Draw.io gibt 403 zurück

**Ursache:** Nicht im VPN verbunden.

**Lösung:**
1. Tailscale-Status prüfen: `tailscale status`
2. Falls nicht verbunden: `sudo systemctl restart tailscaled`

### Draw.io lädt nicht

**Prüfschritte:**
```bash
# Container läuft?
docker ps | grep drawio

# Container Logs prüfen
docker logs drawio

# Lokaler Test (auf VPS)
curl -I http://127.0.0.1:8081
```

### Nginx Fehler

```bash
# Konfiguration testen
sudo nginx -t

# Error Log prüfen
sudo tail -f /var/log/nginx/error.log
```

## Sicherheit

- **VPN-Only:** Nur über Tailscale-Netzwerk erreichbar (100.64.0.0/10)
- **Kein externer Zugriff:** Container bindet nur an 127.0.0.1
- **HTTPS:** Verschlüsselung via Let's Encrypt Zertifikat

---

*Letzte Aktualisierung: 25. Januar 2026*
