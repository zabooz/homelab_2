---
title: pgAdmin
description: Web-basiertes PostgreSQL Management-Tool
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, postgresql, pgadmin, docker, datenbank, database, management
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# pgAdmin

pgAdmin ist ein web-basiertes Management-Tool für PostgreSQL-Datenbanken. Ermöglicht die Verwaltung aller PostgreSQL-Instanzen im Homelab (Forgejo, Refindr, etc.) über eine zentrale Oberfläche.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 125 |
| **Node** | homeserver3 |
| **IP** | 192.168.0.105 |
| **OS** | Debian |
| **RAM** | 4 GB |
| **CPU** | 4 Cores |
| **Disk** | 8 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web-UI** | http://192.168.0.105:5050 |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create 125 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname pgadmin \
  --memory 4196 \
  --cores 4 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.105/24,gw=192.168.0.1 \
  --features nesting=1

pct start 125
pct enter 125
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Compose Setup

```bash
mkdir -p /opt/pgadmin && cd /opt/pgadmin
```

**Datei:** `/opt/pgadmin/docker-compose.yml`

```yaml
services:
  pgadmin:
    image: dpage/pgadmin4
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    restart: unless-stopped

volumes:
  pgadmin_data:
```

### Container starten

```bash
cd /opt/pgadmin
docker compose up -d
```

---

## Ersteinrichtung

1. Browser öffnen: `http://192.168.0.105:5050`
2. Login mit `admin@admin.com` / `admin`
3. **Passwort sofort ändern** unter User Management

### Server hinzufügen

Unter "Add New Server" die PostgreSQL-Instanzen verbinden:

| Name | Host | Port | DB | User |
|------|------|------|-----|------|
| Forgejo | 192.168.0.107 | 5432 | forgejo | forgejo |
| Refindr | 192.168.0.106 | 5433 | preisvergleich | postgres |
| Dolibarr | 192.168.0.149 | 5432 | dolibarr | - |

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep pgadmin

# Logs anschauen
docker logs pgadmin-pgadmin-1 --tail 50

# Container neu starten
cd /opt/pgadmin
docker compose restart

# Update
docker compose pull
docker compose up -d
```

### Backup

```
pgadmin_data Docker Volume    # pgAdmin Konfiguration, gespeicherte Server
```

---

*Siehe auch: [Forgejo](/en/services/forgejo-setup) · [Refindr](/en/services/refindr-setup) · [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW)*
