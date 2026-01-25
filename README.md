# Homelab Dokumentation

LAP-Vorbereitung IT-Systemtechnik

**Author:** Daniel (zabooz)
**Stand:** Januar 2026

---

## Quick Links

| Kategorie | Dokument | Beschreibung |
|-----------|----------|--------------|
| **Übersicht** | [NETWORK_OVERVIEW.md](infrastructure/NETWORK_OVERVIEW.md) | Master-Diagramm, IP-Liste, Quick Reference |
| **Theorie** | [core-concepts.md](core-concepts.md) | Alle LAP-relevanten Netzwerk-Konzepte |

---

## VPN

| Dokument | Inhalt |
|----------|--------|
| [vpn-infrastructure.md](vpn/vpn-infrastructure.md) | Headscale Server Setup, Architektur, Subnet Routing |
| [vpn-client-guide.md](vpn/vpn-client-guide.md) | Client-Installation (Linux, Windows, Mobile), Flags |
| [vpn-troubleshooting.md](vpn/vpn-troubleshooting.md) | Alle VPN-Probleme und Lösungen |

---

## Services

| Dokument | Inhalt |
|----------|--------|
| [homepage-dashboard-setup.md](services/homepage-dashboard-setup.md) | Homepage Dashboard Docker Setup |
| [apache-reverse-proxy-setup.md](services/apache-reverse-proxy-setup.md) | Apache als Reverse Proxy |
| [tailscale-homepage-setup.md](services/tailscale-homepage-setup.md) | Homepage + Tailscale Integration |
| [fog_project_setup.md](services/fog_project_setup_dokumentation_debian_12_lxc.md) | FOG Project PXE/Imaging |

---

## Infrastructure

| Dokument | Inhalt |
|----------|--------|
| [NETWORK_OVERVIEW.md](infrastructure/NETWORK_OVERVIEW.md) | Master-Diagramm, IP-Liste, Quick Reference |
| [LINUXVM.md](infrastructure/LINUXVM.md) | Linux VM Overview & Services |
| [proxmox-netzwerk-setup.md](infrastructure/proxmox-netzwerk-setup.md) | Statische IPs, Netzwerk-Konfiguration |
| [NETWORK_SETUP.md](infrastructure/NETWORK_SETUP.md) | VPS Netzwerk-Architektur |

---

## Operations

| Dokument | Inhalt |
|----------|--------|
| [backup-recovery.md](operations/backup-recovery.md) | Backup-Strategien, Restore-Prozeduren |
| [security-hardening.md](operations/security-hardening.md) | SSH, Firewall, Service-Sicherheit |
| [disaster-recovery.md](operations/disaster-recovery.md) | Was tun wenn... (Ausfallszenarien) |

---

## Verzeichnisstruktur

```
docs/
├── README.md                    # Diese Datei
├── home.md                      # Wiki.js Startseite
├── core-concepts.md             # LAP Exam Konzepte
│
├── infrastructure/
│   ├── NETWORK_OVERVIEW.md      # Master-Diagramm & IP-Liste
│   ├── LINUXVM.md               # Linux VM Overview
│   ├── NETWORK_SETUP.md         # VPS Netzwerk-Architektur
│   └── proxmox-netzwerk-setup.md
│
├── vpn/
│   ├── vpn-infrastructure.md    # Server Setup
│   ├── vpn-client-guide.md      # Client Setup
│   └── vpn-troubleshooting.md   # Probleme & Lösungen
│
├── services/
│   ├── homepage-dashboard-setup.md
│   ├── apache-reverse-proxy-setup.md
│   ├── tailscale-homepage-setup.md
│   ├── wikijs-setup.md
│   ├── stats-api.md
│   ├── paperless-ngx.md
│   └── fog_project_setup_dokumentation_debian_12_lxc.md
│
└── operations/
    ├── backup-recovery.md
    ├── security-hardening.md
    └── disaster-recovery.md
```

---

## Wichtigste Konzepte (LAP)

### Netzwerk-Grundlagen

- **Subnetting/CIDR** - IP-Bereiche berechnen
- **DHCP vs. Statisch** - Wann welches?
- **Routing & Gateway** - Wie Pakete finden ihren Weg
- **NAT/MASQUERADE** - Warum braucht man das?

### VPN

- **Mesh VPN** (Tailscale/Headscale) vs. traditionelle VPNs
- **WireGuard** - Modernes VPN-Protokoll
- **Subnet Routing** - Zugriff auf entfernte Netze
- **Exit-Node** - Internet über VPN

### Linux

- **IP Forwarding** - Router-Funktion aktivieren
- **iptables/nftables** - Firewall und NAT
- **Policy-Based Routing** - Table 52
- **systemd Services** - Dienste verwalten

### Virtualisierung

- **Proxmox VE** - Hypervisor
- **LXC Container** - Leichtgewichtige Virtualisierung
- **KVM/QEMU** - Vollvirtualisierung

---

## Quick Commands

### VPN verbinden

```bash
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns
```

### Netzwerk-Diagnose

```bash
ip route show table 52          # Tailscale Routes
sysctl net.ipv4.ip_forward      # IP Forwarding Status
sudo iptables -t nat -L -n      # NAT Regeln
tailscale status                # VPN Status
```

### Backup erstellen

```bash
vzdump 102 --storage local --compress zstd --mode snapshot
```

---

## Kontakt & Support

- **Headscale Server:** zabooz.duckdns.org
- **VPS IP:** 152.53.111.11
- **Proxmox:** 192.168.0.101

---

*Viel Erfolg bei der LAP-Prüfung!*
