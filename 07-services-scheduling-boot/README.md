# 7. Services, Scheduling & Boot

## A. Services & systemd

Most modern distros use **systemd** as the init system and service manager. Services are called **units** (`.service`, `.socket`, `.timer`, `.mount`, `.target`).

```bash
systemctl status nginx         # is it running? recent logs
sudo systemctl start nginx     # start now
sudo systemctl stop nginx      # stop now
sudo systemctl restart nginx   # restart
sudo systemctl reload nginx    # reload config without full restart
sudo systemctl enable nginx    # start automatically at boot
sudo systemctl disable nginx   # don't start at boot
sudo systemctl enable --now nginx   # enable AND start immediately
systemctl is-active nginx      # active/inactive
systemctl is-enabled nginx     # enabled/disabled
systemctl list-units --type=service        # running services
systemctl list-unit-files --type=service   # all services + state
sudo systemctl daemon-reload   # reload systemd after editing unit files

# System-wide power control
sudo systemctl reboot          # reboot
sudo systemctl poweroff        # shut down
sudo systemctl suspend         # sleep
```

Unit files live in `/etc/systemd/system/` (custom) and `/lib/systemd/system/` (packaged). A minimal service unit:

```ini
[Unit]
Description=My App
After=network.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure
User=alice

[Install]
WantedBy=multi-user.target
```

On older systems you might see SysVinit or Upstart with `service nginx start` and `chkconfig`, but systemd is the norm now.

### Troubleshooting

**A service won't start** — get the reason in this order: `systemctl status servicename` (shows the last few log lines and exit code), then `journalctl -u servicename -e` (full recent logs), then `journalctl -xe` for surrounding context. The actual error is almost always in there.

**"Failed to start ... see 'systemctl status'"** — common causes: a config syntax error (many services offer a test command like `nginx -t` or `sshd -t`), a port already in use (`ss -tulnp | grep :PORT`), a missing file/permission, or wrong `User=` in the unit.

**Edited a unit file but nothing changed** — systemd caches units. Run `sudo systemctl daemon-reload`, then restart the service.

**Service starts then immediately dies** — status shows `activating (auto-restart)` or `failed`. The process is crashing; the journal has the stack trace or error. Check `ExecStart` path and that dependencies (like a database) are up.

**Service is running but "enable" didn't stick** — you started it but didn't enable it, or vice versa. Use `systemctl enable --now name` to do both; verify with `systemctl is-enabled name`.

**Masked service won't start at all** — `Unit is masked`. Unmask it: `sudo systemctl unmask name`.

**Find what's slowing boot** — `systemd-analyze blame` lists services by startup time; `systemd-analyze critical-chain` shows the critical path.

---

## B. Scheduling Tasks (cron, at, timers)

**cron** runs commands on a schedule; **at** runs a one-off command at a future time; systemd **timers** are the modern alternative.

```bash
crontab -e                     # edit your user's cron jobs
crontab -l                     # list your cron jobs
crontab -r                     # remove all your cron jobs
sudo crontab -e -u alice       # edit another user's crontab
```

Cron time format — five fields then the command:

```
┌ minute (0-59)
│ ┌ hour (0-23)
│ │ ┌ day of month (1-31)
│ │ │ ┌ month (1-12)
│ │ │ │ ┌ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command
```

Examples:

```
0 2 * * *      /backup.sh           # every day at 02:00
*/15 * * * *   /check.sh            # every 15 minutes
0 9 * * 1-5    /report.sh           # 09:00 on weekdays
0 0 1 * *      /monthly.sh          # midnight on the 1st of each month
@reboot        /startup.sh          # once at boot
```

System-wide cron lives in `/etc/crontab`, `/etc/cron.d/`, and the `/etc/cron.{hourly,daily,weekly,monthly}/` directories.

`at` for one-time jobs:

```bash
echo "command" | at 14:00      # run at 2 PM today
at now + 1 hour                # then type commands, Ctrl+D to finish
atq                            # list pending at jobs
atrm 3                         # remove at job 3
```

systemd timers:

```bash
systemctl list-timers          # show active timers
systemctl status mytimer.timer # a specific timer
```

### Troubleshooting

**A cron job doesn't run** — the number one cause is **environment**: cron runs with a minimal `PATH` and no shell profile. Use **absolute paths** for every command (`/usr/bin/python3`, not `python3`) and don't rely on variables you set in `.bashrc`. Set them at the top of the crontab if needed.

**How to see if cron even tried** — check the cron log: `grep CRON /var/log/syslog` (Debian) or `journalctl -u cron`/`journalctl -u crond`. If there's no entry, the schedule syntax is wrong or the crontab wasn't saved.

**The job runs but "fails silently"** — cron discards output. Capture it: append `>> /tmp/job.log 2>&1` to the cron line, then read that file to see the real error.

**Editing crontab with the wrong editor** — set your preferred one: `EDITOR=nano crontab -e`.

**Job runs at the wrong time** — check the server's timezone (`timedatectl`); cron uses system time. Also remember day-of-week and day-of-month fields are OR'd together when both are set.

**"You (user) are not allowed to use this program (crontab)"** — you're blocked by `/etc/cron.deny` or missing from `/etc/cron.allow`.

**`at` job never fired** — the `atd` service must be running: `systemctl status atd` (or `atd`), and enable it if not.

---