---
title: Stats API
description: Bun API für System-Monitoring in Homepage
published: true
---

# Stats API für Homepage

Diese Dokumentation beschreibt die Stats APIs, die System-Metriken für das Homepage Dashboard bereitstellen.

## Übersicht

Wir verwenden kleine Bun-APIs um Systemdaten (CPU, RAM, Disk) an Homepage zu senden. Diese laufen auf:

| Server | IP | Port | Beschreibung |
|--------|-----|------|--------------|
| Proxmox | 192.168.0.101 | 4000 | Hypervisor Stats |
| VPS | 100.64.0.5 | 4000 | VPS Stats |

## Proxmox Stats API

### Installation

#### 1. Bun installieren

```bash
apt-get install -y unzip
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
```

#### 2. API erstellen

```bash
mkdir -p /opt/stats-api
```

Datei `/opt/stats-api/index.ts`:

```typescript
import { exec } from "child_process";
import { promisify } from "util";

const execAsync = promisify(exec);

const server = Bun.serve({
  port: 4000,
  async fetch(req) {
    const url = new URL(req.url);
    
    if (url.pathname === "/stats") {
      // Root Disk
      const { stdout: dfOut } = await execAsync("df -B1 /");
      const lines = dfOut.trim().split("\n");
      const parts = lines[1].split(/\s+/);
      const rootTotal = parseInt(parts[1]);
      const rootUsed = parseInt(parts[2]);
      const rootPercent = parseInt(parts[4]);
      
      // LVM Thin Pool (VM Storage)
      const { stdout: lvsOut } = await execAsync("/usr/sbin/lvs --noheadings --units g -o lv_size,data_percent pve/data");
      const lvsParts = lvsOut.trim().split(/\s+/);
      const dataTotal = parseFloat(lvsParts[0]);
      const dataPercent = parseFloat(lvsParts[1]);
      const dataUsed = Math.round(dataTotal * dataPercent / 100);
      
      // Load Average
      const loadavg = await Bun.file("/proc/loadavg").text();
      const loads = loadavg.split(" ");
      
      // Memory
      const meminfo = await Bun.file("/proc/meminfo").text();
      const memLines = meminfo.split("\n");
      const memTotal = parseInt(memLines.find(l => l.startsWith("MemTotal"))?.split(/\s+/)[1] || "0") * 1024;
      const memAvail = parseInt(memLines.find(l => l.startsWith("MemAvailable"))?.split(/\s+/)[1] || "0") * 1024;
      const memUsed = memTotal - memAvail;
      const memPercent = Math.round((memUsed / memTotal) * 100 * 10) / 10;

      const stats = {
        load1: parseFloat(loads[0]),
        load5: parseFloat(loads[1]),
        load15: parseFloat(loads[2]),
        ram: memPercent,
        rootDisplay: `${Math.round(rootUsed / 1024 / 1024 / 1024)}GB / ${Math.round(rootTotal / 1024 / 1024 / 1024)}GB (${rootPercent}%)`,
        dataDisplay: `${dataUsed}GB / ${Math.round(dataTotal)}GB (${Math.round(dataPercent)}%)`
      };

      return Response.json(stats);
    }

    return new Response("Not found", { status: 404 });
  },
});

console.log(`Stats API running on port ${server.port}`);
```

#### 3. Systemd Service erstellen

Datei `/etc/systemd/system/stats-api.service`:

```ini
[Unit]
Description=Stats API for Homepage
After=network.target

[Service]
ExecStart=/root/.bun/bin/bun run /opt/stats-api/index.ts
WorkingDirectory=/opt/stats-api
Restart=always
RestartSec=5
Environment="PATH=/root/.bun/bin:/usr/bin"

[Install]
WantedBy=multi-user.target
```

#### 4. Service aktivieren

```bash
systemctl daemon-reload
systemctl enable stats-api
systemctl start stats-api
```

### API Response

```json
{
  "load1": 0.66,
  "load5": 0.61,
  "load15": 0.55,
  "ram": 44.7,
  "rootDisplay": "19GB / 94GB (22%)",
  "dataDisplay": "146GB / 1754GB (8%)"
}
```

## Homepage Konfiguration

In `services.yaml`:

```yaml
- Proxmox Storage:
    icon: mdi-harddisk
    widget:
      type: customapi
      url: http://192.168.0.101:4000/stats
      refreshInterval: 30
      mappings:
        - field: dataDisplay
          label: VM Storage
        - field: rootDisplay
          label: System
```

## Nützliche Befehle

```bash
# Service Status prüfen
systemctl status stats-api

# Logs anzeigen
journalctl -u stats-api -f

# API testen
curl http://localhost:4000/stats

# Service neustarten
systemctl restart stats-api
```

## Fehlerbehebung

### API antwortet nicht

1. Prüfen ob Service läuft:
```bash
systemctl status stats-api
```

2. Port prüfen:
```bash
ss -tlnp | grep 4000
```

3. Logs prüfen:
```bash
journalctl -u stats-api --no-pager -n 50
```

### LVM Befehl schlägt fehl

Der volle Pfad `/usr/sbin/lvs` muss verwendet werden, da der PATH in systemd eingeschränkt ist.
