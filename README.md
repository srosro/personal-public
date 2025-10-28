## RPI Common setup
### Install common pakages
```
sudo apt install vim npm nmap tailscale python3-venv curl tree screen python3-pip
sudo tailscale up
```

### Wireless connectivity issues
 **Enabling auto-reconnect**
 
  1. Enable Infinite Auto-Reconnect Retries

  By default, NetworkManager gives up after 4 connection attempts. Configure infinite retries so the Pi continuously tries to reconnect after network outages.

  Edit /etc/NetworkManager/NetworkManager.conf:
  [main]
  plugins=ifupdown,keyfile
  autoconnect-retries-default=0

  [ifupdown]
  managed=false

  Apply changes:
  sudo systemctl reload NetworkManager

  2. Configure Primary and Fallback WiFi Networks

  Set up two WiFi networks with different priorities for automatic failover:

  Add primary network (higher priority):
  sudo nmcli connection add type wifi con-name "Practice Patience" \
    ifname wlan0 ssid "Practice Patience" \
    wifi-sec.key-mgmt wpa-psk wifi-sec.psk "YOUR_PASSWORD" \
    connection.autoconnect yes connection.autoconnect-priority 10

  Set fallback network (lower priority):
  sudo nmcli connection modify "Practice Community" connection.autoconnect-priority 5

  Verify configuration:
  nmcli -f NAME,TYPE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show

  NetworkManager will prefer the higher priority network and automatically fall back when unavailable.
 
 **Buggy Driver**

  - RTL8188EUS USB WiFi adapter disconnected due to weak signal/beacon loss
  - The mainline rtl8xxxu driver failed to recover and couldn't scan for networks
  - Required manual driver reload to reconnect (known bug with rtl8xxxu)

  Solution: Switch to lwfinger's rtl8188eu Driver

  1. Disable USB Autosuspend Globally

  sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.backup
  sudo sed -i 's/$/ usbcore.autosuspend=-1/' /boot/firmware/cmdline.txt
  Why: Prevents USB power management from interfering with the WiFi adapter

  2. Blacklist the Buggy rtl8xxxu Driver

  echo "blacklist rtl8xxxu" | sudo tee /etc/modprobe.d/blacklist-rtl8xxxu.conf
  Why: The mainline driver has known issues with disconnection recovery

  3. Install lwfinger's Better Driver

  git clone https://github.com/lwfinger/rtl8188eu.git
  cd rtl8188eu
  make
  sudo make install
  Why: This driver has better stability and roaming support for RTL8188EUS chipsets


### Nightly restarts
Add a cron job to root's crontab that runs /sbin/reboot at a specific time each night (commonly 3 AM or 4 AM when the system is typically idle).
```
0 3 * * * /sbin/reboot
```

### Persistent Journaling
```
odio@rpi2love:~ $ ls -la /var/log/journal/
total 8
drwxr-sr-x+ 2 root systemd-journal 4096 Sep 30 17:05 .
drwxr-xr-x  9 root root            4096 Oct 28 00:52 ..
odio@rpi2love:~ $ sudo journalctl --header | grep "Storage"
odio@rpi2love:~ $ cat /etc/systemd/journald.conf | grep -i storage
#Storage=auto
odio@rpi2love:~ $ sudo mkdir -p /etc/systemd/journald.conf.d/
odio@rpi2love:~ $ sudo tee /etc/systemd/journald.conf.d/99-persistent.conf > /dev/null << 'EOF'
[Journal]
Storage=persistent
SystemMaxUse=100M
SystemKeepFree=1G
MaxRetentionSec=1month
EOF
odio@rpi2love:~ $ sudo systemctl restart systemd-journald
odio@rpi2love:~ $ ls -la /var/log/journal/
total 8
drwxr-sr-x+ 2 root systemd-journal 4096 Sep 30 17:05 .
drwxr-xr-x  9 root root            4096 Oct 28 00:52 ..
odio@rpi2love:~ $ sudo systemctl kill --signal=SIGUSR1 systemd-journald
odio@rpi2love:~ $ sudo journalctl --header | grep "File path"
```
