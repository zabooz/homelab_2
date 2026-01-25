# Wichtige Ports

Häufig verwendete Netzwerk-Ports.

---

## Standard-Ports

| Port | Protokoll | Dienst |
|------|-----------|--------|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3478 | UDP | STUN (NAT-Traversal) |
| 8006 | TCP | Proxmox Web UI |

---

## Homelab-spezifische Ports

| Port | Dienst |
|------|--------|
| 3000 | Wiki.js, Homepage (intern) |
| 4000 | Stats API |
| 8000 | Paperless-ngx |
| 8080 | Headscale UI |
| 8090 | Headscale API |

---

*Siehe auch: [iptables](iptables) | [Diagnose-Befehle](diagnose)*
