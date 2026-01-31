# Linkwarden - Bookmark Manager

**System:** Debian 13 LXC (CT 105)
**IP Address:** 192.168.0.119
**Port:** 3000
**URL:** http://192.168.0.119:3000
**Status:** Online

---

## Was ist Linkwarden?

Selbst-gehosteter Bookmark-Manager zum Speichern, Organisieren und Archivieren von Webseiten. Unterstützt Tags, Collections, Volltext-Suche und automatische Screenshots/Archivierung.

---

## Installation

### Voraussetzungen

- LXC Container mit Debian 13
- Docker + Docker Compose
- Nesting Feature in Proxmox aktiviert (`features: nesting=1` in LXC Config)

### Docker Compose

**Verzeichnis:** `/opt/linkwarden/`

**docker-compose.yml:**
```yaml
services:
  linkwarden:
    image: ghcr.io/linkwarden/linkwarden:latest
    restart: always
    ports:
      - 3000:3000
    env_file:
      - .env
    volumes:
      - linkwarden_data:/data/data
    depends_on:
      - postgres

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: linkwarden_db_pass
      POSTGRES_DB: linkwarden
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  linkwarden_data:
  postgres_data:
```

**.env:**
```env
NEXTAUTH_SECRET=<generierter-secret>
NEXTAUTH_URL=http://192.168.0.119:3000/api/v1/auth
DATABASE_URL=postgresql://postgres:linkwarden_db_pass@postgres:5432/linkwarden
NEXT_PUBLIC_DISABLE_REGISTRATION=false
```

### Starten

```bash
cd /opt/linkwarden
docker compose up -d
```

---

## Erster Login

Beim ersten Aufruf von `http://192.168.0.119:3000` klicke auf **"Register"** und erstelle deinen Admin-Account (Name, E-Mail, Passwort).

Nach der Registrierung kannst du die Registrierung deaktivieren:

```bash
# In /opt/linkwarden/.env ändern:
NEXT_PUBLIC_DISABLE_REGISTRATION=true

# Dann Container neu starten:
cd /opt/linkwarden && docker compose restart
```

---

## Management

```bash
# Status prüfen
docker compose -f /opt/linkwarden/docker-compose.yml ps

# Logs anschauen
docker compose -f /opt/linkwarden/docker-compose.yml logs -f

# Update
cd /opt/linkwarden && docker compose pull && docker compose up -d

# Neustart
cd /opt/linkwarden && docker compose restart
```

---

## Backup

Die Daten liegen in Docker Volumes:
- `linkwarden_linkwarden_data` - Archivierte Webseiten, Screenshots
- `linkwarden_postgres_data` - PostgreSQL Datenbank

```bash
# Datenbank Backup
docker exec linkwarden-postgres-1 pg_dump -U postgres linkwarden > /root/linkwarden_backup.sql

# Datenbank Restore
cat /root/linkwarden_backup.sql | docker exec -i linkwarden-postgres-1 psql -U postgres linkwarden
```

---

*Letzte Aktualisierung: 31. Januar 2026*
