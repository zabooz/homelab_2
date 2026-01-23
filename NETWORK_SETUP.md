# Network Setup Documentation

## Overview

This server runs multiple services behind an Nginx reverse proxy, with Headscale providing a self-hosted Tailscale VPN. Vaultwarden (password manager) is restricted to VPN access only, while other services remain publicly accessible.

## Network Architecture

```
                                    ┌─────────────────────────────────────────┐
                                    │   zaboozMegaFescherSuperServer          │
                                    │                                         │
 ┌──────────────┐                   │   Public IP: 152.53.111.11              │
 │ Public       │                   │   Tailscale IP: 100.64.0.5              │
 │ Internet     │───────────────────┤                                         │
 │              │                   │   ┌─────────────────────────────────┐   │
 └──────────────┘                   │   │ Nginx (Port 443)                │   │
                                    │   │                                 │   │
                                    │   │  /         → Headscale (:8090)  │   │
 ┌──────────────┐                   │   │  /searx/   → SearXNG (:8888)    │   │
 │ VPN Clients  │                   │   │  /vault/   → Vaultwarden (:8000)│   │
 │ (Tailscale)  │───────────────────┤   │             (VPN only!)         │   │
 │ 100.64.0.0/10│                   │   └─────────────────────────────────┘   │
 └──────────────┘                   │                                         │
                                    └─────────────────────────────────────────┘
```

## Services

| Service | URL | Access | Port (internal) |
|---------|-----|--------|-----------------|
| Headscale | https://zabooz.duckdns.org/ | Public | 127.0.0.1:8090 |
| SearXNG | https://zabooz.duckdns.org/searx/ | Public | 127.0.0.1:8888 |
| Vaultwarden | https://zabooz.duckdns.org/vault/ | **VPN only** | 127.0.0.1:8000 |

## VPN Nodes

| Hostname | IP | User | Description |
|----------|-----|------|-------------|
| tailscale | 100.64.0.1 | zabooz | Exit node |
| maschinchen | 100.64.0.2 | zabooz | Laptop |
| jules | 100.64.0.3 | jules | - |
| zabooz-phone | 100.64.0.4 | zabooz | Phone |
| zaboozmegafeschersuperserver | 100.64.0.5 | zabooz | This server |

## How It Works

### DNS Resolution (MagicDNS)

Headscale provides MagicDNS with base domain `headnet.com`. VPN clients can resolve:
- `<hostname>.headnet.com` → Tailscale IP of that node
- `zabooz.duckdns.org` → 100.64.0.5 (overridden via extra_records)

**For VPN clients:**
```
zabooz.duckdns.org → 100.64.0.5 (Tailscale IP)
```

**For public internet:**
```
zabooz.duckdns.org → 152.53.111.11 (Public IP)
```

This means VPN clients route traffic through the encrypted Tailscale tunnel, while public users go through the regular internet.

### Vaultwarden VPN-Only Access

Nginx restricts `/vault/` to Tailscale IP range:

```nginx
location /vault/ {
    allow 100.64.0.0/10;  # Tailscale range
    deny all;             # Block everyone else
    ...
}
```

- VPN client (100.64.0.x) → **200 OK**
- Public IP → **403 Forbidden**

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/headscale/config.yaml` | Headscale configuration, MagicDNS, extra_records |
| `/etc/nginx/sites-available/headscale` | Nginx reverse proxy config |
| `/home/zabooz/vaultwarden/docker-compose.yml` | Vaultwarden Docker setup |

### Key Headscale Settings

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
    - name: "zabooz.duckdns.org"
      type: "A"
      value: "100.64.0.5"
```

### Vaultwarden Environment

```yaml
# /home/zabooz/vaultwarden/docker-compose.yml

environment:
  DOMAIN: "https://zabooz.duckdns.org/vault"
  WEBSOCKET_ENABLED: "true"
  SIGNUPS_ALLOWED: "false"
  ROCKET_PROXY_PATH: "/vault"
```

## SSL/TLS

Let's Encrypt certificate for `zabooz.duckdns.org`:
- Certificate: `/etc/letsencrypt/live/zabooz.duckdns.org/fullchain.pem`
- Key: `/etc/letsencrypt/live/zabooz.duckdns.org/privkey.pem`
- Auto-renewal via certbot

## Maintenance Commands

```bash
# Restart Headscale
sudo systemctl restart headscale

# Check Headscale status
sudo systemctl status headscale

# List VPN nodes
headscale nodes list

# Restart Vaultwarden
cd /home/zabooz/vaultwarden && docker compose restart

# Test Nginx config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# View Nginx logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

## Troubleshooting

### Vaultwarden returns 403

1. Check you're connected to Tailscale: `tailscale status`
2. Verify DNS resolves correctly: `nslookup zabooz.duckdns.org` (should show 100.64.0.5)
3. If DNS is wrong, restart Tailscale: `sudo systemctl restart tailscaled`

### MagicDNS not resolving

1. Check Headscale is running: `sudo systemctl status headscale`
2. Verify config: `grep -A5 "extra_records" /etc/headscale/config.yaml`
3. Restart Headscale: `sudo systemctl restart headscale`
4. Restart client Tailscale daemon

### Vaultwarden not loading

1. Check container: `docker ps | grep vault`
2. Check logs: `docker logs vaultwarden`
3. Verify Nginx proxy: `curl -v http://127.0.0.1:8000/vault/`

## Security Summary

- **Vaultwarden**: Only accessible from VPN (100.64.0.0/10)
- **Headscale/SearXNG**: Publicly accessible
- **All services**: Behind HTTPS with valid Let's Encrypt cert
- **Docker containers**: Bound to 127.0.0.1 only (not exposed to network)
- **Signups disabled**: Vaultwarden is invite-only
