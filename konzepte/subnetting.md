---
title: Subnetting
description: Netzwerke in kleinere Subnetze aufteilen - CIDR und Subnetzmasken
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, subnetting, cidr, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Subnetting

Netzwerke in kleinere Subnetze aufteilen.

---

## Subnetzmasken verstehen

Die Subnetzmaske trennt Netzwerk-Teil von Host-Teil:

**255.255.255.0 in Binär:**
```
11111111.11111111.11111111.00000000
└─────────────────────────┘└──────┘
    Netzwerk-Teil (24 Bit)  Host-Teil (8 Bit)
```

- **1er** = Netzwerk-Teil (fix)
- **0er** = Host-Teil (variabel)

---

## CIDR-Notation

CIDR (Classless Inter-Domain Routing) gibt die Anzahl der Netzwerk-Bits an:

| CIDR | Subnetzmaske | Hosts | Netze aus /24 |
|------|--------------|-------|---------------|
| /8 | 255.0.0.0 | 16.777.214 | 65.536 |
| /16 | 255.255.0.0 | 65.534 | 256 |
| /24 | 255.255.255.0 | 254 | 1 |
| /25 | 255.255.255.128 | 126 | 2 |
| /26 | 255.255.255.192 | 62 | 4 |
| /27 | 255.255.255.224 | 30 | 8 |
| /28 | 255.255.255.240 | 14 | 16 |
| /29 | 255.255.255.248 | 6 | 32 |
| /30 | 255.255.255.252 | 2 | 64 |
| /32 | 255.255.255.255 | 1 | - |

> **Formel:** Hosts = 2^(32-CIDR) - 2 (minus Netzwerk- und Broadcast-Adresse)

---

## Subnetz berechnen

### Beispiel: 192.168.0.0/24

```
Netzwerk-Adresse:  192.168.0.0     (erste Adresse)
Broadcast-Adresse: 192.168.0.255   (letzte Adresse)
Erster Host:       192.168.0.1     (meist Gateway)
Letzter Host:      192.168.0.254
Verfügbare Hosts:  254
```

### Beispiel: 192.168.0.0/26

```
Subnetz 1: 192.168.0.0   - 192.168.0.63   (62 Hosts)
Subnetz 2: 192.168.0.64  - 192.168.0.127  (62 Hosts)
Subnetz 3: 192.168.0.128 - 192.168.0.191  (62 Hosts)
Subnetz 4: 192.168.0.192 - 192.168.0.255  (62 Hosts)
```

---

## Schnelle Berechnung

### Schritt 1: Blockgröße ermitteln

```
/24 → 256 Adressen (Block = 256)
/25 → 128 Adressen (Block = 128)
/26 → 64 Adressen  (Block = 64)
/27 → 32 Adressen  (Block = 32)
/28 → 16 Adressen  (Block = 16)
```

### Schritt 2: Subnetze aufzählen

Für /26 (Block = 64):
- 0, 64, 128, 192 = Netzwerk-Adressen
- 63, 127, 191, 255 = Broadcast-Adressen

### Beispiel-Aufgabe

**Frage:** In welchem Subnetz liegt 192.168.1.100/26?

```
Blockgröße /26 = 64
Subnetze: 0, 64, 128, 192
100 liegt zwischen 64 und 128
→ Subnetz: 192.168.1.64/26
→ Broadcast: 192.168.1.127
→ Host-Range: 192.168.1.65 - 192.168.1.126
```

---

## Wann Subnetting?

| Grund | Beispiel |
|-------|----------|
| **Sicherheit** | Server und Clients trennen |
| **Performance** | Broadcast-Domains verkleinern |
| **Organisation** | Abteilungen trennen |
| **IP-Sparen** | Nur benötigte IPs pro Segment |

---

## Prüfungs-Tipps

1. **Immer 2 abziehen** - Netzwerk + Broadcast = nicht nutzbar
2. **Blockgröße merken** - 256, 128, 64, 32, 16, 8, 4, 2
3. **Binär üben** - Subnetzmaske in Binär schreiben können
4. **/30 für Point-to-Point** - Router-zu-Router braucht nur 2 IPs

---

*Siehe auch: [IP-Adressen](ip-adressen) | [Routing](routing)*
