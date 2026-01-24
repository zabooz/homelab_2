# 🏠 Homepage Dashboard - Setup Dokumentation

**Erstellt:** 24. Januar 2026  
**Author:** Daniel (zabooz)  
**System:** Debian 12 VM (192.168.0.111)  
**Zugriff:** http://192.168.0.111:3000

---

## 📊 Was ist Homepage?

Selbst-gehostetes Application Dashboard mit Service-Überwachung, Widgets und Proxmox-Integration.

**Offizielle Doku:** https://gethomepage.dev

---

## 🚀 Installation

```bash
# Verzeichnis erstellen
mkdir -p ~/homepage/config
cd ~/homepage

# Docker Compose File erstellen
nano docker-compose.yml
```

**docker-compose.yml:**
```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=*
    restart: unless-stopped
```

```bash
# Container starten
docker compose up -d

# Logs checken
docker compose logs -f
```

---

## 📁 Wo finde ich was?

### Verzeichnisstruktur

```
~/homepage/
├── docker-compose.yml          # Docker Container Config
└── config/
    ├── services.yaml           # ← Services hinzufügen (Proxmox, VPS, etc.)
    ├── widgets.yaml            # ← System-Widgets (CPU, RAM, Suche)
    ├── bookmarks.yaml          # ← Lesezeichen/Quick-Links
    ├── settings.yaml           # ← Allgemeine Einstellungen (Titel, Theme, etc.)
    ├── docker.yaml             # Docker-Integration
    └── proxmox.yaml            # Proxmox-API Config
```

**Nach Änderungen:**
```bash
cd ~/homepage
docker compose restart
```

---

## ⚙️ Konfigurationsdateien

### 1️⃣ services.yaml - Services hinzufügen

**Datei:** `~/homepage/config/services.yaml`

```yaml
- Kategorie-Name:
    - Service-Name:
        icon: proxmox.png
        href: https://192.168.0.101:8006
        description: Beschreibung
        widget:                    # Optional - zeigt Live-Daten
          type: proxmox
          url: https://192.168.0.101:8006
          username: root@pam!homepage
          password: API-TOKEN
          node: homeserver
          fields: ["vms", "lxc", "resources.cpu"]
```

**Aktuell konfiguriert:**
- **Infrastructure:** Proxmox (mit Widget)
- **VPS Services:** Vaultwarden, SearXNG, Headscale

---

### 2️⃣ widgets.yaml - System-Widgets

**Datei:** `~/homepage/config/widgets.yaml`

```yaml
- resources:              # System-Ressourcen anzeigen
    cpu: true
    memory: true
    disk: /

- search:                 # Suchleiste oben
    provider: duckduckgo
    target: _blank
```

**Weitere mögliche Widgets:**
- `datetime` - Uhr & Datum
- `openmeteo` - Wetter
- `greeting` - Begrüßung

**Doku:** https://gethomepage.dev/widgets/

---

### 3️⃣ bookmarks.yaml - Lesezeichen

**Datei:** `~/homepage/config/bookmarks.yaml`

```yaml
- Developer:
    - Github:
        - abbr: GH
          href: https://github.com/

- Social:
    - Reddit:
        - abbr: RE
          href: https://reddit.com/
```

---

### 4️⃣ settings.yaml - Allgemeine Einstellungen

**Datei:** `~/homepage/config/settings.yaml`

```yaml
title: Homepage           # Titel im Browser-Tab
theme: dark              # dark / light
color: slate             # Farbschema
layout:
  - Infrastructure       # Kategorie-Reihenfolge
  - VPS Services
```

---

## 🔧 Wichtige Befehle

### Docker Container

```bash
# Status prüfen
docker ps | grep homepage

# Logs anschauen
docker compose logs -f homepage

# Container neustarten (nach Config-Änderung)
cd ~/homepage
docker compose restart

# Container stoppen/starten
docker compose down
docker compose up -d

# Container komplett neu bauen
docker compose down
docker compose pull
docker compose up -d
```

### Config bearbeiten

```bash
# Services hinzufügen
nano ~/homepage/config/services.yaml

# Widgets ändern
nano ~/homepage/config/widgets.yaml

# Bookmarks bearbeiten
nano ~/homepage/config/bookmarks.yaml

# Settings ändern
nano ~/homepage/config/settings.yaml

# Nach jeder Änderung:
cd ~/homepage && docker compose restart
```

---

## 🎯 Proxmox Integration

### API-Token erstellen (auf Proxmox)

```bash
# Auf Proxmox Host (192.168.0.101)
pveum user add root@pam!homepage
pveum user token add root@pam!homepage homepage --privsep 0

# Token wird angezeigt - kopieren!
```

### In Homepage konfigurieren

**In services.yaml:**
```yaml
widget:
  type: proxmox
  url: https://192.168.0.101:8006
  username: root@pam!homepage
  password: DEIN-API-TOKEN-HIER
  node: homeserver
  fields: ["vms", "lxc", "resources.cpu", "resources.mem"]
```

**Mögliche Fields:**
- `vms` - Anzahl VMs
- `lxc` - Anzahl Container
- `resources.cpu` - CPU-Auslastung
- `resources.mem` - RAM-Auslastung
- `resources.disk` - Disk-Auslastung

---

## 🔍 Troubleshooting

### Homepage lädt nicht

```bash
# Container läuft?
docker ps | grep homepage

# Logs checken
docker compose logs homepage

# Port belegt?
sudo netstat -tuln | grep 3000

# Container neu starten
docker compose restart
```

---

## 📚 Nützliche Links

```bash
# API-Token korrekt?
# In services.yaml prüfen

# SSL-Zertifikat-Problem?
# Proxmox nutzt Self-Signed Cert - ist normal

# Node-Name korrekt?
# Muss "homeserver" sein (wie in Proxmox)
```

### Proxmox Widget zeigt keine Daten

```bash
# API-Token korrekt?
# In services.yaml prüfen

# SSL-Zertifikat-Problem?
# Proxmox nutzt Self-Signed Cert - ist normal

# Node-Name korrekt?
# Muss "homeserver" sein (wie in Proxmox)
```

### Config-Änderungen werden nicht übernommen

```bash
# YAML-Syntax korrekt?
# YAML ist einrückungs-empfindlich!

# Container neu starten
cd ~/homepage
docker compose restart

# Cache leeren
docker compose down
docker compose up -d
```

---

## 📚 Nützliche Links

- **Homepage Doku:** https://gethomepage.dev
- **Service Widgets:** https://gethomepage.dev/widgets/services/
- **Info Widgets:** https://gethomepage.dev/widgets/info/
- **Icons:** https://gethomepage.dev/configs/services/#icons
- **Docker Doku:** https://gethomepage.dev/installation/docker/

---

## 🎓 LAP-Relevante Konzepte

### Docker Volumes

```yaml
volumes:
  - ./config:/app/config                    # Host → Container
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

**Erklärung:**
- `./config:/app/config` - Lokale Dateien in Container verfügbar
- `/var/run/docker.sock` - Docker-Socket für Container-Monitoring
- `:ro` - Read-Only (Container kann nicht schreiben)

### YAML-Syntax

**Wichtig:**
- Einrückung mit **Leerzeichen** (keine Tabs!)
- 2 Leerzeichen pro Ebene
- `-` für Listen
- `:` für Key-Value

**Beispiel:**
```yaml
- Kategorie:              # Liste (-)
    - Service:            # Verschachtelte Liste
        key: value        # Key-Value Pair
        list:             # Liste als Value
          - item1
          - item2
```

---

## ✅ Setup-Zusammenfassung

**Installiert:**
- Homepage Dashboard (Docker Container)
- Proxmox Integration mit API

**Konfiguriert:**
- 3 Service-Kategorien (Infrastructure, VPS Services)
- System-Widgets (CPU, RAM, Disk)
- Lesezeichen (Github, Reddit, YouTube)

**Zugriff:**
- http://192.168.0.111:3000

**Status:** ✅ Funktionsfähig

---

*Dokumentiert am 24. Januar 2026 für LAP-Vorbereitung*
