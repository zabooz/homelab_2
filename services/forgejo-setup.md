---
title: Forgejo
description: Selbst-gehosteter Git-Server mit CI/CD Runner für das Homelab
published: true
date: 2026-03-29T00:00:00.000Z
tags: service, git, forgejo, docker, ci-cd, runner, versionierung, version-control
editor: markdown
dateCreated: 2026-03-29T00:00:00.000Z
---

# Forgejo

Forgejo ist ein selbst-gehosteter Git-Server (Gitea-Fork) mit integriertem CI/CD Runner. Läuft mit PostgreSQL-Datenbank und Docker-in-Docker für Pipeline-Execution.

---

## System

| Parameter | Wert |
|-----------|------|
| **Host** | LXC Container (Proxmox) |
| **CT ID** | 127 |
| **Node** | homeserver3 |
| **IP** | 192.168.0.107 |
| **OS** | Debian |
| **RAM** | 8 GB |
| **CPU** | 4 Cores |
| **Disk** | 15 GB |

---

## Zugriff

| Dienst | URL |
|--------|-----|
| **Web-UI** | http://192.168.0.107:3000 |
| **SSH (Git)** | ssh://192.168.0.107:2222 |

---

## Installation

### LXC Container erstellen (Proxmox)

```bash
pct create 127 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname forgejo \
  --memory 8392 \
  --cores 4 \
  --rootfs local-lvm:15 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=192.168.0.107/24,gw=192.168.0.1 \
  --features nesting=1

pct start 127
pct enter 127
```

### Docker installieren

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | bash
```

### Docker Compose Setup

```bash
mkdir -p /opt/forgejo && cd /opt/forgejo
```

**Vorbereitung:**

```bash
# Shared Secret generieren
openssl rand -hex 20
# Output als SHARED_SECRET verwenden

# Sichere Passwörter wählen für DB und Root-Account
```

**Datei:** `/opt/forgejo/docker-compose.yml`

```yaml
volumes:
  docker_certs:

services:

  db:
    image: postgres:17-alpine
    restart: always
    environment:
      POSTGRES_USER: forgejo
      POSTGRES_PASSWORD: <DB_PASSWORD>
      POSTGRES_DB: forgejo
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  docker-in-docker:
    image: data.forgejo.org/oci/docker:dind
    hostname: docker
    privileged: true
    restart: always
    environment:
      DOCKER_TLS_CERTDIR: /certs
    volumes:
      - docker_certs:/certs

  server:
    image: data.forgejo.org/forgejo/forgejo:14.0.3
    restart: always
    depends_on:
      - db
    command: >-
      bash -c '
      /usr/bin/s6-svscan /etc/s6 &
      sleep 10 ;
      su -c "forgejo forgejo-cli actions register --keep-labels --secret <SHARED_SECRET>" git ;
      su -c "forgejo admin user create --admin --username root --password <ROOT_PASSWORD> --email admin@forgejo.home" git ;
      sleep infinity
      '
    environment:
      FORGEJO__database__DB_TYPE: postgres
      FORGEJO__database__HOST: db:5432
      FORGEJO__database__NAME: forgejo
      FORGEJO__database__USER: forgejo
      FORGEJO__database__PASSWD: <DB_PASSWORD>
      FORGEJO__security__INSTALL_LOCK: "true"
      FORGEJO__server__ROOT_URL: http://forgejo.home:3000
      FORGEJO__actions__ENABLED: "true"
      FORGEJO__repository__ENABLE_PUSH_CREATE_USER: "true"
      FORGEJO__repository__DEFAULT_PUSH_CREATE_PRIVATE: "false"
      FORGEJO__repository__DEFAULT_REPO_UNITS: "repo.code,repo.actions"
    volumes:
      - ./data/forgejo:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"

  runner-register:
    image: data.forgejo.org/forgejo/runner:12.7.2
    restart: "no"
    depends_on:
      - server
      - docker-in-docker
    environment:
      DOCKER_HOST: tcp://docker-in-docker:2376
    volumes:
      - ./data/runner:/data
      - docker_certs:/certs:ro
    user: "0:0"
    command: >-
      bash -ec '
      while : ; do
        forgejo-runner create-runner-file --connect --instance http://server:3000 --name homelab-runner --secret <SHARED_SECRET> && break ;
        sleep 1 ;
      done ;
      sed -i -e "s|\"labels\": \[\]|\"labels\": [\"docker:docker://data.forgejo.org/oci/node:24-bookworm\",\"ubuntu-latest:docker://catthehacker/ubuntu:act-24.04\",\"debian-latest:docker://debian:trixie-slim\"]|" .runner ;
      forgejo-runner generate-config > config.yml ;
      sed -i -e "s|network: .*|network: host|" config.yml ;
      sed -i -e "s|^  envs:$$|  envs:\n    DOCKER_HOST: tcp://docker:2376\n    DOCKER_TLS_VERIFY: 1\n    DOCKER_CERT_PATH: /certs/client|" config.yml ;
      sed -i -e "s|^  options:|  options: -v /certs/client:/certs/client|" config.yml ;
      sed -i -e "s|  valid_volumes: \[\]$$|  valid_volumes:\n    - /certs/client|" config.yml ;
      chown -R 1000:1000 /data
      '

  runner-daemon:
    image: data.forgejo.org/forgejo/runner:12.7.2
    restart: always
    depends_on:
      runner-register:
        condition: service_completed_successfully
      docker-in-docker:
        condition: service_started
    environment:
      DOCKER_HOST: tcp://docker:2376
      DOCKER_CERT_PATH: /certs/client
      DOCKER_TLS_VERIFY: "1"
    volumes:
      - ./data/runner:/data
      - docker_certs:/certs
    command: >-
      bash -c '
      while : ; do test -w .runner && forgejo-runner --config config.yml daemon ; sleep 1 ; done
      '
```

### Container starten

```bash
cd /opt/forgejo
docker compose up -d
```

---

## Architektur

```
┌─────────────────────────────────────────────┐
│              Forgejo Stack                   │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Server   │  │ Runner   │  │  DinD    │  │
│  │ :3000    │──│ Daemon   │──│  Docker  │  │
│  │ :2222    │  │          │  │  in      │  │
│  └────┬─────┘  └──────────┘  │  Docker  │  │
│       │                      └──────────┘  │
│  ┌────▼─────┐                              │
│  │PostgreSQL│                              │
│  │  :5432   │                              │
│  └──────────┘                              │
└─────────────────────────────────────────────┘
```

**Komponenten:**
- **Server** - Forgejo Web-UI + Git Server
- **PostgreSQL** - Datenbank
- **Runner Daemon** - Führt CI/CD Pipelines aus
- **Docker-in-Docker** - Container-Runtime für Pipelines

**Runner Labels:**
- `docker` → `node:24-bookworm`
- `ubuntu-latest` → `catthehacker/ubuntu:act-24.04`
- `debian-latest` → `debian:trixie-slim`

---

## Git Remote konfigurieren

```bash
# HTTPS
git remote add forgejo http://192.168.0.107:3000/<user>/<repo>.git

# SSH
git remote add forgejo ssh://git@192.168.0.107:2222/<user>/<repo>.git
```

**Push-to-Create:** Repos werden automatisch erstellt beim ersten Push (ENABLE_PUSH_CREATE_USER aktiviert).

---

## Wartung

### Container Management

```bash
# Status prüfen
docker ps | grep forgejo

# Logs anschauen
docker logs forgejo-server-1 --tail 50
docker logs forgejo-runner-daemon-1 --tail 50

# Container neu starten
cd /opt/forgejo
docker compose restart

# Update auf neue Version
docker compose pull
docker compose up -d
```

### Backup

**Wichtige Daten:**

```
/opt/forgejo/data/forgejo/    # Git Repos, Konfiguration
/opt/forgejo/data/postgres/   # PostgreSQL Datenbank
/opt/forgejo/data/runner/     # Runner Konfiguration
```

---

## Troubleshooting

### Runner registriert sich nicht

```bash
# Runner-Logs prüfen
docker logs forgejo-runner-register-1

# Shared Secret stimmt überein?
# Muss in server command UND runner-register command identisch sein
```

### Web-UI nicht erreichbar

```bash
# Container läuft?
docker ps | grep forgejo-server

# Port 3000 belegt?
ss -tuln | grep 3000
```

---

*Siehe auch: [Netzwerk-Übersicht](/en/infrastructure/NETWORK_OVERVIEW) · [Homepage Dashboard](/en/services/homepage-dashboard-setup)*
