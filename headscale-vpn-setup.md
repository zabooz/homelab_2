# 🚀 Headscale VPN Setup - Komplette Dokumentation

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
sudo headscale users create freunde

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

# Exit-Node
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

---

## 🌍 Proxmox Web-UI von überall erreichen

### Die wichtigste Frage: Brauche ich Tailscale auf Proxmox?

**NEIN! Der LXC Container reicht völlig aus!** ✅

---

### Wie es funktioniert (auch wenn Proxmox KEIN Tailscale hat):

```
┌─────────────────────────────────────────────────────────────┐
│  Café WiFi (fremdes Netz)                                   │
│                                                              │
│  ┌────────────────┐                                         │
│  │  Dein Laptop   │  Browser öffnet:                        │
│  │  Café IP: ???  │  https://192.168.0.101:8006            │
│  │  Tailscale:    │                                         │
│  │  100.64.0.2    │                                         │
│  └────────┬───────┘                                         │
└───────────┼─────────────────────────────────────────────────┘
            │
            │ Routing-Tabelle: "192.168.0.101 → via 100.64.0.1"
            │ Paket wird verschlüsselt (WireGuard)
            │
            ▼
   ╔════════════════════════════════════════╗
   ║         INTERNET (verschlüsselt)       ║
   ╚════════════════════════════════════════╝
            │
            ▼
┌───────────┼─────────────────────────────────────────────────┐
│  Heimnetz │                                                  │
│           │                                                  │
│  ┌────────▼────────┐                                        │
│  │ LXC Container   │  Empfängt verschlüsseltes Paket       │
│  │ Tailscale:      │  Entschlüsselt                         │
│  │ 100.64.0.1      │  "Ah, Ziel ist 192.168.0.101"         │
│  │ Heimnetz:       │  IP Forwarding leitet weiter           │
│  │ 192.168.0.150   │                                        │
│  └────────┬────────┘                                        │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │ Proxmox Host    │  Empfängt Anfrage von 192.168.0.150   │
│  │ 192.168.0.101   │  (Proxmox denkt: lokaler Zugriff!)    │
│  │ Port 8006       │  Sendet Web-UI zurück                 │
│  └────────┬────────┘                                        │
│           │                                                  │
│           │ Antwort zurück zum Container                    │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │ LXC Container   │  Verschlüsselt Antwort                │
│  │ 100.64.0.1      │  Sendet zurück an Laptop              │
│  └────────┬────────┘                                        │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼
   ╔════════════════════════════════════════╗
   ║         INTERNET (verschlüsselt)       ║
   ╚════════════════════════════════════════╝
            │
            ▼
┌───────────┼─────────────────────────────────────────────────┐
│  Café     │                                                  │
│           │                                                  │
│  ┌────────▼───────┐                                         │
│  │  Dein Laptop   │  Browser zeigt Proxmox Web-UI! ✅      │
│  │  100.64.0.2    │                                         │
│  └────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

---

### Praktische Anleitung: Proxmox Browser-Zugriff

#### Schritt 1: Tailscale Status checken

```bash
# Auf deinem Laptop (überall)
tailscale status
```

**Erwartetes Output:**
```
100.64.0.2  maschinchen  zabooz  linux  -
100.64.0.1  tailscale    zabooz  linux  active; offers exit node
```

✅ Beide Nodes müssen **online** sein!

---

#### Schritt 2: Routing checken

```bash
# Route zum Heimnetz vo

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

## 🗺️ ROUTING ERKLÄRT - Von überall auf Proxmox zugreifen

### Die Ausgangssituation

Du sitzt in einem **Café** (fremdes Netz) und willst auf **Proxmox** (Heimnetz) zugreifen.

```
    🏢 CAFÉ                              🏠 ZUHAUSE
┌──────────────┐                    ┌──────────────┐
│  Café WiFi   │                    │  Heimnetz    │
│  10.0.5.0/24 │                    │192.168.0.0/24│
│              │                    │              │
│ ┌──────────┐ │                    │ ┌──────────┐ │
│ │  Laptop  │ │                    │ │ Proxmox  │ │
│ │10.0.5.42 │ │                    │ │  .101    │ │
│ └──────────┘ │                    │ └──────────┘ │
└──────────────┘                    └──────────────┘
       │                                    │
       │                                    │
       ❌ KEIN direkter Weg! ❌            │
       │         Zwei getrennte Netze      │
       └───────────────────────────────────┘
```

**Problem:** Café-Netz und Heimnetz sind **komplett getrennt**!

---

### Lösung: Tailscale Mesh-Netzwerk als Brücke

Tailscale erstellt ein **virtuelles privates Netzwerk (VPN)**, das beide Netze verbindet:

```
    🏢 CAFÉ                                              🏠 ZUHAUSE
┌──────────────┐                                    ┌──────────────┐
│  Café WiFi   │                                    │  Heimnetz    │
│  10.0.5.0/24 │                                    │192.168.0.0/24│
│              │                                    │              │
│ ┌──────────┐ │         ╔═══════════════════╗    │ ┌──────────┐ │
│ │  Laptop  │ │         ║  TAILSCALE MESH   ║    │ │LXC Cont. │ │
│ │10.0.5.42 │◄──────────║   (verschlüsselt) ║────────►│100.64.0.1│ │
│ │          │ │  über   ║   100.64.0.0/10   ║    │ │192.168.  │ │
│ │100.64.0.2│ │Internet ║                   ║    │ │  0.150   │ │
│ └──────────┘ │         ╚═══════════════════╝    │ └────┬─────┘ │
└──────────────┘                                    │      │       │
                                                    │      │       │
                       ✅ Tailscale verbindet! ✅  │ ┌────▼─────┐ │
                                                    │ │ Proxmox  │ │
                                                    │ │  .101    │ │
                                                    │ └──────────┘ │
                                                    └──────────────┘
```

**Lösung:** Tailscale baut einen verschlüsselten Tunnel zwischen Laptop und Container!

---

### Was ist Routing?

**Routing** = Der Weg den ein Datenpaket nimmt, um von A nach B zu kommen.

**Routing-Tabelle** = Eine Liste die sagt: "Für Ziel X, geh über Gateway Y"

#### Routing-Tabelle auf deinem Laptop (MIT Tailscale):

```
┌────────────────────────────────────────────────────────────┐
│                  ROUTING-TABELLE (Laptop)                   │
├──────────────────┬──────────────┬──────────────┬───────────┤
│ Ziel-Netzwerk    │ Via Gateway  │ Interface    │ Bedeutung │
├──────────────────┼──────────────┼──────────────┼───────────┤
│ 0.0.0.0/0        │ 10.0.5.1     │ wlan0        │ Internet  │
│ (Standard)       │ (Café Router)│              │ normal    │
├──────────────────┼──────────────┼──────────────┼───────────┤
│ 100.64.0.0/10    │ direkt       │ tailscale0   │ Tailscale │
│                  │              │              │ Netz      │
├──────────────────┼──────────────┼──────────────┼───────────┤
│ 192.168.0.0/24   │ 100.64.0.1   │ tailscale0   │ Heimnetz  │
│ ← WICHTIG!       │ (Container)  │              │ über VPN! │
└──────────────────┴──────────────┴──────────────┴───────────┘
```

**Die letzte Zeile ist der Trick!**

Wenn du `https://192.168.0.101:8006` aufrufst:
- Laptop checkt: "192.168.0.101 gehört zu 192.168.0.0/24"
- Routing-Tabelle sagt: "Schick's über Gateway 100.64.0.1 (Container)"
- Paket geht über Tailscale zum Container!

---

### Schritt-für-Schritt: Browser öffnet Proxmox

#### SCHRITT 1: Du tippst im Browser

```
https://192.168.0.101:8006
```

**Was passiert auf dem Laptop:**

```
┌─────────────────────────────────────────┐
│          LAPTOP (im Café)               │
│                                         │
│  Browser: "https://192.168.0.101:8006" │
│     ↓                                   │
│  Betriebssystem: "An wen soll ich das  │
│                   Paket schicken?"      │
│     ↓                                   │
│  Routing-Tabelle checken:               │
│  "192.168.0.101 ist in 192.168.0.0/24"  │
│  "Route sagt: via 100.64.0.1"          │
│     ↓                                   │
│  Tailscale Interface: Paket annehmen    │
└─────────────────────────────────────────┘
```

---

#### SCHRITT 2: Paket wird verschlüsselt und gesendet

```
┌─────────────────────────────────────────┐
│          LAPTOP                         │
│                                         │
│  Tailscale Client:                      │
│  ┌─────────────────────────────┐       │
│  │ Original Paket:             │       │
│  │ Von: 100.64.0.2             │       │
│  │ An:  192.168.0.101          │       │
│  │ Port: 8006                  │       │
│  │ Inhalt: HTTPS Request       │       │
│  └─────────────────────────────┘       │
│           ↓                             │
│  WireGuard Verschlüsselung:             │
│  🔒 VERSCHLÜSSELN 🔒                   │
│           ↓                             │
│  ┌─────────────────────────────┐       │
│  │ Verschlüsseltes Paket       │       │
│  │ (niemand kann mitlesen!)    │       │
│  └─────────────────────────────┘       │
│           ↓                             │
│  Sende über Internet an VPS             │
│  (Headscale Server koordiniert)         │
└─────────────────────────────────────────┘
```

**Internet-Weg:**

```
Laptop (Café) → WiFi → Internet → VPS → Internet → Heimnetz → Container
   10.0.5.42          [verschlüsselt]                       100.64.0.1
```

---

#### SCHRITT 3: Container empfängt und routet

```
┌─────────────────────────────────────────────────────┐
│     LXC CONTAINER (im Heimnetz)                     │
│     Hat ZWEI Netzwerk-Interfaces!                   │
│                                                     │
│  ┌─────────────────┐      ┌─────────────────┐     │
│  │  tailscale0     │      │     eth0        │     │
│  │  100.64.0.1     │      │  192.168.0.150  │     │
│  │ (VPN Interface) │      │ (Heimnetz)      │     │
│  └────────┬────────┘      └────────┬────────┘     │
│           │                        │              │
│           │  Paket kommt rein      │              │
│           ▼                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 1. Empfange verschlüsseltes  │ │              │
│  │    Paket auf tailscale0      │ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 2. WireGuard entschlüsselt:  │ │              │
│  │    Von: 100.64.0.2           │ │              │
│  │    An:  192.168.0.101:8006   │ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 3. Kernel prüft:             │ │              │
│  │    "Ziel ist nicht für mich" │ │              │
│  │    "192.168.0.101 ≠ meine IP"│ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 4. IP Forwarding aktiv?      │ │              │
│  │    ✅ JA! (sysctl = 1)       │ │              │
│  │    → Ich darf weiterleiten!  │ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 5. Routing-Tabelle checken:  │ │              │
│  │    192.168.0.101 ist direkt  │ │              │
│  │    erreichbar über eth0!     │ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        │              │
│  ┌──────────────────────────────┐ │              │
│  │ 6. NAT/Masquerading:         │ │              │
│  │    Source IP ändern:         │ │              │
│  │    100.64.0.2 → 192.168.0.150│ │              │
│  │    (sonst kennt Proxmox die  │ │              │
│  │     Source-IP nicht!)        │ │              │
│  └──────────────────────────────┘ │              │
│           ↓                        ▼              │
│  Paket raus über eth0 ──────────►  senden!       │
└─────────────────────────────────────────────────┘
```

**WICHTIG:** Der Container ändert die **Source-IP**!

**Vorher:**
```
Von: 100.64.0.2 (Laptop Tailscale IP)
An:  192.168.0.101 (Proxmox)
```

**Nachher (nach NAT):**
```
Von: 192.168.0.150 (Container Heimnetz IP)
An:  192.168.0.101 (Proxmox)
```

**Warum?** Proxmox kennt `100.64.0.2` nicht, würde das Paket verwerfen!

---

#### SCHRITT 4: Proxmox empfängt und antwortet

```
┌─────────────────────────────────────────┐
│        PROXMOX (192.168.0.101)          │
│                                         │
│  Netzwerk-Interface empfängt Paket:     │
│  ┌─────────────────────────────┐       │
│  │ Von: 192.168.0.150          │       │
│  │ An:  192.168.0.101:8006     │       │
│  │ Inhalt: HTTPS Request       │       │
│  └─────────────────────────────┘       │
│           ↓                             │
│  Proxmox denkt:                         │
│  "Ah, Anfrage aus dem lokalen Netz!"   │
│  "192.168.0.150 ist der Container"     │
│           ↓                             │
│  Web-Server (Port 8006) antwortet:      │
│  ┌─────────────────────────────┐       │
│  │ Von: 192.168.0.101:8006     │       │
│  │ An:  192.168.0.150          │       │
│  │ Inhalt: Proxmox Login-Seite │       │
│  └─────────────────────────────┘       │
│           ↓                             │
│  Sende zurück an 192.168.0.150          │
└─────────────────────────────────────────┘
```

**Proxmox hat KEINE Ahnung dass du im Café sitzt!**

Aus Sicht von Proxmox: "Normaler Zugriff vom Container im lokalen Netz"

---

#### SCHRITT 5: Antwort zurück zum Laptop

```
┌─────────────────────────────────────────────────────┐
│     LXC CONTAINER                                   │
│                                                     │
│  eth0 empfängt Antwort von Proxmox:                 │
│  ┌─────────────────────────────┐                   │
│  │ Von: 192.168.0.101:8006     │                   │
│  │ An:  192.168.0.150          │                   │
│  └─────────────────────────────┘                   │
│           ↓                                         │
│  NAT/Connection Tracking:                           │
│  "Diese Antwort gehört zu einer Session!"          │
│  "Original-Anfrage kam von 100.64.0.2"             │
│           ↓                                         │
│  Source/Destination tauschen und ändern:            │
│  ┌─────────────────────────────┐                   │
│  │ Von: 192.168.0.101:8006     │                   │
│  │ An:  100.64.0.2             │ ← Zurück zu Laptop│
│  └─────────────────────────────┘                   │
│           ↓                                         │
│  WireGuard verschlüsselt Antwort:                   │
│  🔒 VERSCHLÜSSELN 🔒                               │
│           ↓                                         │
│  Sende über Tailscale zurück                        │
└─────────────────────────────────────────────────────┘
         │
         │ [verschlüsselt über Internet]
         ▼
┌─────────────────────────────────────────┐
│          LAPTOP (im Café)               │
│                                         │
│  Tailscale empfängt verschlüsselte      │
│  Antwort und entschlüsselt:             │
│           ↓                             │
│  Browser empfängt Proxmox Login-Seite   │
│           ↓                             │
│  🎉 ERFOLG! Proxmox Web-UI lädt! 🎉    │
└─────────────────────────────────────────┘
```

---

### Die Routing-Magie im Detail

#### Warum funktioniert das?

**1. Tailscale Mesh-Netzwerk:**
- Erstellt virtuelles Netzwerk (100.64.0.0/10)
- Alle Nodes können sich direkt erreichen
- Verschlüsselt mit WireGuard

**2. Subnet Router (LXC Container):**
- Hat Zugriff auf BEIDE Netze:
  - Tailscale: 100.64.0.1
  - Heimnetz: 192.168.0.150
- Kann Pakete zwischen Netzen weiterleiten (IP Forwarding)

**3. Routing-Tabellen:**
- Laptop weiß: "Für Heimnetz geh über Container"
- Container weiß: "Heimnetz ist über eth0 erreichbar"

**4. NAT (Network Address Translation):**
- Container ändert Source-IP
- Proxmox sieht nur Container-IP (192.168.0.150)
- Antworten kommen zum Container zurück
- Container leitet sie zurück zum Laptop

---

### Visual: Zwei-Wege-Kommunikation

#### Hinweg (Laptop → Proxmox):

```
LAPTOP                  CONTAINER               PROXMOX
100.64.0.2              100.64.0.1              192.168.0.101
   │                    192.168.0.150              │
   │                         │                     │
   │  1. Browser Request     │                     │
   │  An: 192.168.0.101:8006 │                     │
   ├──────────────────────►  │                     │
   │  über Tailscale VPN     │                     │
   │  (verschlüsselt)        │                     │
   │                         │  2. Forwarding      │
   │                         │  Source: 192.168.0.150
   │                         ├──────────────────► │
   │                         │  über Heimnetz      │
   │                         │  (unverschlüsselt)  │
```

#### Rückweg (Proxmox → Laptop):

```
PROXMOX             CONTAINER               LAPTOP
192.168.0.101       100.64.0.1              100.64.0.2
   │                192.168.0.150              │
   │                     │                     │
   │  3. Web-UI Antwort  │                     │
   │  An: 192.168.0.150  │                     │
   ├─────────────────►   │                     │
   │  über Heimnetz      │                     │
   │                     │  4. NAT + Routing   │
   │                     │  An: 100.64.0.2     │
   │                     ├──────────────────►  │
   │                     │  über Tailscale VPN │
   │                     │  (verschlüsselt)    │
```

---

### Connection Tracking (Conntrack)

**Wie weiß der Container welche Antwort zu welchem Laptop gehört?**

Der Linux Kernel führt eine **Connection Tracking Table**:

```
┌───────────────────────────────────────────────────────┐
│         CONNECTION TRACKING TABLE (Container)          │
├────────────┬──────────────┬─────────────┬─────────────┤
│ Original   │ Nach NAT     │ Antwort     │ Status      │
├────────────┼──────────────┼─────────────┼─────────────┤
│ 100.64.0.2 │ 192.168.0.150│ 192.168.0   │ ESTABLISHED │
│ :45123     │ :45123       │ .101:8006   │             │
│ →          │ →            │ →           │             │
│ 192.168.0  │ 192.168.0    │ 192.168.0   │             │
│ .101:8006  │ .101:8006    │ .150:45123  │             │
└────────────┴──────────────┴─────────────┴─────────────┘
```

**Was bedeutet das:**
1. Laptop (100.64.0.2:45123) schickt Anfrage
2. Container ändert zu (192.168.0.150:45123) via NAT
3. Proxmox antwortet an (192.168.0.150:45123)
4. Container sieht in Tabelle: "Gehört zu 100.64.0.2!"
5. Leitet zurück an Laptop

---

### Praktisches Beispiel: Traceroute

**Von deinem Laptop aus (im Café):**

```bash
traceroute 192.168.0.101
```

**Output:**

```
traceroute to 192.168.0.101 (192.168.0.101), 30 hops max
 1  100.64.0.1 (100.64.0.1)  25.432 ms  ← Container (über Tailscale VPN)
 2  192.168.0.101 (192.168.0.101)  27.891 ms  ← Proxmox (im Heimnetz)
```

**Was bedeutet das:**
- Hop 1: Paket geht zuerst zum Container (über VPN, deshalb ~25ms)
- Hop 2: Container leitet zum Proxmox (lokales Netz, +2ms)

**Ohne Subnet Router würde Hop 2 nie erreicht werden!**

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

---

## 🌐 STUN, DERP & NAT-Traversal - Wie Tailscale durch Firewalls kommt

### Das NAT-Problem verstehen

**Was ist NAT (Network Address Translation)?**

NAT ist eine Technik die fast jeder Router verwendet, um mehrere Geräte mit **einer** öffentlichen IP-Adresse ins Internet zu bringen.

#### Ohne NAT (theoretisch):

```
┌─────────────┐
│  Router     │  Öffentliche IP: 84.115.223.57
│             │
│  ┌────────┐ │  Öffentliche IP: ???
│  │ Laptop │ │  (braucht eigene öffentliche IP!)
│  └────────┘ │
│             │
│  ┌────────┐ │  Öffentliche IP: ???
│  │ Handy  │ │  (braucht auch eigene!)
│  └────────┘ │
└─────────────┘

Problem: Nicht genug IPv4-Adressen für alle Geräte!
```

#### Mit NAT (Realität):

```
┌─────────────────────────────────────┐
│  Router                              │
│  Öffentliche IP: 84.115.223.57      │
│                                      │
│  ┌────────┐  Private IP: 192.168.0.2│
│  │ Laptop │◄─┐                       │
│  └────────┘  │                       │
│              │  NAT-Tabelle          │
│  ┌────────┐  │  übersetzt            │
│  │ Handy  │◄─┘                       │
│  └────────┘  Private IP: 192.168.0.3│
└─────────────────────────────────────┘

✅ Alle Geräte teilen sich EINE öffentliche IP!
```

**NAT ist super für ausgehende Verbindungen** (du surfst, streamst, etc.)

**ABER:** NAT macht **eingehende Verbindungen** schwierig! 🚧

---

### NAT-Typen und das Verbindungsproblem

Es gibt verschiedene NAT-Typen, die unterschiedlich restriktiv sind:

#### 1. Full Cone NAT (am offensten)

```
┌──────────────────────────────────────────┐
│  Router (Full Cone NAT)                  │
│                                           │
│  Regel: Port 45123 → Laptop              │
│  JEDER von außen darf auf 45123 senden!  │
└──────────────────────────────────────────┘
         ↑
         │ ✅ Direktverbindung möglich!
         │
    [Internet]
```

#### 2. Symmetric NAT (am restriktivsten)

```
┌──────────────────────────────────────────┐
│  Router (Symmetric NAT)                  │
│                                           │
│  Regel: NUR wenn Laptop ZUERST           │
│         an Ziel X gesendet hat,          │
│         darf X zurück senden!            │
│                                           │
│  Andere Geräte → ❌ BLOCKIERT            │
└──────────────────────────────────────────┘
         ↑
         │ ❌ Direktverbindung oft NICHT möglich
         │
    [Internet]
```

---

### Das Peer-to-Peer Problem

**Szenario:** Zwei Tailscale-Nodes wollen sich direkt verbinden

```
    🏢 CAFÉ                                    🏠 HEIMNETZ
┌──────────────┐                          ┌──────────────┐
│  Router NAT  │                          │  Router NAT  │
│  Öff: ???    │                          │  Öff: ???    │
│              │                          │              │
│  Laptop      │                          │  Container   │
│  192.168.1.2 │                          │  192.168.0.150│
└──────────────┘                          └──────────────┘
       │                                         │
       │  Frage: "Wie können wir uns direkt     │
       │          verbinden?"                    │
       │                                         │
       │  Problem: Beide hinter NAT!            │
       │          Kennen jeweils nur            │
       │          ihre PRIVATE IP!              │
       └─────────────────────────────────────────┘
```

**Lösung:** STUN + DERP! 🎯

---

## 🔍 STUN - Session Traversal Utilities for NAT

### Was ist STUN?

**STUN** ist ein Protokoll das einem Gerät sagt:
1. "Deine **öffentliche IP-Adresse** ist X"
2. "Dein Router nutzt **Port Y** für dich"
3. "Dein NAT-Typ ist Z"

### Wie funktioniert STUN?

```
┌─────────────────────────────────────────────────────────────┐
│                    STUN-Server (VPS)                         │
│                    152.53.111.11:3478                        │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    1. "Wer bin ich?"  2. "Du bist:"     3. Speichern
         │                  │                  │
┌────────▼──────────────────▼──────────────────▼─────────┐
│     Laptop (hinter NAT)                                 │
│                                                          │
│     Private IP: 192.168.1.2                             │
│     Öffentliche IP: ??? (weiß ich nicht!)               │
│                                                          │
│     STUN Request senden →                               │
│                    ← STUN Response:                      │
│                      "Deine öffentliche IP: 84.115.x.x" │
│                      "Dein NAT-Port: 45123"            │
│                      "NAT-Typ: Port-Restricted"         │
│                                                          │
│     ✅ Jetzt weiß ich meine öffentliche Adresse!       │
└─────────────────────────────────────────────────────────┘
```

### STUN Schritt-für-Schritt

**1. Laptop sendet STUN-Request:**

```
Von: 192.168.1.2:12345 (private IP, privater Port)
An:  152.53.111.11:3478 (STUN-Server)
Inhalt: "Wer bin ich?"
```

**2. Router macht NAT:**

```
Router sieht: Ausgehende Verbindung von Laptop
Router ändert:
  Von: 192.168.1.2:12345
  Zu:  84.115.223.57:45123 (öffentliche IP + neuer Port)

NAT-Tabelle:
┌────────────────┬─────────────────┐
│ Intern         │ Extern          │
├────────────────┼─────────────────┤
│ 192.168.1.2    │ 84.115.223.57   │
│ :12345         │ :45123          │
└────────────────┴─────────────────┘
```

**3. STUN-Server sieht:**

```
Paket kam an von: 84.115.223.57:45123
(Das ist die öffentliche Adresse des Routers!)
```

**4. STUN-Server antwortet:**

```
An: 84.115.223.57:45123
Inhalt: "Du bist 84.115.223.57:45123"
```

**5. Laptop empfängt Antwort:**

```
✅ "Aha! Meine öffentliche Adresse ist 84.115.223.57:45123"
✅ "Ich teile das mit Headscale"
✅ "Andere Nodes können mich unter dieser Adresse erreichen!"
```

---

### STUN in Aktion - Logs

Erinnerst du dich an das hier vom Container?

```
2026/01/18 20:27:45 portmap: monitor: gateway and self IP changed: gw=192.168.0.1 self=192.168.0.150
2026/01/18 20:27:45 portmap: UPnP discovery response from 192.168.0.17, but gateway IP is 192.168.0.1
```

Das ist **STUN in Aktion**! Der Container:
1. Findet Gateway (Router): 192.168.0.1
2. Macht STUN-Request an Headscale STUN-Server (Port 3478)
3. Erfährt seine öffentliche IP
4. Teilt das mit Headscale

---

## 🚀 DERP - Designated Encrypted Relay for Packets

### Was ist DERP?

**DERP** ist ein **Fallback-Relay-Server** wenn direkte Verbindungen nicht möglich sind.

**Wann braucht man DERP?**

1. **Symmetric NAT** auf beiden Seiten → Direktverbindung unmöglich
2. **Firewalls** blockieren eingehende Verbindungen
3. **Schlechtes Netzwerk** (mobile Daten mit Carrier-Grade NAT)

### DERP vs. Direkte Verbindung

#### Idealszenario: Direkte Verbindung (nach STUN)

```
┌────────────┐                          ┌────────────┐
│   Laptop   │   Direkte Verbindung     │ Container  │
│100.64.0.2  │◄─────────────────────────►│100.64.0.1  │
└────────────┘   WireGuard verschlüsselt └────────────┘
     
✅ Schnell (niedrige Latenz)
✅ Keine zusätzlichen Hops
✅ Peer-to-Peer
```

#### Fallback: DERP Relay

```
┌────────────┐          ┌──────────┐          ┌────────────┐
│   Laptop   │          │   DERP   │          │ Container  │
│100.64.0.2  │◄─────────►│  Server  │◄─────────►│100.64.0.1  │
└────────────┘          │   VPS    │          └────────────┘
                        │Port 443  │
                        └──────────┘
     
🟡 Langsamer (extra Hop über VPS)
🟡 Aber: Funktioniert IMMER
🟡 Trotzdem Ende-zu-Ende verschlüsselt!
```

**WICHTIG:** Auch über DERP ist die Verbindung **Ende-zu-Ende verschlüsselt**!

Der DERP-Server sieht nur verschlüsselte Pakete und kann sie **nicht entschlüsseln**.

---

### DERP Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                    VPS (152.53.111.11)                       │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            Headscale Control Server                │    │
│  │            Port 8090 (hinter Nginx)                │    │
│  │  • Verwaltet Nodes                                 │    │
│  │  • Vergibt IPs                                     │    │
│  │  • Koordiniert Verbindungen                        │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            DERP Server (Embedded)                  │    │
│  │            Port 443 (HTTPS)                        │    │
│  │  • Relay für Pakete                                │    │
│  │  • Wenn direkte Verbindung nicht möglich           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            STUN Server                             │    │
│  │            Port 3478 (UDP)                         │    │
│  │  • Hilft Nodes ihre öffentliche IP zu finden      │    │
│  │  • NAT-Typ Detection                               │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

### DERP Packet Flow

**Laptop (Café) → Container (Heimnetz) über DERP:**

```
SCHRITT 1: Laptop sendet an DERP
┌────────────┐
│   Laptop   │
│            │  WireGuard-verschlüsseltes Paket
│            │  Ziel: Container (100.64.0.1)
└──────┬─────┘
       │
       ↓ HTTPS Connection zu DERP (Port 443)
       │
┌──────▼──────────────────────────────┐
│        DERP Server (VPS)            │
│                                     │
│  Empfängt verschlüsseltes Paket     │
│  Liest: "Für 100.64.0.1"           │
│  (Kann Inhalt NICHT entschlüsseln!) │
│                                     │
│  Checkt: Ist Container verbunden?   │
│  ✅ Ja, Connection zu Container     │
│     existiert                       │
└──────┬──────────────────────────────┘
       │
       ↓ Leitet weiter
       │
┌──────▼─────┐
│ Container  │
│            │  Empfängt verschlüsseltes Paket
│            │  Entschlüsselt mit WireGuard
│            │  ✅ Liest Inhalt
└────────────┘

SCHRITT 2: Container antwortet über DERP
┌────────────┐
│ Container  │
│            │  Verschlüsselt Antwort
└──────┬─────┘
       │
       ↓ HTTPS zu DERP
       │
┌──────▼──────────────────────────────┐
│        DERP Server (VPS)            │
│                                     │
│  Leitet verschlüsseltes Paket       │
│  an Laptop weiter                   │
└──────┬──────────────────────────────┘
       │
       ↓
┌──────▼─────┐
│   Laptop   │  Entschlüsselt Antwort
└────────────┘  ✅ Kommunikation erfolgreich!
```

---

### Warum Port 443 für DERP?

**Port 443 = Standard HTTPS Port**

Vorteile:
- ✅ Fast nie von Firewalls blockiert (Websites brauchen ihn)
- ✅ Sieht aus wie normaler HTTPS-Traffic
- ✅ Funktioniert in restriktiven Netzwerken (Hotels, Firmen, Flughäfen)

Deshalb läuft DERP auf **Port 443** statt auf einem zufälligen Port!

---

## 🔄 NAT-Traversal: Der komplette Ablauf

### Verbindungsaufbau zwischen Laptop und Container

**Phase 1: Nodes registrieren sich bei Headscale**

```
┌────────────┐                    ┌──────────────┐
│   Laptop   │                    │  Container   │
└──────┬─────┘                    └──────┬───────┘
       │                                 │
       ├─────────────────────────────────┤
       │    Beide verbinden sich mit     │
       │    Headscale Control Server     │
       │                                 │
       ↓                                 ↓
┌─────────────────────────────────────────────┐
│       Headscale Control Server (VPS)        │
│                                             │
│  Registrierte Nodes:                        │
│  • Laptop (100.64.0.2)                      │
│    - Öffentliche IP: ???                    │
│  • Container (100.64.0.1)                   │
│    - Öffentliche IP: ???                    │
└─────────────────────────────────────────────┘
```

**Phase 2: STUN - Öffentliche IPs herausfinden**

```
┌────────────┐                    ┌──────────────┐
│   Laptop   │                    │  Container   │
│            │                    │              │
│ STUN ────► │                    │ ◄──── STUN   │
│ Request    │                    │    Request   │
└────────────┘                    └──────────────┘
       │                                 │
       ↓                                 ↓
┌──────────────────────────────────────────────┐
│         STUN Server (Port 3478)              │
│                                              │
│  Laptop kommt von:  84.115.x.x:45123        │
│  Container kommt von: 84.116.y.y:12345      │
└──────────────────────────────────────────────┘
       │                                 │
       ↓                                 ↓
┌────────────┐                    ┌──────────────┐
│   Laptop   │                    │  Container   │
│            │                    │              │
│ "Ich bin   │                    │ "Ich bin     │
│ 84.115.x.x │                    │ 84.116.y.y   │
│ :45123"    │                    │ :12345"      │
└────────────┘                    └──────────────┘
       │                                 │
       └─────────────┬───────────────────┘
                     ↓
          Teilen ihre IPs mit Headscale
                     ↓
┌─────────────────────────────────────────────┐
│       Headscale Control Server              │
│                                             │
│  Nodes mit öffentlichen IPs:                │
│  • Laptop: 84.115.x.x:45123                │
│  • Container: 84.116.y.y:12345             │
│                                             │
│  Headscale teilt allen Nodes diese Info!    │
└─────────────────────────────────────────────┘
```

**Phase 3: ICE/STUN Hole Punching - Direkte Verbindung versuchen**

```
┌────────────┐                    ┌──────────────┐
│   Laptop   │                    │  Container   │
│            │                    │              │
│ "Container │                    │ "Laptop ist  │
│  ist unter │                    │  unter       │
│ 84.116.y.y │                    │ 84.115.x.x   │
│ :12345"    │                    │ :45123"      │
└──────┬─────┘                    └──────┬───────┘
       │                                 │
       │  Beide versuchen gleichzeitig   │
       │  Verbindung aufzubauen          │
       │  ("Hole Punching")              │
       │                                 │
       │     Versuche 1: UDP-Paket       │
       ├─────────────────────────────────►
       │                                 │
       ◄─────────────────────────────────┤
       │     Versuche 2: UDP-Paket       │
       │                                 │
       │  ✅ NAT-"Löcher" sind offen!    │
       │  ✅ Direkte Verbindung möglich! │
       │                                 │
       ◄────────────────────────────────►
       │   WireGuard Encrypted Traffic   │
       │   Peer-to-Peer Connection!      │
```

**Was ist "Hole Punching"?**

Beide Seiten senden **gleichzeitig** Pakete aneinander. Das öffnet temporär "Löcher" in den NATs, sodass Antworten durchkommen.

**Phase 4: Fallback zu DERP (falls Direkt nicht klappt)**

```
┌────────────┐                                ┌──────────────┐
│   Laptop   │                                │  Container   │
└──────┬─────┘                                └──────┬───────┘
       │                                             │
       │  Direkte Verbindung fehlgeschlagen         │
       │  (Symmetric NAT, Firewall, etc.)           │
       │                                             │
       ├─────────────────┬───────────────────────────┤
       │                 ↓                           │
       │        ┌─────────────────┐                 │
       │        │  DERP Server    │                 │
       │        │  (VPS:443)      │                 │
       │        └─────────────────┘                 │
       │                 │                           │
       ↓                 ↓                           ↓
   Verbinde zu DERP   Relay       Verbinde zu DERP
       │             Pakete              │
       ◄──────────────┼──────────────────►
            Encrypted Traffic
         (via DERP Relay)
```

---

## 📊 Vergleich: Direkt vs. DERP

| Aspekt | Direkte Verbindung | DERP Relay |
|--------|-------------------|------------|
| **Latenz** | ✅ Niedrig (5-20ms) | 🟡 Höher (20-100ms) |
| **Durchsatz** | ✅ Maximum | 🟡 Begrenzt durch VPS |
| **Funktioniert immer** | ❌ Nein (NAT-abhängig) | ✅ Ja, immer! |
| **Verschlüsselung** | ✅ Ende-zu-Ende | ✅ Ende-zu-Ende |
| **DERP sieht Inhalt** | N/A | ❌ Nein (verschlüsselt) |
| **Bevorzugt** | ✅ Ja | 🟡 Nur als Fallback |

---

## 🎯 Dein Setup im Detail

### STUN Server im Container

```bash
# Im Container - STUN läuft automatisch
tailscale netcheck
```

Output zeigt:
```
* DERP latency:
  - nue: 59.1ms  (Nuremberg)     ← Nächster DERP
  - fra: 67.2ms  (Frankfurt)
```

### Headscale Config

```yaml
derp:
  server:
    enabled: true                    ← Eigener DERP Server!
    stun_listen_addr: "0.0.0.0:3478" ← STUN Port
    ipv4: 152.53.111.11              ← Öffentliche VPS IP
```

### Wie checken ob Direkt oder DERP?

```bash
# Auf dem Laptop
tailscale status
```

Output:
```
100.64.0.1  tailscale  active; direct 192.168.0.150:41641
                              ^^^^^^ DIREKT verbunden!

# Oder falls über DERP:
100.64.0.1  tailscale  active; relay "nue"
                              ^^^^ Über DERP (Nuremberg)
```

---

## 🔍 Praktisches Beispiel: Verbindungsanalyse

### Laptop → Container Verbindung analysieren

```bash
# Auf dem Laptop
tailscale netcheck
```

**Output erklärt:**

```
Report:
  * UDP: true                           ← UDP funktioniert (gut!)
  * IPv4: yes, 84.115.223.57:45621     ← Öffentliche IP (via STUN)
  * IPv6: no, but OS has support        ← Kein IPv6
  * MappingVariesByDestIP: false        ← NAT-Typ: Port-Restricted
  * PortMapping: UPnP                   ← Router unterstützt UPnP
  * CaptivePortal: false                ← Kein Captive Portal
  * Nearest DERP: Nuremberg             ← Nächster DERP Server
  * DERP latency:
    - nue: 59.1ms  (Nuremberg)         ← 59ms zum DERP
    - fra: 67.2ms  (Frankfurt)
```

**Was bedeutet das:**

1. **UDP: true** → Direkte Verbindungen möglich
2. **IPv4 mit IP:Port** → STUN hat funktioniert
3. **NAT-Typ** → Beschreibt wie restriktiv dein Router ist
4. **UPnP** → Router kann automatisch Ports öffnen
5. **DERP Latency** → Fallback-Zeiten wenn Direkt nicht klappt

---

## 🚦 Verbindungsqualität verstehen

### Best Case: Direkte Verbindung

```
┌────────────┐  WireGuard   ┌──────────────┐
│   Laptop   │◄────────────►│  Container   │
│100.64.0.2  │   5-20ms     │ 100.64.0.1   │
└────────────┘  Peer-to-Peer└──────────────┘

✅ Schnell
✅ Niedrige Latenz
✅ Voller Durchsatz
```

### Fallback: DERP Relay

```
┌────────────┐            ┌──────────┐           ┌──────────────┐
│   Laptop   │            │   DERP   │           │  Container   │
│100.64.0.2  │◄──────────►│  Server  │◄─────────►│ 100.64.0.1   │
└────────────┘  30-50ms    │   VPS    │  30-50ms  └──────────────┘
                           └──────────┘
                         Gesamtlatenz: 60-100ms

🟡 Langsamer
🟡 Extra Hop
✅ Aber: Funktioniert immer!
```

---

## 💡 Warum ist dein Setup besonders gut?

### Tailscale's öffentliche DERP Server

```
Tailscale bietet weltweit DERP Server:
- New York
- London
- Frankfurt
- Tokyo
- etc.

❌ Problem: Du bist von Tailscale Inc. abhängig
❌ Alle Pakete laufen durch ihre Server
```

### Dein eigener DERP Server

```
Du hostest deinen eigenen DERP auf dem VPS:
- 152.53.111.11:443
- Nur für deine Nodes
- Volle Kontrolle

✅ Unabhängig von Tailscale Inc.
✅ Deine eigene Infrastruktur
✅ Keine Third-Party sieht deinen Traffic
✅ Trotzdem Ende-zu-Ende verschlüsselt!
```

---

## 📋 Zusammenfassung: STUN, DERP, NAT

### STUN (Session Traversal Utilities for NAT)
- **Zweck:** Herausfinden der öffentlichen IP und des NAT-Typs
- **Port:** 3478 (UDP)
- **Funktion:** "Du bist unter dieser IP:Port erreichbar"

### DERP (Designated Encrypted Relay for Packets)
- **Zweck:** Fallback-Relay wenn direkte Verbindung nicht möglich
- **Port:** 443 (HTTPS)
- **Funktion:** Leitet verschlüsselte Pakete weiter

### NAT (Network Address Translation)
- **Zweck:** Viele Geräte teilen sich eine öffentliche IP
- **Problem:** Macht eingehende Verbindungen schwierig
- **Lösung:** STUN Hole Punching oder DERP Relay

### NAT-Traversal
- **Phase 1:** Nodes registrieren bei Headscale
- **Phase 2:** STUN ermittelt öffentliche IPs
- **Phase 3:** Hole Punching versucht direkte Verbindung
- **Phase 4:** Fallback zu DERP wenn nötig

---

**Jetzt verstehst du die komplette Magie hinter Tailscale!** 🎩✨

---

## 🔧 Routing Tables, iptables, nftables & IP Forwarding - Die Details

### Was sind Routing Tables?

**Routing Tables** sind Tabellen die dem Betriebssystem sagen: "Für Ziel X, nutze Gateway Y über Interface Z"

Jedes Gerät (Laptop, Server, Router, etc.) hat eine Routing Table!

---

### Routing Table verstehen

#### Routing Table anzeigen

**Linux:**
```bash
# Kurz und übersichtlich
ip route

# Detailliert
ip route show table all

# Oder klassisch
route -n
```

**Beispiel Output:**

```
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
100.64.0.0/10 dev tailscale0 proto kernel scope link src 100.64.0.2
192.168.0.0/24 via 100.64.0.1 dev tailscale0
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.42
```

#### Zeile für Zeile erklärt:

```
┌────────────────────────────────────────────────────────────────┐
│  default via 192.168.1.1 dev wlan0 proto dhcp metric 600      │
└────────────────────────────────────────────────────────────────┘
     │         │             │         │            │
     │         │             │         │            └─ Priorität (niedriger = bevorzugt)
     │         │             │         └─ Protokoll (wie Route erstellt wurde)
     │         │             └─ Interface (Netzwerkkarte)
     │         └─ Gateway IP (Router)
     └─ Ziel (0.0.0.0/0 = alles)

Bedeutung: Für ALLES was nicht spezifischer matcht,
           nutze den Router 192.168.1.1 über WLAN
```

```
┌────────────────────────────────────────────────────────────────┐
│  100.64.0.0/10 dev tailscale0 proto kernel scope link         │
└────────────────────────────────────────────────────────────────┘
     │            │              │            │
     │            │              │            └─ Direkt verbunden (kein Gateway)
     │            │              └─ Kernel hat Route erstellt
     │            └─ Tailscale Interface
     └─ Ziel (Tailscale-Netz)

Bedeutung: Für Tailscale-IPs (100.64.0.x),
           sende direkt über tailscale0 Interface
```

```
┌────────────────────────────────────────────────────────────────┐
│  192.168.0.0/24 via 100.64.0.1 dev tailscale0                 │
└────────────────────────────────────────────────────────────────┘
     │               │              │
     │               │              └─ Über Tailscale Interface
     │               └─ Gateway (LXC Container)
     └─ Ziel (Heimnetz)

Bedeutung: Für Heimnetz (192.168.0.x),
           nutze Container als Gateway über Tailscale
```

---

### Routing-Entscheidung: Wie wählt Linux die Route?

**Prinzip:** **Längster Präfix-Match** gewinnt!

#### Beispiel: Laptop will zu 192.168.0.101

```
Routing Table:
┌──────────────────────┬──────────────┬──────────────┐
│ Ziel                 │ Präfix-Länge │ Gateway      │
├──────────────────────┼──────────────┼──────────────┤
│ 0.0.0.0/0           │ /0  (0 Bits) │ 192.168.1.1  │ ← Default
│ 192.168.0.0/24      │ /24 (24 Bits)│ 100.64.0.1   │ ← Spezifisch!
│ 192.168.1.0/24      │ /24 (24 Bits)│ direkt       │
└──────────────────────┴──────────────┴──────────────┘

Entscheidung für 192.168.0.101:
1. Passt zu 0.0.0.0/0? ✅ Ja (passt zu ALLEM)
2. Passt zu 192.168.0.0/24? ✅ Ja (spezifischer!)
3. Passt zu 192.168.1.0/24? ❌ Nein

→ Wähle 192.168.0.0/24 via 100.64.0.1
  (längster Match = 24 Bits)
```

---

### Routing Table auf dem Container

Der Container hat **ZWEI** Interfaces und entsprechend komplexere Routes:

```bash
# Im Container
ip route show
```

**Output:**

```
default via 192.168.0.1 dev eth0
100.64.0.0/10 dev tailscale0 proto kernel scope link src 100.64.0.1
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.150
```

**Was bedeutet das:**

```
┌──────────────────────────────────────────────────────┐
│         LXC CONTAINER ROUTING                        │
│                                                      │
│  Interface 1: tailscale0 (100.64.0.1)               │
│  Interface 2: eth0 (192.168.0.150)                  │
│                                                      │
│  Routing Entscheidungen:                             │
│                                                      │
│  • Paket zu 100.64.0.x  → tailscale0                │
│  • Paket zu 192.168.0.x → eth0                      │
│  • Paket zu Internet    → eth0 via 192.168.0.1      │
└──────────────────────────────────────────────────────┘
```

---

## 🔥 IP Forwarding - Der Container als Router

### Was ist IP Forwarding?

**IP Forwarding** erlaubt einem Linux-System, Pakete zwischen verschiedenen Netzwerk-Interfaces weiterzuleiten.

**Ohne IP Forwarding:**
```
Paket kommt rein → Kernel checkt: "Ist das für mich?" → Nein → ❌ VERWORFEN
```

**Mit IP Forwarding:**
```
Paket kommt rein → Kernel checkt: "Ist das für mich?" → Nein → Routing Table checken → ✅ WEITERLEITEN
```

---

### IP Forwarding Status checken

```bash
# IPv4 Forwarding
sysctl net.ipv4.ip_forward
# Sollte zeigen: net.ipv4.ip_forward = 1

# IPv6 Forwarding
sysctl net.ipv6.conf.all.forwarding
# Sollte zeigen: net.ipv6.conf.all.forwarding = 1

# ODER: Alle Netzwerk-Einstellungen
sysctl -a | grep forward
```

**Werte:**
- `0` = Aus (Pakete werden NICHT weitergeleitet)
- `1` = An (Pakete werden weitergeleitet)

---

### IP Forwarding aktivieren

#### Temporär (bis zum Reboot)

```bash
# IPv4
sudo sysctl -w net.ipv4.ip_forward=1

# IPv6
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

#### Permanent (überlebt Reboot)

```bash
# Datei editieren
sudo nano /etc/sysctl.conf

# Diese Zeilen hinzufügen oder aktivieren (# entfernen):
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Speichern und anwenden
sudo sysctl -p
```

---

### Warum braucht der Container IP Forwarding?

**Ohne IP Forwarding:**

```
┌────────────┐              ┌──────────────┐
│   Laptop   │              │  Container   │
│100.64.0.2  │              │ 100.64.0.1   │
└──────┬─────┘              └──────┬───────┘
       │                           │
       │  Paket: Ziel 192.168.0.101│
       ├──────────────────────────►│
                                   │
                           Kernel checkt:
                           "Ist 192.168.0.101 meine IP?"
                           Nein → ❌ VERWERFEN
                                   │
                                   X  Paket stirbt hier!
```

**Mit IP Forwarding:**

```
┌────────────┐              ┌──────────────┐              ┌──────────┐
│   Laptop   │              │  Container   │              │ Proxmox  │
│100.64.0.2  │              │ 100.64.0.1   │              │   .101   │
└──────┬─────┘              └──────┬───────┘              └──────────┘
       │                           │
       │  Paket: Ziel 192.168.0.101│
       ├──────────────────────────►│
                                   │
                           Kernel checkt:
                           "Ist 192.168.0.101 meine IP?"
                           Nein → IP Forwarding AN
                           → Routing Table checken
                           → Via eth0 weiterleiten!
                                   │
                                   ├─────────────────────►
                                                   ✅ Paket kommt an!
```

---

## 🛡️ iptables vs. nftables - Firewall & NAT

### Was sind iptables/nftables?

**iptables** und **nftables** sind Linux-Tools für:
1. **Firewall** (Pakete blockieren/erlauben)
2. **NAT** (IP-Adressen ändern)
3. **Packet Filtering** (Pakete filtern/modifizieren)

**nftables** ist der moderne Nachfolger von iptables (seit ~2014)

---

### iptables Basics

#### Wichtige Konzepte: Tables, Chains, Rules

**Tables:**
- `filter` - Firewall (Pakete erlauben/blockieren)
- `nat` - Network Address Translation
- `mangle` - Pakete modifizieren
- `raw` - Connection Tracking bypass

**Chains (in filter table):**
- `INPUT` - Eingehende Pakete FÜR dieses System
- `OUTPUT` - Ausgehende Pakete VON diesem System
- `FORWARD` - Durchlaufende Pakete (werden weitergeleitet)

**Visual:**

```
┌─────────────────────────────────────────────────────┐
│                    LINUX SYSTEM                     │
│                                                     │
│  Paket kommt rein                                   │
│         ↓                                           │
│  ┌──────────────┐                                   │
│  │ INPUT Chain  │  Ist Paket für mich?              │
│  └──────┬───────┘                                   │
│         │ Ja                                         │
│         ↓                                           │
│  Lokale Anwendung (z.B. SSH Server)                │
│                                                     │
│  ┌──────────────┐                                   │
│  │OUTPUT Chain  │  Antwort zurück                   │
│  └──────┬───────┘                                   │
│         ↓                                           │
│  Paket geht raus                                    │
│                                                     │
│  ════════════════════════════════════════════       │
│                                                     │
│  Paket kommt rein                                   │
│         ↓                                           │
│  ┌───────────────┐  Ist Paket für mich?            │
│  │ FORWARD Chain │  Nein → Weiterleiten!           │
│  └───────┬───────┘                                   │
│          ↓                                           │
│  Paket geht raus (an anderes Interface)             │
└─────────────────────────────────────────────────────┘
```

---

### NAT mit iptables - MASQUERADE

**MASQUERADE** = NAT für ausgehende Verbindungen (Source NAT)

Im Container brauchen wir das für Subnet-Routing!

#### Warum?

```
Ohne MASQUERADE:
┌───────┐        ┌───────────┐        ┌─────────┐
│Laptop │        │ Container │        │ Proxmox │
│.0.2   │───────►│ empfängt  │───────►│  .101   │
│       │        │ Paket von │        │         │
│       │        │ 100.64.0.2│        │ "Wer?"  │
│       │        │           │        │ ❌      │
└───────┘        └───────────┘        └─────────┘
                                      Proxmox kennt
                                      100.64.0.2 nicht!
                                      → Verwirft Paket

Mit MASQUERADE:
┌───────┐        ┌───────────┐        ┌─────────┐
│Laptop │        │ Container │        │ Proxmox │
│.0.2   │───────►│ ÄNDERT    │───────►│  .101   │
│       │        │ Source zu │        │         │
│       │        │192.168.0  │        │ "Ah, der│
│       │        │    .150   │        │Container│
│       │◄───────│ NAT Table │◄───────│ ✅      │
└───────┘        └───────────┘        └─────────┘
                 Container merkt sich:
                 "Laptop .0.2 wartet auf Antwort"
```

#### MASQUERADE Rule anzeigen

```bash
# NAT Table checken
sudo iptables -t nat -L -n -v

# POSTROUTING Chain ist wichtig
sudo iptables -t nat -L POSTROUTING -n -v
```

**Typischer Output:**

```
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  123 45678 MASQUERADE all  --  *      eth0    100.64.0.0/10        0.0.0.0/0
```

**Was bedeutet das:**
- Alle Pakete (`all`) 
- Von Tailscale-Netz (`100.64.0.0/10`)
- Die über `eth0` rausgehen (ins Heimnetz)
- → Source-IP wird zu Container-IP geändert (`MASQUERADE`)

---

#### MASQUERADE Rule manuell erstellen

```bash
# Für Subnet Router - Tailscale Pakete masqueraden
sudo iptables -t nat -A POSTROUTING -o eth0 -s 100.64.0.0/10 -j MASQUERADE

# ODER: Für Exit-Node - ALLE Pakete masqueraden
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**Erklärung:**
- `-t nat` = NAT table
- `-A POSTROUTING` = Append to POSTROUTING chain (nachdem Routing-Entscheidung)
- `-o eth0` = Output Interface (Pakete die über eth0 rausgehen)
- `-s 100.64.0.0/10` = Source (nur von Tailscale)
- `-j MASQUERADE` = Jump to MASQUERADE (ändere Source-IP)

---

### iptables Regeln persistent machen

**Problem:** iptables-Regeln gehen beim Reboot verloren!

#### Lösung 1: iptables-persistent (Debian/Ubuntu)

```bash
# Installieren
sudo apt install iptables-persistent

# Aktuelle Regeln speichern
sudo netfilter-persistent save

# Beim Reboot werden Regeln automatisch geladen!
```

#### Lösung 2: Manuell speichern/laden

```bash
# Regeln speichern
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Beim Boot laden (systemd service)
sudo nano /etc/systemd/system/iptables-restore.service
```

---

### nftables - Die moderne Alternative

**nftables** ersetzt iptables, ip6tables, arptables, ebtables mit **einem** Tool!

#### Unterschiede zu iptables

| Feature | iptables | nftables |
|---------|----------|----------|
| **Syntax** | Komplex, viele Tools | Einheitlich, ein Tool |
| **Performance** | Gut | Besser (optimiert) |
| **IPv4/IPv6** | Getrennt | Zusammen |
| **Scripting** | Schwierig | Einfach |

---

#### nftables Basics

```bash
# Status
sudo nft list ruleset

# Tables anzeigen
sudo nft list tables

# Chains in einer Table
sudo nft list table inet filter
```

#### NAT mit nftables

```bash
# NAT Table erstellen
sudo nft add table ip nat

# POSTROUTING Chain erstellen
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }

# MASQUERADE Rule
sudo nft add rule ip nat postrouting oifname "eth0" ip saddr 100.64.0.0/10 masquerade
```

**Erklärung:**
- `oifname "eth0"` = Output Interface (wie `-o eth0` bei iptables)
- `ip saddr 100.64.0.0/10` = Source Address (wie `-s`)
- `masquerade` = MASQUERADE Aktion

---

### Was nutzt dein Setup?

**Auf dem VPS:** Wahrscheinlich UFW (Uncomplicated Firewall)
- UFW ist ein Frontend für iptables
- Einfachere Syntax

**Auf dem Container:** iptables für NAT/MASQUERADE
- Tailscale erstellt automatisch benötigte Regeln
- Für Subnet Router: MASQUERADE wird automatisch gesetzt

---

### iptables Regeln checken

```bash
# Im Container - Alle Regeln anzeigen
sudo iptables -L -n -v

# NAT Table (wichtig für MASQUERADE!)
sudo iptables -t nat -L -n -v

# Nur FORWARD Chain (wichtig für Routing!)
sudo iptables -L FORWARD -n -v
```

---

## 🔍 Debugging: Packet Flow verfolgen

### Wie sehe ich ob Pakete weitergeleitet werden?

#### 1. tcpdump - Netzwerk-Traffic sniffen

```bash
# Im Container - Traffic auf tailscale0 Interface
sudo tcpdump -i tailscale0 -n

# Traffic auf eth0 Interface
sudo tcpdump -i eth0 -n

# Nur ICMP (Ping)
sudo tcpdump -i any icmp -n

# Zu/Von spezifischer IP
sudo tcpdump -i any host 192.168.0.101 -n
```

**Output beim Ping von Laptop zu Proxmox:**

```
# Auf tailscale0:
12:34:56.789 IP 100.64.0.2 > 192.168.0.101: ICMP echo request
12:34:56.791 IP 192.168.0.101 > 100.64.0.2: ICMP echo reply

# Auf eth0 (nach NAT!):
12:34:56.790 IP 192.168.0.150 > 192.168.0.101: ICMP echo request
12:34:56.791 IP 192.168.0.101 > 192.168.0.150: ICMP echo reply
```

**Siehst du:** Source-IP wurde geändert! (100.64.0.2 → 192.168.0.150)

---

#### 2. Conntrack - Connection Tracking anzeigen

```bash
# Aktive Verbindungen
sudo conntrack -L

# Nur ICMP
sudo conntrack -L -p icmp

# Nur zu/von spezifischer IP
sudo conntrack -L | grep 192.168.0.101
```

**Output:**

```
icmp     1 29 src=100.64.0.2 dst=192.168.0.101 type=8 code=0 id=12345 \
              src=192.168.0.101 dst=192.168.0.150 type=0 code=0 id=12345 mark=0
              
Original: 100.64.0.2 → 192.168.0.101
Reply:    192.168.0.101 → 192.168.0.150 (NAT!)
```

---

#### 3. iptables Counter - Paket-Statistiken

```bash
# FORWARD Chain Statistiken
sudo iptables -L FORWARD -n -v

# NAT Statistics
sudo iptables -t nat -L POSTROUTING -n -v
```

**Output:**

```
Chain FORWARD (policy ACCEPT 1234 packets, 567890 bytes)
 pkts bytes target     prot opt in     out     source               destination
  123  8520 ACCEPT     all  --  tailscale0  eth0  0.0.0.0/0            0.0.0.0/0
  120  7890 ACCEPT     all  --  eth0  tailscale0  0.0.0.0/0            0.0.0.0/0
  
Bedeutung: 123 Pakete von Tailscale → Heimnetz
           120 Pakete von Heimnetz → Tailscale
```

---

## 📋 Zusammenfassung: Routing, iptables, IP Forwarding

### Routing Tables
- **Funktion:** Sagen dem System wohin Pakete geschickt werden
- **Checken:** `ip route`
- **Prinzip:** Längster Präfix-Match gewinnt
- **Container:** Hat Routes für Tailscale UND Heimnetz

### IP Forwarding
- **Funktion:** Erlaubt Weiterleitung zwischen Interfaces
- **Checken:** `sysctl net.ipv4.ip_forward`
- **Aktivieren:** `sysctl -w net.ipv4.ip_forward=1`
- **Wichtig:** MUSS im Container AN sein für Subnet Router!

### iptables/nftables
- **Funktion:** Firewall + NAT
- **MASQUERADE:** Ändert Source-IP für ausgehende Pakete
- **Wichtig:** Damit Proxmox die Pakete kennt
- **Checken:** `iptables -t nat -L -n -v`

### NAT (MASQUERADE)
- **Warum:** Proxmox kennt Tailscale-IPs nicht
- **Lösung:** Container ändert Source zu seiner Heimnetz-IP
- **Effekt:** Proxmox denkt, Container hat angefragt
- **Rückweg:** Container leitet Antworten zurück (Conntrack)

### Packet Flow komplett:
1. Laptop sendet (100.64.0.2 → 192.168.0.101)
2. Routing Table: "Über 100.64.0.1"
3. Container empfängt auf tailscale0
4. IP Forwarding: "Weiterleiten erlaubt? ✅"
5. Routing Table: "192.168.0.101 über eth0"
6. iptables NAT: Source ändern (→ 192.168.0.150)
7. Paket raus über eth0
8. Proxmox empfängt und antwortet
9. Container empfängt Antwort
10. Conntrack: "Gehört zu Session mit Laptop!"
11. NAT zurück: Destination ändern (→ 100.64.0.2)
12. Paket über tailscale0 zurück
13. Laptop empfängt ✅

---

**Jetzt verstehst du die komplette Linux-Netzwerk-Magie!** 🎩✨

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
