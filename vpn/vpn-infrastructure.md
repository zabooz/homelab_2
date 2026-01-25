# Headscale VPN Infrastruktur

**Erstellt:** 18. Januar 2026
**Aktualisiert:** 25. Januar 2026
**Author:** Daniel (zabooz)
**Setup:** Self-hosted Tailscale mit Headscale (Docker)

---

## Übersicht

Self-hosted Mesh-VPN mit Headscale als Tailscale-Alternative. Läuft als Docker Container auf dem VPS.

**Siehe auch:**
- [NETWORK_OVERVIEW.md](../infrastructure/NETWORK_OVERVIEW.md) - Master-Diagramm und alle IPs
- [core-concepts.md](../core-concepts.md) - VPN-Konzepte, WireGuard vs OpenVPN
- [vpn-client-guide.md](vpn-client-guide.md) - Client-Installation
- [vpn-troubleshooting.md](vpn-troubleshooting.md) - Problemlösung

---

## Komponenten

| Komponente | IP/Domain | Funktion |
|------------|-----------|----------|
| **VPS** | 152.53.111.11 | Headscale Control Server (Docker) |
| **Domain** | zabooz.duckdns.org | DuckDNS Dynamic DNS |
| **Headscale** | 127.0.0.1:8090 | VPN Control Server |
| **Headscale UI** | 127.0.0.1:8080 | Web-Interface (VPN only) |
| **Nginx** | Port 443 | Reverse Proxy mit HTTPS |
| **DERP Server** | Port 443/3478 | Relay/STUN Server |
| **Tailscale LXC** | 192.168.0.112 / 100.64.0.1 | Exit-Node + Subnet Router |

### Wichtige Nodes

| Hostname | Tailscale IP | LAN IP | Rolle |
|----------|--------------|--------|-------|
| tailscale-router | 100.64.0.1 | 192.168.0.112 | Subnet Router, Exit-Node |
| maschinchen | 100.64.0.2 | - | Laptop (Client) |
| zaboozmegafeschersuperserver | 100.64.0.5 | 152.53.111.11 | VPS (Headscale Server) |

### Features

- Ende-zu-Ende verschlüsselt (WireGuard)
- HTTPS mit Let's Encrypt
- Eigener DERP Server
- Exit-Node (Internet über Heimnetz)
- Subnet Router (Zugriff auf 192.168.0.0/24)
- MagicDNS (headnet.com)

---

## Headscale Server (VPS)

### Docker Setup

Headscale läuft als Docker Container mit Host-Networking:

```bash
# Container Status
docker ps | grep headscale

# Logs anzeigen
docker logs headscale --tail 50

# Container neustarten
docker restart headscale
```

### Wichtige Config (/etc/headscale/config.yaml)

```yaml
server_url: https://zabooz.duckdns.org
listen_addr: 127.0.0.1:8090

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

derp:
  server:
    enabled: true
    region_id: 999
    stun_listen_addr: "0.0.0.0:3478"

dns:
  magic_dns: true
  base_domain: headnet.com
  override_local_dns: true
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1
  extra_records:
    - name: "home.lab"
      type: "A"
      value: "192.168.0.111"
    - name: "vps.lab"
      type: "A"
      value: "100.64.0.5"
    - name: "zabooz.duckdns.org"
      type: "A"
      value: "100.64.0.5"
```

> **Hinweis:** `zabooz.duckdns.org` wird für VPN-Clients auf die Tailscale IP überschrieben, damit VPN-only Services (Vaultwarden, SearXNG) mit gültigem SSL-Zertifikat erreichbar sind.

### Headscale Befehle

Alle Befehle mit `docker exec`:

```bash
# User Management
docker exec headscale headscale users list
docker exec headscale headscale users create <name>

# Node Management
docker exec headscale headscale nodes list
docker exec headscale headscale nodes register --user <user> --key <nodekey>
docker exec headscale headscale nodes delete --identifier <id>

# Routes Management
docker exec headscale headscale routes list
docker exec headscale headscale routes enable --route <id>

# API Keys (für Headscale UI)
docker exec headscale headscale apikeys create --expiration 365d
docker exec headscale headscale apikeys list
```

### DuckDNS Auto-Update

```bash
# Crontab
crontab -e

# Alle 5 Minuten IP updaten
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=zabooz&token=DEIN_TOKEN&ip=" > /dev/null
```

---

## Tailscale LXC (Subnet Router + Exit-Node)

### Proxmox LXC Konfiguration

- **CT ID:** 102
- **Privileged:** YES (für Tailscale nötig)
- **IP:** 192.168.0.112/24
- **Features:** nesting=1, keyctl=1

### Setup

```bash
# Tailscale installieren
curl -fsSL https://tailscale.com/install.sh | sh
systemctl enable --now tailscaled

# Mit Headscale verbinden
tailscale up \
  --login-server=https://zabooz.duckdns.org \
  --advertise-exit-node \
  --advertise-routes=192.168.0.0/24
```

### Routes auf VPS freigeben

```bash
# NodeKey aus Container-Output kopieren, dann auf VPS:
docker exec headscale headscale nodes register --user zabooz --key nodekey:xxxxx

# Routes auflisten und aktivieren
docker exec headscale headscale routes list
docker exec headscale headscale routes enable --route <id>
```

### IP Forwarding (dauerhaft)

LXC Container laden `/etc/sysctl.conf` nicht automatisch. Systemd Service erstellen:

```bash
# /etc/systemd/system/ip-forwarding.service
[Unit]
Description=Enable IP Forwarding
Before=tailscaled.service

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
ExecStart=/sbin/sysctl -w net.ipv6.conf.all.forwarding=1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable ip-forwarding.service
```

### Auto-Start (Proxmox)

```bash
pct set 102 --onboot 1
pct set 102 --startup order=1,up=30
```

---

## Subnet-Routing

### Problem

Heimnetz (192.168.0.0/24) ist getrennt vom Tailscale-Netz (100.64.0.0/10).

### Lösung

Der **Subnet Router** (LXC Container) ist die Brücke:

```
Laptop (100.64.0.2)
    │
    │ Tailscale VPN
    ▼
Tailscale LXC (100.64.0.1 + 192.168.0.112)
    │
    │ Subnet Route: 192.168.0.0/24
    ▼
Proxmox (192.168.0.101), VMs, etc.
```

Mit `--advertise-routes=192.168.0.0/24` gibt der LXC Container das Heimnetz für alle VPN-Clients frei.

---

## Wichtige Ports

| Port | Protokoll | Dienst |
|------|-----------|--------|
| 443 | TCP | HTTPS (Nginx → Headscale) |
| 443 | TCP | DERP Relay |
| 3478 | UDP | STUN (NAT-Traversal) |
| 8090 | TCP | Headscale (intern) |
| 8080 | TCP | Headscale UI (intern) |

---

*Letzte Aktualisierung: Januar 2026*
