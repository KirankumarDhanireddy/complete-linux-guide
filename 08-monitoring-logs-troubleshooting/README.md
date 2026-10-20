# 8. Monitoring, Logs & Troubleshooting

## A. System Monitoring & Performance

```bash
# CPU & processes
top                            # live overview (press 1 for per-core)
htop                           # friendlier interactive monitor
uptime                         # load averages (1/5/15 min)
vmstat 1                       # virtual memory / CPU stats each second
mpstat -P ALL 1                # per-CPU usage (sysstat)

# Memory
free -h                        # RAM and swap usage
cat /proc/meminfo              # detailed memory info

# Disk I/O
iostat -xz 1                   # disk I/O stats (sysstat)
iotop                          # per-process disk I/O (needs root)
df -h                          # disk space
du -sh *                       # sizes in current dir

# Network I/O
iftop                          # live bandwidth per connection
nload                          # simple bandwidth graph
ss -s                          # socket statistics summary

# General / all-in-one
dstat                          # combined CPU/disk/net/memory
sar                            # historical performance data (sysstat)
lscpu                          # CPU details
lsmem                          # memory ranges
```

Load average from `uptime`/`top`: roughly the number of processes wanting CPU. On an N-core machine, a sustained load above N means the system is saturated.

### Troubleshooting

**"System is slow" — where to look first** — open `top`/`htop` and identify the bottleneck by resource:
- **High `%us`/`%sy` CPU** → a process is compute-bound. Sort by CPU (`P` in top) and investigate the top offender.
- **High `%wa` (I/O wait)** → the disk is the bottleneck, not the CPU. Use `iotop` to find the process hammering the disk.
- **Low free memory + high swap activity** → memory pressure (see below).

**High load average but low CPU usage** — the load is coming from processes stuck in I/O wait (state D), not CPU. Look at `%wa` and `iostat -xz 1`; the fix is on the storage/network side.

**Memory looks "full" but the system is fine** — Linux uses free RAM for disk cache. In `free -h`, the number that matters is **available**, not free. Only worry if `available` is low *and* swap is being used heavily.

**Out-of-memory: a process got killed unexpectedly** — the kernel OOM killer struck. Confirm with `dmesg | grep -i oom` or `journalctl -k | grep -i oom`. Add RAM, add swap, or cap the offending process's memory.

**Disk full but you can't find the files** — a deleted-but-open file (see Disks section) or files hidden under an active mount point. `du -sh /*` then drill down; compare with `df`.

**A resource-monitoring command isn't installed** — `htop`, `iotop`, `iftop`, `iostat`/`mpstat` (from `sysstat`), and `dstat` often need installing. Use built-ins (`top`, `vmstat`, `free`, `ss`) meanwhile.

---

## B. Logs & Journaling

On systemd systems, most logs go to the **journal**; many services also write plain-text logs under `/var/log`.

```bash
# systemd journal
journalctl                     # all logs (oldest first)
journalctl -e                  # jump to the end (newest)
journalctl -f                  # follow live (like tail -f)
journalctl -u nginx            # logs for one service
journalctl -b                  # logs since last boot
journalctl -b -1               # logs from the previous boot
journalctl --since "1 hour ago"
journalctl --since "2024-01-01" --until "2024-01-02"
journalctl -p err              # only error priority and worse
journalctl -k                  # kernel messages (dmesg)
journalctl --disk-usage        # how much space the journal uses
sudo journalctl --vacuum-time=7d   # keep only last 7 days

# Classic text logs
ls /var/log/                   # available log files
tail -f /var/log/syslog        # Debian/Ubuntu general log
tail -f /var/log/messages      # RHEL/CentOS general log
cat /var/log/auth.log          # authentication/login events
dmesg                          # kernel ring buffer
dmesg -w                       # follow kernel messages live
```

Log rotation is handled by **logrotate** (`/etc/logrotate.conf`, `/etc/logrotate.d/`), which compresses and prunes old logs so they don't fill the disk.

### Troubleshooting

**The general "something broke, what happened" workflow** — start broad, then narrow: `journalctl -xe` (recent errors with context) → `journalctl -u <service>` (that service) → `journalctl -b` (this boot) or `journalctl -b -1` (the boot before a crash) → `journalctl -p err` (errors only).

**`journalctl` shows nothing / empty** — persistent logging may be off. Check for `/var/log/journal/`; if missing, create it and restart journald: `sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald`. Without it, logs are memory-only and vanish on reboot.

**Logs filled up the disk** — check usage with `journalctl --disk-usage`, then trim: `sudo journalctl --vacuum-time=7d` or `--vacuum-size=500M`. For text logs, verify logrotate is working (`sudo logrotate -d /etc/logrotate.conf` does a dry run).

**Timestamps look wrong in logs** — the system timezone/clock is off. Check `timedatectl` and enable NTP sync.

**Can't read a log: "Permission denied"** — most logs need root. Use `sudo journalctl`/`sudo cat`. To let a user read the journal without full root, add them to the `systemd-journal` group.

**Need logs from a system that won't boot** — boot a live USB, mount the disk, and read `/var/log/` or point journalctl at it: `journalctl -D /mnt/var/log/journal`.

---