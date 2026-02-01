---
title: IP Forwarding
description: Die Router-Funktion in Linux aktivieren
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, routing, linux, forwarding, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# IP Forwarding

Die Router-Funktion in Linux aktivieren.

---

## Was macht IP Forwarding?

```
Aus (= 0): Computer verwirft fremde Pakete
An (= 1):  Computer leitet fremde Pakete weiter → wird zum Router!
```

---

## Konfiguration

```bash
# Status prüfen
sysctl net.ipv4.ip_forward

# Temporär aktivieren
sudo sysctl -w net.ipv4.ip_forward=1

# Dauerhaft aktivieren (/etc/sysctl.conf)
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Anwenden
sudo sysctl -p
```

---

## Router vs. Host

| Merkmal | Host | Router |
|---------|------|--------|
| **IP Forwarding** | Aus | An |
| **Gateway** | Hat immer Gateway | Kann Gateway haben (für Internet) |
| **Interfaces** | Meist 1 | Meist mehrere |
| **Routing** | Schickt alles zum Gateway | Leitet Pakete zwischen Netzen weiter |

**Merksatz:**
```
Host   = Schickt fremde Pakete zum Gateway
Router = Leitet Pakete selbst weiter (IP Forwarding an)
```

**Wichtig:** Auch ein Router kann ein Default-Gateway haben (z.B. für Internet-Traffic). Der Unterschied ist: Ein Router leitet Pakete zwischen seinen Interfaces weiter, ein Host nicht.

---

*Siehe auch: [Routing](routing) | [NAT](nat)*
