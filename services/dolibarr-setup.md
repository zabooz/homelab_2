---
title: Dolibarr ERP
description: Open-Source ERP/CRM fuer IT-Gewerbe - Rechnungen, Inventar, Kundenverwaltung
published: true
date: 2026-03-07T00:00:00.000Z
tags: service, erp, crm, dolibarr, rechnungen, inventar, docker, gewerbe
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# Dolibarr ERP

Dolibarr ist ein Open-Source ERP/CRM-System fuer das IT-Gewerbe. Verwaltet Rechnungen, Angebote, Kunden, Inventar und Buchhaltung.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 118 |
| **Node** | homeserver3 |
| **IP** | 192.168.0.149 |
| **OS** | Debian |
| **RAM** | 4196 MB |
| **CPU** | 2 Cores |
| **Disk** | 25 GB |
| **Web UI** | http://192.168.0.149 |

---

## Docker Compose

**Datei:** `/opt/dolibarr/docker-compose.yml`

```yaml
services:
  dolibarr:
    image: dolibarr/dolibarr:latest
    restart: always
    ports:
      - "80:80"
    environment:
      - DOLI_DB_HOST=db
      - DOLI_DB_USER=dolibarr
      - DOLI_DB_PASSWORD=dolibarr123
      - DOLI_DB_NAME=dolibarr
      - DOLI_ADMIN_LOGIN=admin
      - DOLI_ADMIN_PASSWORD=admin
    volumes:
      - dolibarr_docs:/var/www/documents
      - dolibarr_custom:/var/www/html/custom

  db:
    image: mariadb:latest
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=root123
      - MYSQL_DATABASE=dolibarr
      - MYSQL_USER=dolibarr
      - MYSQL_PASSWORD=dolibarr123
    volumes:
      - dolibarr_db:/var/lib/mysql

volumes:
  dolibarr_docs:
  dolibarr_custom:
  dolibarr_db:
```

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web UI** | http://192.168.0.149 |
| **Login** | admin / admin (beim ersten Start) |

Sprache auf Deutsch: Home → Setup → Translation/Languages → Deutsch

---

## Verwendung

### Module aktivieren

Home → Setup → Modules:
- **Invoices** — Rechnungen
- **Products/Services** — Dienstleistungen/Produkte
- **Third Parties** — Kundenverwaltung
- **Stock/Warehouse** — Inventar (optional)

### Rechnung erstellen

1. **Service anlegen:** Billing → Products/Services → New
   - z.B. "IT Support" mit Stundensatz
2. **Kunde anlegen:** Third Parties → New Customer
3. **Rechnung:** Billing → Customer Invoices → New Invoice
   - Kunde auswaehlen → Service hinzufuegen → Validieren
4. **PDF generieren:** PDF-Icon klicken
5. **Drucken:** PDF oeffnen → Strg+P → DruckenStein (HP LaserJet, 192.168.0.201)

### Kleinunternehmerregelung (Oesterreich)

Bei Jahresumsatz unter 55.000 EUR keine USt noetig:
- Services/Produkte mit **VAT 0%** anlegen
- Hinweis auf Rechnung: "Umsatzsteuerfrei gemaess § 6 Abs. 1 Z 27 UStG"

---

## Rechtliche Grundlagen (Oesterreich 2026)

| Regelung | Grenze |
|----------|--------|
| **Kleinunternehmerregelung** | Unter 55.000 EUR/Jahr → keine USt |
| **Registrierkassenpflicht** | Nur wenn Barumsaetze ueber 7.500 EUR/Jahr |
| **Rechnungsaufbewahrung** | 7 Jahre |
| **Fortlaufende Rechnungsnummern** | Pflicht |

---

## Wartung

```bash
# Status
cd /opt/dolibarr
docker compose ps

# Logs
docker logs dolibarr --tail 50

# Update
docker compose pull
docker compose down && docker compose up -d
```

### Backup

```bash
# Datenbank
docker exec dolibarr-db-1 mysqldump -u dolibarr -pdolibarr123 dolibarr > /backup/dolibarr-db-$(date +%Y%m%d).sql

# Dokumente (Rechnungen PDFs)
docker run --rm -v dolibarr_dolibarr_docs:/data -v /backup:/backup alpine tar czf /backup/dolibarr-docs-$(date +%Y%m%d).tar.gz /data
```

---

*Siehe auch: [Netzwerk-Uebersicht](/en/infrastructure/NETWORK_OVERVIEW)*