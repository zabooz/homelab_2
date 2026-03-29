---
title: Refindr
description: Preisvergleich-App für Produkt-Tracking mit Bun und PostgreSQL
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, preisvergleich, price-comparison, bun, docker, postgresql, refindr
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# Refindr

Refindr ist eine Preisvergleich-App (Codename: preisvergleich) gebaut mit Bun und PostgreSQL. Trackt Produktpreise und vergleicht Angebote.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 126 |
| **Node** | homeserver2 |
| **IP** | 192.168.0.106 |
| **OS** | Debian |
| **RAM** | 4 GB |
| **CPU** | 4 Cores |
| **Disk** | 24 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web-UI** | http://192.168.0.106:3000 |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create 126 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname refindr \
  --memory 4196 \
  --cores 4 \
  --rootfs local-lvm:24 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.106/24,gw=192.168.0.1 \
  --features nesting=1

pct start 126
pct enter 126
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Compose Setup

```bash
mkdir -p /opt/refindr && cd /opt/refindr
```

**Datei:** `/opt/refindr/docker-compose.yml`

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_DB: preisvergleich
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:${POSTGRES_PASSWORD:-changeme}@db:5432/preisvergleich
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

**Datei:** `/opt/refindr/Dockerfile`

```dockerfile
FROM oven/bun:1
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --production
COPY . .
CMD ["bun", "run", "start"]
```

### Container starten

```bash
cd /opt/refindr
docker compose up -d --build
```

---

## Konfiguration

### Umgebungsvariablen

| Variable | Beschreibung | Default |
|----------|-------------|---------|
| `POSTGRES_PASSWORD` | PostgreSQL Passwort | `changeme` |
| `DATABASE_URL` | Datenbank-Verbindung | Auto-generiert |

**Passwort setzen:**

```bash
echo "POSTGRES_PASSWORD=mein_sicheres_passwort" > /opt/refindr/.env
```

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep refindr

# Logs anschauen
docker logs refindr-app-1 --tail 50
docker logs refindr-app-1 -f

# Container neu starten
cd /opt/refindr
docker compose restart

# Neu bauen nach Code-Änderungen
docker compose up -d --build
```

### Backup

**Wichtige Daten:**

```
/opt/refindr/                 # App-Code
pgdata Docker Volume          # PostgreSQL Datenbank
```

```bash
# Datenbank-Dump
docker exec refindr-db-1 pg_dump -U postgres preisvergleich > backup.sql
```

---

## Troubleshooting

### App startet nicht

```bash
# Container-Logs prüfen
docker logs refindr-app-1

# Datenbank erreichbar?
docker exec refindr-db-1 pg_isready -U postgres

# Port 3000 belegt?
ss -tuln | grep 3000
```

---

*Siehe auch: [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [Homepage Dashboard](/en/services/homepage-dashboard-setup)*
