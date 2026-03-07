---
title: Matrix Synapse
description: Selbst-gehosteter Matrix Chat-Server mit Element Web Client
published: true
date: 2026-03-07T00:00:00.000Z
tags: service, matrix, synapse, element, chat, messenger, docker, vps
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# Matrix Synapse

Matrix ist ein dezentrales, verschluesseltes Chat-Protokoll. Synapse ist der Homeserver, Element der Web-Client. Laeuft auf dem VPS und ist nur ueber Tailscale VPN erreichbar.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | VPS (152.53.111.11) |
| **Server** | Synapse (Matrix Homeserver) |
| **Client** | Element Web |
| **Datenbank** | PostgreSQL 15 |
| **Server Name** | zabooz.duckdns.org |
| **Zugriff** | VPN only (100.64.0.0/10) |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Element Web** | https://zabooz.duckdns.org/chat/ |
| **Synapse API** | https://zabooz.duckdns.org/_matrix/ |

---

## Docker Container

| Container | Image | Port (intern) |
|-----------|-------|---------------|
| synapse | matrixdotorg/synapse:latest | 8008 |
| element | vectorim/element-web:latest | 8010 (-> 80) |
| synapse_db | postgres:15-alpine | 5432 |

---

## Konfiguration

### Synapse

**Datei:** `/data/homeserver.yaml` (im Container)

Wichtige Einstellungen:

```yaml
server_name: "zabooz.duckdns.org"
public_baseurl: "https://zabooz.duckdns.org"
enable_registration: false
```

### Element

**Datei:** `/app/config.json` (im Container)

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://zabooz.duckdns.org",
            "server_name": "zabooz.duckdns.org"
        }
    },
    "default_theme": "dark"
}
```

### Nginx Reverse Proxy

```nginx
# Matrix Synapse API - VPN only
location /_matrix {
    allow 100.64.0.0/10;
    deny all;
    proxy_pass http://127.0.0.1:8008;
    client_max_body_size 50M;
}

# Element Web Client - VPN only
location /chat/ {
    allow 100.64.0.0/10;
    deny all;
    proxy_pass http://127.0.0.1:8010/;
}
```

---

## User-Verwaltung

### Neuen User erstellen

```bash
docker exec -it synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008
```

Mit `-a` Flag fuer Admin-Account.

### Passwort resetten

```bash
# Hash generieren
docker exec synapse python -m synapse._scripts.hash_password \
  -c /data/homeserver.yaml -p 'NEUES_PASSWORT'

# In DB updaten
docker exec synapse_db psql -U synapse -c \
  "UPDATE users SET password_hash='HASH' WHERE name='@user:zabooz.duckdns.org';"
```

### User auflisten

```bash
docker exec synapse_db psql -U synapse -c "SELECT name, admin FROM users;"
```

---

## Client-Apps

Element ist auf allen Plattformen verfuegbar:
- **Web:** https://zabooz.duckdns.org/chat/
- **Android:** Element App (Play Store / F-Droid)
- **iOS:** Element App (App Store)
- **Desktop:** Element Desktop

Bei der Anmeldung als Homeserver `https://zabooz.duckdns.org` eintragen.

---

## Wartung

```bash
# Container Status
docker ps | grep -E "synapse|element"

# Logs
docker logs synapse --tail 50
docker logs element --tail 50

# Neustart
docker restart synapse element synapse_db

# Update
docker pull matrixdotorg/synapse:latest
docker pull vectorim/element-web:latest
docker restart synapse element
```

---

## Troubleshooting

### Login geht nicht (403)

Nginx blockt den Request. Pruefen ob Tailscale VPN aktiv ist:

```bash
tailscale status
```

Der Client muss eine IP aus `100.64.0.0/10` haben.

### M_LIMIT_EXCEEDED

Zu viele Login-Versuche. Synapse neu starten:

```bash
docker restart synapse
```

### Federation

Federation ist aktuell nicht konfiguriert (Port 8448 nicht in Nginx). Der Server funktioniert als privater Chat-Server ohne Federation.

---

*Siehe auch: [VPS Konfiguration](/en/infrastructure/VPS) · [VPN Infrastruktur](/en/vpn/vpn-infrastructure)*