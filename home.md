---
title: Startseite
description: Homelab Wiki - Zentrale Dokumentation
published: true
date: 2026-01-25T20:29:09.858Z
tags: homelab, wiki, startseite, overview
editor: markdown
dateCreated: 2026-01-24T15:37:11.693Z
---

# Willkommen im Homelab Wiki

Deine zentrale Dokumentation für die gesamte Homelab-Infrastruktur, Services und Betrieb.

---

## Infrastruktur

| Dokument | Beschreibung |
|----------|--------------|
| [Netzwerk Übersicht](/en/infrastructure/NETWORK_OVERVIEW) | Master-Diagramm, IP-Liste, Quick Reference |
| [VPS](/en/infrastructure/VPS) | VPS Konfiguration & Services |
| [Proxmox Netzwerk Setup](/en/infrastructure/proxmox-netzwerk-setup) | Proxmox Cluster Netzwerk |
| [Halo Strixx](/en/infrastructure/halo-strixx) | AI Workstation (AMD Strix Halo, 128GB UMA) |

---

## LAP Konzepte

| Konzept | Beschreibung |
|---------|--------------|
| [IP-Adressen](/en/konzepte/ip-adressen) | IPv4, IPv6, Private/Public |
| [Subnetting](/en/konzepte/subnetting) | CIDR, Subnetzmasken, Berechnung |
| [DHCP](/en/konzepte/dhcp) | DORA-Prozess, Statisch vs. Dynamisch |
| [Routing](/en/konzepte/routing) | Gateway, Routing-Tabellen, Policy-Based |
| [NAT](/en/konzepte/nat) | SNAT, DNAT, MASQUERADE |
| [IP-Forwarding](/en/konzepte/ip-forwarding) | Router vs. Host |
| [DNS](/en/konzepte/dns) | Hierarchie, MagicDNS |
| [iptables](/en/konzepte/iptables) | Chains, Tables, Firewall |
| [Linux Interfaces](/en/konzepte/linux-interfaces) | Interface-Benennung |
| [VPN-Typen](/en/konzepte/vpn) | WireGuard vs. OpenVPN |
| [Ports](/en/konzepte/ports) | Wichtige Netzwerk-Ports |
| [OSI-Schichtenmodell](/en/konzepte/layer) | Die 7 Schichten, TCP/IP-Modell |
| [Reverse Proxy](/en/konzepte/reverse-proxy) | Forward vs. Reverse, Stream Proxy |
| [Diagnose-Befehle](/en/konzepte/diagnose) | Troubleshooting Commands |
| [Prüfungsfragen](/en/konzepte/pruefungsfragen) | LAP Q&A |

---

## Services

| Service | Beschreibung |
|---------|--------------|
| [Homepage Dashboard](/en/services/homepage-dashboard-setup) | Dashboard Konfiguration |
| [Wiki.js Setup](/en/services/wikijs-setup) | Wiki.js Installation und Git Sync |
| [Stats API](/en/services/stats-api) | Bun API für System-Monitoring |
| [Paperless-ngx](/en/services/paperless-ngx) | Dokumentenverwaltung mit OCR |
| [FOG Project](/en/services/fog_project_setup_dokumentation_debian_12_lxc) | DHCP Server, PXE Boot, Imaging |
| [Linkwarden](/en/services/linkwarden-setup) | Bookmark Manager (192.168.0.119) |
| [Draw.io](/en/services/drawio-setup) | Diagramm-Editor (192.168.0.122) |
| [Home Assistant](/en/services/homeassistant-setup) | Smart Home Steuerung (gestoppt) |
| [Pterodactyl](/en/services/pterodactyl-setup) | Gameserver Panel (gestoppt) |
| [n8n Automation](/en/services/n8n-setup) | Workflow-Automation (192.168.0.116) |
| [Changedetection](/en/services/changedetection-setup) | Website-Monitoring (192.168.0.125) |
| [Gotify](/en/services/gotify-setup) | Push-Benachrichtigungen (192.168.0.126) |
| [Stirling PDF](/en/services/sterlingpdf-setup) | PDF-Tools (192.168.0.131) |
| [ntopng](/en/services/ntopng-setup) | Netzwerk-Monitoring (192.168.0.140) |
| [Crawler4AI](/en/services/crawler4ai-setup) | AI Web Scraping (gestoppt) |
| [Node-RED](/en/services/node-red-setup) | Flow-basierte Automation (gestoppt) |
| [Immich](/en/services/immich-setup) | Google Photos Alternative (192.168.0.144) |
| [AdGuard Home](/en/services/adguard-setup) | DNS-Werbeblocker (192.168.0.137) |
| [Matrix Synapse](/en/services/matrix-setup) | Chat-Server mit Element Web (VPS) |
| [BookStack](/en/services/bookstack-setup) | Wiki fuer Schule (VPS) |
| [Dolibarr ERP](/en/services/dolibarr-setup) | ERP/CRM - Rechnungen, Inventar (192.168.0.149) |
---

## Workflows

| Workflow | Beschreibung |
|----------|--------------|
| [VM Image Update](/en/workflows/vm-image-update) | Automatisches OS-Update & FOG Capture |

---

## VPN

| Anleitung | Beschreibung |
|-----------|--------------|
| [VPN Infrastruktur](/en/vpn/vpn-infrastructure) | VPN Server Setup und Konfiguration |
| [VPN Client Anleitung](/en/vpn/vpn-client-guide) | Wie man sich mit dem VPN verbindet |
| [VPN Fehlerbehebung](/en/vpn/vpn-troubleshooting) | Häufige VPN Probleme und Lösungen |

---

## Betrieb

| Dokument | Beschreibung |
|----------|--------------|
| [Backup & Recovery](/en/operations/backup-recovery) | Backup Prozeduren und Wiederherstellung |
| [Disaster Recovery](/en/operations/disaster-recovery) | Disaster Recovery Planung |
| [Security Hardening](/en/operations/security-hardening) | Sicherheits Best Practices |
