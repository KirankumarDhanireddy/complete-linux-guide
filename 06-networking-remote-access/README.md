# 6. Networking & Remote Access

## A. Networking

**Interfaces** connect the machine to networks (e.g. `eth0`, `enp3s0`, `wlan0`, `lo` loopback). Each has an IP address, and the system uses a **routing table** and **DNS** to reach other hosts.

```bash
# Interfaces & addresses (modern 'ip' suite)
ip a                           # show all interfaces and IPs
ip addr show                   # same, verbose
ip link                        # link-layer info / interface state
ip link set eth0 up            # bring an interface up
ip link set eth0 down          # bring it down
ip addr add 192.168.1.10/24 dev eth0   # assign an IP temporarily

# Routing
ip route                       # show the routing table
ip route add default via 192.168.1.1   # set default gateway
route -n                       # legacy routing table view

# DNS
cat /etc/resolv.conf           # configured DNS servers
nslookup example.com           # DNS lookup
dig example.com                # detailed DNS query
dig +short example.com         # just the answer
host example.com               # simple lookup
getent hosts example.com       # resolve via system config
cat /etc/hosts                 # static hostname->IP map

# Connectivity & diagnostics
ping example.com               # test reachability (Ctrl+C to stop)
ping -c 4 example.com          # send 4 packets then stop
traceroute example.com         # path to a host
mtr example.com                # live traceroute + ping combined
curl https://example.com       # fetch a URL
curl -I https://example.com    # headers only
wget https://example.com/file  # download a file

# Ports & connections
ss -tuln                       # listening TCP/UDP ports (modern)
ss -tulnp                      # include owning process
netstat -tuln                  # legacy equivalent
lsof -i :80                    # what's using port 80
nc -zv host 22                 # test if a port is open (netcat)

# Bandwidth / config helpers
ifconfig                       # legacy interface config (net-tools)
nmcli device status            # NetworkManager device status
nmcli connection show          # connection profiles
hostname                       # show hostname
sudo hostnamectl set-hostname newname   # set hostname
```

The legacy `net-tools` commands (`ifconfig`, `route`, `netstat`) are being replaced by the `iproute2` suite (`ip`, `ss`). Learn `ip` and `ss` for anything current.

### Troubleshooting

Work outward from the machine — this order isolates the problem fast:

1. **Interface up?** `ip a` — is there an interface with an IP other than `lo`? If `state DOWN`, `sudo ip link set eth0 up`. No IP at all → DHCP issue, try `sudo dhclient eth0` or check NetworkManager.
2. **Can you reach the gateway?** `ip route` shows the default gateway; `ping <gateway-ip>`. Fails → local network/cabling/config problem.
3. **Can you reach the internet by IP?** `ping 8.8.8.8`. Works here but not by name → it's **DNS**.
4. **Does DNS resolve?** `dig example.com` or `nslookup example.com`. Broken → fix `/etc/resolv.conf` or your DNS server.

**"Network is unreachable"** — no route to the destination. Check `ip route`; you may be missing a default gateway (`sudo ip route add default via <gw>`).

**Can ping IPs but names fail** — DNS. Confirm `/etc/resolv.conf` has a valid `nameserver`; test with `dig @8.8.8.8 example.com` to bypass local config.

**"Connection refused" vs "Connection timed out"** — *refused* means the host is reachable but nothing is listening on that port (service down, or wrong port) — check with `ss -tulnp` on the server. *Timed out* means packets are being dropped, usually a **firewall** (local or network) — check `ufw`/`firewalld` and any cloud security groups.

**Port is open locally but unreachable remotely** — the service may be bound to `127.0.0.1` only. Check the listen address in `ss -tulnp`; it should be `0.0.0.0:port` (or a real IP) to accept external connections.

**Intermittent drops / high latency** — use `mtr example.com` to see where along the path packets are lost.

**Changes to IP/route disappear after reboot** — you set them at runtime with `ip`, which isn't persistent. Configure them in NetworkManager (`nmcli`) or your distro's network config files.

---

## B. SSH & Remote Access

**SSH** (Secure Shell) gives an encrypted remote shell and file transfer. Key-based authentication is safer than passwords.

```bash
ssh user@host                  # connect to a remote host
ssh -p 2222 user@host          # non-default port
ssh -i ~/.ssh/mykey user@host  # use a specific private key

# Key generation & distribution
ssh-keygen -t ed25519          # generate a key pair (recommended)
ssh-keygen -t rsa -b 4096      # RSA alternative
ssh-copy-id user@host          # install your public key on the server

# File transfer
scp file.txt user@host:/path/          # copy local -> remote
scp user@host:/path/file.txt .         # copy remote -> local
scp -r dir user@host:/path/            # recursive
rsync -avz dir/ user@host:/path/       # efficient sync (only changes)
sftp user@host                         # interactive secure FTP

# Config & agent
~/.ssh/config                          # per-host settings (aliases, keys)
ssh-agent                              # holds decrypted keys in memory
ssh-add ~/.ssh/mykey                   # add a key to the agent
```

Server config lives in `/etc/ssh/sshd_config` (e.g. disable root login, change port, allow only key auth). Restart with `sudo systemctl restart sshd` after changes.

A sample `~/.ssh/config` entry:

```
Host myserver
    HostName 203.0.113.5
    User alice
    Port 2222
    IdentityFile ~/.ssh/mykey
```

Then just `ssh myserver`.

### Troubleshooting

**First step for any SSH problem** — run with verbose output: `ssh -v user@host` (add more `v`s for detail). It shows exactly where the connection or authentication fails.

**"Connection refused"** — sshd isn't running or is on a different port. On the server: `sudo systemctl status sshd`; check the port in `/etc/ssh/sshd_config`.

**"Connection timed out"** — a firewall is dropping traffic or the host is unreachable. Check the local firewall, cloud security group, and that the host is up.

**"Permission denied (publickey)"** — key auth failed. Confirm your public key is in the server's `~/.ssh/authorized_keys`, and that permissions are strict: `chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys` (SSH refuses loose permissions). Verify you're offering the right key with `ssh -i`.

**"Permission denied (password)" / password not accepted** — the server may have `PasswordAuthentication no` set. Use a key, or enable it in `sshd_config` and `systemctl restart sshd`.

**"REMOTE HOST IDENTIFICATION HAS CHANGED" / host key warning** — the server's key changed (reinstall, IP reuse, or a real concern). If expected, remove the old entry: `ssh-keygen -R hostname`.

**"Too many authentication failures"** — your agent is offering many keys and hitting the limit. Force one key: `ssh -i ~/.ssh/mykey -o IdentitiesOnly=yes user@host`.

**Locked out because you disabled password login before testing keys** — use the cloud provider's web console/serial access to fix `sshd_config`, then restart sshd. Always keep one working session open while changing SSH settings.

---