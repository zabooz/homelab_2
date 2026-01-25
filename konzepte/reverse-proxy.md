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

## Beispiel: Apache als Reverse Proxy

```apache
<VirtualHost *:80>
    ServerName 192.168.0.111

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
</VirtualHost>
```

---

*Siehe auch: [Ports](ports) | [Linux Interfaces](linux-interfaces)*
