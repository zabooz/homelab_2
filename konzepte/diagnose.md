# Diagnose-Befehle

Wichtige Befehle für Netzwerk-Troubleshooting.

---

## Netzwerk

```bash
ip addr show              # IP-Konfiguration
ip route show             # Routing-Tabelle
ip route get <ip>         # Route für spezifische IP
ping <ip>                 # Erreichbarkeit testen
traceroute <ip>           # Route verfolgen
ss -tuln                  # Offene Ports
```

---

## DNS

```bash
nslookup <domain>         # DNS-Auflösung
dig <domain>              # Detaillierte DNS-Info
cat /etc/resolv.conf      # Konfigurierte DNS-Server
```

---

## Traffic-Analyse

```bash
tcpdump -i any host <ip>  # Traffic beobachten
tcpdump -i eth0 -n        # Interface-spezifisch
```

---

## Tailscale/Headscale

```bash
tailscale status          # VPN-Status
tailscale netcheck        # Verbindungsqualität
ip route show table 52    # Tailscale Routes
```

---

## Windows

```powershell
ipconfig /all             # IP-Konfiguration
ipconfig /flushdns        # DNS-Cache leeren
route print               # Routing-Tabelle
Test-NetConnection <ip> -Port <port>  # Port-Test
```

---

*Siehe auch: [DNS](dns) | [Routing](routing)*
