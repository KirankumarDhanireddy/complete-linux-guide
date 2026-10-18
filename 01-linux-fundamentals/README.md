<h1 align="center">Linux Fundamentals & System Structure</h1>

## A. What Linux Is & How It's Structured

Linux is a family of open-source operating systems built around the **Linux kernel**. The kernel manages hardware (CPU, memory, devices) and exposes it to programs. Around the kernel sits userspace: shells, utilities, libraries, and applications. A **distribution** (distro) bundles the kernel with a package manager and default tools — e.g. Ubuntu/Debian, Fedora/RHEL/CentOS, Arch, SUSE.

Key layers, from bottom to top: hardware → kernel → system libraries (glibc) → shell and utilities → applications.

Two big distro families to know because commands differ:
- **Debian/Ubuntu** family → uses `apt`, `.deb` packages.
- **Red Hat/Fedora** family → uses `dnf`/`yum`, `.rpm` packages.

Useful "what am I running" commands:

```bash
uname -a               # kernel name, version, architecture
uname -r               # kernel release only
cat /etc/os-release    # distro name and version
hostnamectl            # host, OS, kernel, virtualization info
lsb_release -a         # distro details (if available)
arch                   # CPU architecture (x86_64, aarch64, ...)
uptime                 # how long the system has been up + load
whoami                 # current username
id                     # current user's UID, GID, groups
```

---

## B. The Shell & Terminal Basics

The **shell** is the program that interprets your typed commands. The most common is **Bash** (`/bin/bash`); others include `zsh`, `sh`, `fish`. A **terminal** is the window/app that hosts the shell.

Anatomy of a command:

```
command  -options  arguments
  ls      -l -a     /home
```

- Short options: `-l`, combinable as `-la`.
- Long options: `--all`, `--help`.

Essential navigation and help commands:

```bash
pwd                    # print working directory (where am I)
cd /path/to/dir        # change directory
cd ~                   # go to home directory
cd -                   # go to previous directory
cd ..                  # go up one level
ls                     # list files
ls -l                  # long listing (permissions, owner, size, date)
ls -a                  # include hidden files (dotfiles)
ls -lh                 # human-readable sizes
ls -lt                 # sort by modification time
clear                  # clear the screen (Ctrl+L)
history                # command history
!!                     # repeat last command
!123                   # run command number 123 from history
man ls                 # manual page for a command
ls --help              # quick help
type ls                # what kind of command it is (alias/builtin/binary)
which python3          # path of an executable
whatis grep            # one-line description
apropos network        # search man pages by keyword
```

Command line editing shortcuts (work in Bash): `Ctrl+A` start of line, `Ctrl+E` end of line, `Ctrl+U` delete to start, `Ctrl+K` delete to end, `Ctrl+R` reverse-search history, `Tab` autocomplete, `Ctrl+C` cancel, `Ctrl+D` end of input/logout.

### Troubleshooting

**"command not found"** — the shell can't locate the binary. Check it's installed (`which cmd`, `type cmd`), confirm it's in your `PATH` (`echo $PATH`), and if it's a local script use `./script` not just `script`. After installing something, run `hash -r` to clear the shell's cached command locations.

**"Permission denied" when running a script** — it lacks the execute bit: `chmod +x script.sh`. If it's `Permission denied` on a system path, you likely need `sudo`.

**A command hangs and won't respond** — `Ctrl+C` interrupts it; `Ctrl+Z` suspends it (then `kill %1`). If your whole terminal freezes after a file dump, run `reset` to restore it.

**Terminal shows garbage characters** — you `cat`-ed a binary file. Type `reset` (even blind) and press Enter.

**Tab completion or history stopped working** — your shell config may be broken; check with `bash -n ~/.bashrc` for syntax errors, then `source ~/.bashrc`.

---

## C. Filesystem Hierarchy (Where Things Live)

Linux uses a single tree rooted at `/`. There are no drive letters; devices are mounted into the tree.

| Path | Purpose |
|------|---------|
| `/` | Root of everything |
| `/bin`, `/usr/bin` | Essential user command binaries |
| `/sbin`, `/usr/sbin` | System/admin binaries |
| `/etc` | System-wide configuration files |
| `/home` | Regular users' home directories |
| `/root` | Root user's home directory |
| `/var` | Variable data: logs, mail, spool, caches |
| `/var/log` | Log files |
| `/tmp` | Temporary files (often cleared on reboot) |
| `/usr` | User programs, libraries, docs |
| `/lib`, `/usr/lib` | Shared libraries |
| `/opt` | Optional/third-party software |
| `/dev` | Device files (disks, terminals) |
| `/proc` | Virtual filesystem: kernel & process info |
| `/sys` | Virtual filesystem: hardware/kernel objects |
| `/mnt`, `/media` | Mount points for external/removable media |
| `/boot` | Kernel and bootloader files |
| `/srv` | Data served by the system (web, ftp) |

```bash
ls /                   # see the top-level directories
tree /etc              # tree view (install 'tree' if missing)
du -sh /var/log        # size of a directory
file /bin/ls           # identify a file's type
stat file.txt          # detailed metadata (size, inode, timestamps)
```

---