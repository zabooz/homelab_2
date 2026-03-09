---
title: Samba Netzwerk-Share
description: Samba/SMB Dateifreigabe ueber NVMe Storage im Homelab
published: true
date: 2026-03-09T00:00:00.000Z
tags: service, samba, smb, netzwerk, network, dateifreigabe, file-sharing, storage
editor: markdown
dateCreated: 2026-03-09T00:00:00.000Z
---

# Samba Netzwerk-Share

Samba stellt eine Netzwerk-Dateifreigabe bereit. Der Share liegt auf einem 400 GB NVMe-Volume aus dem `nvme-storage` Pool auf homeserver.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container CT 120 (Proxmox) |
| **Node** | homeserver |
| **IP** | 192.168.0.142 |
| **OS** | Debian |
| **RAM** | 512 MB |
| **CPU** | 2 Cores |
| **Root-Disk** | 4 GB |
| **Share-Disk** | 400 GB NVMe (nvme-storage Pool) |

---

## Zugriff

| Dienst | Adresse |
|--------|---------|
| **SMB Share** | `smb://192.168.0.142/share` |

---

## Mount-Punkt

Der NVMe-Storage ist innerhalb des Containers unter `/mnt/share` eingebunden (400 GB aus dem `nvme-storage` Pool auf homeserver).

```
/mnt/share    400 GB    nvme-storage Pool
```

---

## Wartung

```bash
# Samba Status pruefen
systemctl status smbd

# Samba neu starten
systemctl restart smbd

# Verbundene Clients anzeigen
smbstatus

# Share-Belegung pruefen
df -h /mnt/share
```

### Backup

**Wichtige Daten:**

```
/mnt/share/           # Freigegebene Dateien
/etc/samba/smb.conf   # Samba Konfiguration
```

---

*Siehe auch: [Netzwerk-Uebersicht](/en/infrastructure/NETWORK_OVERVIEW)*
