# Remote CCTV Infrastructure & Dynamic NAT Proxy (Poland Site)

A zero-maintenance, low-power remote camera gateway and management setup deployed on a fanless Mini PC in Poland.

This project implements a **MAC-aware Dynamic Virtual IP (VIP) Proxy** on Debian. It ensures that Reolink IP cameras assigned dynamic DHCP addresses by a local ISP router remain accessible via persistent, static Virtual IPs (`192.168.99.X`) over secure VPN mesh networks (**Tailscale** & **ZeroTier**).

---

## 🔑 Key Features

* **Dynamic MAC-Based NAT Tracking:** Automatically discovers camera dynamic IPs via local `arp-scan` by MAC address and dynamically rewrites `iptables` DNAT rules.
* **Persistent Subnet Routing:** Maps each camera to a fixed Virtual IP (`192.168.99.11-13`) exposed via Tailscale/ZeroTier subnet routers.
* **Flash Memory Protection (NAND Wear Minimization):** Optimized for low-end flash storage on Mini PCs by moving volatile logging (`/var/log` and `journald`) to RAM (`tmpfs`), preventing NAND wear and drive degradation.
* **Zero-Moving-Parts Reliability:** Designed for fanless hardware operating continuously without active thermal fans or physical maintenance access.

---

## 🛠️ Hardware Overview

| Device | Specifications / Details | Function |
| :--- | :--- | :--- |
| **Host Gateway** | Coofun N40 Mini PC (Debian, Fanless, PoE-powered via PD03S Splitter) | Central VPN Gateway & Dynamic NAT Router |
| **PoE Switch** | Reolink RLA-PS1E (5-Port Gigabit, 65W Budget) | Power and connectivity for cameras & host |
| **Camera 1** | Reolink TrackMix PoE (4K PTZ Auto-Tracking, 256GB High-Endurance MicroSD) | Perimeter Security |
| **Camera 2** | Reolink TrackMix PoE (4K PTZ Auto-Tracking, 256GB High-Endurance MicroSD) | Yard Security |
| **Camera 3** | Reolink RLC-520A (5MP Fixed Dome, 256GB High-Endurance MicroSD) | Entryway Security |

---

## 📐 Network Architecture

```
                       [ Tailscale / ZeroTier Mesh Network ]
                                         │
                                         ▼
                      [ Host Gateway (Debian Mini PC) ]
                          Primary VIP Interface: eth0
                         VIP Subnet: 192.168.99.0/24
                                         │
                        (Dynamic NAT via camera-nat-sync)
                                         │
       ┌─────────────────────────────────┼─────────────────────────────────┐
       ▼                                 ▼                                 ▼
[ VIP: 192.168.99.11 ]            [ VIP: 192.168.99.12 ]            [ VIP: 192.168.99.13 ]
 (Camera 1 - TrackMix)             (Camera 2 - TrackMix)             (Camera 3 - RLC-520A)
  MAC: ec:71:db:xx:xx:26            MAC: ec:71:db:xx:xx:95            MAC: ec:71:db:xx:xx:78
```

---

## ⚡ Deployment & Setup Guide

### 1. Enable Flash Memory (NAND) Wear Protections

To protect the host Mini PC's onboard storage from excessive write operations:

#### A. Mount `/var/log` in RAM (`tmpfs`)
Add the following line to `/etc/fstab`:
```conf
tmpfs   /var/log    tmpfs   defaults,noatime,nosuid,nodev,noexec,mode=0755,size=100M    0   0
```

#### B. Restrict Systemd Journal Writing
Modify `/etc/systemd/journald.conf`:
```ini
[Journal]
Storage=volatile
RuntimeMaxUse=30M
```

#### C. Disable Swap and Set `noatime`
Update `/etc/sysctl.conf`:
```conf
vm.swappiness=1
```
In `/etc/fstab`, ensure the root filesystem options include `noatime`.

---

### 2. Configure Virtual IP Interface

Assign the VIP subnet gateway to your host's local network interface (e.g., `eth0`):

```bash
sudo ip addr add 192.168.99.1/24 dev eth0
```

---

### 3. Deploy the Dynamic NAT Synchronization Script

Create the sync script at `/usr/local/bin/camera-nat-sync.sh`:

```bash
#!/bin/bash

# MAC-to-VIP Map Configuration
declare -A VIPS=(
    ["ec:71:db:90:fb:26"]="192.168.99.11"
    ["ec:71:db:b6:55:95"]="192.168.99.12"
    ["ec:71:db:09:9c:78"]="192.168.99.13"
)

# Flush existing PREROUTING NAT rules
iptables -t nat -F PREROUTING

# Ensure outbound MASQUERADE rule is active
iptables -t nat -C POSTROUTING -s 192.168.99.0/24 -j MASQUERADE 2>/dev/null \
    || iptables -t nat -A POSTROUTING -s 192.168.99.0/24 -j MASQUERADE

# Perform local ARP scan
SCAN=$(arp-scan --localnet)

# Dynamic DNAT update loop
for MAC in "${!VIPS[@]}"; do
    VIP="${VIPS[$MAC]}"
    REAL_IP=$(echo "$SCAN" | awk -v mac="$MAC" 'tolower($2) == tolower(mac) {print $1}')

    if [[ -n "$REAL_IP" ]]; then
        echo "Mapping VIP $VIP -> Dynamic Real IP $REAL_IP"
        iptables -t nat -A PREROUTING -d "$VIP" -j DNAT --to-destination "$REAL_IP"
    else
        echo "Warning: Camera with MAC $MAC not found on local network"
    fi
done
```

Make the script executable:
```bash
sudo chmod +x /usr/local/bin/camera-nat-sync.sh
```

---

### 4. Create Systemd Service & Automation Timer

#### Service Unit (`/etc/systemd/system/camera-nat-sync.service`)
```ini
[Unit]
Description=Sync Reolink Camera Dynamic NAT Rules
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/camera-nat-sync.sh
```

#### Timer Unit (`/etc/systemd/system/camera-nat-sync.timer`)
```ini
[Unit]
Description=Run Camera NAT Sync Every Minute

[Timer]
OnBootSec=30
OnUnitActiveSec=60

[Install]
WantedBy=timers.target
```

Enable and start the timer:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now camera-nat-sync.timer
```

---

### 5. Configure VPN Subnet Routing

#### Tailscale
Advertise the static VIP subnet across your Tailnet:
```bash
sudo tailscale up --advertise-routes=192.168.99.0/24
```
*Approve the route in the Tailscale Admin Console.*

#### ZeroTier (Backup Remote Management)
Add a managed route in the ZeroTier Central Console pointing `192.168.99.0/24` to the host node's ZeroTier IP address.

---

## 🔒 Security & Privacy Notice

* All MAC addresses, UIDs, serial numbers, and account credentials in this repository are sanitized placeholder templates.
* Never commit real production secrets, Tailscale auth keys, or camera UIDs to public GitHub repositories.