# 🚀 Headscale VPN Setup - Komplette Dokumentation

**Erstellt:** 18. Januar 2026  
**Author:** Daniel (zabooz)  
**Setup:** Self-hosted Tailscale alternative mit Headscale

---

## 📊 Netzwerk-Architektur

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INTERNET                                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  VPS (152.53.111.11)    │
                    │  zabooz.duckdns.org     │
                    ├─────────────────────────┤
                    │  Nginx (Port 443)       │ ← HTTPS/WebSockets
                    │  ↓                      │
                    │  Headscale (Port 8090)  │ ← Control Server
                    │  DERP Server (Port 443) │ ← Relay/STUN
                    │  DuckDNS Auto-Update    │
                    └────────────┬────────────┘
                                 │
                    Headscale Mesh Network
                    (100.64.0.0/10 CGNAT)
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        │                        │                        │
┌───────▼────────┐      ┌────────▼──────┐      ┌─────────▼────────┐
│  Laptop        │      │ LXC Container │      │ Jules Gerät      │
│  maschinchen   │      │  tailscale    │      │ (neuer Node)     │
│  100.64.0.2    │      │  100.64.0.1   │      │ 100.64.0.3       │
│                │      │               │      │                  │
│ User: zabooz   │      │ User: zabooz  │      │ User: jules      │
└────────────────┘      └───────┬───────┘      └──────────────────┘
                                │
                    Exit-Node + Subnet Router
                    Routes: 0.0.0.0/0, ::/0
                            192.168.0.0/24
                                │
                    ┌───────────▼────────────┐
                    │   HEIMNETZWERK         │
                    │   192.168.0.0/24       │
                    ├────────────────────────┤
                    │ Router: 192.168.0.1    │
                    │ Proxmox: 192.168.0.101 │
                    │ LXC: 192.168.0.150     │
                    │ Debian VM: 192.168.0.158│
                    │ Windows VM: DHCP       │
                    └────────────────────────┘
```

---

## 🎯 Setup-Übersicht

### Komponenten

| Komponente | IP/Domain | Funktion |
|------------|-----------|----------|
| **VPS** | 152.53.111.11 | Headscale Control Server |
| **Domain** | zabooz.duckdns.org | DuckDNS mit Auto-Update |
| **Headscale** | Port 8090 (intern) | VPN Control Server |
| **Nginx** | Port 443 (extern) | Reverse Proxy mit HTTPS |
| **DERP Server** | Port 443 | Relay/STUN Server |
| **LXC Container** | 192.168.0.150 / 100.64.0.1 | Exit-Node + Subnet Router |
| **Laptop** | 100.64.0.2 | Client |
| **Proxmox** | 192.168.0.101 | Virtualisierungs-Host |

### Features

✅ **Ende-zu-Ende verschlüsselt** (WireGuard)  
✅ **HTTPS** mit Let's Encrypt  
✅ **Eigener DERP Server** (keine Abhängigkeit von Tailscale)  
✅ **Exit-Node** (Internet über Heimnetz)  
✅ **Subnet Router** (Zugriff auf komplettes Heimnetz)  
✅ **User-Isolation** (Multi-Tenant fähig)  
✅ **DuckDNS Auto-Update** (Dynamic DNS)

---

## 🔐 Wie das Netzwerk funktioniert

### 1. Control Plane (Headscale Server)
- **VPS** koordiniert alle Nodes
- Vergibt IP-Adressen (100.64.0.x aus CGNAT-Range)
- Verwaltet Routing-Tabellen
- Authentifiziert Nodes über NodeKeys

### 2. Data Plane (Mesh Network)
- Nodes verbinden sich **direkt** miteinander (Peer-to-Peer)
- Falls direkte Verbindung nicht möglich → **DERP Relay** auf VPS
- Verschlüsselung: **WireGuard** (state-of-the-art VPN)
- NAT-Traversal via STUN

### 3. Exit-Node & Subnet Router (LXC Container)
- **Exit-Node:** Routet Internet-Traffic für andere Nodes
- **Subnet Router:** Gibt Zugriff auf Heimnetz (192.168.0.0/24)
- IP Forwarding aktiviert
- Alle verbundenen Nodes können auf Proxmox, VMs etc. zugreifen

---

## 👥 User Management

### Was sind User in Headscale?

- **User = Namespace/Mandant** (ähnlich wie Organisationen)
- Jeder User hat eigene Nodes
- Nodes verschiedener User können sich **standardmäßig nicht sehen**
- Mit ACL Policy kann Cross-User Access erlaubt werden

### User erstellen

```bash
# SSH zum VPS
ssh zabooz@152.53.111.11

# Neuen User erstellen
sudo headscale users create BENUTZERNAME

# Beispiele:
sudo headscale users create zabooz
sudo headscale users create jules
sudo headscale users create familie
sudo headscale users create arbeit
```

### User auflisten

```bash
sudo headscale users list
```

**Output:**
```
ID | Name    | Username | Email | Created            
1  |         | zabooz   |       | 2026-01-18 20:07:05
2  |         | jules    |       | 2026-01-18 22:10:15
3  |         | familie  |       | 2026-01-18 22:15:00
```

### User löschen

```bash
sudo headscale users delete USERNAME
```

---

## 📱 Neues Gerät hinzufügen

### Schritt 1: Tailscale installieren

#### Linux (Debian/Ubuntu)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
```

#### Linux (Arch/CachyOS)
```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
```

#### Windows
- Download: https://tailscale.com/download/windows
- Installieren und starten

#### Android/iOS
- Tailscale App aus dem Store installieren

---

### Schritt 2: Mit Headscale verbinden

#### Linux/Mac
```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

#### Windows (PowerShell als Administrator)
```powershell
tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

**Output:**
```
To authenticate, visit:
  https://zabooz.duckdns.org/register/nodekey:xxxxxxxxxxxxxxxxx

Or run:
  headscale nodes register --key xxxxxxxxxxxxxxxxx --user USERNAME
```

**→ Kopiere den `nodekey:xxxxxxxxx`**

---

### Schritt 3: Node auf dem VPS registrieren

```bash
# SSH zum VPS
ssh zabooz@152.53.111.11

# Node registrieren (User auswählen!)
sudo headscale nodes register --user BENUTZERNAME --key nodekey:xxxxxxxxx

# Beispiele:
sudo headscale nodes register --user zabooz --key nodekey:abc123
sudo headscale nodes register --user jules --key nodekey:def456
```

**Output:**
```
Node GERÄTENAME registered
```

---

### Schritt 4: Überprüfen

```bash
# Alle Nodes anzeigen
sudo headscale nodes list
```

**Output:**
```
ID | Hostname     | Name         | User    | IP addresses    | Connected
1  | tailscale    | tailscale    | zabooz  | 100.64.0.1      | online   
2  | maschinchen  | maschinchen  | zabooz  | 100.64.0.2      | online   
3  | handy-jules  | handy-jules  | jules   | 100.64.0.3      | online   
```

---

## 🛣️ Routes Management

### Routes anzeigen

```bash
sudo headscale nodes list-routes
```

**Output:**
```
ID | Hostname  | Approved                        | Available                       
1  | tailscale | 0.0.0.0/0, 192.168.0.0/24, ::/0 | 0.0.0.0/0, 192.168.0.0/24, ::/0
```

### Routes aktivieren

```bash
# Exit-Node + Subnet Router aktivieren
sudo headscale nodes approve-routes --identifier 1 --routes 0.0.0.0/0,::/0,192.168.0.0/24
```

---

## 🎯 Client-Befehle

### Verbindung testen

```bash
# Status anzeigen
tailscale status

# Ping zu anderem Node
ping 100.64.0.1

# Ping zu Gerät im Heimnetz
ping 192.168.0.101
```

### Exit-Node nutzen

```bash
# Exit-Node aktivieren
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# IP checken (sollte Heim-IP zeigen)
curl ifconfig.me

# Exit-Node deaktivieren
sudo tailscale up --accept-routes
```

### Verbindung trennen

```bash
# Temporär trennen (Key bleibt gültig)
sudo tailscale down

# Komplett ausloggen (Node muss neu registriert werden)
sudo tailscale logout
```

---

## 🔧 Server-Administration

### Headscale Service

```bash
# Status checken
sudo systemctl status headscale

# Neustarten
sudo systemctl restart headscale

# Logs anzeigen
sudo journalctl -u headscale -f

# Logs der letzten 100 Zeilen
sudo journalctl -u headscale -n 100 --no-pager
```

### Nginx Service

```bash
# Status
sudo systemctl status nginx

# Config testen
sudo nginx -t

# Neu laden (ohne Unterbrechung)
sudo systemctl reload nginx

# Neustarten
sudo systemctl restart nginx
```

### DuckDNS Update

```bash
# Manuelles Update
sudo /usr/local/bin/duckdns-update.sh

# Log checken
cat /var/log/duckdns.log

# Cronjob anzeigen
sudo crontab -l
```

---

## 📋 Wichtige Config-Dateien

### Headscale Config
**Pfad:** `/etc/headscale/config.yaml`

**Wichtige Einstellungen:**
```yaml
server_url: https://zabooz.duckdns.org
listen_addr: 127.0.0.1:8090
metrics_listen_addr: 127.0.0.1:9090

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    stun_listen_addr: "0.0.0.0:3478"
    ipv4: 152.53.111.11
  urls: []
```

### Nginx Config
**Pfad:** `/etc/nginx/sites-available/headscale`

**Config:**
```nginx
map $http_upgrade $connection_upgrade {
    default keep-alive;
    'websocket' upgrade;
    '' close;
}

server {
    listen 80;
    server_name zabooz.duckdns.org;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name zabooz.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/zabooz.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/zabooz.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $server_name;
        proxy_redirect http:// https://;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}
```

### LXC Container Config
**Pfad:** `/etc/pve/lxc/102.conf`

**Wichtige Zeilen für TUN/TAP:**
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
```

### DuckDNS Update Script
**Pfad:** `/usr/local/bin/duckdns-update.sh`

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=zabooz&token=1c42529f-69e4-4098-8a8e-f80f1177e13a&ip=" | curl -k -o /var/log/duckdns.log -K -
```

**Cronjob:** (alle 5 Minuten)
```
*/5 * * * * /usr/local/bin/duckdns-update.sh >/dev/null 2>&1
```

---

## 🔥 Cheat Sheet - Häufige Befehle

### VPS (Headscale Server)

```bash
# User Management
sudo headscale users create USERNAME
sudo headscale users list
sudo headscale users delete USERNAME

# Node Management
sudo headscale nodes register --user USERNAME --key nodekey:xxxxx
sudo headscale nodes list
sudo headscale nodes delete --identifier ID
sudo headscale nodes rename --identifier ID --name NEUER_NAME

# Routes Management
sudo headscale nodes list-routes
sudo headscale nodes approve-routes --identifier ID --routes 0.0.0.0/0,::/0,192.168.0.0/24

# Service Management
sudo systemctl status headscale
sudo systemctl restart headscale
sudo systemctl reload nginx
sudo journalctl -u headscale -f

# DuckDNS
sudo /usr/local/bin/duckdns-update.sh
cat /var/log/duckdns.log
```

### Client (Laptop, Handy, etc.)

```bash
# Verbinden
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes

# Status
tailscale status
tailscale netcheck

# Exit-Node
sudo tailscale up --exit-node=100.64.0.1 --accept-routes
sudo tailscale up --accept-routes  # Exit-Node deaktivieren

# Verbindung
sudo tailscale down     # Temporär trennen
sudo tailscale logout   # Komplett ausloggen

# IP Check
curl ifconfig.me
```

---

## 🛡️ Sicherheit & Best Practices

### Firewall (UFW auf VPS)

```bash
# Offene Ports checken
sudo ufw status numbered

# Wichtige Ports:
# - 22/tcp   (SSH)
# - 443/tcp  (HTTPS/Nginx)
# - 3478/udp (STUN)
```

### Let's Encrypt Zertifikat erneuern

```bash
# Automatische Erneuerung testen
sudo certbot renew --dry-run

# Manuell erneuern
sudo certbot renew

# Zertifikats-Info anzeigen
sudo certbot certificates
```

### Backups

**Wichtige Dateien sichern:**
- `/etc/headscale/config.yaml`
- `/var/lib/headscale/db.sqlite` (Headscale Database)
- `/etc/nginx/sites-available/headscale`
- `/etc/letsencrypt/` (SSL Zertifikate)

```bash
# Beispiel Backup-Script
tar -czf headscale-backup-$(date +%Y%m%d).tar.gz \
  /etc/headscale/ \
  /var/lib/headscale/ \
  /etc/nginx/sites-available/headscale
```

---

## 🐛 Troubleshooting

### Problem: Node kann sich nicht verbinden

```bash
# Auf dem VPS - Headscale Logs checken
sudo journalctl -u headscale -n 50 --no-pager

# Häufige Fehler:
# - "No Upgrade header" → Nginx WebSocket Config fehlt
# - "500 Internal Server Error" → Headscale Config-Fehler
```

### Problem: Exit-Node funktioniert nicht

```bash
# Routes checken
sudo headscale nodes list-routes

# IP Forwarding im Container checken
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding

# Sollte beides "1" sein
```

### Problem: DuckDNS nicht aktualisiert

```bash
# Manuelles Update testen
sudo /usr/local/bin/duckdns-update.sh
cat /var/log/duckdns.log

# Sollte "OK" zeigen

# Cronjob checken
sudo crontab -l
```

### Problem: HTTPS funktioniert nicht

```bash
# Nginx Config testen
sudo nginx -t

# Certbot Zertifikat checken
sudo certbot certificates

# Nginx neu laden
sudo systemctl reload nginx
```

---

## 📚 Für LAP-Vorbereitung

### Gelernte Themen

✅ **VPN-Technologie**
- WireGuard Protokoll
- Mesh-Netzwerke
- NAT-Traversal (STUN/DERP)

✅ **Container-Virtualisierung**
- LXC Container in Proxmox
- TUN/TAP Devices
- Unprivileged Containers

✅ **Netzwerk-Konfiguration**
- IP Routing
- IP Forwarding
- Subnet Routing
- Exit-Nodes

✅ **Linux System Administration**
- Systemd Services
- Cronjobs
- Log-Management (journalctl)

✅ **HTTPS/TLS**
- Let's Encrypt
- Certbot
- SSL-Zertifikate

✅ **Reverse Proxy**
- Nginx Konfiguration
- WebSocket-Support
- SSL-Termination

✅ **Firewall**
- UFW (Uncomplicated Firewall)
- iptables
- Port-Management

✅ **DNS**
- DuckDNS (Dynamic DNS)
- A-Records
- DNS-Propagation

---

## 🎯 Architektur-Highlights

### Was macht dieses Setup besonders?

1. **Vollständig selbst-gehostet** - Keine Abhängigkeit von Tailscale Inc.
2. **HTTPS-gesichert** - Professionelle Verschlüsselung mit Let's Encrypt
3. **Eigener DERP Server** - Volle Kontrolle über Relay-Infrastruktur
4. **Multi-User fähig** - User-Isolation für verschiedene Nutzergruppen
5. **Exit-Node** - Internet-Traffic über Heimnetz möglich
6. **Subnet Router** - Zugriff auf komplettes Heimnetzwerk
7. **Auto-Update DNS** - DuckDNS wird automatisch aktualisiert
8. **Hochverfügbar** - Systemd managed Services mit Auto-Restart

### Technologie-Stack

| Layer | Technologie |
|-------|-------------|
| VPN | WireGuard |
| Control Server | Headscale v0.27.1 |
| Web Server | Nginx |
| SSL/TLS | Let's Encrypt (Certbot) |
| Container | LXC (Proxmox) |
| DNS | DuckDNS |
| OS | Debian 12 |
| Firewall | UFW / iptables |

---

## 🔗 Nützliche Links

- **Headscale:** https://github.com/juanfont/headscale
- **Tailscale:** https://tailscale.com
- **DuckDNS:** https://www.duckdns.org
- **Let's Encrypt:** https://letsencrypt.org
- **WireGuard:** https://www.wireguard.com

---

## 📝 Notizen

### Aktuelle Setup-Details

- **VPS IP:** 152.53.111.11
- **Domain:** zabooz.duckdns.org
- **DuckDNS Token:** 1c42529f-69e4-4098-8a8e-f80f1177e13a
- **Headscale Version:** v0.27.1
- **Heimnetz:** 192.168.0.0/24
- **Tailscale Netz:** 100.64.0.0/10

### Aktive User

1. **zabooz** - Hauptuser (Laptop + LXC Container)
2. **jules** - Zweiter User

### Aktive Nodes

1. **tailscale** (100.64.0.1) - LXC Container, Exit-Node + Subnet Router
2. **maschinchen** (100.64.0.2) - Laptop
3. **jules node** (100.64.0.3) - Jules Gerät

---

**Setup erstellt am:** 18. Januar 2026  
**Status:** ✅ Vollständig funktionsfähig  
**Letzte Aktualisierung:** 18. Januar 2026

---

*Viel Erfolg bei der LAP-Prüfung! 🚀*
