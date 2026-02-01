---
title: Linux Interface-Benennung
description: Netzwerk-Interface-Namen verstehen
published: true
date: 2026-01-18T00:00:00.000Z
tags: konzept, concept, netzwerk, network, linux, interfaces, lap
editor: markdown
dateCreated: 2026-01-18T00:00:00.000Z
---

# Linux Interface-Benennung

Netzwerk-Interface-Namen verstehen.

---

## Alte Konvention (vor systemd)

```
eth0, eth1, eth2  → Netzwerkkarten
wlan0             → WLAN
lo                → Loopback
```

**Problem:** Reihenfolge nicht vorhersehbar

---

## Neue Konvention (Predictable Names)

```
en  = Ethernet
wl  = WLAN
lo  = Loopback

Suffix:
s<slot>    = Slot-Nummer (z.B. ens18)
p<bus>     = PCI-Bus (z.B. enp0s3)
```

**Beispiele:**
- `ens18` = Ethernet, Slot 18 (Proxmox VMs)
- `enp0s3` = Ethernet, PCI-Bus 0, Slot 3
- `wlp2s0` = WLAN, PCI-Bus 2, Slot 0
- `eth0` = LXC-Container (alte Benennung)

---

## auto vs. allow-hotplug

```bash
# auto - Immer beim Boot starten (SERVER!)
auto ens18

# allow-hotplug - Nur bei Kabel-Erkennung (Laptop)
allow-hotplug ens18
```

---

*Siehe auch: [Diagnose-Befehle](diagnose)*
