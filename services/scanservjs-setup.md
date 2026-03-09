---
title: ScanServJS
description: Web-basiertes Scanner-Interface mit USB-Passthrough
published: true
date: 2026-03-09T00:00:00.000Z
tags: service, scanner, scanservjs, usb, docker, scan
editor: markdown
dateCreated: 2026-03-09T00:00:00.000Z
---

# ScanServJS

ScanServJS ist ein web-basiertes Scanner-Interface. Der Container laeuft als privilegierter Container mit USB-Passthrough fuer direkten Zugriff auf den Scanner.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container CT 119 (Proxmox) |
| **Node** | homeserver2 |
| **IP** | 192.168.0.148 |
| **OS** | Debian |
| **RAM** | 512 MB |
| **CPU** | 2 Cores |
| **Disk** | 8 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web UI** | http://192.168.0.148 |

---

## Besonderheiten

### Privilegierter Container

Der Container muss als **privilegierter Container** laufen (`unprivileged: 0`), damit USB-Geraete durchgereicht werden koennen.

### USB-Passthrough

In der Proxmox Container-Konfiguration (`/etc/pve/lxc/119.conf`) sind folgende Eintraege fuer den USB-Passthrough noetig:

```conf
lxc.cgroup2.devices.allow: c 189:* rwm
lxc.mount.entry: /dev/bus/usb dev/bus/usb none bind,optional,create=dir
```

Nach Aenderungen an der LXC-Konfiguration muss der Container neu gestartet werden:

```bash
pct stop 119
pct start 119
```

### Scanner pruefen

Innerhalb des Containers:

```bash
# USB-Geraete anzeigen
lsusb

# SANE Scanner erkennen
scanimage -L
```

---

## Wartung

```bash
# Service Status
systemctl status scanservjs

# Logs pruefen
journalctl -u scanservjs --tail 50

# Scanner neu erkennen nach USB-Wechsel
systemctl restart scanservjs
```

---

*Siehe auch: [Netzwerk-Uebersicht](/en/infrastructure/NETWORK_OVERVIEW)*
