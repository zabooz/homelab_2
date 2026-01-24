# 🌐 Apache Reverse Proxy - Setup Dokumentation

**Erstellt:** 24. Januar 2026  
**Author:** Daniel (zabooz)  
**System:** Debian 12 VM (192.168.0.111)  
**Zweck:** Reverse Proxy für Homepage Dashboard

---

## 📊 Was ist ein Reverse Proxy?

Ein **Reverse Proxy** ist ein Server der zwischen Client und Backend-Anwendung sitzt:

```
Client (Browser)
    ↓
Apache (Port 80) ← Du sprichst hiermit
    ↓
Docker Container (Port 3000) ← Apache leitet weiter
```

**Ohne Reverse Proxy:**
```
http://192.168.0.111:3000  ← Port merken!
```

**Mit Reverse Proxy:**
```
http://192.168.0.111       ← Einfach!
```

---

## ✅ Vorteile

- ✅ Keine Ports merken (Port 80 ist Standard)
- ✅ SSL/HTTPS möglich (Let's Encrypt)
- ✅ Mehrere Services auf einem Server
- ✅ Zentrales Logging
- ✅ Load Balancing möglich
- ✅ Security (Backend-IP versteckt)

---

## 🚀 Installation

### 1. Apache installieren

```bash
# Apache2 installieren
sudo apt update
sudo apt install apache2 -y

# Status prüfen
systemctl status apache2
```

### 2. Proxy-Module aktivieren

```bash
# Benötigte Module aktivieren
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod rewrite

# Apache neustarten
sudo systemctl restart apache2
```

---

## ⚙️ Konfiguration

### Apache VirtualHost erstellen

**Datei:** `/etc/apache2/sites-available/homepage.conf`

```bash
# Config-Datei erstellen
sudo nano /etc/apache2/sites-available/homepage.conf
```

**Inhalt:**

```apache
<VirtualHost *:80>
    ServerName 192.168.0.111 
    
    # Proxy-Einstellungen
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
    
    # WebSocket Support (wichtig für Homepage!)
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*) ws://127.0.0.1:3000/$1 [P,L]
    
    # Logging
    ErrorLog ${APACHE_LOG_DIR}/homepage_error.log
    CustomLog ${APACHE_LOG_DIR}/homepage_access.log combined
</VirtualHost>
```

### Konfiguration erklärt

| Directive | Bedeutung |
|-----------|-----------|
| `<VirtualHost *:80>` | Lauscht auf Port 80 (HTTP) |
| `ServerName 192.168.0.111` | Hostname/IP des Servers |
| `ProxyPreserveHost On` | Original-Hostname an Backend weitergeben |
| `ProxyPass / http://127.0.0.1:3000/` | Alle Requests an Docker weiterleiten |
| `ProxyPassReverse /` | Response-Header anpassen |
| `RewriteEngine On` | URL-Rewriting aktivieren |
| `RewriteCond` | Bedingung: Wenn WebSocket-Upgrade |
| `RewriteRule` | Regel: Dann zu ws:// umleiten |

---

## 🔧 Site aktivieren

```bash
# Default-Site deaktivieren (optional)
sudo a2dissite 000-default.conf

# Homepage-Site aktivieren
sudo a2ensite homepage.conf

# Apache Config testen
sudo apache2ctl configtest

# Sollte zeigen: "Syntax OK"

# Apache neustarten
sudo systemctl restart apache2
```

---

## 🧪 Testen

```bash
# Status prüfen
systemctl status apache2

# Port 80 lauscht?
sudo netstat -tuln | grep :80

# Von anderem PC im Netzwerk:
curl http://192.168.0.111

# Oder im Browser:
http://192.168.0.111
```

**Erwartetes Ergebnis:** Homepage Dashboard erscheint!

---

## 📝 Wichtige Befehle

### Apache Service

```bash
# Status anzeigen
systemctl status apache2

# Starten
sudo systemctl start apache2

# Stoppen
sudo systemctl stop apache2

# Neustarten
sudo systemctl restart apache2

# Config neu laden (ohne Neustart)
sudo systemctl reload apache2

# Auto-Start aktivieren
sudo systemctl enable apache2
```

### Config Management

```bash
# Config testen
sudo apache2ctl configtest

# Alle Sites anzeigen
ls -la /etc/apache2/sites-available/

# Aktivierte Sites anzeigen
ls -la /etc/apache2/sites-enabled/

# Site aktivieren
sudo a2ensite <config-name>.conf

# Site deaktivieren
sudo a2dissite <config-name>.conf

# Modul aktivieren
sudo a2enmod <modul-name>

# Modul deaktivieren
sudo a2dismod <modul-name>
```

### Logs

```bash
# Error Log live anschauen
sudo tail -f /var/log/apache2/homepage_error.log

# Access Log live anschauen
sudo tail -f /var/log/apache2/homepage_access.log

# Letzte 50 Zeilen
sudo tail -n 50 /var/log/apache2/homepage_error.log

# Alle Apache Logs
ls -la /var/log/apache2/
```

---

## 🔍 Troubleshooting

### Apache startet nicht

```bash
# Config-Fehler?
sudo apache2ctl configtest

# Logs checken
sudo journalctl -u apache2 -n 50

# Port 80 belegt?
sudo netstat -tuln | grep :80

# Anderen Prozess finden
sudo lsof -i :80
```

### 403 Forbidden

```bash
# Proxy-Module aktiv?
apache2ctl -M | grep proxy

# Sollte zeigen:
# proxy_module
# proxy_http_module

# Falls nicht:
sudo a2enmod proxy proxy_http
sudo systemctl restart apache2
```

### 502 Bad Gateway

```bash
# Docker Container läuft?
docker ps | grep homepage

# Container erreichbar?
curl http://127.0.0.1:3000

# Falls nicht:
cd ~/homepage
docker compose restart
```

### WebSockets funktionieren nicht

```bash
# Rewrite-Modul aktiv?
apache2ctl -M | grep rewrite

# Falls nicht:
sudo a2enmod rewrite
sudo systemctl restart apache2

# WebSocket-Config in VirtualHost?
grep -i websocket /etc/apache2/sites-available/homepage.conf
```

---

## 🔐 SSL/HTTPS (Optional - für später)

### Let's Encrypt Zertifikat

```bash
# Certbot installieren
sudo apt install certbot python3-certbot-apache -y

# Zertifikat erstellen (braucht Domain!)
sudo certbot --apache -d deine-domain.de

# Auto-Renewal testen
sudo certbot renew --dry-run
```

**Wichtig:** Funktioniert nur mit echter Domain (nicht mit IP!)

---

## 🎓 LAP-Relevante Konzepte

### Forward Proxy vs. Reverse Proxy

**Forward Proxy:**
```
Client → Proxy → Internet
(Client kennt Proxy, Server kennt Proxy nicht)
Beispiel: Firmen-Proxy für Internet-Zugang
```

**Reverse Proxy:**
```
Client → Proxy → Backend
(Client kennt nur Proxy, Backend kennt Client nicht)
Beispiel: Apache vor Homepage Docker Container
```

### ProxyPass vs. ProxyPassReverse

**ProxyPass:**
- Leitet **eingehende** Requests weiter
- Client → Apache → Backend

**ProxyPassReverse:**
- Ändert **Response-Header** vom Backend
- Wichtig für Redirects und Location-Header
- Backend antwortet mit "http://127.0.0.1:3000"
- Apache ändert zu "http://192.168.0.111"

### WebSocket Support

**Was sind WebSockets?**
- Bidirektionale, dauerhafte Verbindung
- Anders als HTTP (Request → Response → Ende)
- WebSocket: Verbindung bleibt offen

**Warum wichtig für Homepage?**
- Live-Updates ohne Page-Reload
- Real-time Widgets (CPU, RAM, etc.)

**Apache Config:**
```apache
RewriteCond %{HTTP:Upgrade} =websocket [NC]
RewriteRule /(.*) ws://127.0.0.1:3000/$1 [P,L]
```

---

## 📋 Config-Datei Locations

| Datei/Verzeichnis | Zweck |
|-------------------|-------|
| `/etc/apache2/apache2.conf` | Haupt-Config |
| `/etc/apache2/sites-available/` | Verfügbare Sites (inaktiv) |
| `/etc/apache2/sites-enabled/` | Aktivierte Sites (Symlinks) |
| `/etc/apache2/mods-available/` | Verfügbare Module |
| `/etc/apache2/mods-enabled/` | Aktivierte Module |
| `/var/log/apache2/` | Log-Dateien |

---

## ✅ Setup-Zusammenfassung

**Installiert:**
- Apache2 Webserver
- Proxy-Module (proxy, proxy_http, rewrite)

**Konfiguriert:**
- VirtualHost für 192.168.0.111
- Reverse Proxy zu Docker (Port 3000)
- WebSocket Support

**Zugriff:**
- **Direkt:** http://192.168.0.111:3000
- **Reverse Proxy:** http://192.168.0.111

**Status:** ✅ Funktionsfähig

---

*Dokumentiert am 24. Januar 2026 für LAP-Vorbereitung*
