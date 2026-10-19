# 4. Processes & Package Management

## A. Process Management

A **process** is a running program with a **PID** (process ID). Processes can be foreground, background, or daemons.

```bash
ps                             # your processes in this shell
ps aux                         # all processes, detailed
ps -ef                         # all processes, another format
ps aux | grep nginx            # find a specific process
pgrep nginx                    # PIDs matching a name
top                            # live process/resource monitor
htop                           # nicer interactive monitor (install it)
pidof sshd                     # PID of a program by name

# Signals and killing
kill PID                       # send TERM (polite stop)
kill -9 PID                    # send KILL (force stop)
kill -15 PID                   # send TERM explicitly
killall firefox                # kill all by process name
pkill -f "pattern"             # kill by matching command line

# Job control (within a shell)
command &                      # start in background
jobs                           # list background jobs
fg %1                          # bring job 1 to foreground
bg %1                          # resume job 1 in background
Ctrl+Z                         # suspend current foreground job
nohup command &                # keep running after logout
disown                         # detach a job from the shell

# Priority
nice -n 10 command             # start with lower priority (nicer)
renice -n 5 -p PID             # change priority of a running process
```

Common signals: `SIGTERM (15)` graceful stop, `SIGKILL (9)` force kill, `SIGHUP (1)` reload/hangup, `SIGINT (2)` interrupt (Ctrl+C), `SIGSTOP` pause.

### Troubleshooting

**`kill PID` does nothing / process won't die** — it's ignoring `SIGTERM` or stuck. Escalate to `kill -9 PID` (`SIGKILL`, uncatchable). If even that fails, the process is likely in **uninterruptible sleep (state D)** waiting on I/O — check with `ps -o pid,stat,cmd`; you usually can't kill it without fixing the underlying I/O (often a hung disk/NFS mount).

**A process becomes a zombie (state Z)** — it's already dead but its parent hasn't reaped it. You can't kill a zombie directly; kill or restart the *parent* (`ps -o ppid= -p PID` to find it).

**System is slow / high load** — find the culprit with `top` (press `P` to sort by CPU, `M` by memory) or `htop`. Check load average vs core count; investigate I/O wait (`%wa` in top) with `iotop`.

**"fork: retry: Resource temporarily unavailable"** — you've hit a process/thread limit. Check `ulimit -u` and running counts; a runaway process or fork bomb may be responsible.

**Background job died when you closed the terminal** — it received `SIGHUP`. Next time start it with `nohup cmd &` or inside `tmux`/`screen`, or `disown` it after launching.

**Can't find which process holds a port/file** — `sudo ss -tulnp | grep :PORT` for ports, `sudo lsof PATH` or `fuser PATH` for files.

---

## B. Package Management

Software is installed via a package manager tied to your distro family.

**Debian / Ubuntu (APT):**

```bash
sudo apt update                # refresh package lists
sudo apt upgrade               # upgrade installed packages
sudo apt install nginx         # install a package
sudo apt remove nginx          # remove (keep config)
sudo apt purge nginx           # remove including config
sudo apt autoremove            # remove unused dependencies
apt search keyword             # search packages
apt show nginx                 # package details
dpkg -i package.deb            # install a local .deb
dpkg -l                        # list installed packages
dpkg -L nginx                  # files installed by a package
```

**Red Hat / Fedora / CentOS (DNF/YUM):**

```bash
sudo dnf check-update          # check for updates
sudo dnf install nginx         # install
sudo dnf remove nginx          # remove
sudo dnf update                # update everything
dnf search keyword             # search
dnf info nginx                 # details
rpm -qa                        # list installed packages
rpm -ql nginx                  # files installed by a package
rpm -i package.rpm             # install a local .rpm
```

**Other managers you'll meet:** `pacman` (Arch), `zypper` (SUSE), `snap` and `flatpak` (universal/sandboxed packages), `brew` (Homebrew, also on Linux). Language-specific: `pip` (Python), `npm` (Node), `cargo` (Rust), `gem` (Ruby).

```bash
snap install code --classic    # install a snap
flatpak install flathub org.gimp.GIMP
```

### Troubleshooting

**"Could not get lock /var/lib/dpkg/lock"** — another package operation is running (often an automatic update). Wait for it, or find it with `ps aux | grep -E 'apt|dpkg'`. Only if truly stuck: `sudo rm /var/lib/dpkg/lock*` then `sudo dpkg --configure -a`.

**"Unable to locate package" (apt)** — your package lists are stale or the repo isn't enabled. Run `sudo apt update` first. Check spelling with `apt search`.

**Broken dependencies / half-installed packages** — `sudo apt --fix-broken install`, or `sudo dpkg --configure -a`. On dnf: `sudo dnf distro-sync` and `sudo dnf check`.

**"404 Not Found" during apt update** — a repository moved or your release is end-of-life. Check `/etc/apt/sources.list` and files in `/etc/apt/sources.list.d/`.

**GPG / signature errors** — a repo's signing key is missing or expired. Import the correct key; don't blindly disable verification, since it protects against tampered packages.

**Installed a package but the command isn't found** — it may be in a path not in `PATH`, or the binary name differs from the package name. Find installed files with `dpkg -L pkg` (Debian) or `rpm -ql pkg` (RHEL).

**Disk fills up from cached packages** — clear with `sudo apt clean` / `sudo apt autoremove`, or `sudo dnf clean all`.

---