# VPN Troubleshooting Guide

Sammlung aller VPN-Probleme und Lösungen.

---

## Schnell-Checkliste

Bei VPN-Problemen diese Punkte prüfen:

```bash
# 1. Tailscale läuft?
tailscale status

# 2. Routes genehmigt? (auf VPS)
sudo headscale routes list

# 3. IP Forwarding an? (im Container)
sysctl net.ipv4.ip_forward  # Muss 1 sein

# 4. NAT aktiv? (im Container)
sudo iptables -t nat -L POSTROUTING -n

# 5. Firewall blockiert? (im Container)
sudo iptables -L FORWARD -n
```

---

## Problem: Container offline nach Neustart

### Symptome

```bash
tailscale status
# 100.64.0.1  tailscale  offline, last seen 7m ago
```

### Ursachen

1. **Headscale löschte Node** - Container war >15min offline → Auto-Cleanup
2. **Kein Gateway** - Container konnte Headscale nicht erreichen
3. **IP Forwarding = 0** - Container konnte nicht routen

### Lösungen

#### 1. Gateway permanent in Proxmox Config setzen

LXC Container laden `/etc/network/interfaces` nicht automatisch!

```bash
# /etc/pve/lxc/102.conf
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:D8:A7:B2,ip=192.168.0.112/24,gw=192.168.0.1,type=veth
```

#### 2. IP Forwarding Service erstellen

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

#### 3. Auto-Start aktivieren

```bash
pct set 102 --onboot 1
pct set 102 --startup order=1,up=30
```

---

## Problem: Gateway in Subnet Router konfiguriert

### Symptome

Proxmox und alle Server nicht erreichbar:

```bash
ping 192.168.0.101  # Proxmox     - ❌ FAILED
ping 192.168.0.110  # Windows     - ❌ FAILED
ping 192.168.0.112  # Container   - ✅ SUCCESS
```

### Ursache

**Ein Subnet Router braucht KEIN Gateway!**

| Typ | Gateway? | Verhalten |
|-----|----------|-----------|
| **Normaler Host** | JA | "Was ich nicht kenne → Gateway fragen" |
| **Router** | NEIN | "Ich VERBINDE Netzwerke, ich entscheide selbst!" |

### Was passiert (falsch):

```
Laptop → "Ich will zu 192.168.0.101"
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
Container: "192.168.0.101 ist in 192.168.0.0/24 → über eth0 erreichbar!"
    ↓
NAT/MASQUERADE: Source 100.64.0.2 → 192.168.0.112
    ↓
✅ Proxmox empfängt, antwortet
```

### Lösung

**Container-Konfiguration ohne Gateway:**

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

**Merksatz:**
```
Host = Hat Gateway (fragt bei Unbekanntem)
Router = Kein Gateway (entscheidet selbst)
```

---

## Problem: Routes nicht sichtbar in "ip route"

### Symptome

```bash
ip route
# Zeigt KEINE Tailscale-Routes!
```

### Erklärung

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
```

### Beispiel-Output

```bash
ip route show table 52
# 100.64.0.1 dev tailscale0 table 52        # Container
# 100.64.0.3 dev tailscale0 table 52        # Anderer Node
# 192.168.0.0/24 dev tailscale0 table 52    # Subnet Route ✅
# 100.100.100.100 dev tailscale0 table 52   # Tailscale DNS
```

### Warum macht Tailscale das?

- Keine Konflikte mit normalen Routes
- Saubere Trennung VPN vs. Internet-Traffic
- Automatisches Failover wenn VPN ausfällt

---

## Problem: Keine Verbindung zu Heimnetz-Servern

### Checkliste

```bash
# 1. Routes genehmigt? (auf VPS)
sudo headscale routes list

# 2. IP Forwarding an? (im Container)
sysctl net.ipv4.ip_forward  # Muss 1 sein!

# 3. NAT aktiv? (im Container)
sudo iptables -t nat -L POSTROUTING -n
# Sollte MASQUERADE zeigen

# 4. Firewall blockiert? (im Container)
sudo iptables -L FORWARD -n

# 5. Tailscale läuft? (auf Client)
tailscale status
```

### Falls IP Forwarding aus:

```bash
# Temporär
sudo sysctl -w net.ipv4.ip_forward=1

# Dauerhaft
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Falls NAT fehlt:

```bash
# Im Container
sudo iptables -t nat -A POSTROUTING -o eth0 -s 100.64.0.0/10 -j MASQUERADE

# Dauerhaft machen
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### Falls Firewall blockiert:

```bash
sudo iptables -I FORWARD -i tailscale0 -j ACCEPT
sudo iptables -I FORWARD -o tailscale0 -j ACCEPT
```

---

## Problem: Exit-Node funktioniert nicht

### Symptome

```bash
curl ifconfig.me
# Zeigt immer noch mobile/andere IP statt Heimnetz-IP
```

### Checkliste

```bash
# 1. Exit-Node aktiviert auf VPS?
sudo headscale routes list | grep "0.0.0.0/0"
# Muss "Enabled: true" sein

# 2. Exit-Node auf Client aktiviert?
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# 3. IP checken
curl ifconfig.me  # Sollte deine Heimnetz-IP sein
```

---

## Problem: Headscale Service läuft nicht

### Diagnose

```bash
# Service Status
sudo systemctl status headscale

# Logs anschauen
sudo journalctl -u headscale -f

# Config testen
sudo headscale serve  # Manuell starten zum Debuggen
```

### Häufige Ursachen

1. **Config-Syntax-Fehler** - YAML überprüfen
2. **Port bereits belegt** - `netstat -tuln | grep 8090`
3. **Berechtigungen** - `/var/lib/headscale/` prüfen

---

## Problem: Nodes offline nach Headscale-Neustart

### Lösung

Tailscaled auf allen Nodes neustarten:

```bash
sudo systemctl restart tailscaled
```

---

## Problem: MagicDNS (home.lab) funktioniert nicht

### Checkliste

1. **Headscale läuft?**
   ```bash
   sudo systemctl status headscale
   ```

2. **DNS in Config?**
   ```bash
   grep -A5 "extra_records" /etc/headscale/config.yaml
   ```

3. **Client hat --accept-dns?**
   ```bash
   sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-dns --accept-routes
   ```

4. **Headscale neustarten:**
   ```bash
   sudo systemctl restart headscale
   ```

---

## Packet Flow verstehen

### Der komplette Pfad

```
┌─────────┐        ┌───────────┐        ┌─────────┐
│ Laptop  │───────►│ Container │───────►│ Proxmox │
│.0.2     │        │   .0.1    │        │  .0.101 │
└─────────┘        └───────────┘        └─────────┘

1. Laptop: "Sende Paket zu 192.168.0.101"
2. Routing Table 52: "192.168.0.0/24 via tailscale0"
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

### Warum MASQUERADE?

```
Ohne MASQUERADE:
Laptop sendet:    100.64.0.2 → 192.168.0.101
Proxmox denkt:    "WTF ist 100.64.0.2? Kenne ich nicht!"
                  → Paket verworfen ❌

Mit MASQUERADE:
Container ändert: 100.64.0.2 → 192.168.0.112
Proxmox denkt:    "Ah, Container! Den kenne ich!"
                  → Antwortet an Container ✅
```

---

## Netzwerk Debugging Befehle

### Routing

```bash
ip route                    # Main Table
ip route show table 52      # Tailscale Table
ip route show table all     # Alle Tables
ip route get <ip>           # Welche Route wird verwendet?
```

### iptables

```bash
iptables -L -n -v              # Filter table
iptables -t nat -L -n -v       # NAT table
iptables -L FORWARD -n -v      # Forward chain
```

### Traffic beobachten

```bash
tcpdump -i tailscale0 -n       # Traffic auf Tailscale Interface
tcpdump -i any host <ip>       # Traffic zu bestimmter IP
tcpdump -i any icmp            # Nur ICMP (Ping)
```

### Connections

```bash
ss -tuln                   # Listening Ports
netstat -rn                # Routing Table
ping <ip>                  # Einfacher Test
traceroute <ip>            # Route verfolgen
```

### Tailscale-spezifisch

```bash
tailscale status           # Alle Nodes
tailscale netcheck         # Verbindungsqualität
ip addr show tailscale0    # Interface Details
ip route show table all | grep tailscale  # Alle Tailscale-Routes
```

---

## LAP Prüfungsfragen

### "Warum braucht ein Subnet Router kein Gateway?"

Ein Router verbindet Netzwerke und routet selbst zwischen ihnen. Ein Gateway würde bedeuten "Ich kenne mich nicht aus, frag jemand anderen" - aber der Router SOLL sich auskennen! Er ist selbst das Gateway für andere Geräte.

### "Was ist Policy-Based Routing?"

Linux nutzt mehrere Routing-Tables für verschiedene Zwecke. Die Main-Table für normalen Traffic, Table 52 für Tailscale. Der Kernel entscheidet automatisch welche Table verwendet wird. Das verhindert Konflikte.

### "Warum braucht man NAT/MASQUERADE bei einem Subnet Router?"

Weil das Zielsystem (z.B. Proxmox) die VPN-IP (100.64.0.x) nicht kennt. MASQUERADE ändert die Source-IP zur Container-IP (192.168.0.112), die Proxmox kennt. So kann Proxmox antworten.

### "Was bedeutet 0.0.0.0/0?"

Eine Default-Route - alle Pakete zu unbekannten Zielen werden hierhin geroutet. Das /0 bedeutet "keine Netzwerk-Bits festgelegt", also matched es ALLE IP-Adressen.

---

*Siehe auch: [vpn-infrastructure.md](vpn-infrastructure.md) | [vpn-client-guide.md](vpn-client-guide.md)*
