---
title: Wiki.js Setup
description: Wiki.js installation and Git sync configuration
published: true
---

# Wiki.js Setup Guide

This document covers the installation of Wiki.js with Docker and configuring Git synchronization.

## Overview

Wiki.js is a modern, open-source wiki platform running on Node.js. This setup uses Docker Compose with PostgreSQL as the database and Git for content synchronization.

## Architecture

- **Wiki.js**: Web application on port 3000
- **PostgreSQL 18**: Database backend
- **Git Sync**: Bidirectional sync with GitHub repository

## Installation

### Prerequisites

- Docker and Docker Compose installed
- SSH key pair for Git authentication
- GitHub repository for content storage

### 1. Create Project Directory

```bash
mkdir -p ~/wikiJS/data/wiki
cd ~/wikiJS
```

### 2. Generate SSH Key for Git Sync

```bash
ssh-keygen -t ed25519 -f ~/.ssh/wiki_ed25519 -N '' -C 'wikijs@server'
```

Add the public key to your GitHub repository:
1. Go to: `https://github.com/YOUR_USER/YOUR_REPO/settings/keys`
2. Click "Add deploy key"
3. Paste contents of `~/.ssh/wiki_ed25519.pub`
4. Enable "Allow write access"
5. Save

### 3. Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
services:
  wikijs:
    image: requarks/wiki:2
    container_name: wikijs
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: wikijsrocks
      DB_NAME: wikijs
    depends_on:
      - db
    volumes:
      - ./data/wiki:/wiki/data
      - ~/.ssh/wiki_ed25519:/etc/wiki/.ssh/id_rsa:ro
      - ~/.ssh/wiki_ed25519.pub:/etc/wiki/.ssh/id_rsa.pub:ro

  db:
    image: postgres:18
    container_name: wikijs_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: wikijs
      POSTGRES_PASSWORD: wikijsrocks
      POSTGRES_DB: wikijs
    volumes:
      - ./data/postgres:/var/lib/postgresql
```

### 4. Fix SSH Key Permissions

The SSH key must be readable inside the container:

```bash
chmod 644 ~/.ssh/wiki_ed25519
```

### 5. Add GitHub to Known Hosts

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

### 6. Start Services

```bash
docker compose up -d
```

## Git Sync Configuration

### Initial Setup

1. Access Wiki.js at `http://YOUR_SERVER:3000`
2. Complete the initial setup wizard
3. Go to **Administration** → **Storage**
4. Enable **Git** storage

### Git Storage Settings

| Setting | Value |
|---------|-------|
| Authentication Type | SSH |
| Repository URL | `git@github.com:YOUR_USER/YOUR_REPO.git` |
| Branch | `main` |
| SSH Private Key Path | `/etc/wiki/.ssh/id_rsa` |
| Sync Direction | Bidirectional |
| Sync Schedule | (your preference) |

### Import Existing Content

After configuring Git storage:
1. Go to **Administration** → **Storage** → **Git**
2. Click **"Import Everything"**
3. Wait for all pages to be imported and rendered

## Troubleshooting

### Problem: Git sync not working

**Symptoms:**
- "Permission denied (publickey)" errors
- Sync fails silently

**Common Causes & Solutions:**

#### 1. SSH Key Path Empty

Check the git config inside the container:
```bash
docker exec wikijs git -C /wiki/data/repo config --get core.sshCommand
```

If it shows `ssh -i ""`, the SSH Private Key Path is not set in Wiki.js admin.

**Fix:** Go to Administration → Storage → Git and set SSH Private Key Path to `/etc/wiki/.ssh/id_rsa`

#### 2. SSH Key Not Mounted

```bash
docker exec wikijs ls -la /etc/wiki/.ssh/
```

If directory is empty or missing, check your docker-compose volume mounts.

#### 3. Key File Permissions

If you see "Load key: Permission denied":
```bash
chmod 644 ~/.ssh/wiki_ed25519
```

#### 4. Key Not Added to GitHub

Test SSH connection:
```bash
docker exec wikijs ssh -i /etc/wiki/.ssh/id_rsa -T git@github.com
```

Should show: "Hi USER/REPO\! You've successfully authenticated"

#### 5. Wrong Branch Name

GitHub defaults to `main`, but Wiki.js might default to `master`. 

**Fix:** Set correct branch name in Git storage settings.

#### 6. Docker Created Directories Instead of Mounting Files

If your SSH key paths become directories:
```bash
# Remove the directories
rm -rf ~/.ssh/wiki_ed25519 ~/.ssh/wiki_ed25519.pub

# Create actual key files
ssh-keygen -t ed25519 -f ~/.ssh/wiki_ed25519 -N ''

# Restart containers
docker compose down && docker compose up -d
```

### Problem: Render Error on Pages

**Cause:** Page content exists but render column is NULL in database.

**Fix:** Delete the page from database and reimport:
```bash
docker exec wikijs_db psql -U wikijs -d wikijs -c "DELETE FROM pages WHERE path = 'PAGE_PATH';"
```
Then use "Import Everything" in admin panel.

## Navigation Setup

To show all pages in a sidebar tree:

1. Go to **Administration** → **Navigation**
2. Add new item with Kind: **Page Tree**
3. Save

Or via database:
```bash
docker exec wikijs_db psql -U wikijs -d wikijs -c "UPDATE navigation SET config = '[{\"locale\":\"en\",\"items\":[{\"id\":\"home\",\"icon\":\"mdi-home\",\"kind\":\"link\",\"label\":\"Home\",\"target\":\"/\",\"targetType\":\"home\",\"visibilityMode\":\"all\"},{\"id\":\"tree\",\"icon\":\"mdi-file-tree\",\"kind\":\"tree\",\"label\":\"All Pages\",\"visibilityMode\":\"all\"}]}]' WHERE key = 'site';"
```

## Useful Commands

```bash
# View logs
docker logs wikijs --tail 100

# Check Git status
docker exec wikijs git -C /wiki/data/repo status

# List pages in database
docker exec wikijs_db psql -U wikijs -d wikijs -c "SELECT path, title FROM pages;"

# Test SSH to GitHub
docker exec wikijs ssh -i /etc/wiki/.ssh/id_rsa -T git@github.com

# Manual Git push
docker exec wikijs sh -c "cd /wiki/data/repo && GIT_SSH_COMMAND=\"ssh -i /etc/wiki/.ssh/id_rsa\" git push origin main"
```

## References

- [Wiki.js Documentation](https://docs.requarks.io/)
- [Wiki.js GitHub](https://github.com/requarks/wiki)
