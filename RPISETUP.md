
# Raspberry Pi Kiosk Setup Guide

Complete setup instructions for configuring a Raspberry Pi as a reliable kiosk display with auto-recovery features.

---

## Table of Contents
1. [Initial Package Installation](#1-initial-package-installation)
2. [Hostname Configuration](#2-hostname-configuration)
3. [WiFi Configuration](#3-wifi-configuration)
4. [WiFi Stability Fixes](#4-wifi-stability-fixes)
5. [Chromium Kiosk Service](#5-chromium-kiosk-service)
6. [Tailscale VPN](#6-tailscale-vpn)
7. [System Maintenance](#7-system-maintenance)
8. [Optional Developer Tools](#8-optional-developer-tools)
9. [Verification](#9-verification)

---

## 1. Initial Package Installation

Update system and install required packages:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y vim npm nmap tailscale python3-venv curl tree \
    screen python3-pip chromium network-manager git htop ncdu
```

---

## 2. Hostname Configuration

Set a unique hostname for the device:

```bash
sudo vim /etc/hostname
# Change to: rpi4kiosk (or your preferred name)

sudo vim /etc/hosts
# Update 127.0.1.1 line to match new hostname

sudo reboot
```

---

## 3. WiFi Configuration

### Connect to Primary Network

```bash
# Connect to WiFi
sudo nmcli device wifi connect "SSID Patience" password "YOUR_PASSWORD"

# Configure unlimited auto-reconnect attempts
sudo nmcli connection modify "preconfigured" connection.autoconnect-retries 0

# Set high priority for primary network
sudo nmcli connection modify "preconfigured" connection.autoconnect-priority 10

# Disable WiFi power management (prevents disconnections)
sudo nmcli connection modify "preconfigured" 802-11-wireless.powersave 2
```

### Optional: Add Fallback Network

Configure a secondary WiFi network for automatic failover:

```bash
# Add fallback network with lower priority
sudo nmcli connection add type wifi con-name "SSID Community" \
  ifname wlan0 ssid "SSID Community" \
  wifi-sec.key-mgmt wpa-psk wifi-sec.psk "FALLBACK_PASSWORD" \
  connection.autoconnect yes connection.autoconnect-priority 5
```

### Verify Configuration

```bash
nmcli -f NAME,TYPE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show
```

---

## 4. WiFi Stability Fixes

### Enable Infinite Auto-Reconnect in NetworkManager

Edit NetworkManager configuration:

```bash
sudo vim /etc/NetworkManager/NetworkManager.conf
```

Add/modify:
```ini
[main]
plugins=ifupdown,keyfile
autoconnect-retries-default=0

[ifupdown]
managed=false
```

Apply changes:
```bash
sudo systemctl reload NetworkManager
```

### Fix USB WiFi Adapter Issues (RTL8188EUS/RTL8192)

If using USB WiFi adapters with Realtek chipsets:

#### Disable USB Autosuspend

```bash
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.backup
sudo sed -i 's/$/ usbcore.autosuspend=-1/' /boot/firmware/cmdline.txt
```

#### Disable Driver Power Management

```bash
echo "options rtl8xxxu rtw_power_mgnt=0" | sudo tee /etc/modprobe.d/rtl8xxxu.conf
```

#### Optional: Switch to lwfinger's Driver (for RTL8188EUS)

If experiencing persistent disconnection issues:

```bash
# Blacklist buggy mainline driver
echo "blacklist rtl8xxxu" | sudo tee /etc/modprobe.d/blacklist-rtl8xxxu.conf

# Install better driver
git clone https://github.com/lwfinger/rtl8188eu.git
cd rtl8188eu
make
sudo make install
sudo reboot
```

---

## 5. Chromium Kiosk Service

Create a systemd service that automatically starts Chromium in kiosk mode on boot.

### Create Service File

```bash
sudo vim /etc/systemd/system/yodeck-kiosk.service
```

Paste this content (adjust `User` and URL as needed):

```ini
[Unit]
Description=Yodeck Kiosk Mode
After=network-online.target graphical.target
Wants=network-online.target

[Service]
Environment=DISPLAY=:0
Environment=XAUTHORITY=/home/odio/.Xauthority
Type=simple
User=odio
ExecStartPre=/bin/sleep 5
ExecStartPre=/bin/rm -f /home/odio/.config/chromium/Singleton*
ExecStart=/usr/bin/chromium --kiosk --noerrdialogs --disable-infobars --no-first-run --check-for-update-interval=31536000 https://player.yodeck.com/
Restart=on-failure
RestartSec=10

[Install]
WantedBy=graphical.target
```

**Key features:**
- Waits 5 seconds for system to stabilize
- Automatically removes stale Chromium lock files (prevents startup failures)
- Full-screen kiosk mode with no dialogs
- Auto-restarts on failure

### Enable and Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable yodeck-kiosk.service
sudo systemctl start yodeck-kiosk.service
```

### Check Service Status

```bash
systemctl status yodeck-kiosk.service
ps aux | grep chromium
```

---

## 6. Tailscale VPN

Install and configure Tailscale for remote access:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Verify connection:
```bash
tailscale status
```

---

## 7. System Maintenance

### Enable Persistent Journaling

Configure systemd journal to persist logs across reboots:

```bash
# Create configuration directory
sudo mkdir -p /etc/systemd/journald.conf.d/

# Create persistent logging config
sudo tee /etc/systemd/journald.conf.d/99-persistent.conf > /dev/null << 'EOF'
[Journal]
Storage=persistent
SystemMaxUse=100M
SystemKeepFree=1G
MaxRetentionSec=1month
EOF

# Restart journald
sudo systemctl restart systemd-journald

# Force immediate journal flush
sudo systemctl kill --signal=SIGUSR1 systemd-journald

# Verify persistence
sudo journalctl --header | grep "File path"
ls -la /var/log/journal/
```

### Schedule Nightly Reboots

Add a cron job to reboot the system at 3 AM daily (clears memory, ensures fresh start):

```bash
sudo crontab -e
```

Add this line:
```
0 3 * * * /sbin/reboot
```

---

## 8. Optional Developer Tools

### Claude Code

```bash
sudo npm install -g @anthropic-ai/claude-code
claude --version
```

### SSH Keys

Copy your SSH public key for passwordless access:

```bash
ssh-copy-id odio@rpi4kiosk

# Or manually add to ~/.ssh/authorized_keys
```

### Development Projects

```bash
mkdir ~/Hacking
cd ~/Hacking

# Clone your projects
git clone git@github.com:username/project.git
```

---

## 9. Verification

After setup and reboot, verify everything is working:

```bash
# Check hostname
hostname

# Check WiFi connection
nmcli connection show
iwconfig wlan0

# Check kiosk service
systemctl status yodeck-kiosk.service
ps aux | grep chromium

# Check Tailscale
tailscale status

# Check journal persistence
sudo journalctl --disk-usage
ls -la /var/log/journal/

# Check cron jobs
sudo crontab -l
```

---

## Troubleshooting

### Chromium Won't Start

```bash
# Check service logs
sudo journalctl -u yodeck-kiosk.service -n 50

# Manually remove lock files
rm ~/.config/chromium/Singleton*

# Restart service
sudo systemctl restart yodeck-kiosk.service
```

### WiFi Keeps Disconnecting

```bash
# Check connection status
nmcli connection show "preconfigured"

# View recent WiFi logs
sudo journalctl -u NetworkManager -n 100 | grep wlan

# Check USB WiFi adapter
lsusb | grep -i wireless
dmesg | grep -i wlan | tail -20
```

### Remote Access Issues

```bash
# Check Tailscale status
tailscale status

# Restart Tailscale
sudo systemctl restart tailscaled
```

---

## Key Configuration Files

- `/etc/systemd/system/yodeck-kiosk.service` - Kiosk service definition
- `/etc/NetworkManager/NetworkManager.conf` - Network auto-reconnect config
- `/etc/modprobe.d/rtl8xxxu.conf` - WiFi driver power management
- `/etc/systemd/journald.conf.d/99-persistent.conf` - Persistent logging
- `/etc/hostname` - Device hostname
- `/boot/firmware/cmdline.txt` - USB autosuspend settings

---

## Notes

- The kiosk service includes automatic lock file cleanup to prevent "profile in use" errors
- WiFi is configured for maximum stability with unlimited reconnection attempts
- System reboots nightly to ensure fresh state and clear any memory leaks
- Logs are persisted to help diagnose issues after reboots
- Tailscale provides secure remote access without port forwarding

---

**Setup Date:** October 2025
**Last Updated:** October 28, 2025
