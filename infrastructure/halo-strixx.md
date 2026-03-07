---
title: Halo Strixx - AI Workstation
description: Desktop PC mit AMD Ryzen AI MAX+ 395 (Strix Halo) - AI Workstation
published: true
date: 2026-03-07T00:00:00.000Z
tags: hardware, desktop, ai, workstation, cachy-os, amd, strix-halo, llm, rocm
editor: markdown
dateCreated: 2026-03-07T00:00:00.000Z
---

# Halo Strixx - AI Workstation

Desktop PC, primär für lokale KI/LLM-Workloads.

---

## Hardware

| Komponente | Details |
|------------|---------|
| **CPU** | AMD Ryzen AI MAX+ 395 (Strix Halo) |
| **Kerne** | 16x Zen 5 (32 Threads), bis 5.1 GHz |
| **iGPU** | Radeon 8060S (40 RDNA 3.5 CUs) |
| **NPU** | XDNA 2, 50+ TOPS |
| **RAM** | 128 GB LPDDR5X (UMA - geteilt CPU/GPU) |
| **Storage** | 2 TB NVMe (PCIe 4.0) |
| **VRAM** | bis zu 96 GB (aus UMA-Pool zuweisbar) |
| **TDP** | 55W default, konfigurierbar bis 120W |

### UMA - Unified Memory Architecture

Der Strix Halo nutzt ein einheitliches Speicherpool für CPU und GPU. Das bedeutet:
- GPU kann direkt auf den gesamten RAM zugreifen (kein separater VRAM)
- Ideal für grosse LLM-Modelle (70B+ Parameter bei 128GB RAM)
- Vergleichbar mit Apple M-Serie Architektur

```
CPU (16x Zen 5) ──┐
                  ├── 128 GB LPDDR5X (UMA)
iGPU (40 CUs) ───┘
NPU (XDNA 2) ─────── direkter Zugriff
```

---

## Netzwerk

| Parameter | Wert |
|-----------|------|
| **IP** | 192.168.0.103 |
| **Tailscale IP** | 100.64.0.9 |
| **Verbindung** | WiFi (wlan0) |
| **Hostname** | halo-strixx |

---

## Betriebssystem

| Parameter | Wert |
|-----------|------|
| **OS** | CachyOS (Arch-basiert) |
| **Kernel** | 6.19.2-2-cachyos |
| **Shell** | fish |

---

## Software (AI-relevant)

| Software | Zweck |
|----------|-------|
| **LM Studio** | Lokale LLM-Verwaltung und Inferenz (GUI) |
| **OpenClaw** | Autonomer AI-Agent (lokal self-hosted) |
| **SillyTavern** | LLM-Frontend für Chat/Roleplay |
| **Docker + ROCm** | AMD GPU-beschleunigte LLM-Inferenz (`kyuz0/amd-strix-halo-toolboxes:rocm-6.4.4`) |
| **Distrobox** | Andere Linux-Distributionen in Containern |

### OpenClaw

[OpenClaw](https://openclaw.ai/) ist ein self-hosted autonomer AI-Agent. Er läuft als lokaler WebSocket-Server und verbindet sich mit Messaging-Apps (WhatsApp, Telegram) als Interface. Kann Emails verwalten, Kalender, Shell-Commands ausführen etc.

### ROCm / Docker

```bash
# Laufender Container
docker ps
# llama-rocm   kyuz0/amd-strix-halo-toolboxes:rocm-6.4.4
```

AMD ROCm 6.4.4 für GPU-beschleunigte Inferenz auf der integrierten Radeon 8060S.

---

## Tailscale

Halo Strixx ist im Headscale-Netz eingebunden:

```bash
tailscale up --login-server=https://zabooz.duckdns.org --accept-routes --accept-dns
```

---

*Letzte Aktualisierung: 7. März 2026*
