# Netzwerk-Grundlagen - LAP Referenz

Konsolidierte Sammlung aller prüfungsrelevanten Netzwerk-Konzepte.

---

## IP-Adressierung & Subnetting

### Subnetzmasken verstehen

**255.255.255.0 in Binär:**
```
11111111.11111111.11111111.00000000
└─────────────────────────┘└──────┘
    Netzwerk-Teil (24 Bit)  Host-Teil (8 Bit)
```

### CIDR-Notation

| CIDR | Subnetzmaske | Verfügbare Hosts | Beispiel |
|------|--------------|------------------|----------|
| /24 | 255.255.255.0 | 254 | 192.168.0.0/24 |
| /16 | 255.255.0.0 | 65,534 | 192.168.0.0/16 |
| /8 | 255.0.0.0 | 16,777,214 | 10.0.0.0/8 |
| /32 | 255.255.255.255 | 1 (einzelner Host) | 192.168.0.1/32 |
| /10 | 255.192.0.0 | 4,194,302 | 100.64.0.0/10 (CGNAT) |

### Berechnung für /24

```
Netzwerk:      192.168.0.0
Broadcast:     192.168.0.255
Erster Host:   192.168.0.1 (meist Gateway)
Letzter Host:  192.168.0.254
Verfügbar:     254 Adressen
```

### Private IP-Bereiche (RFC 1918)

| Klasse | Bereich | CIDR | Nutzung |
|--------|---------|------|---------|
| A | 10.0.0.0 - 10.255.255.255 | /8 | Große Netze |
| B | 172.16.0.0 - 172.31.255.255 | /12 | Mittelgroße |
| C | 192.168.0.0 - 192.168.255.255 | /16 | Heimnetze |

### Spezielle Bereiche

| Bereich | Zweck |
|---------|-------|
| 127.0.0.0/8 | Loopback (localhost) |
| 169.254.0.0/16 | Link-Local (APIPA) |
| 100.64.0.0/10 | CGNAT (Carrier-Grade NAT) - Tailscale nutzt dies! |

---

## DHCP (Dynamic Host Configuration Protocol)

### DORA-Prozess

```
1. DISCOVER  →  Client sendet Broadcast: "Ich brauche eine IP!"
2. OFFER     ←  Server bietet IP an: "Du kannst 192.168.0.100 haben"
3. REQUEST   →  Client: "Ja, ich möchte diese IP!"
4. ACK       ←  Server bestätigt: "OK, sie gehört dir für X Stunden"
```

### DHCP vs. Statische IP

| Aspekt | DHCP | Statisch |
|--------|------|----------|
| **Konfiguration** | Automatisch | Manuell |
| **Änderung** | IP kann sich ändern | Bleibt gleich |
| **Verwaltung** | Zentral (DHCP-Server) | Dezentral |
| **Einsatz** | Clients, Laptops | Server, Router, VPN |

### Wann statische IPs?

**Immer bei:**
- Servern (Web, DNS, DHCP)
- Netzwerk-Infrastruktur (Router, Switches)
- VPN-Endpunkten
- Druckern, IoT mit Services

**DHCP bei:**
- Laptops, Desktops
- Smartphones, Tablets
- Gast-Geräte

---

## Routing & Gateway

### Was ist ein Gateway?

- Zugang zu **anderen Netzwerken**
- In Heimnetzen meist der Router
- Alle Pakete zu fremden Netzen gehen an Gateway
- Standard-Route in Routing-Tabelle

### Routing-Tabelle lesen

```bash
ip route show
# default via 192.168.0.1 dev eth0     ← Standardroute (Gateway)
# 192.168.0.0/24 dev eth0 scope link   ← Lokales Netz (direkt erreichbar)
```

**Wichtige Begriffe:**
- **default via** → Standardgateway
- **scope link** → Direkt erreichbar (kein Router nötig)
- **via X** → Über diesen Router erreichbar

### Längster Präfix-Match

Der Kernel wählt die **spezifischste** Route:

```
192.168.0.0/24  ← /24 ist spezifischer als /16
192.168.0.0/16  ← /16 ist spezifischer als /0
0.0.0.0/0       ← Default Route (unspezifischste)
```

### Policy-Based Routing (Table 52)

Linux nutzt **mehrere Routing-Tables** für verschiedene Zwecke:

```bash
# Main Table (Standard)
ip route show table main

# Tailscale Table 52
ip route show table 52

# Welche Route wird verwendet?
ip route get 192.168.0.101
```

**Warum?**
- Keine Konflikte zwischen VPN und normalem Traffic
- Saubere Trennung
- Automatisches Failover

---

## NAT (Network Address Translation)

### NAT-Typen

| Typ | Funktion | Beispiel |
|-----|----------|----------|
| **SNAT** | Ändert Source-IP (ausgehend) | Heimnetz → Internet |
| **DNAT** | Ändert Destination-IP (eingehend) | Port-Forwarding |
| **MASQUERADE** | Dynamisches SNAT | VPN-Router mit DHCP |

### Warum MASQUERADE?

```
Problem:
Laptop sendet:    100.64.0.2 → 192.168.0.101
Proxmox denkt:    "100.64.0.2? Kenne ich nicht!"
                  → Paket verworfen ❌

Lösung mit MASQUERADE:
Container ändert: 100.64.0.2 → 192.168.0.112
Proxmox denkt:    "Ah, Container! Den kenne ich!"
                  → Antwortet ✅
```

### iptables NAT-Regel

```bash
# NAT-Regel anzeigen
sudo iptables -t nat -L POSTROUTING -n -v

# NAT-Regel erstellen
sudo iptables -t nat -A POSTROUTING -o eth0 -s 100.64.0.0/10 -j MASQUERADE
```

---

## IP Forwarding

### Was macht IP Forwarding?

```
Aus (= 0): Computer verwirft fremde Pakete
An (= 1):  Computer leitet fremde Pakete weiter → wird zum Router!
```

### Konfiguration

```bash
# Status prüfen
sysctl net.ipv4.ip_forward

# Temporär aktivieren
sudo sysctl -w net.ipv4.ip_forward=1

# Dauerhaft aktivieren (/etc/sysctl.conf)
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Anwenden
sudo sysctl -p
```

### Router vs. Host

| Merkmal | Host | Router |
|---------|------|--------|
| **IP Forwarding** | Aus | An |
| **Gateway** | Hat Gateway | Kein Gateway |
| **Interfaces** | 1 | Mehrere |
| **Routing** | Fragt Gateway | Entscheidet selbst |

**Merksatz:**
```
Host   = Hat Gateway (fragt bei Unbekanntem)
Router = Kein Gateway (entscheidet selbst)
```

---

## DNS (Domain Name System)

### Funktion

- Übersetzt Domainnamen in IP-Adressen
- google.com → 142.250.185.46

### DNS-Hierarchie

```
1. Lokaler DNS (Router):     192.168.0.1
   ↓ (falls nicht gefunden)
2. Public DNS:               8.8.8.8 (Google)
   ↓
3. Root-DNS-Server           (.)
   ↓
4. TLD-DNS-Server            (.com, .at, .org)
   ↓
5. Authoritative DNS-Server  (google.com)
```

### DNS testen

```bash
# Linux
nslookup google.com
dig google.com

# Konfigurierte DNS-Server
cat /etc/resolv.conf
```

### MagicDNS (Tailscale/Headscale)

Headscale kann eigene DNS-Records bereitstellen:

```yaml
# /etc/headscale/config.yaml
dns:
  extra_records:
    - name: "home.lab"
      type: "A"
      value: "192.168.0.111"
```

---

## iptables Grundlagen

### Chains (Ketten)

| Chain | Wann | Beispiel |
|-------|------|----------|
| **PREROUTING** | Vor Routing-Entscheidung | DNAT |
| **INPUT** | Pakete für lokales System | Firewall |
| **FORWARD** | Pakete die weitergeleitet werden | Router |
| **OUTPUT** | Pakete vom lokalen System | Ausgehende Filter |
| **POSTROUTING** | Nach Routing-Entscheidung | SNAT/MASQUERADE |

### Tables

| Table | Zweck |
|-------|-------|
| **filter** | Firewall-Regeln (Standard) |
| **nat** | NAT-Regeln |
| **mangle** | Paket-Modifikation |

### Wichtige Befehle

```bash
# Filter-Regeln anzeigen
sudo iptables -L -n -v

# NAT-Regeln anzeigen
sudo iptables -t nat -L -n -v

# Forward-Chain anzeigen
sudo iptables -L FORWARD -n -v

# Regel hinzufügen
sudo iptables -A FORWARD -i tailscale0 -j ACCEPT

# Regel am Anfang einfügen
sudo iptables -I FORWARD -i tailscale0 -j ACCEPT

# Regeln dauerhaft speichern
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

## Linux Interface-Benennung

### Alte Konvention (vor systemd)

```
eth0, eth1, eth2  → Netzwerkkarten
wlan0             → WLAN
lo                → Loopback
```

**Problem:** Reihenfolge nicht vorhersehbar

### Neue Konvention (Predictable Names)

```
en  = Ethernet
wl  = WLAN
lo  = Loopback

Suffix:
s<slot>    = Slot-Nummer (z.B. ens18)
p<bus>     = PCI-Bus (z.B. enp0s3)
```

**Beispiele:**
- `ens18` = Ethernet, Slot 18 (Proxmox VMs)
- `enp0s3` = Ethernet, PCI-Bus 0, Slot 3
- `wlp2s0` = WLAN, PCI-Bus 2, Slot 0
- `eth0` = LXC-Container (alte Benennung)

### auto vs. allow-hotplug

```bash
# auto - Immer beim Boot starten (SERVER!)
auto ens18

# allow-hotplug - Nur bei Kabel-Erkennung (Laptop)
allow-hotplug ens18
```

---

## VPN-Typen

| Typ | Beschreibung | Beispiel |
|-----|--------------|----------|
| **Site-to-Site** | Verbindet zwei Netzwerke | Firmenstandorte |
| **Remote Access** | Clients zu einem Netzwerk | Homeoffice-VPN |
| **Mesh VPN** | Alle Nodes direkt verbunden | Tailscale/Headscale |

### WireGuard vs. OpenVPN

| Feature | WireGuard | OpenVPN |
|---------|-----------|---------|
| **Geschwindigkeit** | Sehr schnell | Langsamer |
| **Code-Basis** | 4.000 Zeilen | 100.000+ Zeilen |
| **Konfiguration** | Einfach | Komplex |
| **Protokoll** | UDP | TCP/UDP |

---

## Wichtige Ports

| Port | Protokoll | Dienst |
|------|-----------|--------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3478 | UDP | STUN (NAT-Traversal) |
| 8006 | TCP | Proxmox Web UI |

---

## IPv6 Grundlagen

### Adresstypen

| Typ | Präfix | Beschreibung |
|-----|--------|--------------|
| Link-Local | fe80::/10 | Nur im lokalen Netz |
| Global | 2000::/3 | Internet-routbar |
| Loopback | ::1 | Localhost |

### SLAAC (Stateless Address Autoconfiguration)

1. Router sendet Router Advertisement (RA)
2. Client generiert eigene IPv6-Adresse
3. Kein DHCP nötig

```bash
# In /etc/network/interfaces
iface ens18 inet6 auto  # SLAAC aktiviert
```

---

## Reverse Proxy

### Forward vs. Reverse Proxy

```
Forward Proxy:
Client → Proxy → Internet
(Client kennt Proxy, Server nicht)
Beispiel: Firmen-Proxy für Internet-Zugang

Reverse Proxy:
Client → Proxy → Backend
(Client kennt nur Proxy)
Beispiel: Nginx vor Webserver
```

### Vorteile Reverse Proxy

- Keine Ports merken (Port 80/443)
- SSL/HTTPS terminieren
- Mehrere Services auf einem Server
- Load Balancing
- Security (Backend versteckt)

---

## Diagnose-Befehle

### Netzwerk

```bash
ip addr show              # IP-Konfiguration
ip route show             # Routing-Tabelle
ip route get <ip>         # Route für spezifische IP
ping <ip>                 # Erreichbarkeit testen
traceroute <ip>           # Route verfolgen
ss -tuln                  # Offene Ports
```

### DNS

```bash
nslookup <domain>         # DNS-Auflösung
dig <domain>              # Detaillierte DNS-Info
cat /etc/resolv.conf      # Konfigurierte DNS-Server
```

### Traffic-Analyse

```bash
tcpdump -i any host <ip>  # Traffic beobachten
tcpdump -i eth0 -n        # Interface-spezifisch
```

### Windows

```powershell
ipconfig /all             # IP-Konfiguration
ipconfig /flushdns        # DNS-Cache leeren
route print               # Routing-Tabelle
Test-NetConnection <ip> -Port <port>  # Port-Test
```

---

## Prüfungsfragen

### "Was ist der Unterschied zwischen DHCP und statischer IP?"

DHCP vergibt IP-Adressen automatisch, kann sich aber ändern. Statische IPs werden manuell konfiguriert und bleiben gleich. Server brauchen statische IPs für Erreichbarkeit, Clients können DHCP nutzen.

### "Was ist ein Gateway?"

Ein Gateway ist der Zugang zu anderen Netzwerken. In Heimnetzen ist das meist der Router. Alle Pakete zu unbekannten Zielen werden an das Gateway geschickt.

### "Warum braucht ein Router kein Gateway?"

Ein Router verbindet selbst Netzwerke und trifft eigene Routing-Entscheidungen. Ein Gateway würde bedeuten "frag jemand anderen" - aber der Router IST die Entscheidungsinstanz.

### "Was ist NAT/MASQUERADE?"

NAT ändert IP-Adressen in Paketen. MASQUERADE ist dynamisches Source-NAT, das die Quell-IP durch die eigene ersetzt. Nötig, damit Antworten zurückfinden.

### "Was ist IP Forwarding?"

Die Fähigkeit eines Linux-Systems, Pakete zwischen Interfaces weiterzuleiten. Aus = normaler Host, An = Router. Ohne IP Forwarding verwirft das System fremde Pakete.

### "Was bedeutet 0.0.0.0/0?"

Eine Default-Route, die ALLE IP-Adressen matched. Das /0 bedeutet "keine Netzwerk-Bits festgelegt". Wird für Internet-Traffic und Exit-Nodes verwendet.

---

*Erstellt für LAP IT-Systemtechnik Vorbereitung*
