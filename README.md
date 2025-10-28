## RPI Common setup
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
