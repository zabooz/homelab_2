# IP-Adressierung & Subnetting

LAP-relevante Grundlagen zu IP-Adressen und Subnetzen.

---

## Subnetzmasken verstehen

**255.255.255.0 in Binär:**
```
11111111.11111111.11111111.00000000
└─────────────────────────┘└──────┘
    Netzwerk-Teil (24 Bit)  Host-Teil (8 Bit)
```

---

## CIDR-Notation

| CIDR | Subnetzmaske | Verfügbare Hosts | Beispiel |
|------|--------------|------------------|----------|
| /24 | 255.255.255.0 | 254 | 192.168.0.0/24 |
| /16 | 255.255.0.0 | 65,534 | 192.168.0.0/16 |
| /8 | 255.0.0.0 | 16,777,214 | 10.0.0.0/8 |
| /32 | 255.255.255.255 | 1 (einzelner Host) | 192.168.0.1/32 |
| /10 | 255.192.0.0 | 4,194,302 | 100.64.0.0/10 (CGNAT) |

---

## Berechnung für /24

```
Netzwerk:      192.168.0.0
Broadcast:     192.168.0.255
Erster Host:   192.168.0.1 (meist Gateway)
Letzter Host:  192.168.0.254
Verfügbar:     254 Adressen
```

---

## Private IP-Bereiche (RFC 1918)

| Klasse | Bereich | CIDR | Nutzung |
|--------|---------|------|---------|
| A | 10.0.0.0 - 10.255.255.255 | /8 | Große Netze |
| B | 172.16.0.0 - 172.31.255.255 | /12 | Mittelgroße |
| C | 192.168.0.0 - 192.168.255.255 | /16 | Heimnetze |

---

## Spezielle Bereiche

| Bereich | Zweck |
|---------|-------|
| 127.0.0.0/8 | Loopback (localhost) |
| 169.254.0.0/16 | Link-Local (APIPA) |
| 100.64.0.0/10 | CGNAT (Carrier-Grade NAT) - Tailscale nutzt dies! |

---

*Siehe auch: [DHCP](dhcp) | [Routing](routing)*
