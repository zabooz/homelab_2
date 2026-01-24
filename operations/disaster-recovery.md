# Disaster Recovery

**Aktualisiert:** 24. Januar 2026
**Zweck:** Wiederherstellung nach Systemausfällen

---

## Kritische Dienste - Prioritäten

| Priorität | Dienst | Auswirkung bei Ausfall |
|-----------|--------|------------------------|
| 1 | VPS (Headscale) | Kein VPN-Zugriff auf Heimnetz |
| 2 | Tailscale LXC | Kein Routing zum Heimnetz |
| 3 | Proxmox Host | Alle VMs/LXCs offline |
| 4 | Vaultwarden | Kein Passwort-Zugriff |
| 5 | Homepage | Dashboard nicht verfügbar |

---

## Szenario: VPS geht offline

### Auswirkungen

- Headscale Control Server nicht erreichbar
- Neue VPN-Verbindungen nicht möglich
- **Bestehende** Verbindungen bleiben aktiv (Mesh-Netzwerk!)
- Vaultwarden nicht erreichbar
- zabooz.duckdns.org nicht erreichbar

### Sofortmaßnahmen

1. **VPS-Status prüfen:**
   ```bash
   ping 152.53.111.11
   ssh zabooz@152.53.111.11
   ```

2. **Hoster kontaktieren** falls Server nicht erreichbar

3. **Lokaler Zugriff** über direktes Heimnetz (falls vor Ort)

### Wiederherstellung auf neuem VPS

```bash
# 1. Neuen VPS bereitstellen (Debian 12)

# 2. Domain umleiten
# DuckDNS auf neue IP aktualisieren

# 3. Headscale installieren
wget https://github.com/juanfont/headscale/releases/download/v0.22.3/headscale_0.22.3_linux_amd64.deb
sudo dpkg -i headscale_*.deb

# 4. Backup einspielen
tar -xzf headscale-backup.tar.gz -C /

# 5. Service starten
sudo systemctl enable --now headscale

# 6. Nginx + Let's Encrypt
sudo apt install nginx certbot python3-certbot-nginx
sudo certbot --nginx -d zabooz.duckdns.org

# 7. Vaultwarden Docker
docker compose -f /home/zabooz/vaultwarden/docker-compose.yml up -d

# 8. Alle Clients neu verbinden
# (auf jedem Client)
sudo systemctl restart tailscaled
```

### Kritische Backups für VPS

```
/etc/headscale/config.yaml      # Konfiguration
/var/lib/headscale/db.sqlite    # Node-Datenbank
/home/zabooz/vaultwarden/       # Vaultwarden Daten
/etc/nginx/sites-available/     # Nginx Config
```

---

## Szenario: Proxmox Host fällt aus

### Auswirkungen

- Alle VMs und LXCs offline
- Tailscale LXC (Subnet Router) offline
- Kein Zugriff auf Heimnetz über VPN
- Homepage offline

### Sofortmaßnahmen

1. **Hardware-Problem diagnostizieren:**
   - Netzteil?
   - RAM?
   - Festplatte?
   - Netzwerk?

2. **Falls reparierbar:** Hardware tauschen, Proxmox bootet

3. **Falls Totalausfall:** Neuinstallation nötig

### Wiederherstellung

#### Option A: Gleiche Hardware, Proxmox reinstall

```bash
# 1. Proxmox ISO booten
# 2. Neuinstallation (andere Disk oder vorhandene überschreiben)
# 3. Netzwerk konfigurieren (192.168.0.101)
# 4. Backups von externem Speicher einspielen
```

#### Option B: Neue Hardware

```bash
# 1. Proxmox auf neuer Hardware installieren
# 2. Netzwerk-Bridge vmbr0 konfigurieren
# 3. Storage hinzufügen
# 4. VMs/LXCs aus Backup wiederherstellen:

qmrestore /backup/vzdump-qemu-100-*.tar.zst 100
qmrestore /backup/vzdump-qemu-101-*.tar.zst 101
pct restore 102 /backup/vzdump-lxc-102-*.tar.zst
```

### Priorität bei Wiederherstellung

1. **Tailscale LXC (102)** - VPN-Zugriff wiederherstellen
2. **Debian VM (101)** - Homepage
3. **Windows VM (100)** - Falls benötigt

### Kritische Backups für Proxmox

```
/var/lib/vz/dump/              # VM/LXC Backups
/etc/pve/                       # Proxmox Konfiguration
/etc/network/interfaces         # Netzwerk-Config
```

---

## Szenario: Tailscale LXC offline

### Auswirkungen

- Kein VPN-Routing zum Heimnetz
- Proxmox nur direkt erreichbar
- Bestehende Tailscale-Verbindungen zum Container unterbrochen

### Sofortmaßnahmen

```bash
# 1. Container-Status prüfen (auf Proxmox)
pct status 102

# 2. Container starten falls gestoppt
pct start 102

# 3. In Container einloggen
pct enter 102

# 4. Tailscale-Status prüfen
tailscale status
systemctl status tailscaled

# 5. IP Forwarding prüfen
sysctl net.ipv4.ip_forward  # Muss 1 sein!

# 6. Falls offline bei Headscale, neu registrieren:
sudo tailscale up --login-server=https://zabooz.duckdns.org --advertise-exit-node --advertise-routes=192.168.0.0/24
# Auf VPS: headscale nodes register ...
```

### Wiederherstellung aus Backup

```bash
# Auf Proxmox
pct restore 102 /var/lib/vz/dump/vzdump-lxc-102-*.tar.zst --storage local-lvm

# Nach Restore: Container starten und Tailscale neu verbinden
pct start 102
pct enter 102
sudo tailscale up --login-server=https://zabooz.duckdns.org --advertise-exit-node --advertise-routes=192.168.0.0/24
```

---

## Szenario: Internet-Ausfall zu Hause

### Auswirkungen

- Kein Zugriff von außen auf Heimnetz
- VPS und Vaultwarden weiterhin erreichbar
- Headscale funktioniert weiterhin

### Workaround

1. **Mobile Daten** am Handy nutzen
2. **Tailscale** auf Handy aktiv → Zugriff auf VPS-Dienste
3. **Vaultwarden** bleibt erreichbar
4. Heimnetz nur vor Ort zugänglich

---

## Szenario: DuckDNS Ausfall

### Auswirkungen

- Domain zabooz.duckdns.org nicht auflösbar
- Headscale nicht erreichbar über Domain
- Let's Encrypt Renewal fehlschlägt

### Workaround

1. **Direkte IP** nutzen: `https://152.53.111.11`
   - Browser-Warnung wegen Zertifikat ignorieren
   - Oder: `/etc/hosts` Eintrag

2. **Tailscale-Clients:**
   ```bash
   sudo tailscale up --login-server=https://152.53.111.11 --accept-routes
   ```

### Langfristige Lösung

- Eigene Domain kaufen
- DNS bei Cloudflare/Route53
- Dynamic DNS anders lösen

---

## Kritische Konfigurationen sichern

### Automatisches Config-Backup Script

```bash
#!/bin/bash
# /root/backup-configs.sh

BACKUP_DIR="/backup/configs/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Proxmox Config
cp -r /etc/pve $BACKUP_DIR/pve
cp /etc/network/interfaces $BACKUP_DIR/

# LXC Configs
cp -r /etc/pve/lxc $BACKUP_DIR/lxc-configs

# Dokumentation der aktuellen Zustände
pct list > $BACKUP_DIR/lxc-list.txt
qm list > $BACKUP_DIR/vm-list.txt
ip addr > $BACKUP_DIR/ip-config.txt
```

### Was muss IMMER gesichert sein?

| System | Kritische Daten | Backup-Methode |
|--------|-----------------|----------------|
| VPS | Headscale DB, Vaultwarden | Täglich, Off-Site |
| Proxmox | VM/LXC Backups | Täglich, Lokal |
| Proxmox | /etc/pve | Bei Änderung |
| LXC 102 | Tailscale-Konfiguration | Im vzdump enthalten |

---

## Recovery-Checkliste

### Vor dem Disaster

- [ ] Tägliche Backups laufen
- [ ] Backups werden auf VPS/Off-Site kopiert
- [ ] Restore wurde getestet (monatlich!)
- [ ] Alle Passwörter in Vaultwarden
- [ ] Diese Dokumentation ist aktuell
- [ ] DuckDNS Token ist dokumentiert

### Nach dem Disaster

- [ ] Schadensausmaß feststellen
- [ ] Prioritäten gemäß Liste abarbeiten
- [ ] VPN-Zugriff wiederherstellen (Tailscale LXC)
- [ ] Kritische Dienste prüfen
- [ ] Backup-Integrität der wiederhergestellten Daten prüfen
- [ ] Ursache analysieren und dokumentieren
- [ ] Maßnahmen für Zukunft ableiten

---

## Kontakt-Informationen

| Dienst | Kontakt |
|--------|---------|
| VPS Hoster | (Support-URL eintragen) |
| DuckDNS | https://www.duckdns.org |
| Let's Encrypt | https://letsencrypt.org |

---

## Recovery-Zeiten (geschätzt)

| Szenario | RTO (Recovery Time) |
|----------|---------------------|
| VPS Neustart | 5 Minuten |
| VPS Neuinstallation | 2-4 Stunden |
| Proxmox Neustart | 10 Minuten |
| Proxmox Neuinstallation | 4-8 Stunden |
| Tailscale LXC Restore | 30 Minuten |
| Vaultwarden Restore | 1 Stunde |

---

*Siehe auch: [backup-recovery.md](backup-recovery.md) | [security-hardening.md](security-hardening.md)*
