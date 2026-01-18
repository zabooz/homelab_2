# 🚀 Headscale VPN Setup - Kompakte Dokumentation

**Erstellt:** 18. Januar 2026  
**Author:** Daniel (zabooz)  
**Setup:** Self-hosted Tailscale alternative mit Headscale

---

## ⚡ QUICK START GUIDE - Neues Gerät hinzufügen

### 🎯 Diese Schritte machst du am häufigsten!

#### 1️⃣ User erstellen (auf dem VPS)

```bash
# SSH zum VPS
ssh zabooz@152.53.111.11

# User erstellen
sudo headscale users create BENUTZERNAME

# Beispiele:
sudo headscale users create familie
sudo headscale users create arbeit

# User anzeigen
sudo headscale users list
```

---

#### 2️⃣ Tailscale auf neuem Gerät installieren

**Linux (Debian/Ubuntu):**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
```

**Linux (Arch/CachyOS):**
```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
```

**Windows:**
- Download: https://tailscale.com/download/windows
- Installieren und starten

**Android/iOS:**
- Tailscale App aus dem Store installieren

---

#### 3️⃣ Gerät mit Headscale verbinden

**Linux/Mac:**
```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

**Windows (PowerShell als Admin):**
```powershell
tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

**Das Gerät zeigt dir jetzt einen Key an:**
```
To authenticate, visit:
  https://zabooz.duckdns.org/register/nodekey:abc123def456...

Or run:
  headscale nodes register --key nodekey:abc123def456... --user USERNAME
```

**→ Kopiere den `nodekey:xxxxxxxxx`**

---

#### 4️⃣ Node registrieren (auf dem VPS)

```bash
# SSH zum VPS (falls nicht mehr verbunden)
ssh zabooz@152.53.111.11

# Node registrieren
sudo headscale nodes register --user BENUTZERNAME --key nodekey:xxxxxxxxx

# Beispiele:
sudo headscale nodes register --user zabooz --key nodekey:abc123
sudo headscale nodes register --user familie --key nodekey:def456
```

**Output:**
```
Node GERÄTENAME registered
```

---

#### 5️⃣ Überprüfen

```bash
# Alle Nodes anzeigen
sudo headscale nodes list
```

**Auf dem neuen Gerät:**
```bash
# Status checken
tailscale status

# Proxmox testen
ping 192.168.0.101

# Browser öffnen
firefox https://192.168.0.101:8006
```

---

### 🔧 Häufige Befehle

**VPS (Headscale Server):**
```bash
# User
sudo headscale users create USERNAME
sudo headscale users list

# Nodes
sudo headscale nodes list
sudo headscale nodes register --user USERNAME --key nodekey:xxxxx
sudo headscale nodes delete --identifier ID

# Service
sudo systemctl status headscale
sudo systemctl restart headscale
sudo journalctl -u headscale -f
```

**Client (Laptop, Handy, etc.):**
```bash
# Verbinden
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes

# Status
tailscale status
tailscale netcheck

# Exit-Node nutzen
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# Trennen
sudo tailscale down     # Temporär
sudo tailscale logout   # Komplett
```

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

✅ Ende-zu-Ende verschlüsselt (WireGuard)  
✅ HTTPS mit Let's Encrypt  
✅ Eigener DERP Server (keine Abhängigkeit von Tailscale)  
✅ Exit-Node (Internet über Heimnetz)  
✅ Subnet Router (Zugriff auf komplettes Heimnetz)  
✅ User-Isolation (Multi-Tenant fähig)  
✅ DuckDNS Auto-Update (Dynamic DNS)

---

## 🔐 Wie das Netzwerk funktioniert

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

## 🌐 Subnet-Routing - Kompakte Erklärung

### Was ist ein Subnet?

```
192.168.0.0/24 = Heimnetzwerk

- 192.168.0.0      = Netzwerk-Adresse
- /24              = Subnet-Maske (255.255.255.0)
- 192.168.0.1-254  = Verfügbare Host-Adressen
```

**Problem:** Dein Heimnetz (192.168.0.0/24) ist getrennt vom Tailscale-Netz (100.64.0.0/10)

**Lösung:** Der **Subnet Router** (LXC Container) ist die Brücke zwischen beiden Netzen

---

### Wie funktioniert Subnet-Routing?

#### OHNE Subnet Router:

```
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
└─────────────┘                      └──────────────┘

      ❌ KEIN Zugriff auf Proxmox (192.168.0.101)
```

#### MIT Subnet Router:

```
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
└─────────────┘                      └──────┬───────┘
                                            │ Routes:
      ✅ Zugriff auf Proxmox!                │ 192.168.0.0/24
                                            │
                                     ┌──────▼───────┐
                                     │  Proxmox     │
                                     │ 192.168.0.101│
                                     └──────────────┘
```

---

## 🛠️ LXC Container Setup (Subnet Router + Exit-Node)

### Grundkonfiguration

```bash
# Im LXC Container
ssh root@192.168.0.150

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

---

## 🔍 Troubleshooting - Die wichtigsten Checks

### 1. Routing Tables prüfen

```bash
# Auf dem Laptop - Standard Route-Table
ip route

# WICHTIG: Tailscale nutzt eine SEPARATE Routing-Table (Table 52)!
# Deshalb siehst du die Routes NICHT in 'ip route'!
```

**⚠️ WICHTIG: Tailscale verwendet Policy-Based Routing!**

Tailscale erstellt seine Routes in einer separaten Routing-Table (Table 52), nicht in der Main-Table!

```bash
# Tailscale-Routes anzeigen:
ip route show table 52

# Du solltest sehen:
# 100.64.0.1 dev tailscale0 table 52
# 192.168.0.0/24 dev tailscale0 table 52  ← Subnet Route!

# Prüfen welche Route für eine IP verwendet wird:
ip route get 192.168.0.101
# Output: 192.168.0.101 dev tailscale0 table 52 src 100.64.0.2
```

**Was bedeutet das?**
- Linux nutzt **Policy-Based Routing**
- Normale Internet-Routes → Main Table (`ip route`)
- Tailscale-Routes → Table 52 (`ip route show table 52`)
- Kernel entscheidet automatisch welche Table benutzt wird
- **Das ist normal und sogar besser** (keine Konflikte mit normalen Routes!)

---

### 2. IP Forwarding checken

```bash
# Im Container
sysctl net.ipv4.ip_forward

# Output MUSS 1 sein!
# Falls 0:
sudo sysctl -w net.ipv4.ip_forward=1
```

**Ohne IP Forwarding:** Container nimmt Pakete an, leitet sie aber nicht weiter!

---

### 3. NAT/MASQUERADE prüfen

```bash
# Im Container
sudo iptables -t nat -L POSTROUTING -n -v

# Du solltest sehen:
# MASQUERADE  all  --  *  eth0  100.64.0.0/10  0.0.0.0/0
```

**Was ist MASQUERADE?**
- Ändert die Source-IP von Tailscale-Paketen
- Proxmox sieht Container-IP (192.168.0.150) statt Laptop-IP (100.64.0.2)
- Ohne MASQUERADE: Proxmox verwirft Pakete (unbekannte Source)

---

### 4. Packet Flow testen

```bash
# Im Container - Traffic beobachten
sudo tcpdump -i tailscale0 -n

# Dann vom Laptop:
ping 192.168.0.101

# Du siehst:
# 100.64.0.2 > 192.168.0.101: ICMP echo request
# 192.168.0.101 > 100.64.0.2: ICMP echo reply
```

---

### 5. Firewall checken

```bash
# Im Container
sudo iptables -L FORWARD -n -v

# Es sollte KEINE REJECT/DROP Rule für dein Traffic sein
# Falls doch:
sudo iptables -I FORWARD -i tailscale0 -j ACCEPT
sudo iptables -I FORWARD -o tailscale0 -j ACCEPT
```

---

### 🔍 WICHTIG: "Ich sehe keine Tailscale-Routes in ip route!"

**Das ist NORMAL!** Tailscale nutzt Policy-Based Routing mit Table 52.

```bash
# ❌ FALSCH - zeigt Tailscale-Routes NICHT:
ip route

# ✅ RICHTIG - Tailscale-Routes anzeigen:
ip route show table 52

# ✅ RICHTIG - Prüfen ob Route funktioniert:
ip route get 192.168.0.101
# Sollte zeigen: "dev tailscale0 table 52"

# ✅ RICHTIG - Alle Tailscale-Routes anzeigen:
ip route show table all | grep tailscale

# ✅ RICHTIG - Testen:
ping 192.168.0.101
traceroute 192.168.0.101
```

**Warum macht Tailscale das?**
- ✅ Keine Konflikte mit normalen Routes
- ✅ Saubere Trennung VPN vs. Internet-Traffic
- ✅ Automatisches Failover wenn VPN ausfällt
- ✅ Professionelleres Routing-Setup

**Beispiel-Output von `ip route show table 52`:**
```bash
100.64.0.1 dev tailscale0 table 52        # Container
100.64.0.3 dev tailscale0 table 52        # Anderer Node
192.168.0.0/24 dev tailscale0 table 52    # Subnet Route ✅
100.100.100.100 dev tailscale0 table 52   # Tailscale DNS
```

---

## 🧠 Routing & NAT - Kompakte Theorie

### Der komplette Packet Flow

```
┌─────────┐        ┌───────────┐        ┌─────────┐
│ Laptop  │───────►│ Container │───────►│ Proxmox │
│.0.2     │        │   .0.1    │        │  .0.101 │
└─────────┘        └───────────┘        └─────────┘

1. Laptop: "Sende Paket zu 192.168.0.101"
2. Routing Table: "192.168.0.0/24 via 100.64.0.1"
3. Container empfängt auf tailscale0
4. IP Forwarding: "Weiterleiten erlaubt ✅"
5. Container Routing: "192.168.0.101 über eth0"
6. MASQUERADE: Source-IP ändern (100.64.0.2 → 192.168.0.150)
7. Paket raus über eth0
8. Proxmox empfängt von 192.168.0.150
9. Proxmox antwortet an 192.168.0.150
10. Container empfängt Antwort
11. NAT zurück: Destination ändern (→ 100.64.0.2)
12. Paket zurück über tailscale0
13. Laptop empfängt ✅
```

---

### Warum braucht man MASQUERADE?

**Ohne MASQUERADE:**
```
Laptop sendet:    100.64.0.2 → 192.168.0.101
Proxmox denkt:    "WTF ist 100.64.0.2? Kenne ich nicht!"
                  → Paket verworfen ❌
```

**Mit MASQUERADE:**
```
Container ändert: 100.64.0.2 → 192.168.0.150
Proxmox denkt:    "Ah, Container! Den kenne ich!"
                  → Antwortet an Container
Container ändert: Antwort zurück an 100.64.0.2
                  → Laptop bekommt Antwort ✅
```

---

## 🔧 Wichtige Linux-Konzepte

### 1. Routing Tables

```bash
# Anzeigen
ip route

# Häufige Ausgabe:
default via 192.168.0.1 dev eth0
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.150
100.64.0.0/10 dev tailscale0 scope link
192.168.0.0/24 via 100.64.0.1 dev tailscale0
```

**WICHTIG: Tailscale nutzt Table 52!**

Moderne Tailscale-Versionen verwenden Policy-Based Routing:

```bash
# Main Table (Standard)
ip route show table main

# Tailscale Table 52
ip route show table 52

# Prüfen welche Route verwendet wird
ip route get 192.168.0.101
```

**Wichtige Regeln:**
- **Längster Präfix-Match gewinnt:** /24 schlägt /10
- **Default Route:** Wenn nichts anderes passt → Router (192.168.0.1)
- **Scope link:** Direkt erreichbar (kein Router nötig)
- **Table 52:** Tailscale-Routes isoliert von Main-Routes

---

### 2. IP Forwarding

```bash
# Checken
sysctl net.ipv4.ip_forward

# Temporär aktivieren
sudo sysctl -w net.ipv4.ip_forward=1

# Dauerhaft (in /etc/sysctl.conf):
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

**Was macht es?**
- Erlaubt Pakete zwischen Interfaces weiterzuleiten
- Ohne: System routet nur eigene Pakete
- Mit: System wird zum Router

---

### 3. iptables/nftables

```bash
# iptables Regeln anzeigen
sudo iptables -L -n -v              # Filter table
sudo iptables -t nat -L -n -v       # NAT table

# NAT Rule manuell erstellen
sudo iptables -t nat -A POSTROUTING -o eth0 -s 100.64.0.0/10 -j MASQUERADE

# Regeln dauerhaft machen
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

**iptables Chains:**
- **PREROUTING:** Vor Routing-Entscheidung (DNAT)
- **POSTROUTING:** Nach Routing-Entscheidung (SNAT/MASQUERADE)
- **FORWARD:** Pakete die weitergeleitet werden
- **INPUT:** Pakete für lokales System
- **OUTPUT:** Pakete vom lokalen System

---

## 🚨 Häufige Probleme & Lösungen

### Problem: "Keine Verbindung zu Proxmox"

```bash
# Checkliste:
1. Routes genehmigt? → headscale routes list
2. IP Forwarding an? → sysctl net.ipv4.ip_forward
3. NAT aktiv? → iptables -t nat -L POSTROUTING
4. Firewall blockiert? → iptables -L FORWARD
5. Tailscale läuft? → tailscale status
```

---

### Problem: "Routes nicht in Routing Table"

```bash
# WICHTIG: Tailscale nutzt Table 52, nicht Main Table!

# Prüfen:
ip route show table 52 | grep "192.168.0.0/24"

# Sollte zeigen:
# 192.168.0.0/24 dev tailscale0 table 52

# Falls leer - Tailscale neu starten:
sudo tailscale down
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes

# Dann nochmal checken:
ip route get 192.168.0.101
# Sollte zeigen: "dev tailscale0 table 52"
```

---

### Problem: "Exit-Node funktioniert nicht"

```bash
# Exit-Node aktiviert?
sudo headscale routes list | grep "0.0.0.0/0"

# Exit-Node nutzen (auf Client)
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# IP checken
curl ifconfig.me  # Sollte deine Heimnetz-IP sein
```

---

### Problem: "Headscale Service läuft nicht"

```bash
# Service Status
sudo systemctl status headscale

# Logs anschauen
sudo journalctl -u headscale -f

# Neustarten
sudo systemctl restart headscale

# Config testen
sudo headscale serve  # Manuell starten zum Debuggen
```

---

## 📝 Konfigurationsdateien

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

---

### Nginx Config (/etc/nginx/sites-available/headscale)

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

---

### DuckDNS Auto-Update (Cronjob)

```bash
# Crontab bearbeiten
crontab -e

# Alle 5 Minuten IP updaten
*/5 * * * * curl "https://www.duckdns.org/update?domains=zabooz&token=DEIN_TOKEN&ip="

# Oder als Script:
echo "curl -s 'https://www.duckdns.org/update?domains=zabooz&token=DEIN_TOKEN&ip=' > /dev/null" > /usr/local/bin/duckdns.sh
chmod +x /usr/local/bin/duckdns.sh

# Cronjob:
*/5 * * * * /usr/local/bin/duckdns.sh
```

---

## 🎓 Wichtige Konzepte für die LAP

### 1. VPN-Typen
- **Site-to-Site:** Verbindet zwei Netzwerke (z.B. Firmenstandorte)
- **Remote Access:** Einzelne Clients verbinden sich zu einem Netzwerk
- **Mesh VPN:** Alle Nodes können direkt miteinander kommunizieren (Tailscale/Headscale)

### 2. WireGuard vs. OpenVPN
| Feature | WireGuard | OpenVPN |
|---------|-----------|---------|
| **Geschwindigkeit** | Sehr schnell | Langsamer |
| **Code-Basis** | 4.000 Zeilen | 100.000+ Zeilen |
| **Konfiguration** | Einfach | Komplex |
| **Protokoll** | UDP | TCP/UDP |

### 3. NAT-Typen
- **SNAT (Source NAT):** Ändert Source-IP (ausgehend)
- **DNAT (Destination NAT):** Ändert Destination-IP (eingehend)
- **MASQUERADE:** Dynamisches SNAT (für DHCP-Adressen)

### 4. Wichtige Ports
- **443 (HTTPS):** Headscale Control Server & DERP
- **3478 (UDP):** STUN (NAT-Traversal)
- **8090:** Headscale intern (hinter Nginx)

---

## 📚 Nützliche Befehle - Spickzettel

### Headscale (VPS)

```bash
# User Management
headscale users create <name>
headscale users list
headscale users destroy <name>

# Node Management
headscale nodes list
headscale nodes register --user <user> --key <nodekey>
headscale nodes delete --identifier <id>
headscale nodes expire --identifier <id>

# Routes Management
headscale routes list
headscale routes enable --route <id>
headscale routes disable --route <id>

# Debug
headscale debug create-node --user <user> --name <name>
headscale serve  # Manuell starten
```

---

### Tailscale (Client)

```bash
# Verbindung
tailscale up --login-server=<url> --accept-routes
tailscale down
tailscale logout

# Status
tailscale status
tailscale netcheck  # Verbindungsqualität
tailscale ping <ip>

# Exit-Node
tailscale up --exit-node=<ip>
tailscale exit-node list

# Routes
tailscale status --json | jq '.Peer[].AllowedIPs'
ip route | grep tailscale

# Debug
tailscale debug --logs  # Logs anzeigen
```

---

### Netzwerk Debugging

```bash
# Routing
ip route                    # Main Table
ip route show table 52      # Tailscale Table
ip route show table all     # Alle Tables
ip route get <ip>           # Welche Route wird verwendet?

# IP Forwarding
sysctl net.ipv4.ip_forward
sysctl -w net.ipv4.ip_forward=1

# iptables
iptables -L -n -v
iptables -t nat -L -n -v
iptables -L FORWARD -n -v

# Traffic beobachten
tcpdump -i tailscale0 -n
tcpdump -i any host <ip>
tcpdump -i any icmp

# Connections
ss -tuln  # Listening Ports
netstat -rn  # Routing Table
ping <ip>
traceroute <ip>

# Tailscale-spezifisch
ip route show table all | grep tailscale  # Alle Tailscale-Routes
ip addr show tailscale0                   # Interface Details
```

---

## ✅ Aktuelle Setup-Konfiguration

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
