# 9. Environment, Bash Scripting & Productivity

## A. Environment Variables & Shell Configuration

**Environment variables** carry configuration to programs (e.g. `PATH`, `HOME`, `LANG`).

```bash
printenv                       # list all environment variables
env                            # same
echo $HOME                     # print one variable
echo $PATH                     # the command search path
export MYVAR="value"           # set a variable for this session + children
MYVAR="value"                  # set for the current shell only
unset MYVAR                    # remove a variable
export PATH="$PATH:/opt/bin"   # add a directory to PATH
```

Key variables: `PATH` (where the shell looks for commands), `HOME`, `USER`, `SHELL`, `PWD`, `LANG`/`LC_*` (locale), `EDITOR`, `TERM`.

Startup files (Bash) — where you make settings permanent:
- `~/.bashrc` — runs for each interactive non-login shell (most common place for aliases and exports).
- `~/.bash_profile` / `~/.profile` — runs at login.
- `/etc/profile`, `/etc/bash.bashrc` — system-wide.

```bash
alias ll='ls -alh'             # create a shortcut
alias gs='git status'
source ~/.bashrc               # reload config without logging out
. ~/.bashrc                    # same (dot = source)
```

### Troubleshooting

**Your alias/export/PATH change doesn't take effect** — a new shell reads the config, but your current one doesn't until you `source ~/.bashrc` (or open a new terminal). Also confirm you edited the right file: interactive shells read `~/.bashrc`; login shells read `~/.bash_profile`/`~/.profile`.

**Changes work in the terminal but not in scripts/cron/GUI apps** — those don't read `~/.bashrc`. Put system-wide variables in `/etc/environment` or `/etc/profile.d/*.sh`, and always use absolute paths in cron.

**Broke your shell — every new terminal errors or won't open** — you have a syntax error in a startup file. Open a shell that skips them: `bash --norc --noprofile`, then fix the file. Check syntax without running with `bash -n ~/.bashrc`.

**Duplicated `PATH` entries** — you appended to `PATH` inside `~/.bashrc`, which runs every time a subshell starts. Guard it or set it in a login file instead. Inspect with `echo $PATH | tr ':' '\n'`.

**A variable is set but a program doesn't see it** — it wasn't exported. `MYVAR=x` is local to the shell; child processes need `export MYVAR=x`.

**`$?` shows non-zero after a command** — that's the exit status of the *last* command; non-zero means it failed. Check it immediately after the command, since the next command overwrites it.

---

## B. Bash Scripting Essentials

A **script** is a text file of commands. Start it with a **shebang** and make it executable.

```bash
#!/bin/bash
# my first script

name="World"                       # variable (no spaces around =)
echo "Hello, $name!"               # use with $

# Read input
read -p "Enter your name: " user
echo "Hi, $user"

# Command substitution
today=$(date +%F)
echo "Today is $today"

# Arguments: $0 script name, $1..$9 args, $# count, $@ all, $? last exit code
echo "First argument: $1"
echo "Number of args: $#"

# Conditionals
if [ "$user" = "root" ]; then
    echo "You are root"
elif [ -f /etc/passwd ]; then
    echo "passwd exists"
else
    echo "Something else"
fi

# Numeric comparison: -eq -ne -lt -le -gt -ge
# String: = != -z (empty) -n (non-empty)
# Files: -f (file) -d (dir) -e (exists) -r/-w/-x (perms)

# Loops
for i in 1 2 3; do
    echo "Number $i"
done

for file in *.txt; do
    echo "Found $file"
done

count=0
while [ $count -lt 5 ]; do
    echo "count=$count"
    count=$((count + 1))          # arithmetic
done

# Case statement
case "$1" in
    start) echo "starting" ;;
    stop)  echo "stopping" ;;
    *)     echo "usage: $0 {start|stop}" ;;
esac

# Functions
greet() {
    echo "Hello, $1"
    return 0
}
greet "Alice"

# Exit status
exit 0                             # 0 = success, non-zero = error
```

Running and debugging:

```bash
chmod +x script.sh             # make executable
./script.sh                    # run it
bash script.sh                 # run without +x
bash -x script.sh              # trace execution (debug)
set -e                         # exit on any error (put near top)
set -u                         # error on undefined variables
```

Handy building blocks: `$(...)` command substitution, `$((...))` arithmetic, `[[ ... ]]` (preferred test in bash), `&&`/`||` for conditional flow, and `trap 'cmd' EXIT` to run cleanup on exit.

### Troubleshooting

**Debug any script** — run it with `bash -x script.sh` to trace each expanded command, or add `set -x` at the top (and `set +x` to stop). Add `set -euo pipefail` near the top of serious scripts: exit on error, error on undefined variables, and catch failures inside pipelines.

**"unexpected end of file" / "unexpected token"** — an unclosed quote, `if`/`fi`, `do`/`done`, or brace. Check syntax without running: `bash -n script.sh`. Watch for Windows line endings (`\r`), which cause bizarre errors — fix with `dos2unix` or `sed -i 's/\r$//'`.

**"[: too many arguments" or "unary operator expected"** — an unquoted variable that's empty or contains spaces. Always quote: `[ "$var" = "x" ]`, and prefer `[[ ... ]]` in bash, which is more forgiving.

**"command not found" for a variable assignment** — you put spaces around `=`. It must be `name=value`, no spaces.

**Variable set inside a loop is empty afterward** — a `while ... | read` loop runs in a subshell, so changes don't persist. Use process substitution instead: `while read x; do ...; done < <(command)`.

**Comparing numbers gives wrong results** — `[ "$a" -gt "$b" ]` for integers; `>`/`<` inside `[ ]` do *string* comparison. For arithmetic use `(( a > b ))`.

**Script works when you run it but not from cron/another dir** — it relies on the current directory or environment. Use absolute paths, and `cd` to a known location at the start.

---

## C. Security Basics (Firewall, sudo, SELinux/AppArmor)

**Firewall** — controls inbound/outbound traffic. The kernel layer is `netfilter`; front-ends differ by distro.

```bash
# UFW (Ubuntu's simple firewall)
sudo ufw status                # current rules
sudo ufw enable                # turn on the firewall
sudo ufw allow 22              # allow SSH
sudo ufw allow 80/tcp          # allow HTTP
sudo ufw deny 23               # block a port
sudo ufw delete allow 80       # remove a rule

# firewalld (RHEL/Fedora)
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload

# iptables / nftables (low-level)
sudo iptables -L -n            # list rules
sudo nft list ruleset          # nftables (modern replacement)
```

**sudo** — grants controlled root access. Configuration is in `/etc/sudoers` (edit only with `visudo`) and `/etc/sudoers.d/`.

```bash
sudo command                   # run a single command as root
sudo -l                        # what am I allowed to run?
sudo visudo                    # safely edit sudoers
```

**Mandatory Access Control** — an extra layer beyond file permissions:
- **SELinux** (RHEL/Fedora): `getenforce`, `sudo setenforce 0` (permissive), `sestatus`, `ls -Z` (contexts).
- **AppArmor** (Ubuntu/SUSE): `sudo aa-status`, profiles in `/etc/apparmor.d/`.

Good habits: keep the system updated, use SSH keys not passwords, disable root SSH login, run services as non-root users, open only the ports you need, and check `journalctl`/`/var/log/auth.log` for suspicious logins.

### Troubleshooting

**Blocked yourself out with a firewall rule** — this is why you *always* keep a second session open when changing firewall/SSH rules. If locked out remotely, use the cloud console/serial access to disable the firewall (`sudo ufw disable` / `sudo systemctl stop firewalld`), fix the rule, re-enable.

**Traffic still blocked after adding an "allow" rule** — rule order matters; an earlier deny may win. List rules with numbers (`sudo ufw status numbered`, `sudo iptables -L -n --line-numbers`) and check ordering. With firewalld, remember `--permanent` changes need `--reload`.

**Everything's permitted but connections still fail** — check for a *second* layer: cloud security groups, a network firewall, or the service binding only to localhost (see Networking).

**A program mysteriously fails to read a file it has permissions for** — SELinux/AppArmor may be blocking it, not classic permissions. Check `getenforce`; look for `AVC` denials in `journalctl` or `/var/log/audit/audit.log`. Test by temporarily setting permissive mode (`sudo setenforce 0`) — if it works, the fix is a proper SELinux context (`restorecon`, `chcon`, or a policy), **not** leaving SELinux off.

**"sudo: command not found" or sudo broken** — the sudoers file has a syntax error (only ever edit it with `visudo`, which validates). Recover via root: `pkexec visudo` or boot to single-user mode.

**Spot suspicious activity** — failed logins: `grep -i failed /var/log/auth.log` or `journalctl -u sshd | grep -i fail`; current logins: `who`, `last`; listening services you didn't expect: `ss -tulnp`.

---

## D. Handy One-Liners & Keyboard Shortcuts

Everyday one-liners:

```bash
# Disk hogs: biggest items in current directory
du -ah . | sort -rh | head -20

# What's listening on which port + process
sudo ss -tulnp

# Find and kill a process by name
pkill -f "python myapp.py"

# Watch a command update every 2 seconds
watch -n 2 'df -h'

# Count files in a directory
ls -1 | wc -l

# Replace text across many files
grep -rl "oldtext" . | xargs sed -i 's/oldtext/newtext/g'

# Free up page cache (careful; usually unnecessary)
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

# Show the 10 largest files under /var
find /var -type f -printf '%s %p\n' | sort -rn | head

# Quick HTTP server in the current folder (Python 3)
python3 -m http.server 8000

# Test a remote port quickly
nc -zv example.com 443
```

Terminal shortcuts worth memorizing: `Tab` complete, `Ctrl+R` search history, `Ctrl+C` cancel, `Ctrl+Z` suspend, `Ctrl+A`/`Ctrl+E` line start/end, `Ctrl+U`/`Ctrl+K` delete to start/end, `Ctrl+L` clear, `!!` last command, `!$` last argument of last command, `sudo !!` re-run last command as root.

---