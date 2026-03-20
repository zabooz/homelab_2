---
title: Home Assistant
description: Smart Home Plattform auf Proxmox mit Zigbee2MQTT
published: true
date: 2026-03-20T00:00:00.000Z
tags: service, home-assistant, smart-home, proxmox, zigbee, zigbee2mqtt, mqtt, sonoff
editor: markdown
dateCreated: 2026-02-02T00:00:00.000Z
---

# Home Assistant

**Host:** VM 305 auf homeserver (HAOS)
**Port:** 8123
**URL:** http://192.168.0.114:8123
**Zugriff:** Heimnetz / VPN

## Beschreibung

Home Assistant ist eine Open-Source Smart Home Plattform zur Steuerung und Automatisierung von IoT-Geräten. Die Instanz läuft als HAOS VM auf dem Proxmox Host (192.168.0.101).

## Zigbee Setup

### Hardware

- **Dongle:** Sonoff Zigbee 3.0 USB Dongle Plus (CP210x / Silicon Labs Chip)
- **Coordinator:** Texas Instruments CC2652
- **USB-Passthrough:** Über Proxmox → VM 305 → Hardware → USB Device (Vendor/Device ID: `10c4:ea60`)
- **Gerätepfad in der VM:** `/dev/ttyUSB0`

### Zigbee2MQTT (aktiv)

Zigbee2MQTT ist die aktive Zigbee-Integration, läuft als Home Assistant Add-on. Wurde gewählt weil ZHA Probleme mit der IASZone-Konfiguration von batteriebetriebenen Sensoren hatte (Timeouts, State nicht aktualisiert).

**Add-on Repository:** `https://github.com/zigbee2mqtt/hassio-zigbee2mqtt`

**Abhängigkeiten:**
- Mosquitto MQTT Broker (Add-on) — läuft als `mqtt://core-mosquitto`
- MQTT Integration in HA (Broker: `core-mosquitto`, Port: `1883`)
- MQTT-User für Authentifizierung (unter Settings → People → Users angelegt)

**Konfiguration:**
- Serial Port: `/dev/ttyUSB0`
- Adapter Type: zstack
- Baudrate: 115200
- MQTT Server: `mqtt://core-mosquitto`
- Frontend Port: 8099
- Zigbee Channel: 11

### ZHA (deaktiviert)

ZHA (Zigbee Home Automation) ist die eingebaute HA-Integration, wurde aber deaktiviert. ZHA und Zigbee2MQTT können **nicht gleichzeitig** denselben Dongle nutzen. ZHA hatte Probleme mit dem SNZB-04P (bekannter Bug: IASZone-Konfiguration bricht ab wegen Sleep-Timeouts bei batteriebetriebenen Geräten).

## Geräte

| Gerät | Typ | Zigbee-Rolle | Standort | Entity |
|-------|-----|-------------|----------|--------|
| Sonoff SNZB-04P | Tür-/Fenstersensor | End Device | Abstellkammer | `binary_sensor.abstellkammer_door` |
| A60 DIM T | Dimmbare Lampe | **Router** (Mesh) | Abstellkammer | `light.abstellkammer_lampe` |

### Zigbee Mesh

Die Lampe (netzstrombetrieben) fungiert als Zigbee-Router und erweitert das Mesh-Netzwerk. Batteriebetriebene Geräte wie der SNZB-04P sind End Devices und routen nicht weiter. Mehr Router-Geräte (Lampen, Steckdosen) = bessere Reichweite und Stabilität.

### Hinweis: Sonoff SNZB-04P Pairing

Der SNZB-04P schläft als batteriebetriebenes Gerät häufig ein, was die Konfiguration nach dem Pairing verzögert. Während der Konfigurationsphase regelmäßig den **Reset-Knopf kurz drücken** um den Sensor wach zu halten.

## Siehe auch

- [Netzwerk Übersicht](/en/infrastructure/NETWORK_OVERVIEW) - IP-Adressen und Topologie
- [Proxmox Netzwerk Setup](/en/infrastructure/proxmox-netzwerk-setup) - Virtualisierung
