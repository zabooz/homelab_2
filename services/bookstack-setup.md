---
title: BookStack
description: Wiki und Dokumentationsplattform fuer Schule
published: true
date: 2026-03-07T00:00:00.000Z
tags: service, wiki, bookstack, dokumentation, docker, vps, schule
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# BookStack

BookStack ist eine Wiki- und Dokumentationsplattform, die auf dem VPS fuer schulbezogene Dokumentation genutzt wird.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | VPS (152.53.111.11) |
| **URL** | https://zabooz.duckdns.org/wiki/ |
| **Image** | lscr.io/linuxserver/bookstack:latest |
| **Datenbank** | MariaDB |
| **Zugriff** | Public (kein VPN noetig) |

---

## Docker Container

| Container | Image | Port (intern) |
|-----------|-------|---------------|
| bookstack | lscr.io/linuxserver/bookstack:latest | 9000 (-> 80) |
| bookstack_db | lscr.io/linuxserver/mariadb:latest | 3306 |

---

## Nginx Reverse Proxy

```nginx
location /wiki/ {
    proxy_pass http://127.0.0.1:9000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

---

## Wartung

```bash
# Container Status
docker ps | grep bookstack

# Logs
docker logs bookstack --tail 50

# Update
docker pull lscr.io/linuxserver/bookstack:latest
docker pull lscr.io/linuxserver/mariadb:latest
docker restart bookstack bookstack_db
```

---

*Siehe auch: [VPS Konfiguration](/en/infrastructure/VPS) · [Wiki.js](/en/services/wikijs-setup)*