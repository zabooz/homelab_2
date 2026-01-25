# Netzwerk-Übersicht

Master-Referenz für die gesamte Infrastruktur.

---

## Infrastruktur-Diagramm

```
                              INTERNET
                                  │
                                  │
              ┌───────────────────┴───────────────────┐
              │                                       │
              │         VPS (152.53.111.11)           │
              │         zabooz.duckdns.org            │
              │                                       │
              │    ┌─────────────────────────────┐    │
              │    │ Nginx (443)                 │    │
              │    │  ├── / → Headscale (8090)   │    │
              │    │  ├── /vault/ → Vaultwarden  │    │
              │    │  ├── /searx/ → SearXNG      │    │
              │    │  └── /web → Headscale UI    │    │
              │    └─────────────────────────────┘    │
              │                                       │
              │    Tailscale IP: 100.64.0.5           │
              └───────────────────┬───────────────────┘
                                  │
                    Headscale Mesh VPN (WireGuard)
                         100.64.0.0/10
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       │                          │                          │
       │                          │                          │
┌──────▼──────┐          ┌────────▼────────┐         ┌───────▼───────┐
│   Laptop    │          │  Tailscale LXC  │         │    Handy      │
│ maschinchen │          │   100.64.0.1    │         │  zabooz-phone │
│ 100.64.0.2  │          │  192.168.0.112  │         │  100.64.0.4   │
└─────────────┘          │                 │         └───────────────┘
                         │  Exit-Node +    │
                         │  Subnet Router  │
                         └────────┬────────┘
                                  │
                       ┌──────────▼──────────┐
                       │  HEIMNETZWERK       │
                       │  192.168.0.0/24     │
                       │  Gateway: .1        │
                       │                     │
                       │  ┌───────────────┐  │
                       │  │ Proxmox .101  │  │
                       │  │   homeserver  │  │
                       │  │               │  │
                       │  │ ┌───────────┐ │  │
                       │  │ │Win VM .110│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Deb VM .111│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │TS LXC .112│ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │FOG  .113  │ │  │
                       │  │ ├───────────┤ │  │
                       │  │ │Paper .115 │ │  │
                       │  │ └───────────┘ │  │
                       │  └───────────────┘  │
                       └─────────────────────┘
```

---

## IP-Adressen Master-Liste

### VPS (Internet)

| Hostname | Public IP | Tailscale IP | Dienste |
|----------|-----------|--------------|---------|
| zaboozMegaFescherSuperServer | 152.53.111.11 | 100.64.0.5 | Headscale, Vaultwarden, SearXNG |

### Heimnetzwerk (192.168.0.0/24)

| Gerät | IP | MAC | Typ | Funktion |
|-------|-----|-----|-----|----------|
| **Router** | 192.168.0.1 | - | Gateway | Internet-Gateway (DHCP deaktiviert) |
| **Proxmox** | 192.168.0.101 | ec:b1:d7:72:c9:d1 | Host | Virtualisierungs-Host |
| **Windows VM** | 192.168.0.110 | bc:24:11:a9:bf:52 | VM | Windows Server 2025 |
| **Debian VM** | 192.168.0.111 | bc:24:11:1a:24:3d | VM | Homepage Dashboard |
| **Tailscale LXC** | 192.168.0.112 | bc:24:11:d8:a7:b2 | LXC | VPN Exit-Node/Subnet Router |
| **FOG Server** | 192.168.0.113 | - | LXC | DHCP Server, PXE/Imaging |
| **Paperless-ngx** | 192.168.0.115 | - | LXC | Dokumentenverwaltung |

### Tailscale VPN (100.64.0.0/10)

| Hostname | Tailscale IP | User | Rolle |
|----------|--------------|------|-------|
| tailscale | 100.64.0.1 | zabooz | Subnet Router, Exit-Node |
| maschinchen | 100.64.0.2 | zabooz | Laptop |
| jules | 100.64.0.3 | jules | Client |
| zabooz-phone | 100.64.0.4 | zabooz | Handy |
| zaboozmegafeschersuperserver | 100.64.0.5 | zabooz | VPS |

---

## IP-Bereiche

```
192.168.0.1         - Gateway/Router (kein DHCP!)
192.168.0.100-119   - Statische IPs (Server/Infrastruktur)
192.168.0.120-199   - Reserviert für Erweiterungen
192.168.0.200-250   - FOG DHCP Pool (für alle Clients + PXE Boot)
```

## DHCP

| Parameter | Wert |
|-----------|------|
| **DHCP Server** | FOG (192.168.0.113) |
| **Router DHCP** | Deaktiviert |
| **DHCP Range** | 192.168.0.200 - 192.168.0.250 |
| **Gateway** | 192.168.0.1 |
| **DNS** | 192.168.0.1 (Router) |

> **Wichtig:** Router-DHCP ist deaktiviert. FOG übernimmt DHCP für das gesamte Netzwerk inkl. PXE Boot.

---

## Dienste-Übersicht

### VPS-Dienste

| Dienst | URL | Port (intern) | Zugriff |
|--------|-----|---------------|---------|
| Headscale | https://zabooz.duckdns.org/ | 8090 | Public |
| Headscale UI | https://zabooz.duckdns.org/web/ | 8080 | **VPN only** |
| SearXNG | https://zabooz.duckdns.org/searx/ | 8888 | **VPN only** |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | 8000 | **VPN only** |

### Heimnetz-Dienste

| Dienst | URL | Host | Port |
|--------|-----|------|------|
| Proxmox | https://192.168.0.101:8006 | Proxmox | 8006 |
| Homepage | http://192.168.0.111 | Debian VM | 80 |
| Homepage (direkt) | http://192.168.0.111:3000 | Debian VM | 3000 |
| FOG Project | http://192.168.0.113/fog/management | FOG LXC | 80 |
| Paperless-ngx | http://192.168.0.115:8000 | Paperless LXC | 8000 |

### MagicDNS (über Tailscale)

| Name | Auflösung | Zweck |
|------|-----------|-------|
| home.lab | 192.168.0.111 | Homepage Dashboard |
| vps.lab | 100.64.0.5 | VPS via Tailscale |
| zabooz.duckdns.org | 100.64.0.5 | VPS (überschreibt Public DNS für VPN-Clients) |

> **Hinweis:** `zabooz.duckdns.org` wird für VPN-Clients auf die Tailscale IP überschrieben, damit VPN-only Services mit gültigem SSL-Zertifikat erreichbar sind.

---

## Netzwerk-Parameter

| Parameter | Wert |
|-----------|------|
| **Netzwerk** | 192.168.0.0/24 |
| **Subnetzmaske** | 255.255.255.0 |
| **Gateway** | 192.168.0.1 |
| **DNS (primär)** | 192.168.0.1 |
| **DNS (sekundär)** | 8.8.8.8 |
| **Tailscale-Netz** | 100.64.0.0/10 |

---

## Service-Abhängigkeiten

```
┌─────────────────────────────────────────────────┐
│                    Vaultwarden                   │
│            (braucht: Headscale, Nginx)           │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                    Headscale                     │
│         (braucht: Nginx, Let's Encrypt)          │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                 Tailscale LXC                    │
│          (braucht: Headscale, Proxmox)           │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│                    Proxmox                       │
│              (braucht: Hardware)                 │
└─────────────────────────────────────────────────┘
```

**Startreihenfolge bei komplettem Neustart:**
1. Proxmox Host
2. Tailscale LXC (102)
3. VPS (Headscale) muss bereits laufen
4. Andere VMs/LXCs

---

## Quick Reference Card

### VPN verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns
```

### Wichtige IPs

```
VPS:         152.53.111.11 / 100.64.0.5
Proxmox:     192.168.0.101
Subnet LXC:  192.168.0.112 / 100.64.0.1
Homepage:    192.168.0.111
```

### Management-URLs

```
Proxmox:     https://192.168.0.101:8006
Vaultwarden: https://zabooz.duckdns.org/vault/
Homepage:    http://home.lab  (über VPN)
```

---

*Letzte Aktualisierung: 25. Januar 2026*
