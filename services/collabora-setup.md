---
title: Collabora Online
description: Selbst-gehosteter Office-Editor (CODE) für kollaboratives Arbeiten
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, office, collabora, docker, editor, dokumente, documents, code
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# Collabora Online

Collabora Online Development Edition (CODE) ist ein selbst-gehosteter Office-Editor. Ermöglicht das Bearbeiten von Dokumenten (Writer, Calc, Impress) direkt im Browser, z.B. eingebunden in OpenCloud oder andere Plattformen.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 122 |
| **Node** | homeserver3 |
| **IP** | 192.168.0.150 |
| **OS** | Debian |
| **RAM** | 4 GB |
| **CPU** | 4 Cores |
| **Disk** | 8 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web-UI / Admin** | http://192.168.0.150 |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create 122 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname collabora \
  --memory 4384 \
  --cores 4 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=192.168.0.150/24,gw=192.168.0.1 \
  --features nesting=1

pct start 122
pct enter 122
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Container starten

```bash
docker run -d \
  --name collabora \
  --restart unless-stopped \
  -p 80:9980 \
  -e "extra_params=--o:ssl.enable=false" \
  collabora/code
```

> **Hinweis:** SSL ist deaktiviert (intern im LAN). Bei Zugriff über VPN/Internet sollte ein Reverse Proxy mit SSL davor geschaltet werden.

---

## Integration

Collabora wird als externer Editor in Cloud-Plattformen eingebunden:

| Plattform | WOPI-URL |
|-----------|----------|
| OpenCloud | http://192.168.0.150 |

Die Plattform verbindet sich über das WOPI-Protokoll mit Collabora für Live-Editing.

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep collabora

# Logs anschauen
docker logs collabora --tail 50
docker logs collabora -f

# Container neu starten
docker restart collabora

# Update auf neue Version
docker pull collabora/code
docker stop collabora && docker rm collabora
# Dann docker run Befehl oben wiederholen
```

---

## Troubleshooting

### Collabora nicht erreichbar

```bash
# Container läuft?
docker ps | grep collabora

# Port 80 belegt?
ss -tuln | grep 80

# Interner Port prüfen
docker exec collabora ss -tlnp | grep 9980
```

### Dokument-Bearbeitung funktioniert nicht

- WOPI-Host muss in der Collabora-Config als erlaubt eingetragen sein
- Netzwerk-Konnektivität zwischen Collabora und der Cloud-Plattform prüfen

---

*Siehe auch: [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [Homepage Dashboard](/en/services/homepage-dashboard-setup)*
