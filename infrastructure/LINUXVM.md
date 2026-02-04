---
title: Linux VM Overview
description: Zentrale Debian VM und ihre Services
published: true
date: 2026-01-25T00:00:00.000Z
tags: infrastructure, linux, debian, vm, proxmox, server
editor: markdown
dateCreated: 2026-01-25T00:00:00.000Z
---

# Linux VM Overview

**System:** Debian 13 "Trixie"
**IP Address:** 192.168.0.111
**Role:** VPS Stats API Host
**Status:** Online

---

## System Architecture

Diese VM hostet die VPS Stats API. Die früheren Services (Homepage, Wiki.js, Draw.io) wurden in eigene LXC-Container migriert.

| Service | Neuer Host |
|---------|------------|
| Homepage Dashboard | 192.168.0.123 (LXC) |
| Wiki.js | 192.168.0.124 (LXC) |
| Draw.io | 192.168.0.122 (LXC) |

---

## Hosted Services

| Service | Port | URL | Type | Status |
|:---|:---|:---|:---|:---|
| **VPS Stats API** | :4000 | `localhost:4000/stats` | Native (Bun) | Active |
| **Apache** | - | - | Native (Systemd) | Disabled |

---

## Service Details

### VPS Stats API
A lightweight custom API built with **Bun** to provide system metrics to the Homepage dashboard.
- **Port:** 4000
- **Directory:** `~/vps-stats-api`
- **Technology:** TypeScript / Bun
- **Endpoint:** `GET /stats`
- **Documentation:** [Stats API](/en/services/stats-api)

### Apache (Disabled)
Apache ist installiert aber deaktiviert. Kann bei Bedarf als Reverse Proxy aktiviert werden.
- **Config Path:** `/etc/apache2/sites-available/`
- **Status:** Disabled (syntax error in config)

---

## Management & Maintenance

### System Updates
```bash
sudo apt update && sudo apt upgrade -y
```

### VPS Stats API Control
```bash
# Check if running
lsof -i :4000

# Start manually (if not in systemd)
cd ~/vps-stats-api
bun install
bun index.ts
```

---

## Directory Map

```text
/root/
├── vps-stats-api/         # Custom Stats API source code
└── ...
```

---

*Letzte Aktualisierung: Januar 2026*
