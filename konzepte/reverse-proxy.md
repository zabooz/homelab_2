---
title: Reverse Proxy
description: HTTP Reverse Proxy, Nginx Stream Proxy, TCP/UDP Grundlagen
published: true
date: 2026-02-01T00:00:00.000Z
tags: konzept, concept, netzwerk, network, nginx, reverse-proxy, stream-proxy, tcp, udp, layer4, layer7, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Reverse Proxy

Webserver als Vermittler zwischen Clients und Backend.

---

## Forward vs. Reverse Proxy

```
Forward Proxy:
Client → Proxy → Internet
(Client kennt Proxy, Server nicht)
Beispiel: Firmen-Proxy für Internet-Zugang

Reverse Proxy:
Client → Proxy → Backend
(Client kennt nur Proxy)
Beispiel: Nginx vor Webserver
```

---

## Vorteile Reverse Proxy

- Keine Ports merken (Port 80/443)
- SSL/HTTPS terminieren
- Mehrere Services auf einem Server
- Load Balancing
- Security (Backend versteckt)

---

## Layer 7 (HTTP) vs. Layer 4 (Stream) Proxy

Ein Reverse Proxy kann auf verschiedenen Netzwerk-Schichten arbeiten:

```
OSI-Modell (vereinfacht):

Layer 7 - Application   → HTTP, HTTPS (Webseiten, APIs)
Layer 4 - Transport      → TCP, UDP (rohe Verbindungen)
Layer 3 - Network        → IP (Routing)
```

### Layer 7 - HTTP Reverse Proxy (Standard)

Der Proxy **versteht** HTTP. Er kann:
- URLs auswerten (`/vault/` → Vaultwarden, `/searx/` → SearXNG)
- HTTP-Header setzen/ändern (Host, X-Forwarded-For)
- SSL/TLS terminieren (HTTPS → HTTP intern)
- Caching, Compression, Rate-Limiting

```
Client ──HTTPS──► Nginx ──HTTP──► Backend (:8000)
                    │
                    ├─ /vault/  → Vaultwarden
                    ├─ /searx/  → SearXNG
                    └─ /draw/   → Draw.io
```

**Nginx Konfiguration** (in `http {}` Block):
```nginx
location /vault/ {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
}
```

### Layer 4 - Stream Proxy (TCP/UDP)

Der Proxy versteht **kein** HTTP. Er leitet rohe TCP/UDP-Pakete weiter. Nützlich für:
- **Gameserver** (Veloren, Xonotic, Minecraft)
- **Datenbanken** (MySQL, PostgreSQL)
- **Mail-Server** (SMTP, IMAP)
- Alles was kein HTTP spricht

```
Spieler ──TCP:14004──► Nginx ──TCP──► Gameserver (192.168.0.120:14004)
```

**Nginx Konfiguration** (in `stream {}` Block, **außerhalb** von `http {}`):
```nginx
stream {
    server {
        listen 14004;                        # Port auf dem VPS
        proxy_pass 192.168.0.120:14004;      # Weiterleitung zum Backend
    }
}
```

### Unterschied auf einen Blick

| | Layer 7 (HTTP) | Layer 4 (Stream) |
|--|----------------|-------------------|
| **Versteht** | HTTP-Protokoll | Nur TCP/UDP |
| **Routing** | Nach URL-Pfad (`/vault/`) | Nach Port (:14004) |
| **SSL** | Kann terminieren | Kann durchleiten (passthrough) |
| **Header** | Kann lesen/ändern | Keine Header (rohe Bytes) |
| **Nginx Block** | `http { }` | `stream { }` |
| **Anwendung** | Webseiten, APIs | Gameserver, Datenbanken |

---

## TCP vs. UDP

Zwei Transport-Protokolle, die bestimmen **wie** Daten übertragen werden:

### TCP (Transmission Control Protocol)

- **Verbindungsorientiert**: Erst Verbindung aufbauen (3-Way-Handshake), dann Daten senden
- **Zuverlässig**: Verlorene Pakete werden erneut gesendet
- **Reihenfolge garantiert**: Pakete kommen in der richtigen Reihenfolge an
- **Langsamer** durch den Overhead

```
Client           Server
  │── SYN ──────►│       1. "Ich will verbinden"
  │◄── SYN-ACK ──│       2. "OK, ich bin bereit"
  │── ACK ──────►│       3. "Verbindung steht"
  │── Daten ────►│       4. Daten senden
  │◄── ACK ──────│       5. "Angekommen!"
```

**Verwendet für:** HTTP/S, SSH, E-Mail, Dateitransfer, die meisten Gameserver

### UDP (User Datagram Protocol)

- **Verbindungslos**: Einfach Pakete senden, kein Handshake
- **Unzuverlässig**: Verlorene Pakete werden NICHT erneut gesendet
- **Keine Reihenfolge**: Pakete können in beliebiger Reihenfolge ankommen
- **Schneller** durch weniger Overhead

```
Client           Server
  │── Daten ────►│       Einfach senden
  │── Daten ────►│       Kein Warten auf Bestätigung
  │── Daten ────►│       Manche Pakete gehen verloren? Egal!
```

**Verwendet für:** DNS, VoIP, Video-Streaming, manche Gameserver (Echtzeit-Spiele)

### Warum nutzen Gameserver manchmal beides?

| Datentyp | Protokoll | Grund |
|----------|-----------|-------|
| Spieler-Position, Bewegung | UDP | Echtzeit, veraltete Daten sind wertlos |
| Chat, Inventar, Login | TCP | Muss zuverlässig ankommen |
| Datei-Downloads (Maps) | TCP | Darf nichts fehlen |

> **Im Homelab:** Veloren nutzt TCP. Xonotic nutzt TCP + UDP (Quake-Engine). Im Zweifel beide Protokolle weiterleiten.

---

## Beispiel: Apache als HTTP Reverse Proxy

```apache
<VirtualHost *:80>
    ServerName 192.168.0.111

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
</VirtualHost>
```

---

## Praxis: Gameserver über VPS erreichbar machen

So sind die Gameserver im Homelab konfiguriert:

```
                    INTERNET
                       │
            ┌──────────▼──────────┐
            │  VPS (152.53.111.11)│
            │                     │
            │  Nginx Stream:      │
            │  :14004 ──┐         │
            │  :14005 ──┤         │
            └───────────┼─────────┘
                        │ Tailscale Tunnel
            ┌───────────▼─────────┐
            │  Pterodactyl        │
            │  (192.168.0.120)    │
            │                     │
            │  Veloren  :14004    │
            │  Xonotic  :14005   │
            └─────────────────────┘
```

**Zwei Zugangswege:**
- **Lokal/VPN**: `192.168.0.120:14004` (direkt im Heimnetz oder über Tailscale)
- **Internet**: `152.53.111.11:14004` (über VPS Stream Proxy)

Spieler von außen brauchen kein VPN - sie verbinden sich direkt zur öffentlichen VPS-IP.

---

*Siehe auch: [Ports](ports) | [Linux Interfaces](linux-interfaces) | [VPN](vpn) | [VPS](../infrastructure/VPS)*
