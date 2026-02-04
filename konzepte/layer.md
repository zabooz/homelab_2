---
title: OSI-Schichtenmodell
description: Die 7 Schichten des OSI-Modells und TCP/IP-Modell
published: true
date: 2026-02-01T00:00:00.000Z
tags: konzept, concept, netzwerk, network, osi, layer, schichten, tcp/ip, protokoll, protocol, lap
editor: markdown
dateCreated: 2026-02-01T00:00:00.000Z
---

# OSI-Schichtenmodell (7 Schichten)

Das OSI-Modell (Open Systems Interconnection) beschreibt, wie Daten von einer Anwendung über das Netzwerk zu einer anderen Anwendung gelangen. Jede Schicht hat eine bestimmte Aufgabe.

---

## Übersicht

```
Schicht 7 │ Application   │ Anwendung      │ HTTP, HTTPS, DNS, SMTP, SSH
Schicht 6 │ Presentation  │ Darstellung    │ SSL/TLS, JPEG, ASCII, Verschlüsselung
Schicht 5 │ Session       │ Sitzung        │ Verbindungsauf-/abbau, Sessions
Schicht 4 │ Transport     │ Transport      │ TCP, UDP
Schicht 3 │ Network       │ Vermittlung    │ IP, ICMP, Routing
Schicht 2 │ Data Link     │ Sicherung      │ Ethernet, MAC-Adressen, Switches
Schicht 1 │ Physical      │ Bitübertragung │ Kabel, WLAN, Signale, Hubs
```

> **Merksatz (von oben nach unten):** **A**lle **P**rofessoren **s**aufen **t**equila **n**ach **d**er **P**rüfung
> **Merksatz (von unten nach oben):** **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

---

## Die 7 Schichten im Detail

### Schicht 1 - Physical (Bitübertragung)

**Aufgabe:** Rohe Bits (0 und 1) als elektrische Signale, Licht oder Funk übertragen.

```
Computer A                              Computer B
    │                                       │
    └──── 01101001 ──── Kupferkabel ────────┘
          Spannung an/aus = 1/0
```

**Was passiert hier:**
- Umwandlung von Daten in physische Signale (Strom, Licht, Funk)
- Definiert Kabeltypen, Stecker, Frequenzen, Spannungen

**Geräte:** Hubs, Repeater, Kabel, Netzwerkkarten (physisch)

**Beispiele:**
| Medium | Signal | Einsatz |
|--------|--------|---------|
| Kupferkabel (Cat5/6) | Elektrische Spannung | LAN |
| Glasfaser | Lichtimpulse | WAN, Rechenzentren |
| WLAN (802.11) | Funkwellen | Wireless |

---

### Schicht 2 - Data Link (Sicherung)

**Aufgabe:** Zuverlässige Übertragung zwischen **direkt verbundenen** Geräten im selben Netzwerk. Adressierung über MAC-Adressen.

```
PC-A (MAC: aa:bb:cc:11:22:33)
  │
  └──── Ethernet Frame ────► Switch ────► PC-B (MAC: dd:ee:ff:44:55:66)
        [Ziel-MAC | Quell-MAC | Daten | Prüfsumme]
```

**Was passiert hier:**
- Verpackt Daten in **Frames** (Rahmen)
- MAC-Adressen identifizieren Geräte eindeutig (Hardware-Adresse)
- Fehlererkennung durch Prüfsummen (CRC)
- Switch lernt welche MAC an welchem Port hängt

**Geräte:** Switches, Bridges, Netzwerkkarten (logisch)

**Protokolle:** Ethernet (802.3), WLAN (802.11), ARP

> **Im Homelab:** Dein Switch verbindet Proxmox, Router, etc. Er arbeitet auf Layer 2 und leitet Frames anhand von MAC-Adressen weiter.

---

### Schicht 3 - Network (Vermittlung)

**Aufgabe:** Pakete über **verschiedene Netzwerke** zum Ziel routen. Adressierung über IP-Adressen.

```
PC-A (192.168.0.111)                          VPS (152.53.111.11)
  │                                              │
  └──► Router ──► Internet ──► Router ──────────┘
       192.168.0.1         Routing-Tabellen
                           bestimmen den Weg
```

**Was passiert hier:**
- Verpackt Daten in **Pakete** mit Quell- und Ziel-IP
- Router entscheiden anhand der Routing-Tabelle wohin das Paket geht
- Jeder Hop (Router) trifft eine neue Routing-Entscheidung

**Geräte:** Router, Layer-3-Switches

**Protokolle:** IP (IPv4/IPv6), ICMP (ping), ARP

**Unterschied zu Layer 2:**
| | Layer 2 | Layer 3 |
|--|---------|---------|
| Adresse | MAC (Hardware) | IP (Logisch) |
| Bereich | Lokales Netzwerk | Über Netzwerke hinweg |
| Gerät | Switch | Router |

> **Im Homelab:** Pakete von deinem PC (192.168.0.x) zum VPS (152.53.111.11) werden über mehrere Router durch das Internet geleitet.

---

### Schicht 4 - Transport

**Aufgabe:** Ende-zu-Ende Verbindung zwischen zwei Anwendungen. Entscheidet **wie** Daten transportiert werden (zuverlässig oder schnell).

```
Browser (Port 54321)  ──────────────►  Webserver (Port 443)
                      TCP-Verbindung
                      Daten kommen an!
```

**Zwei Protokolle:**

| | TCP | UDP |
|--|-----|-----|
| **Verbindung** | Ja (3-Way-Handshake) | Nein (einfach senden) |
| **Zuverlässig** | Ja (Bestätigung, Neuübertragung) | Nein (Fire & Forget) |
| **Reihenfolge** | Garantiert | Nicht garantiert |
| **Geschwindigkeit** | Langsamer (Overhead) | Schneller |
| **Einsatz** | HTTP, SSH, E-Mail, Dateitransfer | DNS, VoIP, Streaming, Games |

**Ports** identifizieren die Anwendung auf einem Host:
- Quell-Port (zufällig, z.B. 54321) + Ziel-Port (bekannt, z.B. 443)
- Portnummern 0-1023: Well-known Ports (brauchen root)
- Portnummern 1024-65535: Frei verwendbar

> **Im Homelab:** Veloren-Gameserver nutzt TCP auf Port 14004. Der Nginx Stream Proxy auf dem VPS arbeitet auf Layer 4 - er versteht kein HTTP, sondern leitet TCP/UDP-Pakete weiter.

---

### Schicht 5 - Session (Sitzung)

**Aufgabe:** Verbindungen (Sessions) zwischen Anwendungen aufbauen, verwalten und beenden.

```
Client                     Server
  │── Session aufbauen ──►│
  │◄── Session OK ────────│
  │── Daten senden ──────►│
  │── Daten senden ──────►│
  │── Session beenden ───►│
```

**Was passiert hier:**
- Steuert den Dialog zwischen zwei Systemen
- Synchronisationspunkte setzen (Checkpoints)
- Bei Unterbrechung: Session kann wiederhergestellt werden

**Beispiele:**
- RPC (Remote Procedure Call)
- NetBIOS Sessions
- Login-Sessions auf Webseiten

> **Praxis:** Layer 5 ist in modernen Protokollen oft mit Layer 4 oder 7 verschmolzen. TCP übernimmt Session-Funktionen, und HTTP verwaltet eigene Sessions (Cookies).

---

### Schicht 6 - Presentation (Darstellung)

**Aufgabe:** Daten in ein Format umwandeln, das die Anwendung versteht. Verschlüsselung, Kompression, Zeichensätze.

```
Klartext "Passwort123"
    │
    ▼ Verschlüsselung (TLS)
"a7f3b2c9e1d8..."
    │
    ▼ Kompression
"a7f3..." (kleiner)
    │
    ▼ Senden
```

**Was passiert hier:**
- **Verschlüsselung/Entschlüsselung:** SSL/TLS
- **Kompression:** Daten kleiner machen für schnellere Übertragung
- **Zeichensatzumwandlung:** ASCII, UTF-8, EBCDIC
- **Datenformat:** JPEG, PNG, JSON, XML

**Beispiele:**
- HTTPS = HTTP + TLS (Layer 6 verschlüsselt Layer 7 Daten)
- Bilddateien: JPEG-Kompression
- Textcodierung: UTF-8

> **Praxis:** Wie Layer 5 ist Layer 6 in modernen Stacks oft in die Anwendung oder Bibliotheken integriert (z.B. OpenSSL für TLS).

---

### Schicht 7 - Application (Anwendung)

**Aufgabe:** Schnittstelle zwischen Netzwerk und Benutzer/Anwendung. Die Protokolle, mit denen Programme direkt arbeiten.

```
Browser ──HTTP GET /──► Nginx ──► Webseite anzeigen
E-Mail  ──SMTP────────► Mailserver ──► E-Mail zustellen
Terminal──SSH──────────► Server ──► Shell-Zugang
```

**Was passiert hier:**
- Anwendungsprotokolle definieren wie Programme kommunizieren
- Benutzer interagiert (in)direkt mit dieser Schicht

**Wichtige Protokolle:**
| Protokoll | Port | Funktion |
|-----------|------|----------|
| HTTP/HTTPS | 80/443 | Webseiten |
| SSH | 22 | Sichere Shell |
| DNS | 53 | Namensauflösung |
| SMTP | 25 | E-Mail senden |
| IMAP | 143 | E-Mail empfangen |
| DHCP | 67/68 | IP-Adressvergabe |
| FTP | 21 | Dateitransfer |

> **Im Homelab:** Nginx HTTP Reverse Proxy arbeitet auf Layer 7 - er versteht HTTP, kann URLs auswerten (`/vault/`, `/searx/`) und Header setzen. Im Gegensatz dazu arbeitet der Stream Proxy auf Layer 4.

---

## OSI vs. TCP/IP-Modell

In der Praxis wird oft das vereinfachte TCP/IP-Modell (4 Schichten) verwendet:

```
        OSI-Modell              TCP/IP-Modell
    ┌───────────────┐      ┌───────────────────┐
    │ 7 Application │      │                   │
    ├───────────────┤      │   Application     │
    │ 6 Presentation│      │   (HTTP, DNS,     │
    ├───────────────┤      │    SSH, SMTP)      │
    │ 5 Session     │      │                   │
    ├───────────────┤      ├───────────────────┤
    │ 4 Transport   │      │   Transport       │
    │               │      │   (TCP, UDP)       │
    ├───────────────┤      ├───────────────────┤
    │ 3 Network     │      │   Internet        │
    │               │      │   (IP, ICMP)       │
    ├───────────────┤      ├───────────────────┤
    │ 2 Data Link   │      │   Network Access  │
    ├───────────────┤      │   (Ethernet,       │
    │ 1 Physical    │      │    WLAN)           │
    └───────────────┘      └───────────────────┘
```

**Warum zwei Modelle?**
- **OSI:** Theoretisches Referenzmodell (7 Schichten, gut für Prüfungen)
- **TCP/IP:** Praktisches Modell (4 Schichten, wie das Internet wirklich funktioniert)
- Layer 5, 6, 7 sind im TCP/IP-Modell zu einer Schicht zusammengefasst

---

## Datenkapselung (Encapsulation)

Jede Schicht verpackt die Daten der darüberliegenden Schicht mit eigenem Header:

```
Layer 7:                              [Daten]
Layer 4:                    [TCP-Header | Daten]          → Segment
Layer 3:           [IP-Header | TCP-Header | Daten]       → Paket
Layer 2:  [MAC-Header | IP-Header | TCP-Header | Daten | CRC]  → Frame
Layer 1:   01001010110010101001010101001010101...          → Bits
```

**Bezeichnungen je Schicht:**
| Schicht | Dateneinheit | Englisch |
|---------|-------------|----------|
| Layer 7 | Daten | Data |
| Layer 4 | Segment (TCP) / Datagramm (UDP) | Segment / Datagram |
| Layer 3 | Paket | Packet |
| Layer 2 | Frame (Rahmen) | Frame |
| Layer 1 | Bits | Bits |

---

## Praxis-Bezug im Homelab

```
Spieler verbindet sich zu Veloren (152.53.111.11:14004):

Layer 7: Veloren Game-Protokoll (Anwendungsdaten)
Layer 4: TCP, Ziel-Port 14004 ← Nginx Stream Proxy arbeitet hier
Layer 3: IP: Ziel 152.53.111.11 → routing → 192.168.0.120
Layer 2: Ethernet Frame mit MAC-Adresse des nächsten Hops
Layer 1: Elektrische Signale über Kabel / Funk
```

---

*Siehe auch: [Reverse Proxy](/en/konzepte/reverse-proxy) | [Ports](/en/konzepte/ports) | [Routing](/en/konzepte/routing) | [IP-Adressen](/en/konzepte/ip-adressen)*
