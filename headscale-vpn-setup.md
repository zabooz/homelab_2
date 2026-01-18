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

## 🌐 Subnetz-Routing & Exit-Nodes - Ausführliche Erklärung

### Was ist ein Subnetz (Subnet)?

Ein **Subnetz** ist ein logisch getrennter Bereich eines Netzwerks. In unserem Fall:

```
192.168.0.0/24 = Heimnetzwerk

Aufschlüsselung:
- 192.168.0.0      = Netzwerk-Adresse
- /24              = Subnet-Maske (255.255.255.0)
- 192.168.0.1-254  = Verfügbare Host-Adressen
- 192.168.0.255    = Broadcast-Adresse

Bedeutung von /24:
- 24 Bits sind für das Netzwerk reserviert
- 8 Bits bleiben für Hosts (2^8 - 2 = 254 nutzbare IPs)
```

**Warum ist das wichtig?**
- Dein Heimnetz (192.168.0.0/24) ist **physisch getrennt** vom Tailscale-Netz (100.64.0.0/10)
- Ohne Routing können Geräte im Tailscale-Netz **nicht** auf dein Heimnetz zugreifen
- Der **Subnet Router** (LXC Container) ist die Brücke zwischen beiden Netzen

---

### Wie funktioniert Subnet-Routing?

#### Szenario OHNE Subnet Router:

```
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
└─────────────┘                      └──────┬───────┘
                                            │
      ❌ KEIN Zugriff auf:                  │ Heimnetz
                                            │
                                     ┌──────▼───────┐
                                     │  Proxmox     │
                                     │ 192.168.0.101│
                                     └──────────────┘
```

**Problem:** Laptop kann nur mit anderen Tailscale-Nodes kommunizieren (100.64.0.x), aber **nicht** mit Geräten im Heimnetz (192.168.0.x)

---

#### Szenario MIT Subnet Router:

```
┌─────────────┐                      ┌──────────────┐
│   Laptop    │  Tailscale VPN       │ LXC Container│
│ 100.64.0.2  │ ◄──────────────────► │ 100.64.0.1   │
│             │                      │ 192.168.0.150│ ← Hat BEIDE IPs!
└─────────────┘                      └──────┬───────┘
                                            │
      ✅ Zugriff auf alles:                 │ IP Forwarding
         100.64.0.x                         │ aktiviert
         192.168.0.x                        │
                                     ┌──────▼───────┐
                                     │  Proxmox     │
                                     │ 192.168.0.101│
                                     │              │
                                     │ Debian VM    │
                                     │ 192.168.0.158│
                                     └──────────────┘
```

**Lösung:** Der Container fungiert als **Router** zwischen den Netzen!

---

### Schritt-für-Schritt: Was passiert beim Subnet-Routing?

#### 1. Advertised Routes (Ankündigen)

Wenn der LXC Container mit Tailscale startet:

```bash
tailscale up \
  --login-server=https://zabooz.duckdns.org \
  --advertise-routes=192.168.0.0/24 \    ← "Ich kann 192.168.0.0/24 erreichen!"
  --accept-routes                        ← "Ich akzeptiere auch Routes von anderen"
```

**Was passiert:**
- Container sagt Headscale: "Hey, ich habe Zugriff auf das Netz 192.168.0.0/24"
- Headscale speichert diese Info: "Container 100.64.0.1 kann zu 192.168.0.0/24 routen"
- Aber: Route ist noch **NICHT aktiv** (muss erst approved werden!)

#### 2. Approve Routes (Aktivieren)

Auf dem VPS:

```bash
sudo headscale nodes list-routes
```

Output zeigt:
```
ID | Hostname  | Approved | Available        | Serving
1  | tailscale |          | 192.168.0.0/24   |        
```

**Jetzt aktivieren:**

```bash
sudo headscale nodes approve-routes --identifier 1 --routes 192.168.0.0/24
```

**Was passiert:**
- Headscale sagt allen Nodes: "Wenn ihr zu 192.168.0.0/24 wollt, geht über 100.64.0.1"
- Alle Clients bekommen diese Routing-Info automatisch
- Route wird in den Routing-Tabellen der Clients eingetragen

#### 3. Client akzeptiert Routes

Auf dem Laptop:

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
#                                                            ^^^^^^^^^^^^^^^^
#                                                            Wichtig!
```

**Was passiert:**
- Laptop akzeptiert die Route: "Okay, für 192.168.0.0/24 nutze ich 100.64.0.1 als Gateway"
- Routing-Tabelle wird aktualisiert

**Routing-Tabelle auf dem Laptop** (vereinfacht):

```
Ziel               Gateway         Interface
100.64.0.0/10  →   direkt      →   tailscale0
192.168.0.0/24 →   100.64.0.1  →   tailscale0   ← Neue Route!
0.0.0.0/0      →   Router      →   eth0/wlan0
```

---

### Praktisches Beispiel: Laptop greift auf Proxmox zu

**Schritt-für-Schritt was passiert:**

```
1. Laptop (100.64.0.2) will Proxmox (192.168.0.101) erreichen
   ↓
2. Laptop checkt Routing-Tabelle:
   "192.168.0.101 gehört zu 192.168.0.0/24"
   "Route sagt: Gateway ist 100.64.0.1"
   ↓
3. Laptop schickt Paket über Tailscale zu 100.64.0.1
   [Verschlüsselt mit WireGuard]
   ↓
4. Container (100.64.0.1) empfängt Paket
   "Ziel ist 192.168.0.101"
   "Das ist in meinem lokalen Netz!"
   ↓
5. Container leitet Paket weiter (IP Forwarding)
   [Über eth0 Interface: 192.168.0.150]
   ↓
6. Proxmox (192.168.0.101) empfängt Paket
   "Quelle ist 192.168.0.150" (Container IP)
   ↓
7. Proxmox antwortet zurück an 192.168.0.150
   ↓
8. Container leitet Antwort zurück über Tailscale
   ↓
9. Laptop empfängt Antwort
   ✅ Verbindung hergestellt!
```

---

### IP Forwarding - Was ist das?

**IP Forwarding** ist die Fähigkeit eines Geräts, Pakete zwischen verschiedenen Netzwerk-Interfaces weiterzuleiten.

#### Ohne IP Forwarding:
```
Paket kommt rein (tailscale0) → Container → ❌ VERWORFEN
```

#### Mit IP Forwarding:
```
Paket kommt rein (tailscale0) → Container → ✅ Weitergeleitet → eth0 → Heimnetz
```

**Aktiviert im Container:**

```bash
# Temporär aktivieren
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# Permanent aktivieren
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

**Checken ob aktiv:**
```bash
sysctl net.ipv4.ip_forward
# Sollte zeigen: net.ipv4.ip_forward = 1
```

---

### Exit-Node - Internet über Heimnetz

Ein **Exit-Node** routet **ALLEN** Internet-Traffic über sich selbst.

#### Normale Verbindung (ohne Exit-Node):

```
Laptop → eigenes Internet → Ziel-Website
         (z.B. Café WiFi)
```

#### Mit Exit-Node:

```
Laptop → Tailscale VPN → Exit-Node (100.64.0.1) → Heimnetz Internet → Ziel-Website
         (verschlüsselt)
```

**Warum ist das nützlich?**

1. **Sicherheit:** In unsicheren Netzen (Café, Hotel) ist Traffic verschlüsselt bis zum Heimnetz
2. **Geo-Location:** Websites sehen deine Heim-IP statt Café-IP
3. **Zugriff:** Du nutzt die Internet-Verbindung deines Heimnetzes

---

### Exit-Node Routen erklärt

Wenn der Container als Exit-Node advertised:

```bash
tailscale up \
  --advertise-exit-node \              ← "Ich kann ALLE Internet-Pakete routen!"
  --advertise-routes=192.168.0.0/24    ← "Und auch das lokale Netz!"
```

**Das bedeutet in der Routing-Tabelle:**

```
0.0.0.0/0       → "Default Route" → ALLES Internet
::/0            → "Default Route IPv6" → ALLES Internet (IPv6)
192.168.0.0/24  → Spezifisches lokales Netz
```

**Auf dem VPS aktivieren:**

```bash
sudo headscale nodes approve-routes --identifier 1 --routes 0.0.0.0/0,::/0,192.168.0.0/24
```

**Auf dem Laptop nutzen:**

```bash
# Exit-Node aktivieren
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# IP checken (zeigt jetzt Heim-IP!)
curl ifconfig.me

# Exit-Node deaktivieren
sudo tailscale up --accept-routes
```

---

### Routing-Tabelle verstehen

**Routing-Tabelle auf dem Laptop anzeigen:**

```bash
ip route show

# Oder detaillierter:
ip route show table all
```

**Beispiel-Output MIT aktiviertem Subnet Router:**

```
default via 192.168.1.1 dev wlan0          ← Standard Internet über WLAN
100.64.0.0/10 dev tailscale0               ← Tailscale-Netz
192.168.0.0/24 via 100.64.0.1 dev tailscale0  ← Heimnetz über Container!
```

**Was bedeutet das:**

| Ziel | Via | Interface | Bedeutung |
|------|-----|-----------|-----------|
| `default` | 192.168.1.1 | wlan0 | Internet geht normal über WLAN |
| `100.64.0.0/10` | direkt | tailscale0 | Tailscale-IPs direkt über VPN |
| `192.168.0.0/24` | 100.64.0.1 | tailscale0 | Heimnetz über Container |

---

### NAT (Network Address Translation) im Container

Der Container muss auch **NAT** machen, damit die Antworten zurückkommen.

**Problem ohne NAT:**

```
Laptop (100.64.0.2) → Container → Proxmox (192.168.0.101)
                                  ↓
                                  "Wer ist 100.64.0.2?"
                                  "Kenne ich nicht!"
                                  ❌ Paket verworfen
```

**Lösung mit NAT (Masquerading):**

```
Laptop (100.64.0.2) → Container → NAT → Proxmox (192.168.0.101)
                                         ↓
                      Source wird zu 192.168.0.150 (Container IP)
                                         ↓
                      "Ah, 192.168.0.150 kenne ich!"
                                         ↓
                      Antwort → Container → NAT → Laptop
                                ✅ Funktioniert!
```

**NAT wird automatisch von iptables/nftables gemacht** wenn IP Forwarding aktiv ist.

---

### Zusammenfassung: Route Types

| Route Type | CIDR | Was es macht | Beispiel |
|------------|------|--------------|----------|
| **Specific Subnet** | 192.168.0.0/24 | Zugriff auf bestimmtes Netz | Heimnetz |
| **Default IPv4** | 0.0.0.0/0 | ALLES Internet (IPv4) | Exit-Node |
| **Default IPv6** | ::/0 | ALLES Internet (IPv6) | Exit-Node |
| **Single Host** | 192.168.0.101/32 | Nur EIN Gerät | Nur Proxmox |

---

### Visual: Packet Flow beim Subnet-Routing

```
┌──────────────────────────────────────────────────────────────────┐
│                     LAPTOP (100.64.0.2)                          │
│                                                                   │
│  Anwendung: "ping 192.168.0.101"                                │
│       ↓                                                           │
│  Routing-Tabelle checken:                                        │
│  "192.168.0.101 → via 100.64.0.1 (tailscale0)"                  │
│       ↓                                                           │
│  Tailscale Client: Paket verschlüsseln (WireGuard)              │
│       ↓                                                           │
│  Netzwerk: Sende zu 100.64.0.1                                   │
└──────────────┬────────────────────────────────────────────────────┘
               │
               │ [verschlüsseltes Paket über Internet/VPN]
               │
┌──────────────▼────────────────────────────────────────────────────┐
│              LXC CONTAINER (100.64.0.1 / 192.168.0.150)          │
│                                                                   │
│  Tailscale empfängt: Paket entschlüsseln                         │
│       ↓                                                           │
│  Kernel: IP Forwarding aktiv?                                    │
│       ↓ JA                                                        │
│  Routing: Ziel 192.168.0.101 ist im lokalen Netz                │
│       ↓                                                           │
│  NAT/Masquerading: Source = 192.168.0.150                        │
│       ↓                                                           │
│  eth0: Sende Paket ins Heimnetz                                  │
└──────────────┬────────────────────────────────────────────────────┘
               │
               │ [Paket im Heimnetz]
               │
┌──────────────▼────────────────────────────────────────────────────┐
│                 PROXMOX (192.168.0.101)                          │
│                                                                   │
│  Empfängt Paket von 192.168.0.150                                │
│       ↓                                                           │
│  Antwortet zurück an 192.168.0.150                               │
└──────────────┬────────────────────────────────────────────────────┘
               │
               │ [Antwort zurück zum Container]
               │
┌──────────────▼────────────────────────────────────────────────────┐
│              LXC CONTAINER                                        │
│                                                                   │
│  eth0 empfängt Antwort                                           │
│       ↓                                                           │
│  NAT/Conntrack: "Gehört zu Session mit 100.64.0.2"              │
│       ↓                                                           │
│  Tailscale: Verschlüsseln und zurück senden                      │
└──────────────┬────────────────────────────────────────────────────┘
               │
               │ [verschlüsselt zurück]
               │
┌──────────────▼────────────────────────────────────────────────────┐
│                     LAPTOP (100.64.0.2)                          │
│                                                                   │
│  Tailscale: Entschlüsseln                                        │
│       ↓                                                           │
│  Anwendung: "64 bytes from 192.168.0.101: icmp_seq=1"           │
│       ↓                                                           │
│  ✅ ERFOLG!                                                       │
└───────────────────────────────────────────────────────────────────┘
```

---

### Praktische Tests zum Verstehen

#### Test 1: Route ist da, aber nicht approved

```bash
# Container advertised Route, aber VPS hat sie nicht approved
tailscale status  # auf Laptop

# Zeigt:
# 100.64.0.1  tailscale  ← Container ist da
# Aber KEINE Route zu 192.168.0.0/24!

ping 192.168.0.101
# → Timeout / Network unreachable
```

#### Test 2: Route approved, aber Client akzeptiert sie nicht

```bash
# VPS hat Route approved, aber Laptop mit --accept-routes vergessen
sudo tailscale up --login-server=https://zabooz.duckdns.org
#                                    (ohne --accept-routes!)

ping 192.168.0.101
# → Timeout / Network unreachable
```

#### Test 3: Alles richtig konfiguriert

```bash
# Auf VPS: Route approved
sudo headscale nodes approve-routes --identifier 1 --routes 192.168.0.0/24

# Auf Laptop: --accept-routes gesetzt
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes

# Test:
ping 192.168.0.101
# ✅ 64 bytes from 192.168.0.101: icmp_seq=1 ttl=63 time=2.55 ms
```

---

### Debugging: Route funktioniert nicht

#### Schritt 1: Ist Route advertised?

```bash
# Auf VPS
sudo headscale nodes list-routes
```

Sollte zeigen:
```
ID | Hostname  | Available        
1  | tailscale | 192.168.0.0/24  ← Route ist advertised
```

Falls NICHT → Im Container nochmal `tailscale up` mit `--advertise-routes`

#### Schritt 2: Ist Route approved?

```bash
sudo headscale nodes list-routes
```

Sollte zeigen:
```
ID | Hostname  | Approved         | Serving
1  | tailscale | 192.168.0.0/24   | 192.168.0.0/24  ← Route ist aktiv
```

Falls NICHT → Route approven

#### Schritt 3: Akzeptiert Client die Route?

```bash
# Auf Laptop
tailscale status
```

Sollte zeigen:
```
100.64.0.1  tailscale  zabooz  linux  active; offers exit node
```

Und in Routing-Tabelle:
```bash
ip route | grep 192.168.0
# Sollte zeigen:
# 192.168.0.0/24 via 100.64.0.1 dev tailscale0
```

Falls NICHT → `tailscale up --accept-routes`

#### Schritt 4: IP Forwarding im Container?

```bash
# Im Container
sysctl net.ipv4.ip_forward
# MUSS zeigen: net.ipv4.ip_forward = 1
```

Falls 0 → IP Forwarding aktivieren

---

### Wichtige Konzepte nochmal zusammengefasst

1. **Advertise Route** = "Ich KANN zu diesem Netz routen"
2. **Approve Route** = "Okay, andere dürfen dich als Gateway nutzen"
3. **Accept Routes** = "Ich WILL diese Routes in meiner Routing-Tabelle"
4. **IP Forwarding** = "Ich DARF Pakete zwischen Interfaces weiterleiten"
5. **NAT/Masquerading** = "Ich ändere Source-IP damit Antworten zurückkommen"

**Alle 5 müssen zusammenspielen, sonst funktioniert Subnet-Routing nicht!**

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
