# Headscale VPN Infrastructure

**Erstellt:** 18. Januar 2026
**Aktualisiert:** 24. Januar 2026
**Author:** Daniel (zabooz)
**Setup:** Self-hosted Tailscale alternative mit Headscale

---

## Netzwerk-Architektur

```
                                    INTERNET
                                       │
                       ┌───────────────┴───────────────┐
                       │  VPS (152.53.111.11)          │
                       │  zabooz.duckdns.org           │
                       ├───────────────────────────────┤
                       │  Nginx (Port 443)             │ ← HTTPS/WebSockets
                       │  ↓                            │
                       │  Headscale (Port 8090)        │ ← Control Server
                       │  DERP Server (Port 443)       │ ← Relay/STUN
                       │  DuckDNS Auto-Update          │
                       └───────────────┬───────────────┘
                                       │
                       Headscale Mesh Network
                       (100.64.0.0/10 CGNAT)
                                       │
       ┌───────────────────────────────┼───────────────────────────────┐
       │                               │                               │
┌──────▼───────┐              ┌────────▼──────┐              ┌─────────▼────────┐
│  Laptop      │              │ LXC Container │              │ Weitere Nodes    │
│  maschinchen │              │  tailscale    │              │                  │
│  100.64.0.2  │              │  100.64.0.1   │              │ 100.64.0.x       │
│              │              │               │              │                  │
│ User: zabooz │              │ User: zabooz  │              │ User: xyz        │
└──────────────┘              └───────┬───────┘              └──────────────────┘
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
                          │ LXC: 192.168.0.112     │
                          │ Debian VM: 192.168.0.111│
                          └────────────────────────┘
```

---

## Komponenten-Übersicht

| Komponente | IP/Domain | Funktion |
|------------|-----------|----------|
| **VPS** | 152.53.111.11 | Headscale Control Server |
| **Domain** | zabooz.duckdns.org | DuckDNS mit Auto-Update |
| **Headscale** | Port 8090 (intern) | VPN Control Server |
| **Nginx** | Port 443 (extern) | Reverse Proxy mit HTTPS |
| **DERP Server** | Port 443 | Relay/STUN Server |
| **LXC Container** | 192.168.0.112 / 100.64.0.1 | Exit-Node + Subnet Router |
| **Laptop** | 100.64.0.2 | Client |
| **Proxmox** | 192.168.0.101 | Virtualisierungs-Host |

### Features

- Ende-zu-Ende verschlüsselt (WireGuard)
- HTTPS mit Let's Encrypt
- Eigener DERP Server (keine Abhängigkeit von Tailscale)
- Exit-Node (Internet über Heimnetz)
- Subnet Router (Zugriff auf komplettes Heimnetz)
- User-Isolation (Multi-Tenant fähig)
- DuckDNS Auto-Update (Dynamic DNS)

---

## Wie das Netzwerk funktioniert

### 1. Control Plane (Headscale Server)

- VPS koordiniert alle Nodes
- Vergibt IP-Adressen (100.64.0.x aus CGNAT-Range)
- Verwaltet Routing-Tabellen
- Authentifiziert Nodes über NodeKeys

### 2. Data Plane (Mesh Network)

- Nodes verbinden sich **direkt** miteinander (Peer-to-Peer)
- Falls direkte Verbindung nicht möglich → DERP Relay auf VPS
- Verschlüsselung: WireGuard (state-of-the-art VPN)
- NAT-Traversal via STUN

### 3. Exit-Node & Subnet Router (LXC Container)

- **Exit-Node:** Routet Internet-Traffic für andere Nodes
- **Subnet Router:** Gibt Zugriff auf Heimnetz (192.168.0.0/24)
- IP Forwarding aktiviert
- Alle verbundenen Nodes können auf Proxmox, VMs etc. zugreifen

---

## Headscale Server Setup (VPS)

### Voraussetzungen

- VPS mit öffentlicher IP
- Domain (z.B. via DuckDNS)
- Nginx als Reverse Proxy
- Let's Encrypt SSL-Zertifikat

### Headscale Installation

```bash
# Headscale herunterladen (aktuelle Version prüfen!)
wget https://github.com/juanfont/headscale/releases/download/v0.22.3/headscale_0.22.3_linux_amd64.deb
sudo dpkg -i headscale_*.deb

# Service aktivieren
sudo systemctl enable --now headscale
```

### Headscale Config (/etc/headscale/config.yaml)

```yaml
server_url: https://zabooz.duckdns.org
listen_addr: 127.0.0.1:8090
metrics_listen_addr: 127.0.0.1:9090

grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false

ip_prefixes:
  - 100.64.0.0/10

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "home"
    region_name: "Home DERP"
    stun_listen_addr: "0.0.0.0:3478"

  urls:
    - https://zabooz.duckdns.org/derp/derp-map

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

dns:
  base_domain: zabooz.duckdns.org
  magic_dns: true
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
```

### Nginx Reverse Proxy (/etc/nginx/sites-available/headscale)

```nginx
server {
    listen 443 ssl http2;
    server_name zabooz.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/zabooz.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/zabooz.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
    }
}

server {
    listen 80;
    server_name zabooz.duckdns.org;
    return 301 https://$host$request_uri;
}
```

### DuckDNS Auto-Update (Cronjob)

```bash
# Crontab bearbeiten
crontab -e

# Alle 5 Minuten IP updaten
*/5 * * * * curl "https://www.duckdns.org/update?domains=zabooz&token=DEIN_TOKEN&ip="
```

---

## LXC Container Setup (Subnet Router + Exit-Node)

### Proxmox LXC Konfiguration

- **CT ID:** 102
- **Privileged:** YES (für Tailscale nötig)
- **IP:** 192.168.0.112/24
- **Features:** nesting=1, keyctl=1

### Grundkonfiguration

```bash
# Im LXC Container
ssh root@192.168.0.112

# Tailscale installieren
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled

# Mit Headscale verbinden
sudo tailscale up \
  --login-server=https://zabooz.duckdns.org \
  --advertise-exit-node \
  --advertise-routes=192.168.0.0/24
```

### Auf dem VPS registrieren

```bash
# NodeKey kopieren (vom Container Output)
ssh zabooz@152.53.111.11

# Node registrieren
sudo headscale nodes register --user zabooz --key nodekey:xxxxxxxxx

# Routes approven
sudo headscale routes list
sudo headscale routes enable --route 1  # Exit-Node (0.0.0.0/0)
sudo headscale routes enable --route 2  # Subnet (192.168.0.0/24)
```

### IP Forwarding aktivieren (dauerhaft)

```bash
# Im Container
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Checken
sysctl net.ipv4.ip_forward  # Muss 1 sein
```

### IP Forwarding Service (für LXC-Container)

LXC Container laden `/etc/sysctl.conf` nicht automatisch. Erstelle einen systemd Service:

```bash
# /etc/systemd/system/ip-forwarding.service
[Unit]
Description=Enable IP Forwarding
Before=tailscaled.service

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable ip-forwarding.service
```

### Auto-Start aktivieren (Proxmox)

```bash
pct set 102 --onboot 1
pct set 102 --startup order=1,up=30
```

---

## Subnet-Routing erklärt

### Was ist ein Subnet?

```
192.168.0.0/24 = Heimnetzwerk

- 192.168.0.0      = Netzwerk-Adresse
- /24              = Subnet-Maske (255.255.255.0)
- 192.168.0.1-254  = Verfügbare Host-Adressen
```

**Problem:** Dein Heimnetz (192.168.0.0/24) ist getrennt vom Tailscale-Netz (100.64.0.0/10)

**Lösung:** Der **Subnet Router** (LXC Container) ist die Brücke zwischen beiden Netzen

### Visualisierung

```
OHNE Subnet Router:
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
└─────────────┘                      └──────────────┘
      ❌ KEIN Zugriff auf Proxmox (192.168.0.101)

MIT Subnet Router:
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
└─────────────┘                      └──────┬───────┘
                                            │ Routes:
      ✅ Zugriff auf Proxmox!               │ 192.168.0.0/24
                                            │
                                     ┌──────▼───────┐
                                     │  Proxmox     │
                                     │ 192.168.0.101│
                                     └──────────────┘
```

---

## Headscale Befehle (VPS)

### User Management

```bash
headscale users create <name>
headscale users list
headscale users destroy <name>
```

### Node Management

```bash
headscale nodes list
headscale nodes register --user <user> --key <nodekey>
headscale nodes delete --identifier <id>
headscale nodes expire --identifier <id>
```

### Routes Management

```bash
headscale routes list
headscale routes enable --route <id>
headscale routes disable --route <id>
```

### Service Management

```bash
sudo systemctl status headscale
sudo systemctl restart headscale
sudo journalctl -u headscale -f
```

---

## Aktive Konfiguration

### Aktive User

1. **zabooz** - Hauptuser (Laptop + LXC Container)
2. **jules** - Zweiter User

### Aktive Nodes

| Hostname | Tailscale IP | LAN IP | Rolle |
|----------|--------------|--------|-------|
| tailscale | 100.64.0.1 | 192.168.0.112 | Subnet router, exit node |
| maschinchen | 100.64.0.2 | variabel | Client |
| zaboozmegafeschersuperserver | 100.64.0.5 | 152.53.111.11 | VPS/Coordinator |

---

## LAP-Relevante Konzepte

### VPN-Typen

- **Site-to-Site:** Verbindet zwei Netzwerke (z.B. Firmenstandorte)
- **Remote Access:** Einzelne Clients verbinden sich zu einem Netzwerk
- **Mesh VPN:** Alle Nodes können direkt miteinander kommunizieren (Tailscale/Headscale)

### WireGuard vs. OpenVPN

| Feature | WireGuard | OpenVPN |
|---------|-----------|---------|
| **Geschwindigkeit** | Sehr schnell | Langsamer |
| **Code-Basis** | 4.000 Zeilen | 100.000+ Zeilen |
| **Konfiguration** | Einfach | Komplex |
| **Protokoll** | UDP | TCP/UDP |

### Wichtige Ports

- **443 (HTTPS):** Headscale Control Server & DERP
- **3478 (UDP):** STUN (NAT-Traversal)
- **8090:** Headscale intern (hinter Nginx)

---

*Siehe auch: [vpn-client-guide.md](vpn-client-guide.md) | [vpn-troubleshooting.md](vpn-troubleshooting.md)*
