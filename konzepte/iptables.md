---
title: iptables Grundlagen
description: Linux Firewall und Paketfilterung
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, firewall, iptables, sicherheit, security, linux, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# iptables Grundlagen

Linux Firewall und Paketfilterung.

---

## Chains (Ketten)

| Chain | Wann | Beispiel |
|-------|------|----------|
| **PREROUTING** | Vor Routing-Entscheidung | DNAT |
| **INPUT** | Pakete für lokales System | Firewall |
| **FORWARD** | Pakete die weitergeleitet werden | Router |
| **OUTPUT** | Pakete vom lokalen System | Ausgehende Filter |
| **POSTROUTING** | Nach Routing-Entscheidung | SNAT/MASQUERADE |

---

## Tables

| Table | Zweck |
|-------|-------|
| **filter** | Firewall-Regeln (Standard) |
| **nat** | NAT-Regeln |
| **mangle** | Paket-Modifikation |

---

## Wichtige Befehle

```bash
# Filter-Regeln anzeigen
sudo iptables -L -n -v

# NAT-Regeln anzeigen
sudo iptables -t nat -L -n -v

# Forward-Chain anzeigen
sudo iptables -L FORWARD -n -v

# Regel hinzufügen
sudo iptables -A FORWARD -i tailscale0 -j ACCEPT

# Regel am Anfang einfügen
sudo iptables -I FORWARD -i tailscale0 -j ACCEPT

# Regeln dauerhaft speichern
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

*Siehe auch: [NAT](/en/konzepte/nat) | [IP-Forwarding](/en/konzepte/ip-forwarding)*
