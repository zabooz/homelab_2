# DNS (Domain Name System)

Namensauflösung im Netzwerk.

---

## Funktion

- Übersetzt Domainnamen in IP-Adressen
- google.com → 142.250.185.46

---

## DNS-Hierarchie

```
1. Lokaler DNS (Router):     192.168.0.1
   ↓ (falls nicht gefunden)
2. Public DNS:               8.8.8.8 (Google)
   ↓
3. Root-DNS-Server           (.)
   ↓
4. TLD-DNS-Server            (.com, .at, .org)
   ↓
5. Authoritative DNS-Server  (google.com)
```

---

## DNS testen

```bash
# Linux
nslookup google.com
dig google.com

# Konfigurierte DNS-Server
cat /etc/resolv.conf
```

---

## MagicDNS (Tailscale/Headscale)

Headscale kann eigene DNS-Records bereitstellen:

```yaml
# /etc/headscale/config.yaml
dns:
  extra_records:
    - name: "home.lab"
      type: "A"
      value: "192.168.0.111"
```

---

*Siehe auch: [Diagnose-Befehle](diagnose)*
