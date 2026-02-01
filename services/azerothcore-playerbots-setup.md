---
title: AzerothCore Playerbots Setup
description: AzerothCore WotLK 3.3.5a Server mit Playerbots Modul - Installation, Konfiguration und Bot-Kommandos
published: true
date: 2026-02-01T18:00:00.000Z
tags: gameserver, azerothcore, wow, wotlk, playerbots, docker, gaming
editor: markdown
dateCreated: 2026-02-01T18:00:00.000Z
---

# AzerothCore WotLK - Playerbots Setup

## Overview
AzerothCore 3.3.5a WotLK server with mod-playerbots module running in Docker.
Fork: `mod-playerbots/azerothcore-wotlk` (Playerbot branch)

## Installation

### Clone
```bash
git clone https://github.com/mod-playerbots/azerothcore-wotlk.git --branch=Playerbot ~/playerbot/azerothcore-wotlk
cd ~/playerbot/azerothcore-wotlk/modules
git clone https://github.com/mod-playerbots/mod-playerbots.git --branch=master
```

### Build & Start
```bash
cd ~/playerbot/azerothcore-wotlk
docker compose up -d --build
```
Build compiles the full C++ server + playerbots module. Takes a while depending on CPU cores.

### Verify Module Loaded
```bash
docker compose logs ac-worldserver 2>&1 | grep -i playerbot
```
Should show playerbot loading messages, not just the branch name.

---

## Post-Build Configuration

### 1. Set Realmlist IP
```bash
docker compose exec ac-database mysql -uroot -ppassword -e "UPDATE acore_auth.realmlist SET address='YOUR.SERVER.IP' WHERE id=1;"
```

### 2. Create Game Account
```bash
docker attach ac-worldserver
```
```
account create YOURUSERNAME YOURPASSWORD
account set gmlevel YOURUSERNAME 3 -1
```
Detach: `Ctrl+P` then `Ctrl+Q`

### 3. Client Realmlist
Edit `realmlist.wtf` in WoW client folder:
```
set realmlist YOUR.SERVER.IP
```

### 4. Playerbots Config
Copy config out of container:
```bash
docker compose cp ac-worldserver:/azerothcore/env/dist/etc/playerbots.conf.dist ./playerbots.conf
```

Edit:
```bash
nano playerbots.conf
```
```ini
AiPlayerbot.Enabled = 1
AiPlayerbot.AllowPlayerBots = 1
```

Mount via `docker-compose.override.yml`:
```yaml
services:
  ac-worldserver:
    volumes:
      - ./playerbots.conf:/azerothcore/env/dist/etc/playerbots.conf
```

Restart:
```bash
docker compose down && docker compose up -d
```

---

## Playing with Bots (5-Man Group)

### Setup Alts
1. Log in, create 4 alt characters (1 tank, 1 healer, 2 DPS)
2. Log into each alt once briefly so they exist in the DB
3. Log back onto your main

### Add Bots (in-game)
```
.playerbots bot add AltName1,AltName2,AltName3,AltName4
```

### Invite to Party
```
/invite AltName1
/invite AltName2
/invite AltName3
/invite AltName4
```

### Summon & Setup (party chat)
```
summon
maintenance
autogear
```

### Set Roles (whisper each bot)
```
/w TankName co +tank
/w HealerName co +heal
/w DpsName1 co +dps
/w DpsName2 co +dps
```

### Set Talents (whisper each bot)
```
/w BotName talents spec list
/w BotName talents spec protection
```

### Combat
- Target a mob, party chat: `attack`
- Bots fight automatically
- Party chat: `follow` to regroup

### After Server Restart
Re-add bots:
```
.playerbots bot add AltName1,AltName2,AltName3,AltName4
```

---

## Command Reference

### Bot Management
| Command | Description |
|---|---|
| `.playerbots bot add Name1,Name2` | Login alt bots |
| `.playerbots bot addaccount AccountName` | Login entire account as bots |
| `.playerbots bot addclass warrior` | Spawn random bot of class |
| `.playerbots bot remove Name1,Name2` | Logout specific bots |
| `.playerbots bot remove *` | Logout all party bots |
| `.playerbots bot add *` | Login all party/raid alts |
| `.playerbots bot list` | List your alt bots |
| `.playerbots bot init=rare Name` | Respawn bot with rare gear |

### Party Commands (party chat - all bots hear)
| Command | Description |
|---|---|
| `summon` | Teleport bots to you |
| `follow` | Bots follow you |
| `stay` | Bots stay in place |
| `attack` | Attack your target |
| `grind` | Attack anything nearby |
| `flee` | Run to player ignoring threats |
| `release` | Release spirit when dead |
| `revive` | Revive at spirit healer |
| `leave` | Leave party |
| `maintenance` | Learn spells, repair, buy consumables |
| `autogear` | Auto-equip best gear |
| `reset botAI` | Reset bot settings to default |

### Combat Strategies (whisper bot)
| Command | Description |
|---|---|
| `co +tank` | Tank mode |
| `co +heal` | Healer mode |
| `co +dps` | DPS mode |
| `co +aoe` | Use AoE abilities |
| `co +threat` | Threat management |
| `co -strategy` | Remove a strategy |
| `co ?` | Show current strategies |

### Non-Combat Strategies (whisper bot)
| Command | Description |
|---|---|
| `nc +loot` | Enable looting |
| `nc +food` | Use food/drink |
| `nc +pvp` | PvP mode |
| `nc ?` | Show current non-combat strategies |

### Talents
| Command | Description |
|---|---|
| `talents` | Check current spec |
| `talents spec list` | Show available specs |
| `talents spec [name]` | Change talent spec |

### Loot
| Command | Description |
|---|---|
| `ll all` | Loot everything |
| `ll normal` | Loot non-BOP items |
| `ll gray` | Loot gray items only |
| `ll quest` | Loot quest items only |
| `s *` | Sell gray items |
| `s vendor` | Sell all sellable items |

### Items
| Command | Description |
|---|---|
| `e [item]` | Equip item |
| `ue [item]` | Unequip item |
| `u [item]` | Use item |
| `b [item]` | Buy item |
| `destroy [item]` | Destroy item |
| `2g 3s 5c` | Give gold to bot |

### Quests
| Command | Description |
|---|---|
| `quests` | Show quest summary |
| `accept [quest]` | Accept quest |
| `accept *` | Accept all available quests |
| `drop [quest]` | Abandon quest |
| `talk` | Talk to NPC |

### Group Targeting
| Command | Description |
|---|---|
| `@tank [command]` | Command tanks only |
| `@heal [command]` | Command healers only |
| `@dps [command]` | Command DPS only |
| `@group1 [command]` | Command group 1 only |

### Raid Strategies (auto-applied in raids)
| Command | Description |
|---|---|
| `naxx` | Naxxramas |
| `icc` | Icecrown Citadel |
| `uld` | Ulduar |
| `moltencore` | Molten Core |
| `bwl` | Blackwing Lair |

### Account Linking (for alts on other accounts)
```
.playerbots account setKey MySecretKey
.playerbots account link OtherAccount MySecretKey
.playerbots account linkedAccounts
.playerbots account unlink OtherAccount
```

### Server Console Commands
| Command | Description |
|---|---|
| `playerbot rndbot stats` | Random bot statistics |
| `playerbot rndbot reload` | Reload config without restart |
| `playerbot rndbot init` | Re-roll all random bots |

### In-Game Help
```
/w BotName help
```

---

## Playerbots Config Reference (playerbots.conf)

```ini
AiPlayerbot.Enabled = 1                      # Enable module
AiPlayerbot.AllowPlayerBots = 1              # Allow player bot commands
AiPlayerbot.RandomBotMinLevel = 1            # Random bot min level
AiPlayerbot.RandomBotMaxLevel = 80           # Random bot max level
AiPlayerbot.RandombotStartingLevel = 5       # Random bot starting level
AiPlayerbot.RandomBotMaps = 0,1,530,571      # Maps: EK, Kalimdor, Outland, Northrend
AiPlayerbot.DeleteRandomBotAccounts = 0      # Keep bot accounts between restarts
AiPlayerbot.AutoDoQuests = 1                 # Bots auto-quest
AiPlayerbot.BotActiveAlone = 100             # Bot activity level
```

---

## Adding More Modules

### Compatible Modules
- **mod-ah-bot** - Auction House bot (fills AH with items)
- **mod-transmog** - Transmogrification
- **mod-solocraft** - Scale dungeons for solo play
- **mod-1v1-arena** - 1v1 arena PvP

Note: Do NOT use Auctionator with playerbots (crashes). Use mod-ah-bot instead.

### Install a Module
```bash
cd ~/playerbot/azerothcore-wotlk/modules
git clone https://github.com/azerothcore/mod-ah-bot.git
cd ~/playerbot/azerothcore-wotlk
docker compose up -d --build
```
Requires rebuild every time a new module is added.

---

## Docker Management

```bash
docker compose up -d          # Start server
docker compose down           # Stop server
docker compose logs -f ac-worldserver   # Watch logs
docker attach ac-worldserver  # Attach to console (Ctrl+P Ctrl+Q to detach)
docker compose up -d --build  # Rebuild after changes
```

## Ports
| Port | Service |
|---|---|
| 3724 | Authserver |
| 8085 | Worldserver |
| 7878 | SOAP |
| 3306 | MySQL |
