# Tailscale Subnet Router - Gateway Problem

**Datum:** 19. Januar 2026
**Problem:** Proxmox und alle Server nicht erreichbar nach statischer IP-Konfiguration
**Ursache:** Gateway im Subnet Router konfiguriert
**Status:** ✅ Gelöst

---

## Was ist passiert?

Nach der Umstellung des Tailscale LXC Containers von DHCP auf statische IP war plötzlich nichts mehr erreichbar:

```bash
# Vom Laptop aus:
ping 192.168.0.101  # Proxmox     - ❌ FAILED (100% packet loss)
ping 192.168.0.110  # Windows     - ❌ FAILED
ping 192.168.0.111  # Debian      - ❌ FAILED
ping 192.168.0.112  # Container   - ✅ SUCCESS
```

**Die verwirrende Situation:**
- Laptop war im Internet (über Tailscale Exit-Node)
- Laptop hatte lokale IP 192.168.0.95 (DHCP)
- Laptop war im gleichen Netz wie alle Server
- Trotzdem: Keine Server erreichbar!

---

## Root Cause

### Die fehlerhafte Konfiguration

```bash
# /etc/network/interfaces im LXC Container
auto eth0
iface eth0 inet static
    address 192.168.0.112
    netmask 255.255.255.0
    gateway 192.168.0.1        # ← FEHLER!
    dns-nameservers 192.168.0.1 8.8.8.8
```

### Warum ist das ein Problem?

**Ein Subnet Router braucht KEIN Gateway!**

| Typ | Gateway? | Verhalten |
|-----|----------|-----------|
| **Normaler Host** (PC, Server) | ✅ JA | "Was ich nicht kenne → Gateway fragen" |
| **Router** (VPN, Firewall) | ❌ NEIN | "Ich VERBINDE Netzwerke, ich entscheide selbst!" |

### Was passierte (falsch):

```
Laptop → "Ich will zu 192.168.0.101"
    ↓
Tailscale Route: 192.168.0.0/24 dev tailscale0
    ↓
Container empfängt auf tailscale0
    ↓
Container: "192.168.0.101? Kenne ich nicht! → Gateway fragen!"
    ↓
Router (192.168.0.1): "Paket von 100.64.0.x? WTF?"
    ↓
❌ Paket verworfen
```

### Was passieren sollte (richtig):

```
Laptop → "Ich will zu 192.168.0.101"
    ↓
Tailscale Route: 192.168.0.0/24 dev tailscale0
    ↓
Container empfängt auf tailscale0
    ↓
Container: "192.168.0.101 ist in 192.168.0.0/24 → über eth0 erreichbar!"
    ↓
NAT/MASQUERADE: Source 100.64.0.2 → 192.168.0.112
    ↓
Proxmox empfängt, antwortet
    ↓
✅ Laptop erhält Antwort
```

---

## Die Lösung

### Korrekte Container-Konfiguration

```bash
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.0.112
    netmask 255.255.255.0
    # KEIN gateway!
    # KEIN dns-nameservers!
```

### Fix durchführen

```bash
# 1. Im Container Config bearbeiten
ssh root@192.168.0.112
sudo nano /etc/network/interfaces
# Gateway-Zeile löschen, speichern

# 2. Container neu starten
sudo reboot

# 3. Auf dem Laptop Tailscale neu verbinden
sudo tailscale down
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes

# 4. Testen
ping 192.168.0.101  # ✅ Proxmox
ping 192.168.0.110  # ✅ Windows
ping 192.168.0.111  # ✅ Debian
```

---

## Wichtige Konzepte

### 1. Router vs. Host

**Host (mit Gateway):**
- Normale PCs, Laptops, Server
- "Für alles Unbekannte → Gateway fragen"
- Hat EIN Netzwerk-Interface

**Router (ohne Gateway):**
- VPN Exit-Nodes, Firewalls, Subnet Router
- "Ich VERBINDE Netzwerke, ich entscheide selbst"
- Hat MEHRERE Netzwerk-Interfaces

### 2. Policy-Based Routing (Table 52)

```bash
# Tailscale nutzt eigene Routing-Table
ip route show table 52
192.168.0.0/24 dev tailscale0  # ← Subnet Route

# Main Table bleibt unberührt
ip route show
default via 192.168.0.1 dev wlan0
```

**Vorteil:** Keine Konflikte zwischen VPN und normalem Traffic

### 3. NAT/MASQUERADE

```bash
# Im Container
iptables -t nat -L POSTROUTING -n
MASQUERADE  all  --  100.64.0.0/10  0.0.0.0/0
```

**Was es macht:**
- Ändert Source-IP von VPN-Clients (100.64.0.x)
- Zu Container-IP (192.168.0.112)
- Damit Ziel-Server antworten können

### 4. IP Forwarding

```bash
# Muss 1 sein!
sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

**Ohne:** Container verwirft fremde Pakete
**Mit:** Container leitet Pakete zwischen Interfaces weiter

### 5. Was ist 0.0.0.0/0?

| Kontext | Bedeutung |
|---------|-----------|
| Routing Table | Default Route - "alles was ich nicht kenne" |
| Exit-Node | Kompletter Internet-Traffic |
| Server Binding | "Höre auf allen Interfaces" |

```bash
# Exit-Node advertised:
--advertise-routes=0.0.0.0/0  # = Alles
--advertise-routes=192.168.0.0/24  # = Nur Heimnetz
```

---

## Troubleshooting-Checkliste

```bash
# 1. Container: Kein Gateway?
cat /etc/network/interfaces
# Sollte KEIN "gateway" haben

# 2. Container: IP Forwarding aktiv?
sysctl net.ipv4.ip_forward
# Muss 1 sein

# 3. Container: Routing Table (kein default!)
ip route show
# 192.168.0.0/24 dev eth0
# 100.64.0.0/10 dev tailscale0
# KEIN "default via"!

# 4. Container: NAT/MASQUERADE?
iptables -t nat -L POSTROUTING -n
# MASQUERADE  all  --  *  eth0  100.64.0.0/10

# 5. Headscale: Routes approved?
sudo headscale routes list
# Alle Routes müssen "Enabled: true" sein

# 6. Laptop: Tailscale Routes aktiv?
ip route show table 52 | grep 192.168.0
# 192.168.0.0/24 dev tailscale0
```

---

## LAP-Wissen

### Frage: "Warum braucht ein Subnet Router kein Gateway?"

**Antwort:** Ein Router verbindet Netzwerke und routet selbst zwischen ihnen. Ein Gateway würde bedeuten "Ich kenne mich nicht aus, frag jemand anderen" - aber der Router SOLL sich auskennen! Er ist selbst das Gateway für andere Geräte.

### Frage: "Was ist Policy-Based Routing?"

**Antwort:** Linux nutzt mehrere Routing-Tables für verschiedene Zwecke. Die Main-Table für normalen Traffic, Table 52 für Tailscale. Der Kernel entscheidet automatisch welche Table verwendet wird. Das verhindert Konflikte.

### Frage: "Warum braucht man NAT/MASQUERADE bei einem Subnet Router?"

**Antwort:** Weil das Zielsystem (z.B. Proxmox) die VPN-IP (100.64.0.x) nicht kennt. MASQUERADE ändert die Source-IP zur Container-IP (192.168.0.112), die Proxmox kennt. So kann Proxmox antworten.

### Frage: "Was bedeutet 0.0.0.0/0?"

**Antwort:** Eine Default-Route - alle Pakete zu unbekannten Zielen werden hierhin geroutet. Das /0 bedeutet "keine Netzwerk-Bits festgelegt", also matched es ALLE IP-Adressen.

---

## Merksatz

```
🖥️ Host = Hat Gateway (fragt bei Unbekanntem)
🌐 Router = Kein Gateway (entscheidet selbst)
```

**Ein System kann entweder Host ODER Router sein - nicht beides!**

---

## Netzwerk-Übersicht (Referenz)

| Gerät | IP | Typ |
|-------|-----|-----|
| Router | 192.168.0.1 | Gateway |
| Proxmox | 192.168.0.101 | Host (statisch) |
| Windows Server | 192.168.0.110 | Host (statisch) |
| Debian Server | 192.168.0.111 | Host (statisch) |
| Tailscale LXC | 192.168.0.112 | **Router** (statisch, KEIN Gateway!) |
| Laptop | 192.168.0.95 | Host (DHCP) |

---

*Dokumentiert am 19. Januar 2026 - Troubleshooting-Session für LAP-Vorbereitung*
