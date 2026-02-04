---
title: VM Image Update Workflow
description: Automatischer OS-Update und FOG Image Capture Workflow für Master-VMs
published: true
date: 2026-02-03T00:00:00.000Z
tags: service, n8n, workflow, automation, fog, proxmox, pxe, imaging, automatisierung
editor: markdown
dateCreated: 2026-02-03T00:00:00.000Z
---

# VM Image Update Workflow

Dieser n8n-Workflow automatisiert den kompletten Prozess des OS-Updates für Master-VMs und erstellt anschließend automatisch neue [FOG](/en/services/fog_project_setup_dokumentation_debian_12_lxc) Images für das Deployment.

**Trigger:** Alle 5 Tage, Mitternacht (Cron: `0 0 */5 * *`)
**Workflow-Dauer:** 1-3 Stunden (abhängig von Update-Größe)

---

## Verarbeitete VMs

| VMID | Name | Distro | FOG Host ID | Update Command |
|------|------|--------|-------------|----------------|
| 500 | cachy-master | CachyOS (Arch) | 9 | `rm -f /var/lib/pacman/db.lck && pacman -Syu --noconfirm` |
| 501 | ubuntu-master | Ubuntu | 13 | `DEBIAN_FRONTEND=noninteractive apt-get update && apt-get dist-upgrade -y` |
| 502 | fedora-master | Fedora | 10 | `dnf upgrade -y --assumeyes` |
| 503 | mint-master | Linux Mint | 11 | `DEBIAN_FRONTEND=noninteractive apt-get update && apt-get dist-upgrade -y` |
| 504 | debian-master | Debian | 12 | `DEBIAN_FRONTEND=noninteractive apt-get update && apt-get dist-upgrade -y` |
| 507 | debian-server-master | Debian Server | 14 | `DEBIAN_FRONTEND=noninteractive apt-get update && apt-get dist-upgrade -y` |
| 508 | parrot-master | Parrot OS | 16 | `DEBIAN_FRONTEND=noninteractive apt-get update && apt-get dist-upgrade -y` |

> Debian/Ubuntu-basierte Distros verwenden zusätzlich `-o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"` um interaktive Prompts zu unterdrücken.

---

## Workflow-Ablauf

```
1. Schedule Trigger (alle 5 Tage, 00:00)
2. VM-Liste definieren (Edit Fields)
3. Array in Einzelitems trennen (Split Out)
4. Für jede VM sequentiell (Batch Size = 1):
   a. VM starten (Proxmox API)
   b. 2 Min warten (Boot Time)
   c. OS Updates installieren (qm guest exec, Timeout: 30 Min)
   d. VM herunterfahren (Proxmox API)
   e. 1 Min warten (Shutdown Time)
   f. FOG Capture Task erstellen (FOG API)
   g. 5 Sek warten (Task Registration)
   h. VM starten (PXE Boot → FOG Capture → auto Shutdown)
5. Nächste VM (zurück zu 4a)
```

---

## Workflow-Schritte im Detail

| # | Node | Typ | API/Endpoint | Details |
|---|------|-----|-------------|---------|
| 1 | Schedule Trigger | Cron | — | `0 0 */5 * *` (Europe/Vienna) |
| 2 | vms | Edit Fields | — | Definiert Array mit 7 VM-Objekten (vmid, fogHostId, name, updateCmd) |
| 3 | split_vms | Split Out | — | Trennt `vm_list` Array in einzelne Items |
| 4 | Loop Over Items | Loop | — | Sequentielle Verarbeitung (Batch Size: 1) |
| 5 | start_vm | HTTP Request | `POST .../qemu/{vmid}/status/start` | Proxmox API, SSL ignorieren (Self-Signed) |
| 6 | Wait | Wait | — | 2 Minuten (VM Boot + QEMU Guest Agent) |
| 7 | update_os | Execute Command | SSH → `qm guest exec {vmid} -- {updateCmd}` | Timeout: 30 Min (1800000ms), via Proxmox SSH |
| 8 | shutdown_vm | HTTP Request | `POST .../qemu/{vmid}/status/shutdown` | Proxmox API |
| 9 | Wait1 | Wait | — | 1 Minute (Shutdown abwarten) |
| 10 | set_capture_task | HTTP Request | `POST http://192.168.0.113/fog/host/{fogHostId}/task` | Body: `{"taskTypeID": 2, "shutdown": true}` |
| 11 | Wait2 | Wait | — | 5 Sekunden (FOG Task Registration) |
| 12 | start_vm1 | HTTP Request | `POST .../qemu/{vmid}/status/start` | VM bootet PXE → FOG Capture → auto Shutdown |

**Proxmox API Base URL:** `https://192.168.0.101:8006/api2/json/nodes/homeserver`

### PXE Boot Ablauf (nach Node 12)

1. VM bootet via PXE (Network Boot vor Disk in Boot Order)
2. FOG erkennt pending Capture Task
3. Image wird von VM-Disk erstellt
4. Image auf FOG Server gespeichert
5. VM wird automatisch heruntergefahren (`shutdown: true`)

---

## Fehlerbehandlung

### Retry Settings

Alle HTTP Request Nodes (start_vm, shutdown_vm, set_capture_task, start_vm1):

```
Retry On Fail: Enabled
Max Tries: 3
Wait Between Tries: 10 Sekunden
```

---

## Troubleshooting

### VM startet nicht

**Symptome:** HTTP 500 bei start_vm

1. VM bereits gestartet? → `qm status <vmid>`
2. Proxmox API Token gültig?
3. Node Name korrekt? (homeserver)

### OS Update schlägt fehl

**Symptome:** update_os Node Timeout oder Error

1. **Guest Agent nicht installiert:**
   ```bash
   apt install qemu-guest-agent  # Debian/Ubuntu
   pacman -S qemu-guest-agent    # Arch/CachyOS
   dnf install qemu-guest-agent  # Fedora
   ```
2. **Timeout zu kurz** — auf 30 Minuten erhöhen (1800000ms)
3. **VM nicht vollständig gebootet** — Wait nach start_vm auf 3 Minuten erhöhen
4. **Paketmanager locked** (Arch) — `rm -f /var/lib/pacman/db.lck` im updateCmd
5. **Interactive Prompts** (Debian/Ubuntu) — `DEBIAN_FRONTEND=noninteractive` im bash -c Wrapper

### FOG Capture Task wird nicht erstellt

1. **FOG API Tokens prüfen:**
   ```bash
   curl -H "fog-api-token: TOKEN" \
        -H "fog-user-token: TOKEN" \
        http://192.168.0.113/fog/host
   ```
2. **FOG Host ID korrekt?** — In FOG Web-UI: Hosts → Host anzeigen → ID notieren
3. **VM bereits heruntergefahren?** — Capture braucht gestoppte VM

### PXE Boot startet nicht

1. **Boot Order in VM prüfen:**
   ```bash
   qm config <vmid> | grep boot
   ```
   Network Boot muss vor Disk sein
2. **FOG Task nicht registriert:** In FOG Web-UI Active Tasks prüfen, Wait2 verlängern

---

## Monitoring

### Execution Logs

**n8n Web-UI → Executions:**
- Jede Iteration sichtbar (VM 500, 501, etc.)
- Node-Outputs für Debugging
- Execution Time pro VM

### Log-Dateien in VMs

```bash
# Debian/Ubuntu
tail -f /var/log/apt/term.log

# Arch/CachyOS
tail -f /var/log/pacman.log

# Fedora
tail -f /var/log/dnf.log
```

### Proxmox

```bash
# VM Console öffnen
qm terminal <vmid>

# Guest Agent Status
qm agent <vmid> ping
```

---

## Sicherheit

### SSH Command Injection Prevention

- `updateCmd` stammt aus kontrollierter Quelle (Edit Fields Node im Workflow)
- Keine User-Inputs in updateCmd — Commands sind vordefiniert

### SSH Host Key Regeneration

Alle geklonten VMs haben identische SSH Host Keys. Vor FOG Capture SSH Keys löschen:

```bash
# In Master-VM vor Capture
rm -f /etc/ssh/ssh_host_*
```

Neue Keys werden beim ersten Boot automatisch generiert.

---

*Siehe auch: [n8n Setup](/en/services/n8n-setup) · [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc) · [Proxmox Netzwerk](/en/infrastructure/proxmox-netzwerk-setup) · [Alle Workflows](/en/workflows)*
