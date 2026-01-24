# Tailscale + Homepage Setup Documentation

**Date:** 2026-01-24
**VPS:** zaboozMegaFescherSuperServer (152.53.111.11)

---

## Overview

This setup allows accessing home LAN services (like `192.168.0.111`) from anywhere via Tailscale/Headscale, with a custom DNS name `home.lab`.

## Architecture

```
[Client Device]
    ↓ (Tailscale connection)
[VPS - Headscale coordinator at zabooz.duckdns.org]
    ↓ (routes via Tailscale network)
[tailscale-router on Proxmox - 100.64.0.1 / 192.168.0.112]
    ↓ (subnet routing)
[Home LAN - 192.168.0.0/24]
    ↓
[Debian VM - 192.168.0.111 - Homepage Dashboard]
```

---

## Components

### 1. Headscale (VPS)
- **Location:** Docker container on VPS
- **Config:** `/etc/headscale/config.yaml`
- **Data:** `/var/lib/headscale/`

### 2. Tailscale Router (Proxmox LXC)
- **Tailscale IP:** 100.64.0.1
- **LAN IP:** 192.168.0.112
- **Advertised routes:** `192.168.0.0/24`, `0.0.0.0/0` (exit node)

### 3. Homepage Dashboard (Debian VM)
- **LAN IP:** 192.168.0.111
- **DNS Name:** `home.lab`
- **Port:** 80 (mapped to container port 3000)

---

## Configuration Changes Made

### 1. Headscale DNS Record (VPS)

**File:** `/etc/headscale/config.yaml`

Added custom DNS record for `home.lab`:

```yaml
dns:
  extra_records:
    - name: "zabooz.duckdns.org"
      type: "A"
      value: "100.64.0.5"
    - name: "home.lab"
      type: "A"
      value: "192.168.0.111"
```

**Restart required:**
```bash
docker restart headscale
```

---

### 2. Homepage Container (Debian VM)

**Location:** `192.168.0.111`
**Config directory:** `/root/homepage/config/`

#### Removed Apache (was blocking port 80):
```bash
systemctl stop apache2
systemctl disable apache2
```

#### Docker container:
```bash
docker run -d \
  --name homepage \
  --restart unless-stopped \
  -p 80:3000 \
  -e 'HOMEPAGE_ALLOWED_HOSTS=home.lab,192.168.0.111,localhost' \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /root/homepage/config:/app/config \
  ghcr.io/gethomepage/homepage:latest
```

**Important:** The `HOMEPAGE_ALLOWED_HOSTS` environment variable is required for custom hostnames.

---

### 3. Homepage Services Config

**File:** `/root/homepage/config/services.yaml`

```yaml
---
- Infrastructure:
    - Proxmox:
        icon: proxmox.png
        href: https://192.168.0.101:8006
        description: Hypervisor
        ping: 192.168.0.101
        widget:
          type: proxmox
          url: https://192.168.0.101:8006
          username: root@pam!homepage
          password: <API_TOKEN>
          node: homeserver
          fields: ["vms", "lxc", "resources.cpu", "resources.mem", "resources.disk"]
    - Tailscale Router:
        icon: tailscale.png
        description: Subnet Router (LXC)
        ping: 192.168.0.112
    - Debian VM:
        icon: si-debian
        description: Homepage Host
        ping: 192.168.0.111

- VPS Services:
    - Vaultwarden:
        icon: vaultwarden.png
        href: https://zabooz.duckdns.org/vault/
        description: Password Manager
        siteMonitor: https://zabooz.duckdns.org/vault/

    - SearXNG:
        icon: searxng.png
        href: https://zabooz.duckdns.org/searx/
        description: Private Search Engine
        siteMonitor: https://zabooz.duckdns.org/searx/

    - Headscale:
        icon: tailscale.png
        description: VPN Control Server
        href: https://zabooz.duckdns.org/web/
        siteMonitor: https://zabooz.duckdns.org/web/
```

**Notes:**
- Local services use `ping:` (ICMP)
- VPS services use `siteMonitor:` (HTTP) because VPS blocks ICMP

---

### 4. Homepage Settings

**File:** `/root/homepage/config/settings.yaml`

```yaml
---
title: Zabooz's Homelab
headerStyle: boxed

allowedHosts:
  - home.lab
  - 192.168.0.111
  - localhost

providers:
  docker:
    socket: /var/run/docker.sock
```

---

## Tailscale Client Setup (New Device)

### Quick Setup

**1. Install Tailscale:**
- Linux: `curl -fsSL https://tailscale.com/install.sh | sh`
- Android/iOS: Install from App Store / Play Store
- Windows: https://tailscale.com/download

**2. Connect:**
```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-dns --accept-routes
```

**3. Authorize:** Open the URL shown in browser, then verify with `tailscale status`

---

### Required Flags

| Flag | What it does | Without it |
|------|--------------|------------|
| `--login-server=https://zabooz.duckdns.org` | Connect to Headscale (not Tailscale) | Won't connect |
| `--accept-dns` | Use Headscale DNS (MagicDNS) | `home.lab` won't resolve |
| `--accept-routes` | Accept subnet routes from tailscale-router | Can't reach `192.168.0.x` |

### Optional Flags

| Flag | What it does |
|------|--------------|
| `--exit-node=100.64.0.1` | Route ALL traffic through home network |
| `--reset` | Reset settings (use instead of `--force-reauth`) |

---

### Mobile (Android/iOS)

1. Open Tailscale → Settings → "Use custom coordination server"
2. Enter: `https://zabooz.duckdns.org`
3. Enable: **Accept DNS** + **Accept Routes**
4. Connect and authorize

---

## Network Overview

| Device | Tailscale IP | LAN IP | Role |
|--------|--------------|--------|------|
| VPS (Headscale) | 100.64.0.5 | 152.53.111.11 | Coordinator |
| tailscale-router | 100.64.0.1 | 192.168.0.112 | Subnet router, exit node |
| maschinchen | 100.64.0.2 | 192.168.0.203 | Client |
| Debian VM | 100.64.0.6 | 192.168.0.111 | Homepage host |
| Proxmox | - | 192.168.0.101 | Hypervisor |

---

## Troubleshooting

### Nodes showing offline after Headscale restart
Restart tailscaled on each node:
```bash
sudo systemctl restart tailscaled
```

### Homepage host validation error
Ensure `HOMEPAGE_ALLOWED_HOSTS` includes the hostname:
```bash
docker stop homepage && docker rm homepage
docker run -d --name homepage ... -e 'HOMEPAGE_ALLOWED_HOSTS=home.lab,...'
```

### Can't reach home LAN from outside
1. Check tailscale-router is online: `tailscale status`
2. Verify routes are approved in Headscale:
   ```bash
   docker exec headscale headscale routes list
   ```
3. Ensure client has `--accept-routes`

---

## Files Reference

| Location | File | Purpose |
|----------|------|---------|
| VPS | `/etc/headscale/config.yaml` | Headscale config (DNS, DERP, etc.) |
| Debian VM | `/root/homepage/config/services.yaml` | Homepage services |
| Debian VM | `/root/homepage/config/settings.yaml` | Homepage settings |
