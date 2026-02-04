---
title: FOG Project Setup
description: DHCP Server, PXE Boot und Imaging mit FOG Project
published: true
date: 2026-01-20T00:00:00.000Z
tags: service, dhcp, pxe, imaging, netzwerk, network, proxmox, lxc
editor: markdown
dateCreated: 2026-01-20T00:00:00.000Z
---

# 📝 FOG Project – Setup Dokumentation

**Erstellt:** 20. Januar 2026  
**Author:** Daniel (zabooz)  
**System:** FOG 1.5.10 auf Debian 12 (LXC)

---

## 🎯 Was ist FOG?
**FOG Project** ist eine Open-Source-Lösung für OS-Deployment über das Netzwerk.

**Features:**
- Image Capture & Deploy
- PXE Network Boot
- Multicast (mehrere PCs gleichzeitig)
- Hardware-Inventar
- Unterstützt Windows, Linux, macOS

**Siehe auch:**
- [DHCP](/en/konzepte/dhcp) - DORA-Prozess, DHCP-Server Grundlagen
- [IP-Adressen](/en/konzepte/ip-adressen) - Statische vs. Dynamische IPs

---

## 🌐 Netzwerk-Setup

- **Router / Gateway:** `192.168.0.1`  
  ⚠️ DHCP **DEAKTIVIERT**
- **FOG Server:** `192.168.0.113`
- **DHCP Range (FOG):** `192.168.0.200 – 192.168.0.250`

**Statische IPs (bleiben erhalten):**
- Proxmox: `192.168.0.101`
- Andere Server: `192.168.0.110 – 192.168.0.112`

> ❗ Wichtig: Router-DHCP **muss AUS** sein – FOG übernimmt DHCP vollständig.

---

## 🔧 LXC Container (Proxmox)

- **CT ID:** 103
- **Hostname:** `fog-server`
- **Template:** Debian 12
- **Privileged:** ✅ YES (**wichtig!**)
- **IP:** `192.168.0.113/24`
- **RAM:** 4 GB
- **CPU:** 2 Cores
- **Disk:** 200 GB
- **Features:**
  - `nesting=YES`
  - `keyctl=YES`

---

## 📦 Installation

```bash
ssh root@192.168.0.113

apt update && apt upgrade -y
apt install git -y

cd /root
git clone https://github.com/FOGProject/fogproject.git
cd fogproject/bin
./installfog.sh
```

### Wichtige Antworten im Installer
```
Linux: Debian (2)
Type: Normal (N)
IP: 192.168.0.113
Router: 192.168.0.1
DNS: N (Router macht DNS)
FOG DHCP: Y
Range: 192.168.0.200 - 192.168.0.250
```

---

## ⚙️ Wichtige Konfigurationsdateien

### DHCP Server
**Datei:** `/etc/dhcp/dhcpd.conf`

```conf
subnet 192.168.0.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    range dynamic-bootp 192.168.0.200 192.168.0.250;
    default-lease-time 21600;
    max-lease-time 43200;
    option routers 192.168.0.1;
    next-server 192.168.0.113;
}

class "Legacy" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00000";
    filename "undionly.kkpxe";
}

class "UEFI-64-1" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00007";
    filename "snponly.efi";
}
```

**Erklärung:**
- `range` → IP-Bereich für PXE-Clients
- `option routers` → Gateway bleibt der Router
- `next-server` → FOG/TFTP Server

---

### FOG Settings
**Datei:** `/opt/fog/.fogsettings`

```bash
# Wird bei der Installation erstellt
# Enthält alle FOG-Einstellungen
# Bei Neuinstallation wird diese Datei wiederverwendet
```

> 🧹 Bei Problemen:  
> `rm /opt/fog/.fogsettings`

---

## 🌐 DNS Handling

### Option 1: Router macht DNS (empfohlen / gewählt)
```conf
# Keine DNS-Option gesetzt
# Clients fragen automatisch den Router
```

**Vorteile:**
- Einfach
- Router kennt lokale Hosts
- Nutzt Provider-DNS

### Option 2: DNS explizit setzen
```conf
option domain-name-servers 192.168.0.1, 8.8.8.8;
```

### Option 3: AdGuard / Pi-hole (später)
```conf
option domain-name-servers 192.168.0.120, 192.168.0.1;
```

---

## 🚀 PXE Boot Prozess

```
1. PC startet → F12 → Network Boot
2. DHCP Request (Broadcast)
3. FOG DHCP antwortet:
   - IP aus Range
   - Gateway: 192.168.0.1
   - Boot-Server: 192.168.0.113
   - Boot-File: undionly.kpxe / snponly.efi
4. Download via TFTP
5. iPXE Menü erscheint
6. Capture / Deploy / Register
```

---

## 🖼️ Image-Strategien

### Linux Images

- Kein Sysprep nötig
- Hardware-unabhängig
- Sehr stabil

**Workflow:**
```
1. Linux installieren + konfigurieren
2. Image capturen
3. Auf andere Hardware deployen
```

---

### Windows Images

#### ✅ Mit Sysprep (empfohlen)
```
1. Windows + Software installieren
2. sysprep.exe → OOBE + Generalize + Shutdown
3. Capture
4. Deploy → hardwareflexibel
```

#### ❌ Ohne Sysprep
```
- Nur identische Hardware
- Schnell, aber unflexibel
```

---

## 📋 Workflow

### Image erstellen
```
FOG Web → Image Management → Create
FOG Web → Host Management → Create
Host → Basic Tasks → Capture
```

### Image deployen
```
PXE Boot → Quick Registration
FOG Web → Host → Deploy
```

---

## ⚙️ Services

```bash
systemctl status FOGMulticastManager
systemctl status FOGImageReplicator
systemctl status FOGScheduler
systemctl status isc-dhcp-server
systemctl status apache2
systemctl status mariadb
```

---

## 🎓 Wichtige Konzepte

- **DHCP Option 66:** TFTP Server
- **DHCP Option 67:** Boot-Datei
- **PXE:** Preboot Execution Environment
- **TFTP:** Boot-Datei-Transfer
- **Resizable Images:** Diskgrößen-unabhängig
- **Multicast:** Ein Stream, viele Clients

---

## ✅ Checkliste

- [x] LXC Container privileged
- [x] FOG installiert
- [x] Web-UI erreichbar
- [x] Passwort geändert
- [x] Router-DHCP deaktiviert
- [x] PXE Boot erfolgreich

---

## 🌐 Web Interface

- **URL:** http://192.168.0.113/fog/management
- **Login:** `fog` / *dein Passwort*

