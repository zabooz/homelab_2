# Proxmox Netzwerk-Setup - Statische IP-Konfiguration

**Erstellt:** 19. Januar 2026
**Author:** Daniel (zabooz)
**Zweck:** LAP-Vorbereitung IT-Systemtechnik

---

## Übersicht

Anleitung zur Konfiguration statischer IP-Adressen für VMs und LXC-Container in Proxmox VE.

**Siehe auch:**
- [NETWORK_OVERVIEW.md](../NETWORK_OVERVIEW.md) - Master-Diagramm und IP-Liste
- [core-concepts.md](../core-concepts.md) - Netzwerk-Grundlagen (DHCP, Subnetting, etc.)

---

## IP-Adressen

| Gerät | IP-Adresse | Typ | Interface |
|-------|------------|-----|-----------|
| Proxmox Host | 192.168.0.101 | Statisch | vmbr0 |
| Windows Server (VM 100) | 192.168.0.110 | Statisch | Ethernet |
| Debian Server (VM 101) | 192.168.0.111 | Statisch | ens18 |
| Tailscale LXC (CT 102) | 192.168.0.112 | Statisch | eth0 |
| FOG Server (LXC 103) | 192.168.0.113 | Statisch | eth0 |

**Netzwerk-Parameter:**
- Gateway: 192.168.0.1
- Subnetzmaske: 255.255.255.0 (/24)
- DNS: 192.168.0.1, 8.8.8.8

---

## Konfiguration

### 1. Proxmox VE Host

Bereits konfiguriert - Bridge vmbr0 mit statischer IP.

```bash
# IP-Konfiguration anzeigen
ip addr show vmbr0
```

---

### 2. Windows Server (VM 100)

#### GUI-Methode:

1. `Win+R` → `ncpa.cpl` → Enter
2. Rechtsklick "Ethernet" → Eigenschaften
3. Doppelklick "IPv4 (TCP/IPv4)"
4. Eintragen:
   ```
   IP-Adresse:        192.168.0.110
   Subnetzmaske:      255.255.255.0
   Standardgateway:   192.168.0.1

   Bevorzugter DNS:   192.168.0.1
   Alternativer DNS:  8.8.8.8
   ```
5. OK → OK → Schließen

#### PowerShell-Methode:

```powershell
# Statische IP setzen
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.0.110 -PrefixLength 24 -DefaultGateway 192.168.0.1

# DNS-Server setzen
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.0.1","8.8.8.8")

# Verifizieren
ipconfig /all
ping 192.168.0.1
ping google.com
```

---

### 3. Debian Server (VM 101)

```bash
# Backup erstellen
sudo cp /etc/network/interfaces /etc/network/interfaces.bak

# Datei bearbeiten
sudo nano /etc/network/interfaces
```

**Konfiguration:**

```bash
# Loopback
auto lo
iface lo inet loopback

# Statische IP
auto ens18
iface ens18 inet static
    address 192.168.0.111
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8

# IPv6 (SLAAC)
iface ens18 inet6 auto
```

**Wichtig:** `auto` statt `allow-hotplug` für Server!

```bash
# Netzwerk neustarten
sudo systemctl restart networking

# Verifizieren
ip addr show ens18
ping -c 4 192.168.0.1
ping -c 4 google.com
```

---

### 4. Tailscale LXC (CT 102)

```bash
# Datei bearbeiten
nano /etc/network/interfaces
```

**Konfiguration:**

```bash
# Loopback
auto lo
iface lo inet loopback

# Statische IP
auto eth0
iface eth0 inet static
    address 192.168.0.112
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 192.168.0.1 8.8.8.8
```

```bash
# Netzwerk neustarten
systemctl restart networking

# Verifizieren
ip addr show eth0
tailscale status
```

**Hinweis:** Die Tailscale-IP (100.64.0.1) bleibt unverändert!

---

### 5. FOG Server (LXC 103)

Gleiche Methode wie Tailscale LXC, aber mit IP 192.168.0.113.

---

## Proxmox-spezifische Befehle

```bash
# Bridge-Informationen
brctl show
ip link show vmbr0

# VM/LXC-Management
qm list                         # Alle VMs
pct list                        # Alle LXC-Container
qm status <vmid>                # VM-Status
pct status <ctid>               # Container-Status

# Netzwerk-Config anzeigen
cat /etc/network/interfaces

# Netzwerk neu laden (VORSICHT - bricht SSH!)
ifreload -a
```

---

## Troubleshooting

### Keine Verbindung nach Umstellung

1. **IP prüfen:**
   ```bash
   ip addr show <interface>    # Linux
   ipconfig /all               # Windows
   ```

2. **Gateway erreichbar?**
   ```bash
   ping 192.168.0.1
   ```
   Wenn nicht → IP/Subnetzmaske falsch

3. **Internet aber kein DNS?**
   ```bash
   ping 8.8.8.8      # Funktioniert?
   ping google.com   # Funktioniert nicht?
   ```
   → DNS-Problem, `dns-nameservers` in config prüfen

4. **Routing-Tabelle:**
   ```bash
   ip route show     # Linux
   route print       # Windows
   ```
   Default-Route vorhanden?

### Netzwerk nach Reboot weg (Linux)

**Ursache:** `allow-hotplug` statt `auto`

**Lösung:** In `/etc/network/interfaces`:
```bash
auto ens18          # RICHTIG für Server
# allow-hotplug ens18  # FALSCH für Server
```

### IP-Konflikt

```bash
# Konflikt finden
sudo arping -D -I ens18 192.168.0.110

# Alle IPs im Netz scannen
nmap -sn 192.168.0.0/24
```

---

## Verifizierung

Nach der Konfiguration von jedem System aus testen:

```bash
ping 192.168.0.1     # Gateway
ping 192.168.0.101   # Proxmox
ping 192.168.0.110   # Windows
ping 192.168.0.111   # Debian
ping 192.168.0.112   # Tailscale LXC
ping 8.8.8.8         # Internet
ping google.com      # DNS
```

Alle Pings sollten erfolgreich sein mit < 1ms im lokalen Netz.

---

*Letzte Aktualisierung: Januar 2026*
