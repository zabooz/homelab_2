# 🖥️ Proxmox Netzwerk-Setup - Statische IP-Konfiguration

**Erstellt:** 19. Januar 2026  
**Author:** Daniel (zabooz)  
**Zweck:** LAP-Vorbereitung IT-Systemtechnik  
**Setup:** Proxmox VE mit VMs/LXC und statischen IP-Adressen

---

## 📊 Netzwerk-Übersicht

### Aktuelle Infrastruktur

```
┌─────────────────────────────────────────────────────┐
│           HEIMNETZWERK (192.168.0.0/24)             │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Router/Gateway: 192.168.0.1                        │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Proxmox VE Host                             │  │
│  │  IP: 192.168.0.101 (statisch)                │  │
│  │  Bridge: vmbr0                                │  │
│  │                                               │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  VM 100: Windows Server 2025           │  │  │
│  │  │  IP: 192.168.0.110 (statisch)          │  │  │
│  │  │  Adapter: Ethernet (bc:24:11:a9:bf:52) │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │                                               │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  VM 101: Debian Linux Server           │  │  │
│  │  │  IP: 192.168.0.111 (statisch)          │  │  │
│  │  │  Adapter: ens18 (bc:24:11:1a:24:3d)    │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │                                               │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │  LXC 102: Tailscale Node               │  │  │
│  │  │  IP: 192.168.0.112 (statisch)          │  │  │
│  │  │  Adapter: eth0 (bc:24:11:d8:a7:b2)     │  │  │
│  │  │  Tailscale: 100.64.0.1                 │  │  │
│  │  │  Funktion: Exit-Node + Subnet Router   │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 🎯 IP-Adress-Schema

### Planung und Begründung

| Gerät | IP-Adresse | MAC-Adresse | Typ | Begründung |
|-------|------------|-------------|-----|------------|
| **Router** | 192.168.0.1 | - | Gateway | Standard-Gateway |
| **Proxmox Host** | 192.168.0.101 | ec:b1:d7:72:c9:d1 | Statisch | Hypervisor - zentrale Verwaltung |
| **Windows Server** | 192.168.0.110 | bc:24:11:a9:bf:52 | Statisch | Server-VM - stabile Erreichbarkeit |
| **Linux Server** | 192.168.0.111 | bc:24:11:1a:24:3d | Statisch | Server-VM - stabile Erreichbarkeit |
| **Tailscale LXC** | 192.168.0.112 | bc:24:11:d8:a7:b2 | Statisch | VPN Exit-Node - feste IP erforderlich |

### Netzwerk-Parameter

- **Netzwerk:** 192.168.0.0/24
- **Subnetzmaske:** 255.255.255.0 (CIDR: /24)
- **Gateway:** 192.168.0.1
- **DNS-Server:** 
  - Primär: 192.168.0.1 (Router)
  - Sekundär: 8.8.8.8 (Google DNS)
- **Verfügbare Hosts:** 192.168.0.1 - 192.168.0.254 (254 Adressen)

### IP-Bereich Aufteilung

```
192.168.0.1       - Gateway/Router
192.168.0.2-99    - DHCP-Pool (dynamische Geräte)
192.168.0.100-119 - Statische IPs (Server/Infrastruktur)
192.168.0.120-254 - Reserviert für Erweiterungen
```

---

## 🔧 Konfiguration der Systeme

### 1. Proxmox VE Host

**Bereits konfiguriert - keine Änderungen nötig**

```bash
# IP-Konfiguration anzeigen
root@homeserver:~# ip addr show vmbr0
3: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    inet 192.168.0.101/24 scope global vmbr0
```

**Netzwerk-Interface:** vmbr0 (Bridge)  
**Physisches Interface:** nic0 (enp0s31f6)

---

### 2. Windows Server 2025 (VM 100)

**Umstellung von DHCP (192.168.0.200) auf statisch (192.168.0.110)**

#### Konfiguration über GUI:

1. **Netzwerkadapter öffnen:**
   - Rechtsklick auf Start → "Ausführen" (oder `Win+R`)
   - Eingabe: `ncpa.cpl`
   - Enter

2. **Adapter-Eigenschaften:**
   - Rechtsklick auf "Ethernet" → "Eigenschaften"
   - Doppelklick auf "Internetprotokoll Version 4 (TCP/IPv4)"

3. **Statische IP konfigurieren:**
   ```
   ● Folgende IP-Adresse verwenden:
   
   IP-Adresse:           192.168.0.110
   Subnetzmaske:         255.255.255.0
   Standardgateway:      192.168.0.1
   
   ● Folgende DNS-Serveradressen verwenden:
   
   Bevorzugter DNS:      192.168.0.1
   Alternativer DNS:     8.8.8.8
   ```

4. **Speichern:** OK → OK → Schließen

#### Alternative: PowerShell-Konfiguration

```powershell
# Adapter-Name ermitteln
Get-NetAdapter

# Statische IP setzen
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.0.110 -PrefixLength 24 -DefaultGateway 192.168.0.1

# DNS-Server setzen
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.0.1","8.8.8.8")

# Konfiguration prüfen
Get-NetIPAddress -InterfaceAlias "Ethernet"
```

#### Verifizierung:

```powershell
# IP-Konfiguration anzeigen
ipconfig /all

# Gateway testen
ping 192.168.0.1

# Internet-Verbindung testen
ping 8.8.8.8

# DNS-Auflösung testen
ping google.com
```

---

### 3. Debian Linux Server (VM 101)

**Umstellung von DHCP (192.168.0.158) auf statisch (192.168.0.111)**

#### Konfigurationsdatei bearbeiten:

```bash
# Backup erstellen
sudo cp /etc/network/interfaces /etc/network/interfaces.bak

# Datei bearbeiten
sudo nano /etc/network/interfaces
```

#### Konfiguration:

**Vorher (DHCP):**
```bash
# The primary network interface
allow-hotplug ens18
iface ens18 inet dhcp
```

**Nachher (Statisch):**
```bash
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface - STATIC IP
auto ens18
iface ens18 inet static
    address 192.168.0.111
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8

# IPv6 autoconfiguration
iface ens18 inet6 auto
```

#### Wichtige Unterschiede:

| Parameter | Vorher | Nachher | Erklärung |
|-----------|--------|---------|-----------|
| **Interface Start** | `allow-hotplug` | `auto` | `auto` = Startet immer beim Boot (Server-Standard) |
| **IP-Vergabe** | `inet dhcp` | `inet static` | Statische statt dynamische IP |
| **IPv6** | `inet6 auto` | `inet6 auto` | Bleibt aktiviert (SLAAC) |

#### Netzwerk neustarten:

```bash
# Methode 1: Service neustarten
sudo systemctl restart networking

# Methode 2: System neustarten (empfohlen)
sudo reboot
```

#### Verifizierung:

```bash
# IP-Konfiguration anzeigen
ip addr show ens18

# Sollte zeigen:
# inet 192.168.0.111/24 scope global ens18

# Gateway testen
ping -c 4 192.168.0.1

# Internet testen
ping -c 4 8.8.8.8

# DNS testen
ping -c 4 google.com

# Routing-Tabelle anzeigen
ip route show
```

---

### 4. Tailscale LXC Container (CT 102)

**Umstellung von DHCP (192.168.0.150) auf statisch (192.168.0.112)**

#### Konfiguration:

```bash
# Backup erstellen
cp /etc/network/interfaces /etc/network/interfaces.bak

# Datei bearbeiten
nano /etc/network/interfaces
```

#### Konfigurationsdatei:

```bash
# Loopback
auto lo
iface lo inet loopback

# Statische IP für eth0
auto eth0
iface eth0 inet static
    address 192.168.0.112
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8
```

#### Netzwerk neustarten:

```bash
# Service neustarten
systemctl restart networking

# Oder LXC rebooten
reboot
```

#### Verifizierung:

```bash
# IP-Konfiguration anzeigen
ip addr show eth0

# Sollte zeigen:
# inet 192.168.0.112/24 scope global eth0
# Tailscale-Interface bleibt unverändert:
# tailscale0: inet 100.64.0.1/32

# Tailscale-Status prüfen
tailscale status

# Netzwerk-Tests
ping -c 4 192.168.0.1    # Gateway
ping -c 4 192.168.0.101  # Proxmox
ping -c 4 8.8.8.8        # Internet
ping -c 4 google.com     # DNS
```

**Wichtig:** Die Tailscale-IP (100.64.0.1) bleibt unverändert! Nur die lokale IP ändert sich.

---

## 📝 Wichtige Konzepte für die LAP

### 1. Statische vs. Dynamische IP-Adressen

#### DHCP (Dynamic Host Configuration Protocol)

**Funktionsweise:**
- Client sendet DHCP-Discover Broadcast
- DHCP-Server antwortet mit DHCP-Offer
- Client sendet DHCP-Request
- Server bestätigt mit DHCP-ACK

**Vorteile:**
- ✅ Automatische Konfiguration
- ✅ Zentrale Verwaltung
- ✅ Einfach für Client-Geräte

**Nachteile:**
- ❌ IP-Adresse kann sich ändern
- ❌ Nicht vorhersehbar
- ❌ Problematisch für Server

#### Statische IP-Adressen

**Konfiguration:**
- Manuell in der Netzwerkkonfiguration gesetzt
- IP bleibt dauerhaft gleich
- Keine Abhängigkeit von DHCP-Server

**Vorteile:**
- ✅ Vorhersehbar und stabil
- ✅ Einfaches Troubleshooting
- ✅ Notwendig für Server und Netzwerkdienste
- ✅ DNS-Einträge möglich

**Nachteile:**
- ❌ Manuelle Konfiguration nötig
- ❌ IP-Konflikte möglich bei falscher Planung

### 2. Wann statische IPs verwenden?

**Immer bei:**
- 🖥️ Servern (Web, Mail, DNS, etc.)
- 🌐 Netzwerk-Infrastruktur (Router, Switches, Access Points)
- 🔒 VPN-Endpunkten (wie Headscale/Tailscale Exit-Node)
- 📡 IoT-Geräten die Services anbieten
- 🖨️ Netzwerkdruckern

**DHCP verwenden bei:**
- 💻 Laptops und Desktop-PCs
- 📱 Smartphones und Tablets
- 👥 Gast-Geräte

### 3. Interface-Benennung unter Linux

#### Alte Namenskonvention (vor systemd):
```
eth0, eth1, eth2  → Erste, zweite, dritte Netzwerkkarte
wlan0             → WLAN-Interface
lo                → Loopback
```

**Problem:** Reihenfolge nicht vorhersehbar (eth0/eth1 konnten bei Neustart tauschen)

#### Neue Namenskonvention (Predictable Network Names):

**Schema:**
```
en  = Ethernet
wl  = WLAN
lo  = Loopback (unveränderbar)

Suffix:
s<slot>    = Slot-Nummer (z.B. ens18)
p<bus>     = PCI-Bus (z.B. enp0s3)
```

**Beispiele:**
- `ens18` = Ethernet, Slot 18 (Proxmox-Standard für VMs)
- `enp0s3` = Ethernet, PCI-Bus 0, Slot 3
- `wlp2s0` = WLAN, PCI-Bus 2, Slot 0
- `eth0` = LXC-Container (nutzen oft alte Benennung)

**Vorteil:** Konsistent über Neustarts hinweg

### 4. Netzwerk-Interfaces unter Linux

#### `auto` vs. `allow-hotplug`

**`auto` (empfohlen für Server):**
```bash
auto ens18
```
- Interface wird **immer** beim Boot gestartet
- Auch wenn kein Kabel eingesteckt ist
- Standard für Server

**`allow-hotplug` (für Desktop/Laptop):**
```bash
allow-hotplug ens18
```
- Interface wird nur gestartet wenn **Kabel erkannt** wird
- Gut für Geräte die oft getrennt werden
- Nicht für Server verwenden!

#### `/etc/network/interfaces` Struktur:

```bash
# Loopback (immer vorhanden)
auto lo
iface lo inet loopback

# Physisches Interface
auto <interface>
iface <interface> inet static
    address <IP-Adresse>
    netmask <Subnetzmaske>
    gateway <Gateway-IP>
    dns-nameservers <DNS1> <DNS2>
```

### 5. Subnetzmasken und CIDR-Notation

#### Subnetzmaske verstehen:

**255.255.255.0:**
```
11111111.11111111.11111111.00000000
└─────────────────────────┘└──────┘
    Netzwerk-Teil (24 Bit)  Host-Teil (8 Bit)
```

#### CIDR-Notation:

| CIDR | Subnetzmaske | Verfügbare Hosts | Beispiel |
|------|--------------|------------------|----------|
| /24 | 255.255.255.0 | 254 | 192.168.0.0/24 |
| /16 | 255.255.0.0 | 65,534 | 192.168.0.0/16 |
| /8 | 255.0.0.0 | 16,777,214 | 10.0.0.0/8 |

**Berechnung für /24:**
- Netzwerk: 192.168.0.0
- Broadcast: 192.168.0.255
- Erster Host: 192.168.0.1 (meist Gateway)
- Letzter Host: 192.168.0.254
- **Verfügbar:** 254 Adressen

### 6. Gateway und Routing

**Gateway (Standardgateway):**
- Zugang zu anderen Netzwerken
- In Heimnetzen meist der Router
- Alle Pakete zu fremden Netzen gehen an Gateway

**Routing-Tabelle anzeigen:**
```bash
# Linux
ip route show

# Beispiel-Output:
default via 192.168.0.1 dev ens18  ← Standardroute
192.168.0.0/24 dev ens18 scope link  ← Lokales Netz
```

### 7. DNS (Domain Name System)

**Funktion:**
- Übersetzt Domainnamen in IP-Adressen
- google.com → 142.250.185.46

**DNS-Server Hierarchie:**
```
1. Lokaler DNS (Router): 192.168.0.1
   ↓ (falls nicht gefunden)
2. ISP DNS oder Public DNS: 8.8.8.8 (Google)
   ↓
3. Root-DNS-Server
   ↓
4. TLD-DNS-Server (.com, .at, etc.)
   ↓
5. Authoritative DNS-Server
```

**DNS testen:**
```bash
# Linux
nslookup google.com
dig google.com

# Windows
nslookup google.com
```

### 8. IPv6 Grundlagen

**IPv6-Adress-Typen:**
```
Link-Local:  fe80::/10  → Nur im lokalen Netz gültig
Global:      2000::/3   → Internet-routbar
Loopback:    ::1        → Localhost (wie 127.0.0.1)
```

**SLAAC (Stateless Address Autoconfiguration):**
- Router sendet Router Advertisement (RA)
- Client generiert eigene IPv6-Adresse
- Kein DHCP nötig (aber DHCPv6 existiert auch)

**In unserer Config:**
```bash
iface ens18 inet6 auto
```
→ Aktiviert SLAAC, Client holt sich automatisch IPv6

---

## 🧪 Test-Protokoll

### Verbindungstests zwischen allen Systemen

**Von jedem System aus:**

```bash
# Proxmox Host testen
ping 192.168.0.101

# Windows Server testen
ping 192.168.0.110

# Linux Server testen
ping 192.168.0.111

# Tailscale LXC testen
ping 192.168.0.112

# Gateway testen
ping 192.168.0.1

# Internet-Konnektivität
ping 8.8.8.8

# DNS-Auflösung
ping google.com
```

### Erwartete Ergebnisse:

✅ Alle Pings sollten erfolgreich sein (0% packet loss)  
✅ Antwortzeiten im lokalen Netz: < 1ms  
✅ DNS-Auflösung funktioniert  
✅ Internet-Zugang von allen Systemen

---

## 🔍 Troubleshooting

### Problem: Keine Netzwerk-Verbindung nach Umstellung

**Checkliste:**

1. **IP-Konfiguration prüfen:**
   ```bash
   # Linux
   ip addr show <interface>
   
   # Windows
   ipconfig /all
   ```

2. **Gateway erreichbar?**
   ```bash
   ping 192.168.0.1
   ```
   - Wenn nicht → IP/Subnetzmaske falsch

3. **DNS funktioniert?**
   ```bash
   ping 8.8.8.8  # Sollte funktionieren
   ping google.com  # Testet DNS
   ```
   - Wenn 8.8.8.8 geht, aber google.com nicht → DNS-Problem

4. **Routing-Tabelle prüfen:**
   ```bash
   # Linux
   ip route show
   
   # Windows
   route print
   ```
   - Default-Route vorhanden?

5. **Firewall blockiert?**
   ```bash
   # Linux (temporär deaktivieren zum Testen)
   sudo systemctl stop ufw
   
   # Windows
   Windows Defender Firewall temporär deaktivieren
   ```

### Problem: IP-Konflikt

**Symptom:** Intermittierende Verbindungsabbrüche, doppelte IP-Warnung

**Lösung:**
```bash
# IP-Konflikt im Netz finden (Linux)
sudo arping -D -I ens18 192.168.0.110

# Alle aktiven IPs im Netz scannen
nmap -sn 192.168.0.0/24
```

**Vermeidung:**
- IP-Schema dokumentieren
- DHCP-Pool von statischen IPs trennen
- Statische IPs außerhalb DHCP-Range

### Problem: DNS funktioniert nicht

**Symptom:** `ping 8.8.8.8` funktioniert, aber `ping google.com` nicht

**Linux-Lösung:**
```bash
# DNS-Server prüfen
cat /etc/resolv.conf

# Sollte enthalten:
nameserver 192.168.0.1
nameserver 8.8.8.8

# Falls nicht, in /etc/network/interfaces ergänzen:
dns-nameservers 192.168.0.1 8.8.8.8
```

**Windows-Lösung:**
- Netzwerkadapter-Eigenschaften → TCP/IPv4
- DNS-Server manuell eintragen

### Problem: Netzwerk nach Reboot weg (Linux)

**Ursache:** Interface startet nicht automatisch

**Lösung:**
```bash
# In /etc/network/interfaces prüfen:
auto ens18  # ← Muss vorhanden sein!
iface ens18 inet static
```

Nicht `allow-hotplug` bei Servern verwenden!

---

## 📚 Nützliche Befehle

### Linux Netzwerk-Diagnose

```bash
# IP-Konfiguration
ip addr show                    # Alle Interfaces
ip addr show <interface>        # Spezifisches Interface
ip link show                    # Interface-Status

# Routing
ip route show                   # Routing-Tabelle
ip route get <ip>               # Welche Route für IP?

# DNS
nslookup <domain>               # DNS-Auflösung
dig <domain>                    # Detaillierte DNS-Info
cat /etc/resolv.conf            # Konfigurierte DNS-Server

# Verbindungstests
ping -c 4 <ip>                  # ICMP-Test (4 Pakete)
traceroute <ip>                 # Route verfolgen
mtr <ip>                        # Kombiniert ping + traceroute

# Port-Tests
nc -zv <ip> <port>              # Port-Test
telnet <ip> <port>              # Verbindungstest
ss -tuln                        # Offene Ports

# Netzwerk-Services
systemctl status networking     # Netzwerk-Service-Status
systemctl restart networking    # Netzwerk neu starten

# Interface-Verwaltung
ip link set <if> up             # Interface aktivieren
ip link set <if> down           # Interface deaktivieren
```

### Windows Netzwerk-Diagnose

```powershell
# IP-Konfiguration
ipconfig                        # Basis-Info
ipconfig /all                   # Detaillierte Info
ipconfig /release               # DHCP-Adresse freigeben
ipconfig /renew                 # Neue DHCP-Adresse holen
ipconfig /flushdns              # DNS-Cache leeren

# Verbindungstests
ping <ip>                       # ICMP-Test
tracert <ip>                    # Route verfolgen
pathping <ip>                   # Erweiterte Route-Analyse

# DNS
nslookup <domain>               # DNS-Auflösung
ipconfig /displaydns            # DNS-Cache anzeigen

# Port-Tests
Test-NetConnection <ip> -Port <port>  # PowerShell
telnet <ip> <port>              # CMD (telnet-client muss installiert sein)

# Routing
route print                     # Routing-Tabelle
netstat -rn                     # Routing-Tabelle (alternativ)

# Adapter-Verwaltung
Get-NetAdapter                  # Alle Adapter (PowerShell)
Get-NetIPAddress                # IP-Konfiguration (PowerShell)
Get-NetRoute                    # Routing-Tabelle (PowerShell)

# Firewall
netsh advfirewall show allprofiles  # Firewall-Status
```

### Proxmox-spezifische Befehle

```bash
# Network-Konfiguration
cat /etc/network/interfaces     # Netzwerk-Config anzeigen
nano /etc/network/interfaces    # Netzwerk-Config bearbeiten

# Bridge-Informationen
brctl show                      # Alle Bridges
ip link show vmbr0              # Bridge-Details

# VM/LXC-Management
qm list                         # Alle VMs auflisten
pct list                        # Alle LXC-Container auflisten
qm status <vmid>                # VM-Status
pct status <ctid>               # Container-Status

# Netzwerk neu starten (VORSICHT - bricht SSH-Verbindung!)
systemctl restart networking
ifreload -a                     # Alternativer, sanfterer Restart
```

---

## ✅ Zusammenfassung

### Was wurde erreicht:

1. ✅ **Netzwerk-Architektur geplant**
   - Sauberes IP-Schema (110-119 für Server)
   - Getrennte Bereiche für DHCP und statische IPs

2. ✅ **Alle VMs auf statische IPs umgestellt**
   - Windows Server: 192.168.0.110
   - Linux Server: 192.168.0.111
   - Tailscale LXC: 192.168.0.112

3. ✅ **Konfiguration dokumentiert**
   - Schritt-für-Schritt Anleitungen
   - Befehle für Linux und Windows
   - Troubleshooting-Guide

4. ✅ **Funktionalität verifiziert**
   - Alle Systeme erreichbar
   - Internet-Zugang funktioniert
   - DNS-Auflösung klappt

### Vorteile des Setups:

- 🎯 **Vorhersehbar:** Jedes System hat eine feste IP
- 🔧 **Wartbar:** Einfaches Troubleshooting durch klare Struktur
- 📈 **Skalierbar:** Platz für 20 weitere statische Server-IPs
- 📝 **Dokumentiert:** Komplette Anleitung für Nachvollziehbarkeit
- 🎓 **LAP-gerecht:** Zeigt Verständnis von Netzwerkplanung

### Für die LAP wichtig:

**Mögliche Prüfungsfragen:**

1. **"Warum haben Sie statische IPs verwendet?"**
   - Server-Infrastruktur braucht Stabilität
   - VPN Exit-Node benötigt feste Adresse
   - Einfacheres Troubleshooting
   - Professioneller Produktiv-Standard

2. **"Was ist der Unterschied zwischen `auto` und `allow-hotplug`?"**
   - `auto`: Startet immer beim Boot (Server)
   - `allow-hotplug`: Nur bei Kabel-Erkennung (Laptop)

3. **"Wie funktioniert DHCP?"**
   - Discover → Offer → Request → ACK
   - Automatische IP-Vergabe
   - Lease-Time (Ablaufzeit)

4. **"Was ist ein Gateway?"**
   - Zugang zu anderen Netzwerken
   - Routet Pakete an fremde Netze
   - Standard-Route in Routing-Tabelle

5. **"Wie prüfen Sie die Netzwerk-Verbindung?"**
   - Ping zum Gateway
   - Ping zu DNS (8.8.8.8)
   - Ping zu Domain (DNS-Test)
   - Routing-Tabelle checken

---

**Dokumentation erstellt am:** 19. Januar 2026  
**Status:** ✅ Vollständig funktionsfähig  
**Getestet:** Alle Systeme erreichbar und funktional

---

*Viel Erfolg bei der LAP-Prüfung! 🚀*
