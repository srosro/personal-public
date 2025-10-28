## RPI Common setup

### Wireless connectivity issues
 Problem

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
