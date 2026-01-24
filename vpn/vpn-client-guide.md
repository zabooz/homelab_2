# Tailscale Client Setup Guide

**Server:** `https://zabooz.duckdns.org`

---

## Quick Start - Neues Gerät hinzufügen

### 1. User erstellen (auf dem VPS)

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

### 2. Tailscale installieren

### 3. Gerät verbinden

### 4. Node registrieren (auf dem VPS)

### 5. Überprüfen

---

## Download & Installation

| Platform | Link |
|----------|------|
| Linux | https://tailscale.com/download/linux |
| Windows | https://tailscale.com/download/windows |
| macOS | https://tailscale.com/download/mac |
| Android | https://play.google.com/store/apps/details?id=com.tailscale.ipn |
| iOS | https://apps.apple.com/app/tailscale/id1470499037 |

---

## Linux

### Quick Install (auto-detects distro)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Manual Install

**Debian / Ubuntu:**
```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt update
sudo apt install tailscale
```

**Fedora:**
```bash
sudo dnf config-manager --add-repo https://pkgs.tailscale.com/stable/fedora/tailscale.repo
sudo dnf install tailscale
sudo systemctl enable --now tailscaled
```

**Arch / CachyOS:**
```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
```

### Verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-dns --accept-routes
```

Das Gerät zeigt dir jetzt einen Key an:
```
To authenticate, visit:
  https://zabooz.duckdns.org/register/nodekey:abc123def456...

Or run:
  headscale nodes register --key nodekey:abc123def456... --user USERNAME
```

**Kopiere den `nodekey:xxxxxxxxx`** und registriere auf dem VPS.

---

## Windows

### Installation

1. Download: https://tailscale.com/download/windows
2. Installieren und starten

### Verbinden (PowerShell als Admin)

```powershell
tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

### GUI-Methode

1. Tailscale Icon im System Tray → Settings
2. "Use custom coordination server" aktivieren
3. URL eingeben: `https://zabooz.duckdns.org`
4. Enable: **Accept Routes** + **Accept DNS**
5. Connect

---

## macOS

1. Download von https://tailscale.com/download/mac
2. Install und öffnen
3. Settings → "Use custom coordination server"
4. URL: `https://zabooz.duckdns.org`
5. Enable: Accept Routes + Accept DNS
6. Connect und authorize

---

## Android / iOS

1. Tailscale App installieren (App Store / Play Store)
2. **Vor dem Einloggen:** Settings → "Use custom coordination server"
3. URL eingeben: `https://zabooz.duckdns.org`
4. Enable: **Accept DNS** + **Accept Routes**
5. Connect und authorize

---

## Node Registrierung (VPS)

Nach dem Verbinden eines neuen Geräts:

```bash
# SSH zum VPS
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

## Verifizierung

### Auf dem neuen Gerät

```bash
# Status checken
tailscale status

# Proxmox testen
ping 192.168.0.101

# Browser öffnen
firefox https://192.168.0.101:8006
```

### Weitere Checks

```bash
# Verbindungsqualität
tailscale netcheck

# Ping über Tailscale
tailscale ping <ip>

# DNS testen (MagicDNS)
ping home.lab
```

---

## Flags Referenz

### Erforderliche Flags

| Flag | Zweck | Ohne diesen Flag |
|------|-------|------------------|
| `--login-server=https://zabooz.duckdns.org` | Mit Headscale verbinden | Verbindet zu Tailscale statt Headscale |
| `--accept-dns` | MagicDNS nutzen | `home.lab` funktioniert nicht |
| `--accept-routes` | Subnet Routes akzeptieren | Kein Zugriff auf `192.168.0.x` |

### Optionale Flags

| Flag | Zweck |
|------|-------|
| `--exit-node=100.64.0.1` | Kompletten Traffic über Heimnetz routen |
| `--reset` | Einstellungen zurücksetzen |
| `--advertise-routes=<subnet>` | Selbst als Subnet Router agieren |
| `--advertise-exit-node` | Selbst als Exit-Node agieren |

---

## Häufige Befehle

### Verbindung

```bash
# Verbinden mit allen Optionen
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns

# Temporär trennen
sudo tailscale down

# Komplett abmelden
sudo tailscale logout

# Neu verbinden nach Logout
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```

### Status

```bash
# Alle verbundenen Nodes
tailscale status

# Verbindungsqualität
tailscale netcheck

# Detaillierte Infos (JSON)
tailscale status --json | jq '.Peer[].AllowedIPs'
```

### Exit-Node nutzen

```bash
# Exit-Node aktivieren (kompletter Traffic über Heimnetz)
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# Verfügbare Exit-Nodes auflisten
tailscale exit-node list

# Exit-Node deaktivieren
sudo tailscale up --exit-node= --accept-routes

# Eigene IP checken (sollte Heimnetz-IP sein)
curl ifconfig.me
```

### Service

```bash
# Tailscale Daemon Status
systemctl status tailscaled

# Daemon neustarten
sudo systemctl restart tailscaled

# Logs
sudo journalctl -u tailscaled -f
```

---

## Netzwerk-Übersicht

| Gerät | Tailscale IP | LAN IP | Rolle |
|-------|--------------|--------|-------|
| VPS (Headscale) | 100.64.0.5 | 152.53.111.11 | Coordinator |
| tailscale-router | 100.64.0.1 | 192.168.0.112 | Subnet router, exit node |
| maschinchen | 100.64.0.2 | variabel | Client |
| Debian VM | 100.64.0.6 | 192.168.0.111 | Homepage host |
| Proxmox | - | 192.168.0.101 | Hypervisor |

---

## Erreichbare Services

Nach erfolgreicher Verbindung:

| Service | URL | Zugriff |
|---------|-----|---------|
| Proxmox | https://192.168.0.101:8006 | Subnet Routing |
| Homepage | http://home.lab | MagicDNS + Subnet |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | VPN only |

---

*Siehe auch: [vpn-infrastructure.md](vpn-infrastructure.md) | [vpn-troubleshooting.md](vpn-troubleshooting.md)*
