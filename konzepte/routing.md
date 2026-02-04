---
title: Routing & Gateway
description: Wie Pakete ihren Weg durch Netzwerke finden
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, routing, gateway, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Routing & Gateway

Wie Pakete ihren Weg durch Netzwerke finden.

---

## Was ist ein Gateway?

- Zugang zu **anderen Netzwerken**
- In Heimnetzen meist der Router
- Alle Pakete zu fremden Netzen gehen an Gateway
- Standard-Route in Routing-Tabelle

---

## Routing-Tabelle lesen

```bash
ip route show
# default via 192.168.0.1 dev eth0     ← Standardroute (Gateway)
# 192.168.0.0/24 dev eth0 scope link   ← Lokales Netz (direkt erreichbar)
```

**Wichtige Begriffe:**
- **default via** → Standardgateway
- **scope link** → Direkt erreichbar (kein Router nötig)
- **via X** → Über diesen Router erreichbar

---

## Längster Präfix-Match

Der Kernel wählt die **spezifischste** Route:

```
192.168.0.0/24  ← /24 ist spezifischer als /16
192.168.0.0/16  ← /16 ist spezifischer als /0
0.0.0.0/0       ← Default Route (unspezifischste)
```

---

## Policy-Based Routing (Table 52)

Linux nutzt **mehrere Routing-Tables** für verschiedene Zwecke:

```bash
# Main Table (Standard)
ip route show table main

# Tailscale Table 52
ip route show table 52

# Welche Route wird verwendet?
ip route get 192.168.0.101
```

**Warum?**
- Keine Konflikte zwischen VPN und normalem Traffic
- Saubere Trennung
- Automatisches Failover

---

*Siehe auch: [NAT](/en/konzepte/nat) | [IP-Forwarding](/en/konzepte/ip-forwarding)*
