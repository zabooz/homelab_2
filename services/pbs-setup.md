---
title: Proxmox Backup Server
description: Backup-Lösung für Proxmox VMs und LXC Container
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, backup, proxmox, pbs, infrastructure, sicherung
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# Proxmox Backup Server (PBS)

Proxmox Backup Server ist die dedizierte Backup-Lösung für den Proxmox Cluster. Sichert VMs und LXC Container von allen drei Cluster-Nodes mit Deduplizierung und Verschlüsselung.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 120 |
| **Node** | homeserver |
| **IP** | 192.168.0.180 |
| **OS** | Debian |
| **RAM** | 2 GB |
| **CPU** | 2 Cores |
| **Disk** | 8 GB (System) |
| **Version** | PBS 4.1.5 |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web-UI** | https://192.168.0.180:8007 |

---

## Datastore-Anbindung in Proxmox

PBS ist als Storage in den Proxmox Nodes eingebunden:

| Storage-Name | Node | Status |
|-------------|------|--------|
| pbs-hs1 | homeserver | deaktiviert |
| pbs-hs2 | homeserver2 | deaktiviert |
| pbs-hs3 | homeserver3 | aktiv (~1.8 TB verfügbar) |

### Storage in Proxmox hinzufügen

Auf jedem Proxmox Node unter **Datacenter → Storage → Add → Proxmox Backup Server**:

| Parameter | Wert |
|-----------|------|
| ID | pbs-hs1 (pro Node anpassen) |
| Server | 192.168.0.180 |
| Datastore | (Name des PBS Datastores) |
| Username | (PBS API User) |
| Fingerprint | (PBS SSL Fingerprint) |

---

## Wartung

### Web-UI

Die gesamte Verwaltung (Datastores, Backup-Jobs, Pruning, Garbage Collection) erfolgt über die Web-UI unter `https://192.168.0.180:8007`.

### Nützliche Befehle

```bash
# PBS Version prüfen
proxmox-backup-manager version

# Datastore Status
proxmox-backup-manager datastore list

# Garbage Collection starten
proxmox-backup-manager garbage-collection start <datastore>
```

---

## Troubleshooting

### PBS Web-UI nicht erreichbar

```bash
# Service läuft?
systemctl status proxmox-backup-proxy

# Port 8007 offen?
ss -tuln | grep 8007

# Service neustarten
systemctl restart proxmox-backup-proxy
systemctl restart proxmox-backup
```

### Backup schlägt fehl

- Netzwerk-Konnektivität zwischen Proxmox Node und PBS prüfen
- Datastore-Speicherplatz prüfen (Web-UI → Dashboard)
- Fingerprint korrekt in Proxmox Storage-Config?

---

*Siehe auch: [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [Backup & Recovery](/en/operations/backup-recovery)*
