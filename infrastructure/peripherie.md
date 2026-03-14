---
title: Peripherie
description: Drucker, Scanner und andere Peripheriegeraete im Heimnetz
published: true
date: 2026-03-14T00:00:00.000Z
tags: infrastructure, peripherie, drucker, printer, scanner, hardware, netzwerk
editor: markdown
dateCreated: 2026-03-08T00:00:00.000Z
---

# Peripherie

Drucker, Scanner und andere Peripheriegeraete im Heimnetz.

---

## Drucker

### HP LaserJet Pro M118dw (DruckenStein)

| Parameter | Wert |
|-----------|------|
| **Name** | DruckenStein |
| **Modell** | HP LaserJet Pro M118dw (M118-M119) |
| **MAC** | 38:22:e2:8d:a4:69 |
| **Hostname** | NPI8DA469.local |
| **IP** | DHCP (zuletzt 192.168.0.201) |
| **Typ** | Schwarz/Weiss Laserdrucker |
| **Protokoll** | IPP (driverless) |
| **CUPS URI** | `ipp://NPI8DA469.local/ipp/print` |
| **Verbindung** | WLAN |
| **Duplex** | Ja |
| **Native Formate** | PDF, PostScript, PCL, PCL-XL, JPEG, URF |

### CUPS-Konfiguration

Der Drucker wird im **driverless-Modus** (`-m everywhere`) betrieben. Das bedeutet: PDFs werden direkt an den Drucker gesendet, ohne Konvertierung ueber Ghostscript.

```bash
# Drucker einrichten (driverless, direkt PDF)
lpadmin -p DruckenStein -E -v ipp://NPI8DA469.local/ipp/print -m everywhere

# CUPS Web UI
http://localhost:631

# Drucker Status
lpstat -p -l

# Testseite drucken
lp -d DruckenStein /usr/share/cups/data/testprint

# Alle PDFs in einem Ordner drucken
lp -d DruckenStein /pfad/zum/ordner/*.pdf

# Druckauftraege anzeigen / abbrechen
lpstat -o DruckenStein
cancel -a DruckenStein
```

### Bekannte Probleme

| Problem | Ursache | Loesung |
|---------|---------|---------|
| **gstopxl filter failed** | Ghostscript crasht bei PCL-XL-Konvertierung | Drucker im driverless-Modus neu einrichten: `lpadmin -p DruckenStein -E -v ipp://NPI8DA469.local/ipp/print -m everywhere` — sendet PDF nativ, kein Ghostscript noetig |
| **Drucker nicht gefunden (DNS-SD)** | CUPS nutzt langen DNS-SD URI (`dnssd://HP%20LaserJet...`) der nicht aufgeloest werden kann | URI auf stabilen Hostnamen umstellen: `lpadmin -p DruckenStein -v ipp://NPI8DA469.local/ipp/print` — alternativ direkte IP: `ipp://192.168.0.201/ipp/print` |
| **Jobs stecken in Queue** | Kombination aus DNS-Fehler und Filter-Crash | `cancel -a DruckenStein` und Drucker mit `lpadmin` neu konfigurieren (siehe oben) |
| **Brave kann PDFs nicht drucken** | Brave blockiert lokale Netzwerk-Requests | PDF mit System-Viewer oeffnen oder `brave://flags` → "Block insecure private network requests" → Disabled |
| **IP aendert sich** | Drucker bekommt IP per DHCP | Hostname `NPI8DA469.local` statt IP verwenden (bleibt stabil). Alternativ: MAC-Reservierung in FOG DHCP (MAC: `38:22:e2:8d:a4:69`) |

> **Tipp:** Der Hostname `NPI8DA469.local` wird ueber mDNS/Avahi aufgeloest und ist an die MAC-Adresse gebunden. Er bleibt stabil auch wenn sich die DHCP-IP aendert. Fuer maximale Stabilitaet: MAC-Reservierung in FOG DHCP anlegen.

---

## Scanner

### Canon CanoScan LiDE 400 (scanservjs)

| Parameter | Wert |
|-----------|------|
| **Modell** | Canon CanoScan LiDE 400 |
| **Host** | LXC 119 auf homeserver2 (privilegiert) |
| **IP** | 192.168.0.148 |
| **DNS** | scanner.lab |
| **Web UI** | http://scanner.lab |
| **Verbindung** | USB an homeserver2 |
| **SANE Backend** | pixma |

### scanservjs

Web-basierte Scan-Oberflaeche. Scannen von jedem Geraet im Netzwerk ueber den Browser (auch Handy).

**Datei:** `/opt/scanservjs/docker-compose.yml`

```yaml
services:
  scanservjs:
    image: sbs20/scanservjs:latest
    restart: always
    privileged: true
    ports:
      - "80:8080"
    devices:
      - /dev/bus/usb:/dev/bus/usb
    volumes:
      - scans:/var/lib/scanservjs/output
      - config:/var/lib/scanservjs/config
    environment:
      - PUID=0
      - PGID=0

volumes:
  scans:
  config:
```

**Wichtig:**
- LXC muss **privilegiert** sein (`unprivileged: 0`) fuer USB-Zugriff
- USB-Passthrough in LXC-Config:
  ```
  lxc.cgroup2.devices.allow: c 189:* rwm
  lxc.mount.entry: /dev/bus/usb dev/bus/usb none bind,optional,create=dir
  ```

### Wartung

```bash
# Status
cd /opt/scanservjs
docker compose ps

# Logs
docker logs scanservjs-scanservjs-1 --tail 50

# Update
docker compose pull
docker compose down && docker compose up -d
```

---

*Siehe auch: [Netzwerk-Uebersicht](/en/infrastructure/NETWORK_OVERVIEW) · [DHCP](/en/konzepte/dhcp) · [Paperless-ngx](/en/services/paperless-ngx)*
