# 3. Users, Permissions & Security

## A. Users, Groups & Permissions

Linux is multi-user. Every user has a **UID**, a primary **group** (GID), and optional secondary groups. Account data lives in `/etc/passwd`, group data in `/etc/group`, and (hashed) passwords in `/etc/shadow`.

```bash
whoami                         # current username
id                             # UID, GID, groups
id username                    # info for another user
who                            # who is logged in
w                              # who is logged in + what they're doing
last                           # recent login history

# Managing users (need sudo/root)
sudo useradd -m -s /bin/bash alice   # create user with home dir + shell
sudo adduser alice                   # interactive, friendlier (Debian)
sudo passwd alice                    # set/change a password
sudo usermod -aG sudo alice          # add alice to the 'sudo' group
sudo usermod -l newname oldname      # rename a user
sudo userdel alice                   # delete user
sudo userdel -r alice                # delete user AND home directory

# Managing groups
sudo groupadd developers             # create group
sudo groupdel developers             # delete group
sudo gpasswd -a alice developers     # add alice to a group
groups alice                         # list a user's groups

# Switching identity
su - alice                           # switch to another user (full login)
sudo command                         # run one command as root
sudo -i                              # interactive root shell
sudo su -                            # become root
```

`/etc/passwd` fields: `username:x:UID:GID:comment:home:shell`. The `x` means the password hash is stored in `/etc/shadow`.

### Troubleshooting

**Added a user to a group but they still can't access the resource** — group membership is only picked up at login. Have them log out and back in, or run `newgrp groupname` in the current shell. Verify with `id` (not just `/etc/group`).

**"user is not in the sudoers file"** — the account isn't allowed sudo. As root, run `usermod -aG sudo alice` (Debian) or `usermod -aG wheel alice` (RHEL), then re-login.

**Forgot the root/user password** — boot into rescue/single-user mode (see the Boot section), remount root read-write, and run `passwd username`.

**Can't create/delete a user: "user is currently used by process"** — that user has running processes or is logged in. `pkill -u alice` to end them, then `userdel`. Add `-f` to force if needed.

**Login works but drops straight back to the prompt** — usually a bad shell or missing home dir. Check the last two `/etc/passwd` fields; fix with `usermod -s /bin/bash user` or recreate the home directory.

**`su` fails with "Authentication failure"** — you're typing the *target* user's password (for `su alice`) or root's password (for `su -`), not your own. Use `sudo -i` if you don't know root's password.

---

## B. File Ownership & Special Permissions

Every file has an **owner (user)**, a **group**, and permission bits for three classes: **user (u)**, **group (g)**, **others (o)**.

Reading an `ls -l` line:

```
-rwxr-xr--  1  alice  devs  4096  Jan 1  file
│└┬┘└┬┘└┬┘     └─┬─┘  └┬─┘
│ u   g  o       owner group
type
```

Permission meanings — for files: `r` read, `w` write, `x` execute. For directories: `r` list contents, `w` create/delete files inside, `x` enter/traverse the directory.

Numeric (octal) values: `r=4`, `w=2`, `x=1`. Sum per class.
- `755` = `rwxr-xr-x` (owner all, others read/execute)
- `644` = `rw-r--r--` (owner read/write, others read)
- `700` = `rwx------` (owner only)
- `777` = everyone everything (rarely appropriate)

```bash
chmod 755 script.sh            # set permissions numerically
chmod u+x script.sh            # add execute for the user (owner)
chmod go-w file                # remove write from group and others
chmod -R 644 /dir              # recursive
chown alice file               # change owner
chown alice:devs file          # change owner and group
chown -R alice:devs /dir       # recursive
chgrp devs file                # change group only
umask                          # show default permission mask
umask 022                      # new files 644, new dirs 755
```

Special permission bits:
- **SetUID (4xxx)** — a program runs with the owner's privileges (e.g. `passwd`). `chmod u+s file`.
- **SetGID (2xxx)** — runs with the group's privileges; on a directory, new files inherit its group. `chmod g+s dir`.
- **Sticky bit (1xxx)** — on a shared directory (like `/tmp`), only the file's owner can delete it. `chmod +t dir`.

```bash
chmod 4755 program             # SetUID + 755
chmod 2775 shareddir           # SetGID + 775
chmod 1777 /shared             # sticky + 777
lsattr file                    # list extended attributes
chattr +i file                 # make file immutable (can't edit/delete)
getfacl file                   # view Access Control Lists
setfacl -m u:bob:rw file       # grant bob read/write via ACL
```

### Troubleshooting

**"Permission denied" even though the file shows `rwx` for you** — check the *whole path*. You need execute (`x`) on every parent directory to reach the file. `namei -l /full/path/to/file` shows permissions at each level.

**A script has `x` set but still won't run** — the shebang line may point to a missing interpreter, or the file has Windows line endings (`\r\n`). Check with `file script.sh`; fix with `dos2unix script.sh` or `sed -i 's/\r$//' script.sh`.

**`chmod`/`chown` says "Operation not permitted" as root** — the file is immutable. Check `lsattr file`; remove the flag with `chattr -i file`.

**Recursively changed permissions and broke things** — `chmod -R 777` or `755` on files *and* directories is a common footgun. Set them separately: `find /dir -type d -exec chmod 755 {} \;` and `find /dir -type f -exec chmod 644 {} \;`.

**A SetUID program stopped working after copy/edit** — copying often drops the SetUID bit and changes ownership. Re-apply `chmod u+s` and `chown` the correct owner.

**Files created in a shared directory keep the wrong group** — set the SetGID bit on the directory (`chmod g+s dir`) so new files inherit the directory's group.

---