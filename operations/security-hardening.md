# Security Hardening Guide

**Aktualisiert:** 24. Januar 2026
**Zweck:** Sicherheitskonfiguration für alle Systeme

---

## SSH Hardening

### Grundlegende Absicherung

**Datei:** `/etc/ssh/sshd_config`

```bash
# Root-Login deaktivieren
PermitRootLogin no

# Nur Key-Authentifizierung
PasswordAuthentication no
PubkeyAuthentication yes

# Leere Passwörter verbieten
PermitEmptyPasswords no

# Login-Versuche begrenzen
MaxAuthTries 3
MaxSessions 3

# Idle-Timeout (10 Minuten)
ClientAliveInterval 300
ClientAliveCountMax 2

# Protokoll Version 2 erzwingen (Standard in modernen Versionen)
Protocol 2
```

### SSH-Port ändern (optional)

```bash
# /etc/ssh/sshd_config
Port 2222    # Statt 22

# Service neustarten
sudo systemctl restart sshd

# Verbindung testen
ssh -p 2222 user@host
```

### SSH-Key Setup

```bash
# Auf Client: Key generieren
ssh-keygen -t ed25519 -C "beschreibung"

# Key auf Server kopieren
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Oder manuell:
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### SSH-Config anwenden

```bash
# Syntax prüfen
sudo sshd -t

# Service neustarten
sudo systemctl restart sshd
```

---

## Fail2ban

### Installation

```bash
sudo apt update
sudo apt install fail2ban -y
```

### Konfiguration

**Datei:** `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
# Generelle Einstellungen
bantime = 1h
findtime = 10m
maxretry = 3
ignoreip = 127.0.0.1/8 192.168.0.0/24 100.64.0.0/10

# E-Mail-Benachrichtigung (optional)
# destemail = admin@example.com
# sender = fail2ban@example.com
# action = %(action_mwl)s

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 24h
```

### Fail2ban verwalten

```bash
# Status anzeigen
sudo fail2ban-client status

# SSH-Jail Status
sudo fail2ban-client status sshd

# IP entbannen
sudo fail2ban-client set sshd unbanip <IP>

# Alle gebannten IPs
sudo fail2ban-client get sshd banned

# Service neustarten
sudo systemctl restart fail2ban
```

---

## Firewall (UFW)

### UFW auf VPS

```bash
# Installation
sudo apt install ufw -y

# Default Policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH erlauben (WICHTIG: VOR aktivieren!)
sudo ufw allow ssh
# Oder mit anderem Port:
sudo ufw allow 2222/tcp

# HTTPS erlauben
sudo ufw allow 443/tcp

# HTTP erlauben (für Let's Encrypt)
sudo ufw allow 80/tcp

# STUN für VPN
sudo ufw allow 3478/udp

# Tailscale Interface erlauben
sudo ufw allow in on tailscale0

# UFW aktivieren
sudo ufw enable

# Status anzeigen
sudo ufw status verbose
```

### UFW für Heimnetz-Zugriff

```bash
# Gesamtes Heimnetz erlauben
sudo ufw allow from 192.168.0.0/24

# Nur bestimmte IPs
sudo ufw allow from 192.168.0.101  # Proxmox

# Tailscale-Netz erlauben
sudo ufw allow from 100.64.0.0/10
```

---

## iptables (Proxmox/LXC)

### Grundlegende Absicherung

```bash
# Bestehende Verbindungen erlauben
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Loopback erlauben
iptables -A INPUT -i lo -j ACCEPT

# SSH erlauben
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Proxmox Web-UI (nur aus Heimnetz)
iptables -A INPUT -p tcp --dport 8006 -s 192.168.0.0/24 -j ACCEPT

# ICMP (Ping) erlauben
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Alles andere verwerfen
iptables -A INPUT -j DROP
```

### Regeln persistent machen

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

## Service-spezifische Sicherheit

### Vaultwarden

**VPN-Only Zugriff via Nginx:**

```nginx
location /vault/ {
    # Nur Tailscale-Netz erlauben
    allow 100.64.0.0/10;
    deny all;

    proxy_pass http://127.0.0.1:8000;
    # ... weitere proxy settings
}
```

**Vaultwarden Einstellungen:**

```yaml
# docker-compose.yml
environment:
  SIGNUPS_ALLOWED: "false"           # Keine neuen Registrierungen
  INVITATIONS_ALLOWED: "true"        # Einladungen erlauben
  ADMIN_TOKEN: "SECURE_RANDOM_TOKEN" # Admin-Panel absichern
  WEBSOCKET_ENABLED: "true"
```

### Headscale

- Läuft nur auf localhost (127.0.0.1:8090)
- Nginx als Reverse Proxy mit HTTPS
- Let's Encrypt Zertifikat

```yaml
# /etc/headscale/config.yaml
listen_addr: 127.0.0.1:8090  # Nur lokal!
```

### Proxmox

- Web-UI nur aus lokalem Netz zugänglich
- API-Tokens statt Passwörter für Automatisierung
- 2FA für Admin-Accounts aktivieren

```bash
# API-Token für Homepage Widget
pveum user token add root@pam homepage --privsep 0
```

---

## Regelmäßige Updates

### Automatische Security Updates (Debian/Ubuntu)

```bash
# Unattended Upgrades installieren
sudo apt install unattended-upgrades apt-listchanges

# Konfigurieren
sudo dpkg-reconfigure unattended-upgrades

# Aktivieren
sudo systemctl enable unattended-upgrades
```

### Manuelle Updates

```bash
# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Nur Security Updates
sudo apt-get -s dist-upgrade | grep "^Inst" | grep -i security

# Proxmox Updates
apt update && apt dist-upgrade -y

# Docker Container aktualisieren
docker compose pull
docker compose up -d
```

### Update-Zeitplan

| System | Frequenz | Methode |
|--------|----------|---------|
| VPS (Debian) | Wöchentlich | Manuell + Unattended |
| Proxmox Host | Monatlich | Manuell mit Backup |
| VMs/LXCs | Wöchentlich | Manuell |
| Docker Images | Bei Release | docker compose pull |

---

## Logging & Monitoring

### Wichtige Log-Dateien

```bash
# SSH-Logins
/var/log/auth.log

# System-Logs
/var/log/syslog

# Nginx Access/Error
/var/log/nginx/access.log
/var/log/nginx/error.log

# Fail2ban
/var/log/fail2ban.log

# Docker Container
docker logs <container_name>
```

### Log-Überwachung

```bash
# SSH-Login-Versuche live
tail -f /var/log/auth.log | grep sshd

# Fehlgeschlagene Logins
grep "Failed password" /var/log/auth.log

# Erfolgreiche Logins
grep "Accepted" /var/log/auth.log

# Nginx Fehler
tail -f /var/log/nginx/error.log
```

---

## Passwort-Richtlinien

### Starke Passwörter

- Mindestens 16 Zeichen
- Groß/Kleinbuchstaben, Zahlen, Sonderzeichen
- Keine Wörter aus dem Wörterbuch
- Für jeden Service ein anderes Passwort
- Passwort-Manager nutzen (Vaultwarden!)

### Passwort-Komplexität erzwingen

```bash
# PAM-Modul installieren
sudo apt install libpam-pwquality

# /etc/pam.d/common-password
password requisite pam_pwquality.so retry=3 minlen=12 difok=3
```

---

## Netzwerk-Segmentierung

### VLANs (falls Router unterstützt)

| VLAN | Netz | Zweck |
|------|------|-------|
| 1 | 192.168.0.0/24 | Management |
| 10 | 192.168.10.0/24 | Server |
| 20 | 192.168.20.0/24 | Clients |
| 99 | 192.168.99.0/24 | Gäste |

### Tailscale ACLs

Headscale unterstützt ACLs für Zugriffskontrolle:

```json
{
  "acls": [
    {
      "action": "accept",
      "users": ["zabooz"],
      "ports": ["*:*"]
    },
    {
      "action": "accept",
      "users": ["jules"],
      "ports": ["tailscale:*"]
    }
  ]
}
```

---

## Security Checkliste

### VPS

- [ ] SSH nur mit Keys
- [ ] SSH Root-Login deaktiviert
- [ ] Fail2ban installiert und konfiguriert
- [ ] UFW aktiv mit minimalen Regeln
- [ ] Automatische Security Updates
- [ ] Vaultwarden nur über VPN erreichbar
- [ ] Let's Encrypt Zertifikat aktiv

### Proxmox

- [ ] Web-UI nur aus lokalem Netz
- [ ] Starke Passwörter für alle Accounts
- [ ] Regelmäßige Backups
- [ ] Updates eingespielt

### Allgemein

- [ ] Passwort-Manager im Einsatz
- [ ] Separate Passwörter für jeden Service
- [ ] 2FA wo möglich aktiviert
- [ ] Logs werden überwacht
- [ ] Backup-Integrität getestet

---

## Incident Response

### Bei verdächtiger Aktivität

1. **Isolieren:** Betroffenes System vom Netz trennen
2. **Analysieren:** Logs prüfen, was ist passiert?
3. **Eindämmen:** Weitere Ausbreitung verhindern
4. **Beseitigen:** Schadsoftware/Zugang entfernen
5. **Wiederherstellen:** Aus sauberem Backup
6. **Dokumentieren:** Was ist passiert, was wurde getan?

### Nützliche Befehle

```bash
# Aktive Verbindungen
ss -tuln
netstat -tuln

# Laufende Prozesse
ps aux
top

# Letzte Logins
last
lastb  # Fehlgeschlagene

# Offene Ports
nmap -sV localhost

# Dateiänderungen der letzten 24h
find / -mtime -1 -type f 2>/dev/null
```

---

*Siehe auch: [backup-recovery.md](backup-recovery.md) | [disaster-recovery.md](disaster-recovery.md)*
