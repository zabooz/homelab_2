# LAP Prüfungsfragen

Häufige Fragen und Antworten für die IT-Systemtechnik Prüfung.

---

## "Was ist der Unterschied zwischen DHCP und statischer IP?"

DHCP vergibt IP-Adressen automatisch, kann sich aber ändern. Statische IPs werden manuell konfiguriert und bleiben gleich. Server brauchen statische IPs für Erreichbarkeit, Clients können DHCP nutzen.

**Siehe:** [DHCP](dhcp) | [Subnetting](subnetting)

---

## "Was ist ein Gateway?"

Ein Gateway ist der Zugang zu anderen Netzwerken. In Heimnetzen ist das meist der Router. Alle Pakete zu unbekannten Zielen werden an das Gateway geschickt.

**Siehe:** [Routing](routing)

---

## "Warum braucht ein Router kein Gateway?"

Ein Router verbindet selbst Netzwerke und trifft eigene Routing-Entscheidungen. Ein Gateway würde bedeuten "frag jemand anderen" - aber der Router IST die Entscheidungsinstanz.

**Siehe:** [Routing](routing) | [IP-Forwarding](ip-forwarding)

---

## "Was ist NAT/MASQUERADE?"

NAT ändert IP-Adressen in Paketen. MASQUERADE ist dynamisches Source-NAT, das die Quell-IP durch die eigene ersetzt. Nötig, damit Antworten zurückfinden.

**Siehe:** [NAT](nat)

---

## "Was ist IP Forwarding?"

Die Fähigkeit eines Linux-Systems, Pakete zwischen Interfaces weiterzuleiten. Aus = normaler Host, An = Router. Ohne IP Forwarding verwirft das System fremde Pakete.

**Siehe:** [IP-Forwarding](ip-forwarding)

---

## "Was bedeutet 0.0.0.0/0?"

Eine Default-Route, die ALLE IP-Adressen matched. Das /0 bedeutet "keine Netzwerk-Bits festgelegt". Wird für Internet-Traffic und Exit-Nodes verwendet.

**Siehe:** [Routing](routing) | [Subnetting](subnetting)

---

## "Was ist der Unterschied zwischen Forward und Reverse Proxy?"

**Forward Proxy:** Client → Proxy → Internet (Client kennt Proxy)
**Reverse Proxy:** Client → Proxy → Backend (Client kennt nur Proxy)

**Siehe:** [Reverse Proxy](reverse-proxy)

---

## "Was ist CIDR?"

Classless Inter-Domain Routing. Notation wie /24 gibt an, wie viele Bits für das Netzwerk reserviert sind. /24 = 256 Adressen, /16 = 65.536 Adressen.

**Siehe:** [Subnetting](subnetting)

---

*Viel Erfolg bei der LAP-Prüfung!*
