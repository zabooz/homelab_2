---
title: VPN-Typen
description: Virtual Private Networks - WireGuard, OpenVPN, Mesh VPN
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, vpn, wireguard, verschlüsselung, encryption, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# VPN-Typen

Virtual Private Networks verstehen.

---

## VPN-Architekturen

| Typ | Beschreibung | Beispiel |
|-----|--------------|----------|
| **Site-to-Site** | Verbindet zwei Netzwerke | Firmenstandorte |
| **Remote Access** | Clients zu einem Netzwerk | Homeoffice-VPN |
| **Mesh VPN** | Alle Nodes direkt verbunden | Tailscale/Headscale |

---

## WireGuard vs. OpenVPN

| Feature | WireGuard | OpenVPN |
|---------|-----------|---------|
| **Geschwindigkeit** | Sehr schnell | Langsamer |
| **Code-Basis** | 4.000 Zeilen | 100.000+ Zeilen |
| **Konfiguration** | Einfach | Komplex |
| **Protokoll** | UDP | TCP/UDP |

---

*Siehe auch: [VPN Infrastruktur](../vpn/vpn-infrastructure.md) | [VPN Client Guide](../vpn/vpn-client-guide.md)*
