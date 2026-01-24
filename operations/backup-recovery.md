# Backup & Recovery

**Aktualisiert:** 24. Januar 2026
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
| Proxmox VMs/LXC | vzdump | Lokal + VPS | Täglich |
| Headscale Config | rsync | VPS → lokal | Täglich |
| Vaultwarden | Docker Volume | VPS + lokal | Täglich |
| Homepage Config | Git | Repository | Bei Änderung |

---

## Proxmox Backup (vzdump)

### Manuelles Backup

```bash
# Einzelne VM/LXC sichern
vzdump <vmid> --storage local --compress zstd --mode snapshot

# Beispiele
vzdump 100 --storage local --compress zstd --mode snapshot  # Windows VM
vzdump 101 --storage local --compress zstd --mode snapshot  # Debian VM
vzdump 102 --storage local --compress zstd --mode snapshot  # Tailscale LXC

# Mit Beschreibung
vzdump 100 --storage local --compress zstd --notes "Pre-Update Backup"
```

### Backup-Modi

| Modus | Beschreibung | Downtime |
|-------|--------------|----------|
| `snapshot` | Live-Backup (empfohlen) | Keine |
| `suspend` | RAM auf Disk, dann Backup | Kurz |
| `stop` | VM/LXC stoppen | Länger |

### Automatisches Backup (Cronjob)

```bash
# /etc/cron.d/proxmox-backup
# Täglich um 03:00 alle VMs/LXCs sichern
0 3 * * * root vzdump --all --storage local --compress zstd --mode snapshot --mailnotification failure
```

### Über Web-UI konfigurieren

1. Datacenter → Backup → Add
2. Storage: local
3. Schedule: Täglich 03:00
4. Selection: Alle oder spezifische VMs
5. Mode: Snapshot
6. Compression: ZSTD
7. Notification: Bei Fehler

### Backup-Verzeichnis

```bash
# Standard-Speicherort
ls /var/lib/vz/dump/

# Backup-Dateien
vzdump-lxc-102-2026_01_24-03_00_01.tar.zst
vzdump-lxc-102-2026_01_24-03_00_01.log
```

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

### Proxmox VM/LXC wiederherstellen

```bash
# Über CLI
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.tar.zst 100 --storage local-lvm
pct restore 102 /var/lib/vz/dump/vzdump-lxc-102-*.tar.zst --storage local-lvm

# Mit Überschreiben
qmrestore ... --force
pct restore ... --force
```

### Über Web-UI

1. Storage → Backups
2. Backup auswählen → Restore
3. Target Storage wählen
4. Optional: Unique MAC-Adressen

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

### Proxmox Backup verifizieren

```bash
# Backup-Integrität prüfen
vzdump --verify /var/lib/vz/dump/vzdump-*.tar.zst

# Backup-Inhalt auflisten
tar -tvf /var/lib/vz/dump/vzdump-lxc-102-*.tar.zst
```

### Regelmäßige Restore-Tests

**Monatlich empfohlen:**

1. Test-VM aus Backup erstellen (andere VMID)
2. Funktionalität prüfen
3. Test-VM löschen

```bash
# Test-Restore in andere VMID
pct restore 999 /var/lib/vz/dump/vzdump-lxc-102-*.tar.zst \
    --storage local-lvm

# Nach Test löschen
pct destroy 999
```

---

## Backup-Monitoring

### Backup-Status prüfen

```bash
# Letzte Backups anzeigen
ls -la /var/lib/vz/dump/

# Backup-Logs prüfen
cat /var/lib/vz/dump/vzdump-*.log | tail -50

# Speicherplatz prüfen
df -h /var/lib/vz/dump/
```

### Benachrichtigung bei Fehler

Proxmox kann E-Mails senden:

1. Datacenter → Backup → Options
2. Mail notification: failure
3. E-Mail-Adresse konfigurieren

---

## Speicherplatz-Management

### Alte Backups löschen

```bash
# Backups älter als 30 Tage löschen
find /var/lib/vz/dump/ -name "vzdump-*.tar.zst" -mtime +30 -delete

# Nur die letzten 5 Backups pro VM behalten
# (manuelle Aufräumaktion nötig)
```

### Backup-Rotation in Proxmox

```bash
# /etc/vzdump.conf
maxfiles: 5    # Maximale Backups pro VM behalten
```

---

## Wichtige Hinweise

1. **Backups testen!** - Ein Backup das nicht wiederhergestellt werden kann ist wertlos
2. **Off-Site Kopie** - Mindestens ein Backup außerhalb des Proxmox-Hosts
3. **Dokumentation** - Restore-Prozeduren dokumentieren
4. **Verschlüsselung** - Sensitive Backups verschlüsseln
5. **Monitoring** - Backup-Jobs überwachen

---

*Siehe auch: [disaster-recovery.md](disaster-recovery.md) | [security-hardening.md](security-hardening.md)*
