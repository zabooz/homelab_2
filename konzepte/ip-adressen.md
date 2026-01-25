# IP-Adressen

Grundlagen zu IPv4 und IPv6 Adressen.

---

## IPv4

### Format

- 32 Bit, 4 Oktette
- Dezimal mit Punkten: `192.168.0.1`
- Jedes Oktett: 0-255

### Private IP-Bereiche (RFC 1918)

| Klasse | Bereich | CIDR | Nutzung |
|--------|---------|------|---------|
| A | 10.0.0.0 - 10.255.255.255 | /8 | Große Netze |
| B | 172.16.0.0 - 172.31.255.255 | /12 | Mittelgroße |
| C | 192.168.0.0 - 192.168.255.255 | /16 | Heimnetze |

### Spezielle Bereiche

| Bereich | Zweck |
|---------|-------|
| 127.0.0.0/8 | Loopback (localhost) |
| 169.254.0.0/16 | Link-Local (APIPA) - Kein DHCP verfügbar |
| 100.64.0.0/10 | CGNAT (Carrier-Grade NAT) - Tailscale nutzt dies! |
| 0.0.0.0 | "Alle Adressen" / Unspezifiziert |
| 255.255.255.255 | Broadcast |

### Public vs. Private

| Typ | Beschreibung | Beispiel |
|-----|--------------|----------|
| **Private** | Nur intern erreichbar, nicht im Internet | 192.168.0.1 |
| **Public** | Im Internet erreichbar, weltweit eindeutig | 152.53.111.11 |

> **Wichtig:** Private IPs brauchen NAT um ins Internet zu kommen!

---

## IPv6

### Format

- 128 Bit, 8 Gruppen à 16 Bit
- Hexadezimal mit Doppelpunkten: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Kurzform: `2001:db8:85a3::8a2e:370:7334` (führende Nullen + `::` für Null-Gruppen)

### Adresstypen

| Typ | Präfix | Beschreibung |
|-----|--------|--------------|
| Link-Local | fe80::/10 | Nur im lokalen Netz, automatisch generiert |
| Global Unicast | 2000::/3 | Internet-routbar (wie Public IPv4) |
| Unique Local | fc00::/7 | Privat (wie RFC 1918 bei IPv4) |
| Loopback | ::1 | Localhost |

### SLAAC (Stateless Address Autoconfiguration)

IPv6 braucht kein DHCP:

1. Router sendet Router Advertisement (RA)
2. Client generiert eigene IPv6-Adresse aus Präfix + MAC
3. Automatisch, kein Server nötig

```bash
# In /etc/network/interfaces
iface ens18 inet6 auto  # SLAAC aktiviert
```

---

## IPv4 vs. IPv6 Vergleich

| Aspekt | IPv4 | IPv6 |
|--------|------|------|
| **Länge** | 32 Bit | 128 Bit |
| **Adressen** | ~4,3 Milliarden | ~340 Sextillionen |
| **Format** | 192.168.0.1 | 2001:db8::1 |
| **NAT** | Oft nötig | Nicht nötig |
| **Autokonfiguration** | DHCP | SLAAC |
| **Header** | Variabel, komplex | Fix, einfacher |

---

*Siehe auch: [Subnetting](subnetting) | [DHCP](dhcp) | [NAT](nat)*
