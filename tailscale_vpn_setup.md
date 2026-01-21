---
# 🚀 Tailscale VPN - Setup & Fixes

**Datum:** 21. Januar 2026  
**Author:** Daniel (zabooz)  
**Zweck:** LAP-Vorbereitung

## 🔴 Die Probleme
- Container war offline:
```
bashtailscale status
# 100.64.0.1  tailscale  offline, last seen 7m ago
```
**Ursachen:**
1. Headscale löschte Node - Container war 15min offline → Auto-Cleanup
2. Kein Gateway - Container konnte Headscale nicht erreichen
3. IP Forwarding = 0 - Container konnte nicht routen

## ✅ Die Lösungen

### 1. Gateway permanent (Proxmox Config)
```
bash# /etc/pve/lxc/102.conf
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:D8:A7:B2,ip=192.168.0.112/24,gw=192.168.0.1,type=veth
```
**Warum?** LXC lädt /etc/network/interfaces nicht automatisch!

### 2. IP Forwarding Service
```
bash# /etc/systemd/system/ip-forwarding.service
[Unit]
Description=Enable IP Forwarding
Before=tailscaled.service

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
```
bash# systemctl daemon-reload
systemctl enable ip-forwarding.service
```

### 3. Auto-Start aktivieren
```
bash# pct set 102 --onboot 1
pct set 102 --startup order=1,up=30
```

---

## 🌐 Wie es funktioniert

### Packet Flow
```
Laptop (100.64.0.2) 
    ↓ via Tailscale VPN
Container (100.64.0.1)
    ↓ IP Forwarding = 1 (leitet weiter)
    ↓ NAT: Source 100.64.0.2 → 192.168.0.112
Proxmox (192.168.0.101)
    ↓ Antwortet an 192.168.0.112
Container
    ↓ Reverse NAT: Dest → 100.64.0.2
Laptop empfängt ✅
```

### Policy-Based Routing (Table 52)
- Laptop nutzt 2 Routing-Tables:
```
bash# Main Table - normaler Traffic:
ip route show
# default via 172.17.99.219 dev wlan0

# Tailscale Table 52 - VPN-Traffic:
ip route show table 52
# 192.168.0.0/24 dev tailscale0
```
- Kernel entscheidet automatisch welche Table!

### Wichtige Konzepte
**IP Forwarding:**
- Aus (= 0): Computer verwirft fremde Pakete
- An (= 1): Computer leitet fremde Pakete weiter (Router!)
```
bash# Ohne IP Forwarding:
Paket zu 192.168.0.101 → "Nicht für mich" → VERWORFEN ❌

# Mit IP Forwarding:
Paket zu 192.168.0.101 → "Weiterleiten!" → GEROUTET ✅
```

### NAT/MASQUERADE
**Warum?**
```
Proxmox kennt 100.64.0.x nicht!
→ Container ändert Source-IP: 100.64.0.2 → 192.168.0.112
→ Proxmox kann antworten ✅
```

### Headscale Auto-Cleanup
**Nodes offline > 15min → gelöscht!**
```
18:50 - Container disconnected
19:07 - Node deleted (15min timeout)
→ Muss neu registriert werden
```

## 🧪 Test-Checkliste
```
bash# Im Container:
ip route show  # Gateway da?
sysctl net.ipv4.ip_forward  # = 1?
systemctl status ip-forwarding.service  # läuft?
tailscale status  # online?

# Auf Proxmox:
pct config 102 | grep onboot  # = 1?

# Vom Laptop (Hotspot):
ping 192.168.0.101  # 0% loss?
```

## 🔧 Troubleshooting
- Container offline?
```
bash# pct start 102
pct enter 102
systemctl restart tailscaled
```
- Kein Subnet-Routing?
```
bash# Container:
sysctl -w net.ipv4.ip_forward=1
```
- Laptop:
```
sudo tailscale up --login-server=https://zabooz.duckdns.org --accept-routes
```
- Gateway fehlt?
```
bash# Proxmox Config checken:
nano /etc/pve/