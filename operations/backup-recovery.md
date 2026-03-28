---
title: Backup & Recovery
description: Backup Prozeduren und Wiederherstellung
published: true
date: 2026-01-24T00:00:00.000Z
tags: operations, backup, recovery, wiederherstellung, sicherung, betrieb
editor: markdown
dateCreated: 2026-01-24T00:00:00.000Z
---

# Backup & Recovery

**Aktualisiert:** 28. März 2026
**Zweck:** Backup-Strategien und Wiederherstellungsprozeduren

---

## Backup-Strategie Übersicht

### 3-2-1 Regel

- **3** Kopien der Daten
- **2** verschiedene Medien
- **1** Kopie off-site

### Infrastruktur-Komponenten

| Komponente | Backup-Methode | Ziel | Frequenz |
|------------|----------------|------|----------|
| Proxmox VMs/LXC | PBS (Proxmox Backup Server) | pbs-backup (192.168.0.180) | Täglich |
| Headscale Config | rsync | VPS → lokal | Täglich |
| Vaultwarden | Docker Volume | VPS + lokal | Täglich |
| Homepage Config | Git | Repository | Bei Änderung |

---

## Proxmox Backup Server (PBS)

Der Proxmox Backup Server läuft als LXC (VMID 120) auf homeserver (192.168.0.101) und ist als shared Storage `pbs-backup` in allen drei Cluster-Nodes eingebunden.

- **IP:** 192.168.0.180
- **Web-UI:** https://192.168.0.180:8007
- **Storage:** ~1.8 TB verfügbar
- **Fingerprint:** `9c:90:5f:99:8c:9d:40:6f:d0:ec:68:dc:81:7f:ff:e0:c5:83:3d:1b:6c:3d:18:78:30:3e:42:72:76:b6:c8:a0`

### Vorteile gegenüber vzdump

- **Inkrementelle Backups** — nur geänderte Blöcke werden übertragen
- **Deduplizierung** — spart erheblich Speicherplatz
- **Integrität** — automatische Checksummen-Verifizierung
- **Zentrales Management** — ein Dashboard für alle drei Nodes
- **Pruning** — automatische Aufbewahrungsregeln

### Backup über Proxmox Web-UI

1. Datacenter → Backup → Add
2. Storage: **pbs-backup**
3. Schedule: Täglich 03:00
4. Selection: Alle oder spezifische VMs
5. Mode: Snapshot
6. Notification: Bei Fehler

### Manuelles Backup (vzdump an PBS)

```bash
# Einzelne VM/LXC an PBS sichern
vzdump <vmid> --storage pbs-backup --mode snapshot

# Beispiel
vzdump 102 --storage pbs-backup --mode snapshot  # Tailscale LXC
```

### Backup-Modi

| Modus | Beschreibung | Downtime |
|-------|--------------|----------|
| `snapshot` | Live-Backup (empfohlen) | Keine |
| `suspend` | RAM auf Disk, dann Backup | Kurz |
| `stop` | VM/LXC stoppen | Länger |

---

## VPS Backup (Off-Site)

### Kritische Dateien auf VPS

```
/etc/headscale/config.yaml      # Headscale Konfiguration
/var/lib/headscale/db.sqlite    # Headscale Datenbank
/home/zabooz/vaultwarden/       # Vaultwarden Daten
/etc/nginx/sites-available/     # Nginx Konfiguration
/etc/letsencrypt/               # SSL-Zertifikate
```

### Backup-Script für VPS

```bash
#!/bin/bash
# /root/backup.sh

BACKUP_DIR="/root/backups"
DATE=$(date +%Y%m%d)
REMOTE_USER="zabooz"
REMOTE_HOST="192.168.0.111"  # Über VPN erreichbar

mkdir -p $BACKUP_DIR

# Headscale Backup
tar -czf $BACKUP_DIR/headscale-$DATE.tar.gz \
    /etc/headscale/ \
    /var/lib/headscale/

# Vaultwarden Backup
docker stop vaultwarden
tar -czf $BACKUP_DIR/vaultwarden-$DATE.tar.gz \
    /home/zabooz/vaultwarden/
docker start vaultwarden

# Nginx Backup
tar -czf $BACKUP_DIR/nginx-$DATE.tar.gz \
    /etc/nginx/sites-available/

# Zertifikate Backup
tar -czf $BACKUP_DIR/letsencrypt-$DATE.tar.gz \
    /etc/letsencrypt/

# Auf Heimserver kopieren (über VPN)
rsync -avz $BACKUP_DIR/ $REMOTE_USER@$REMOTE_HOST:/backup/vps/

# Alte Backups löschen (älter als 7 Tage)
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

### Cronjob für VPS-Backup

```bash
# crontab -e
0 4 * * * /root/backup.sh >> /var/log/backup.log 2>&1
```

---

## Vaultwarden Backup

### Kritische Daten

```
/home/zabooz/vaultwarden/
├── vw-data/
│   ├── db.sqlite3        # Haupt-Datenbank (KRITISCH!)
│   ├── db.sqlite3-wal    # Write-Ahead Log
│   ├── rsa_key.pem       # RSA Private Key
│   ├── rsa_key.pub.pem   # RSA Public Key
│   └── attachments/      # Dateianhänge
└── docker-compose.yml    # Container-Konfiguration
```

### Manuelles Backup

```bash
# Container stoppen für konsistentes Backup
docker stop vaultwarden

# Backup erstellen
tar -czf vaultwarden-backup-$(date +%Y%m%d).tar.gz \
    /home/zabooz/vaultwarden/

# Container wieder starten
docker start vaultwarden
```

### Automatisches Backup-Script

```bash
#!/bin/bash
# /home/zabooz/backup-vaultwarden.sh

BACKUP_DIR="/home/zabooz/backups"
DATE=$(date +%Y%m%d_%H%M)

mkdir -p $BACKUP_DIR

# Container kurz stoppen
docker stop vaultwarden

# Datenbank kopieren
cp /home/zabooz/vaultwarden/vw-data/db.sqlite3 \
   $BACKUP_DIR/vaultwarden-$DATE.sqlite3

# Container wieder starten
docker start vaultwarden

# Alte Backups löschen (behalte letzte 30)
ls -t $BACKUP_DIR/vaultwarden-*.sqlite3 | tail -n +31 | xargs -r rm

echo "Vaultwarden backup completed: $DATE"
```

---

## Restore-Prozeduren

### Proxmox VM/LXC wiederherstellen (PBS)

**Über Web-UI (empfohlen):**

1. Storage → pbs-backup → Content
2. Backup auswählen → Restore
3. Target Storage wählen (local-lvm oder ssd-storage)
4. Optional: Unique MAC-Adressen

**Über CLI:**

```bash
# LXC aus PBS wiederherstellen
pct restore 102 pbs-backup:backup/ct/102/2026-03-28T03:00:00Z --storage local-lvm

# VM aus PBS wiederherstellen
qmrestore pbs-backup:backup/vm/305/2026-03-28T03:00:00Z 305 --storage ssd-storage

# Mit Überschreiben
pct restore ... --force
qmrestore ... --force
```

### Headscale wiederherstellen (VPS)

```bash
# Backup entpacken
tar -xzf headscale-YYYYMMDD.tar.gz -C /

# Service neustarten
sudo systemctl restart headscale

# Nodes überprüfen
sudo headscale nodes list
```

### Vaultwarden wiederherstellen

```bash
# Container stoppen
docker stop vaultwarden

# Alte Daten sichern
mv /home/zabooz/vaultwarden/vw-data /home/zabooz/vaultwarden/vw-data.old

# Backup einspielen
tar -xzf vaultwarden-backup-YYYYMMDD.tar.gz -C /

# Container starten
docker start vaultwarden

# Testen
curl https://zabooz.duckdns.org/vault/
```

---

## Backup-Integrität testen

### PBS Verify

PBS verifiziert Backup-Integrität automatisch via Checksummen. Manuell:

1. PBS Web-UI (https://192.168.0.180:8007) → Datastore → Verify
2. Oder im Proxmox: Storage → pbs-backup → Content → Verify

### Regelmäßige Restore-Tests

**Monatlich empfohlen:**

1. Test-VM aus Backup erstellen (andere VMID)
2. Funktionalität prüfen
3. Test-VM löschen

```bash
# Test-Restore in andere VMID
pct restore 999 pbs-backup:backup/ct/102/2026-03-28T03:00:00Z \
    --storage local-lvm

# Nach Test löschen
pct destroy 999
```

---

## Backup-Monitoring

### PBS Dashboard

Die PBS Web-UI (https://192.168.0.180:8007) zeigt:
- Letzte Backups und deren Status
- Speicherauslastung und Deduplizierungsrate
- Verify-Ergebnisse

### Im Proxmox Cluster

- Datacenter → Backup → letzte Backup-Jobs prüfen
- Storage → pbs-backup → Content → alle Backups aller Nodes

### Benachrichtigung bei Fehler

Proxmox kann E-Mails senden:

1. Datacenter → Backup → Options
2. Mail notification: failure
3. E-Mail-Adresse konfigurieren

---

## Speicherplatz-Management

### PBS Pruning (Aufbewahrungsregeln)

PBS unterstützt automatisches Pruning mit Aufbewahrungsregeln:

1. PBS Web-UI → Datastore → Prune & GC
2. Regeln konfigurieren (z.B. keep-last: 7, keep-weekly: 4, keep-monthly: 3)

### Garbage Collection

Nach dem Pruning müssen nicht mehr referenzierte Chunks aufgeräumt werden:

1. PBS Web-UI → Datastore → Prune & GC → Start Garbage Collection
2. Oder per Cronjob in PBS konfigurieren

---

## Wichtige Hinweise

1. **Backups testen!** - Ein Backup das nicht wiederhergestellt werden kann ist wertlos
2. **Off-Site Kopie** - Mindestens ein Backup außerhalb des Proxmox-Hosts
3. **Dokumentation** - Restore-Prozeduren dokumentieren
4. **Verschlüsselung** - Sensitive Backups verschlüsseln
5. **Monitoring** - Backup-Jobs überwachen

---

*Siehe auch: [Disaster Recovery](/en/operations/disaster-recovery) | [Security Hardening](/en/operations/security-hardening)*
