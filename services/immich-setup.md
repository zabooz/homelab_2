---
title: Immich Setup
description: Self-hosted Google Photos Alternative mit AI-gestützter Bilderkennung
published: true
date: 2026-03-06T00:00:00.000Z
tags: services, immich, photos, fotos, google-photos, ai, docker
editor: markdown
dateCreated: 2026-03-06T00:00:00.000Z
---

# Immich

**IP:** 192.168.0.144
**Node:** homeserver3 (CT 116)
**Port:** 2283

---

## Übersicht

Immich ist eine selbst-gehostete Google Photos Alternative mit automatischem Foto-Backup, Gesichtserkennung und AI-basierter Bildsuche. Alle AI-Features laufen lokal.

## Zugriff

| Dienst | URL |
|--------|-----|
| Web UI | http://192.168.0.144 |
| Mobile App | Android / iOS (Server-URL eintragen) |

## Installation

```bash
mkdir ~/immich-app && cd ~/immich-app
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### .env Konfiguration

```env
UPLOAD_LOCATION=./library
DB_DATA_LOCATION=./postgres
DB_PASSWORD=<sicheres-passwort>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
IMMICH_VERSION=v2
TZ=Europe/Vienna
```

### Starten

```bash
docker compose up -d
```

Beim ersten Aufruf von `http://192.168.0.144` wird der Admin-Account erstellt.

## Features

- Auto-Backup vom Handy (Android + iOS App)
- Gesichtserkennung (lokal, kein Cloud-Dienst)
- AI-Bildsuche via CLIP (z.B. "Hund", "Strand")
- Timeline, Karte, geteilte Alben
- Partner-Sharing

## Wartung

```bash
cd ~/immich-app

# Status
docker compose ps

# Logs
docker compose logs -f

# Update
docker compose pull && docker compose up -d
```

---

*Letzte Aktualisierung: 6. März 2026*
