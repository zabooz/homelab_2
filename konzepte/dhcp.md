# DHCP (Dynamic Host Configuration Protocol)

Automatische IP-Adressvergabe im Netzwerk.

---

## DORA-Prozess

```
1. DISCOVER  →  Client sendet Broadcast: "Ich brauche eine IP!"
2. OFFER     ←  Server bietet IP an: "Du kannst 192.168.0.100 haben"
3. REQUEST   →  Client: "Ja, ich möchte diese IP!"
4. ACK       ←  Server bestätigt: "OK, sie gehört dir für X Stunden"
```

---

## DHCP vs. Statische IP

| Aspekt | DHCP | Statisch |
|--------|------|----------|
| **Konfiguration** | Automatisch | Manuell |
| **Änderung** | IP kann sich ändern | Bleibt gleich |
| **Verwaltung** | Zentral (DHCP-Server) | Dezentral |
| **Einsatz** | Clients, Laptops | Server, Router, VPN |

---

## Wann statische IPs?

**Immer bei:**
- Servern (Web, DNS, DHCP)
- Netzwerk-Infrastruktur (Router, Switches)
- VPN-Endpunkten
- Druckern, IoT mit Services

**DHCP bei:**
- Laptops, Desktops
- Smartphones, Tablets
- Gast-Geräte

---

*Siehe auch: [IP-Adressen](ip-adressen) | [Routing](routing)*
