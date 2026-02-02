---
title: NAT
description: Network Address Translation - Adressübersetzung zwischen Netzwerken
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, nat, firewall, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# NAT (Network Address Translation)

Adressübersetzung zwischen Netzwerken.

---

## NAT-Typen

| Typ | Funktion | Beispiel |
|-----|----------|----------|
| **SNAT** | Ändert Source-IP (ausgehend) | Heimnetz → Internet |
| **DNAT** | Ändert Destination-IP (eingehend) | Port-Forwarding |
| **MASQUERADE** | Dynamisches SNAT | VPN-Router mit DHCP |

---

## Warum MASQUERADE?

```
Problem:
Laptop sendet:    100.64.0.2 → 192.168.0.101
Proxmox denkt:    "100.64.0.2? Kenne ich nicht!"
                  → Paket verworfen ❌

Lösung mit MASQUERADE:
Container ändert: 100.64.0.2 → 192.168.0.112
Proxmox denkt:    "Ah, Container! Den kenne ich!"
                  → Antwortet ✅
```

---

## iptables NAT-Regel

```bash
# NAT-Regel anzeigen
sudo iptables -t nat -L POSTROUTING -n -v

# NAT-Regel erstellen
sudo iptables -t nat -A POSTROUTING -o eth0 -s 100.64.0.0/10 -j MASQUERADE
```

---

*Siehe auch: [IP-Forwarding](/konzepte/ip-forwarding) | [iptables](/konzepte/iptables)*
