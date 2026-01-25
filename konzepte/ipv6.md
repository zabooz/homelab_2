# IPv6 Grundlagen

Grundlagen des Internet Protocol Version 6.

---

## Adresstypen

| Typ | Präfix | Beschreibung |
|-----|--------|--------------|
| Link-Local | fe80::/10 | Nur im lokalen Netz |
| Global | 2000::/3 | Internet-routbar |
| Loopback | ::1 | Localhost |

---

## SLAAC (Stateless Address Autoconfiguration)

1. Router sendet Router Advertisement (RA)
2. Client generiert eigene IPv6-Adresse
3. Kein DHCP nötig

```bash
# In /etc/network/interfaces
iface ens18 inet6 auto  # SLAAC aktiviert
```

---

## IPv6 vs. IPv4

| Aspekt | IPv4 | IPv6 |
|--------|------|------|
| **Länge** | 32 Bit | 128 Bit |
| **Format** | 192.168.0.1 | 2001:db8::1 |
| **NAT** | Oft nötig | Nicht nötig |
| **Autokonfiguration** | DHCP | SLAAC |

---

*Siehe auch: [IP-Subnetting](ip-subnetting) | [DHCP](dhcp)*
