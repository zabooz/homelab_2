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

## 🖥️ Master-VMs (Proxmox homeserver2)

OS-Images werden in Proxmox-VMs auf homeserver2 (192.168.0.102) erstellt und via FOG captured.

| VMID | Name | OS | Bootloader | Status |
|------|------|----|------------|--------|
| 500 | elementaryOS-master | elementary OS | GRUB | OEM-Image (Secure Boot deaktiviert) |
| 501 | arch-master | Arch Linux | Limine | Image (siehe Limine-Hinweise unten) |
| 503 | fedora-master | Fedora 43 Workstation | GRUB | OEM-Image |
| 504 | linuxMint-master | Linux Mint | GRUB | OEM-Image |

### VM-Konfiguration (alle VMs gleich)

```
bios: ovmf
machine: q35
cpu: host
cores: 4
scsihw: virtio-scsi-single
scsi0: ssd-storage:32G
efidisk0: ssd-storage:4M,efitype=4m
net0: virtio,bridge=vmbr0
boot: order=net0;scsi0
```

> **Wichtig: VM-Erstellung**
> Alle Master-VMs müssen mit `cpu: host` erstellt werden. Andere CPU-Typen (z.B. `x86-64-v2-AES`) können dazu führen, dass PXE Boot in OVMF nicht funktioniert. Bei PXE-Problemen: VM komplett neu erstellen statt reparieren.

> **Wichtig: EFI Disk**
> Die EFI Disk (`efitype=4m`) darf NICHT mit `pre-enrolled-keys` erstellt werden, da sonst PXE Boot nicht funktioniert. Standard `efitype=4m` ohne Zusatzoptionen verwenden.

### FOG EFI Exit Type

Für UEFI-VMs und UEFI-Hardware muss in FOG der **EFI Exit Type** korrekt eingestellt sein:

- **FOG Web UI → Host Management → [Host] → General → EFI Exit Type**
- **Empfohlen: `Sanboot`** für Proxmox OVMF und die meisten UEFI-Systeme
- `Sanboot` lässt iPXE die lokale Disk direkt booten (zuverlässiger als `Exit`)
- `Refind` funktioniert ebenfalls, ist aber langsamer (Zwischenschritt über rEFInd Boot-Manager)

| Exit Type | Beschreibung | Empfehlung |
|-----------|-------------|------------|
| Sanboot | iPXE bootet Disk direkt | Empfohlen |
| Refind | Zwischenschritt über rEFInd | Alternative |
| Exit | iPXE beendet sich, Firmware übernimmt | Funktioniert oft nicht mit OVMF |
| Grub | Chainload über GRUB EFI | Möglich |

> **Bemerkung: elementary OS und Secure Boot**
> elementary OS (VM 500) wurde mit `pre-enrolled-keys=1` erstellt. Secure Boot muss im OVMF Setup (ESC beim Boot → Device Manager → Secure Boot Configuration) **deaktiviert** werden, damit sowohl FOG PXE Boot als auch normaler Disk Boot funktionieren. Die `pre-enrolled-keys=1` Einstellung in der Proxmox-Config kann bleiben — Secure Boot wird nur in der NVRAM deaktiviert.

### Limine Bootloader und FOG (Arch Linux)

Limine installiert sich unter `EFI/limine/BOOTX64.EFI` und registriert einen NVRAM-Eintrag via `efibootmgr`. Das funktioniert beim direkten Disk-Boot, aber **nicht nach FOG-Deployment**, weil:

1. **NVRAM-Einträge gehen verloren** — FOG deployed nur die Disk-Partitionen, nicht die UEFI-NVRAM. Auf dem Ziel-PC existiert kein Boot-Eintrag für `EFI/limine/`.
2. **Sanboot sucht den UEFI-Fallback-Pfad** — iPXE/Sanboot bootet die Disk über den Standard-Pfad `EFI/BOOT/BOOTX64.EFI`. Wenn dort nichts liegt, findet Sanboot (und auch Refind/Grub) den Bootloader nicht.

**Fix:** Eine Kopie der Limine-EFI-Datei als UEFI-Fallback anlegen. Die originale Limine-Installation unter `EFI/limine/` bleibt unverändert:

```bash
mkdir -p /boot/EFI/BOOT
cp /boot/EFI/limine/BOOTX64.EFI /boot/EFI/BOOT/BOOTX64.EFI
```

> **Wichtig:** Dieser Schritt muss vor jedem FOG Capture wiederholt werden, falls Limine aktualisiert wurde (z.B. nach `pacman -Syu`), damit die Fallback-Kopie aktuell bleibt.

### Bootloader root= Parameter

Bootloader-Configs dürfen **nicht** `/dev/sdaX` oder `/dev/nvme0n1pX` als `root=` verwenden — Devicenamen können sich auf anderer Hardware ändern. Stattdessen `PARTUUID` nutzen:

```bash
# PARTUUID ermitteln:
blkid /dev/sda3

# In limine.conf (oder GRUB etc.):
# Statt:  root=/dev/sda3
# Nutze:  root=PARTUUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## 🖼️ Image-Strategien

### Linux OEM Images (empfohlen für Deployment)

OEM-Modus installiert das System ohne persönlichen Benutzer. Beim ersten Boot durch den Endbenutzer erscheint ein Einrichtungs-Wizard.

**Unterstützte Distros:**

| Distro | OEM-Modus | Mechanismus |
|--------|-----------|-------------|
| Linux Mint | Expliziter OEM-Modus im Installer | Calamares |
| Ubuntu | Expliziter OEM-Modus im Installer | Ubiquity/Subiquity |
| Fedora | User-Erstellung überspringen | GNOME Initial Setup beim ersten Boot |

**Workflow:**
```
1. Linux in Master-VM installieren (OEM-Modus / ohne User)
2. initramfs universell machen (siehe unten)
3. Limine: EFI/BOOT Fallback anlegen (siehe oben)
4. truncate -s 0 /etc/machine-id
5. Herunterfahren (NICHT rebooten!)
6. VM über PXE/FOG booten → Image capturen
7. Image auf Ziel-PCs deployen
```

> **Wichtig: initramfs für echte Hardware**
> - **Mint/Ubuntu (aktuell):** Nutzt `initramfs-tools`, das standardmäßig portable initrds baut — enthält alle gängigen Hardware-Treiber (AHCI, NVMe, SATA, USB). Funktioniert ohne Änderung auf beliebiger Hardware.
> - **Fedora:** Nutzt `dracut` mit `--hostonly` als Default — enthält **nur** die Treiber der aktuellen Maschine (= VirtIO in der VM). Auf echtem PC mit AHCI/NVMe **kommt Kernel Panic** `block(0,0)`. Fix: `dracut --regenerate-all --force --no-hostonly` vor dem Capture ausführen.
> - **Arch/CachyOS:** Nutzt `mkinitcpio` mit `autodetect`-Hook als Default — gleiches Problem wie Fedora, nur VirtIO-Treiber im initramfs. Fix: `autodetect` aus der HOOKS-Zeile in `/etc/mkinitcpio.conf` entfernen und `mkinitcpio -P` ausführen. Ohne diesen Fix kommt Kernel Panic (`root fs not found`) auf echter Hardware.
> - **Hinweis:** Ubuntu 25.10+ wechselt auf dracut. Zukünftige Mint-Versionen könnten ebenfalls betroffen sein — dann gilt der gleiche Fix wie bei Fedora.

> **Wichtig: machine-id**
> `/etc/machine-id` muss vor dem Capture geleert werden (`truncate -s 0 /etc/machine-id`). Sonst bekommen alle deployten PCs die gleiche ID → DHCP-Konflikte, identische D-Bus IDs.

---

### Linux Images (ohne OEM)

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

#### Mit Sysprep (empfohlen)
```
1. Windows + Software installieren
2. sysprep.exe → OOBE + Generalize + Shutdown
3. Capture
4. Deploy → hardwareflexibel
```

#### Ohne Sysprep
```
- Nur identische Hardware
- Schnell, aber unflexibel
```

---

## 📋 Workflow

### Image erstellen
```
FOG Web → Image Management → Create
  - Image Type: Single Disk - Resizable
  - Image Partition Type: GPT (für UEFI)
  - OS Type: Linux
FOG Web → Host Management → Create
  - EFI Exit Type: Sanboot (für UEFI)
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

