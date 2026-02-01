# VPS - zaboozMegaFescherSuperServer

**IP:** 152.53.111.11 (Public) | 100.64.0.5 (Tailscale)
**Domain:** zabooz.duckdns.org
**OS:** Debian 13

**Siehe auch:**
- [Reverse Proxy](../konzepte/reverse-proxy.md) - Forward vs. Reverse Proxy
- [Ports](../konzepte/ports.md) - Wichtige Netzwerk-Ports
- [VPN-Typen](../konzepte/vpn.md) - Mesh VPN, WireGuard

## Übersicht

Dieser Server betreibt mehrere Dienste hinter einem Nginx Reverse Proxy. Headscale stellt ein selbst-gehostetes Tailscale VPN bereit. Vaultwarden (Passwort-Manager) ist nur über VPN erreichbar, während andere Dienste öffentlich zugänglich sind.

## Netzwerk-Architektur

```
                                    ┌─────────────────────────────────────────┐
                                    │   zaboozMegaFescherSuperServer          │
                                    │                                         │
 ┌──────────────┐                   │   Public IP: 152.53.111.11              │
 │ Öffentliches │                   │   Tailscale IP: 100.64.0.5              │
 │ Internet     │───────────────────┤                                         │
 │              │                   │   ┌─────────────────────────────────┐   │
 └──────────────┘                   │   │ Nginx (Port 443)                │   │
                                    │   │                                 │   │
                                    │   │  /         → Headscale (:8090)  │   │
 ┌──────────────┐                   │   │  /web      → Headscale UI(:8080)│   │
 │ VPN Clients  │                   │   │  /searx/   → SearXNG (:8888)    │   │
 │ (Tailscale)  │───────────────────┤   │  /vault/   → Vaultwarden (:8000)│   │
 │ 100.64.0.0/10│                   │   │  /draw/    → Draw.io (:8081)    │   │
 └──────────────┘                   │   │             (Nur VPN!)          │   │
                                    │   └─────────────────────────────────┘   │
                                    │                                         │
                                    └─────────────────────────────────────────┘
```

## Dienste

| Dienst | URL | Zugriff | Port (intern) |
|--------|-----|---------|---------------|
| Headscale | https://zabooz.duckdns.org/ | Öffentlich | 127.0.0.1:8090 |
| Headscale UI | https://zabooz.duckdns.org/web | **Nur VPN** | 127.0.0.1:8080 |
| SearXNG | https://zabooz.duckdns.org/searx/ | **Nur VPN** | 127.0.0.1:8888 |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | **Nur VPN** | 127.0.0.1:8000 |
| Draw.io | https://zabooz.duckdns.org/draw/ | **Nur VPN** | 127.0.0.1:8081 |

## VPN Nodes

| Hostname | IP | Benutzer | Beschreibung |
|----------|-----|----------|--------------|
| tailscale | 100.64.0.1 | zabooz | Exit-Node, Subnet Router |
| maschinchen | 100.64.0.2 | zabooz | Laptop |
| jules | 100.64.0.3 | jules | - |
| zabooz-phone | 100.64.0.4 | zabooz | Handy |
| zaboozmegafeschersuperserver | 100.64.0.5 | zabooz | Dieser Server |

## Funktionsweise

### DNS-Auflösung (MagicDNS)

Headscale stellt MagicDNS mit der Basis-Domain `headnet.com` bereit. VPN-Clients können auflösen:
- `<hostname>.headnet.com` → Tailscale IP des Nodes
- `home.lab` → 192.168.0.111 (Homepage Dashboard)
- `vps.lab` → 100.64.0.5 (VPS via Tailscale)
- `zabooz.duckdns.org` → 100.64.0.5 (VPS via Tailscale, für VPN-only Services)

**DNS-Auflösung (zwei Wege):**
```
Ohne VPN:  zabooz.duckdns.org → 152.53.111.11 (Öffentliche IP via DuckDNS)
Mit VPN:   zabooz.duckdns.org → 100.64.0.5    (Tailscale IP via MagicDNS)
```

> **Warum zwei Auflösungen?** Nginx erlaubt VPN-only Services (`/vault/`, `/searx/`, `/web`) nur für Tailscale IPs (100.64.0.0/10). Wenn VPN-Clients `zabooz.duckdns.org` über MagicDNS auflösen, geht der Traffic durch den Tailscale-Tunnel und Nginx sieht die Tailscale-IP → Zugriff erlaubt.
>
> **Edge Case:** Falls VPN-Verbindung abbricht und nicht reconnecten kann (DNS-Cache zeigt auf Tailscale IP), DNS-Cache leeren: `resolvectl flush-caches`

### Vaultwarden Nur-VPN-Zugriff

Nginx beschränkt `/vault/` auf den Tailscale IP-Bereich:

```nginx
location /vault/ {
    allow 100.64.0.0/10;  # Tailscale Bereich
    deny all;             # Alle anderen blockieren
    ...
}
```

- VPN-Client (100.64.0.x) → **200 OK**
- Öffentliche IP → **403 Forbidden**

## Konfigurations-Dateien

| Datei | Zweck |
|-------|-------|
| `/etc/headscale/config.yaml` | Headscale Konfiguration, MagicDNS, extra_records |
| `/etc/nginx/sites-available/default` | Nginx Reverse Proxy Konfiguration |
| `/home/zabooz/vaultwarden/docker-compose.yml` | Vaultwarden Docker Setup |
| `/home/zabooz/draw.io/docker-compose.yml` | Draw.io Docker Setup |

### Wichtige Headscale Einstellungen

```yaml
# /etc/headscale/config.yaml

server_url: https://zabooz.duckdns.org
listen_addr: 127.0.0.1:8090

dns:
  magic_dns: true
  base_domain: headnet.com
  override_local_dns: true
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1
  extra_records:
    - name: "home.lab"
      type: "A"
      value: "192.168.0.111"
    - name: "vps.lab"
      type: "A"
      value: "100.64.0.5"
    - name: "zabooz.duckdns.org"
      type: "A"
      value: "100.64.0.5"
```

### Vaultwarden Umgebung

```yaml
# /home/zabooz/vaultwarden/docker-compose.yml

environment:
  DOMAIN: "https://zabooz.duckdns.org/vault"
  WEBSOCKET_ENABLED: "true"
  SIGNUPS_ALLOWED: "false"
  ROCKET_PROXY_PATH: "/vault"
```

## SSL/TLS

Let's Encrypt Zertifikat für `zabooz.duckdns.org`:
- Zertifikat: `/etc/letsencrypt/live/zabooz.duckdns.org/fullchain.pem`
- Schlüssel: `/etc/letsencrypt/live/zabooz.duckdns.org/privkey.pem`
- Automatische Erneuerung via certbot

## Wartungs-Befehle

```bash
# Headscale neustarten
docker restart headscale

# Headscale Status prüfen
docker ps | grep headscale
docker logs headscale --tail 20

# VPN Nodes auflisten
docker exec headscale headscale nodes list

# Vaultwarden neustarten
cd /home/zabooz/vaultwarden && docker compose restart

# Nginx Konfiguration testen
sudo nginx -t

# Nginx neu laden
sudo systemctl reload nginx

# Nginx Logs anzeigen
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

## Fehlerbehebung

### Vaultwarden gibt 403 zurück

1. Prüfen ob Tailscale verbunden ist: `tailscale status`
2. Verifizieren dass deine Tailscale IP im 100.64.0.0/10 Bereich ist
3. Falls nicht verbunden, Tailscale neustarten: `sudo systemctl restart tailscaled`

### MagicDNS löst nicht auf

1. Prüfen ob Headscale läuft: `docker ps | grep headscale`
2. Konfiguration verifizieren: `grep -A5 "extra_records" /etc/headscale/config.yaml`
3. Headscale neustarten: `docker restart headscale`
4. Client Tailscale Daemon neustarten

### Vaultwarden lädt nicht

1. Container prüfen: `docker ps | grep vault`
2. Logs prüfen: `docker logs vaultwarden`
3. Nginx Proxy verifizieren: `curl -v http://127.0.0.1:8000/vault/`

## Sicherheits-Zusammenfassung

- **Vaultwarden**: Nur über VPN erreichbar (100.64.0.0/10)
- **SearXNG**: Nur über VPN erreichbar (100.64.0.0/10)
- **Draw.io**: Nur über VPN erreichbar (100.64.0.0/10)
- **Headscale**: Öffentlich erreichbar (muss für VPN-Verbindung erreichbar sein)
- **Alle Dienste**: Hinter HTTPS mit gültigem Let's Encrypt Zertifikat
- **Docker Container**: Nur an 127.0.0.1 gebunden (nicht im Netzwerk exponiert)
- **Registrierung deaktiviert**: Vaultwarden ist nur auf Einladung

---

*Letzte Aktualisierung: 25. Januar 2026*
