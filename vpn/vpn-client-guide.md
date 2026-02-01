---
title: VPN Client Anleitung
description: Tailscale Client Setup und Verbindung zum VPN
published: true
date: 2026-01-18T00:00:00.000Z
tags: vpn, tailscale, client, anleitung, guide
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Tailscale Client Setup Guide

**Server:** `https://zabooz.duckdns.org`

**Siehe auch:**
- [VPS](../infrastructure/VPS.md) - VPS wo Headscale läuft (152.53.111.11)
- [NETWORK_OVERVIEW.md](../infrastructure/NETWORK_OVERVIEW.md) - Alle IPs und Dienste
- [DNS](../konzepte/dns.md) - MagicDNS, Namensauflösung
- [Routing](../konzepte/routing.md) - Subnet Routes verstehen
- [vpn-infrastructure.md](vpn-infrastructure.md) - Server Setup
- [vpn-troubleshooting.md](vpn-troubleshooting.md) - Problemlösung

---

## Quick Start

### 1. User erstellen (VPS)

```bash
ssh zabooz@152.53.111.11

# User erstellen
docker exec headscale headscale users create BENUTZERNAME

# User anzeigen
docker exec headscale headscale users list
```

### 2. Tailscale auf Client installieren

Siehe Abschnitte unten für Linux/Windows/macOS/Mobile.

### 3. Client verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-dns --accept-routes
```

### 4. Node registrieren (VPS)

```bash
# NodeKey vom Client-Output kopieren
docker exec headscale headscale nodes register --user BENUTZERNAME --key nodekey:xxxxxxxxx
```

### 5. Testen

```bash
tailscale status
ping 192.168.0.101  # Proxmox
```

---

## Installation

| Platform | Link |
|----------|------|
| Linux | https://tailscale.com/download/linux |
| Windows | https://tailscale.com/download/windows |
| macOS | https://tailscale.com/download/mac |
| Android | Play Store: Tailscale |
| iOS | App Store: Tailscale |

---

## Linux

### Quick Install

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

**Arch / CachyOS:**
```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
```

### Verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-dns --accept-routes
```

Kopiere den `nodekey:xxxxxxxxx` aus dem Output und registriere auf dem VPS.

---

## Windows

### Installation

1. Download: https://tailscale.com/download/windows
2. Installieren und starten

### Verbinden (PowerShell als Admin)

```powershell
tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns
```

### GUI-Methode

1. Tailscale Icon im System Tray → Settings
2. "Use custom coordination server" aktivieren
3. URL: `https://zabooz.duckdns.org`
4. Enable: **Accept Routes** + **Accept DNS**
5. Connect

---

## macOS

1. Download von https://tailscale.com/download/mac
2. Settings → "Use custom coordination server"
3. URL: `https://zabooz.duckdns.org`
4. Enable: Accept Routes + Accept DNS
5. Connect

---

## Android / iOS

1. Tailscale App installieren
2. **Vor dem Einloggen:** Settings → "Use custom coordination server"
3. URL: `https://zabooz.duckdns.org`
4. Enable: **Accept DNS** + **Accept Routes**
5. Connect

---

## Node Registrierung (VPS)

Nach dem Verbinden eines neuen Geräts:

```bash
ssh zabooz@152.53.111.11

# Node registrieren
docker exec headscale headscale nodes register --user BENUTZERNAME --key nodekey:xxxxxxxxx

# Alle Nodes anzeigen
docker exec headscale headscale nodes list
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
# Verbinden
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns

# Temporär trennen
sudo tailscale down

# Komplett abmelden
sudo tailscale logout
```

### Status

```bash
# Alle verbundenen Nodes
tailscale status

# Verbindungsqualität
tailscale netcheck

# DNS testen
ping home.lab
```

### Exit-Node nutzen

```bash
# Exit-Node aktivieren (kompletter Traffic über Heimnetz)
sudo tailscale up --exit-node=100.64.0.1 --accept-routes

# Exit-Node deaktivieren
sudo tailscale up --exit-node= --accept-routes

# Eigene öffentliche IP checken
curl ifconfig.me
```

### Service (Linux)

```bash
systemctl status tailscaled
sudo systemctl restart tailscaled
sudo journalctl -u tailscaled -f
```

---

## Erreichbare Services

Nach erfolgreicher Verbindung:

| Service | URL | Zugriff |
|---------|-----|---------|
| Proxmox | https://192.168.0.101:8006 | Subnet Routing |
| Homepage | http://home.lab | MagicDNS |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | VPN only |
| SearXNG | https://zabooz.duckdns.org/searx/ | VPN only |
| Headscale UI | https://zabooz.duckdns.org/web | VPN only |

---

*Letzte Aktualisierung: Januar 2026*
