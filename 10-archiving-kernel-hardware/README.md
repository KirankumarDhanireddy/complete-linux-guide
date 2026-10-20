# 10. Archiving, Kernel & Hardware

## A. Archiving & Compression

**tar** bundles many files into one archive; compression tools (`gzip`, `bzip2`, `xz`, `zip`) shrink them.

```bash
# tar (the 'flags' spell out the action)
tar -cvf archive.tar files/        # create an (uncompressed) archive
tar -xvf archive.tar               # extract
tar -tvf archive.tar               # list contents without extracting
tar -czvf archive.tar.gz files/    # create gzip-compressed (.tar.gz)
tar -xzvf archive.tar.gz           # extract gzip
tar -cjvf archive.tar.bz2 files/   # bzip2 compression
tar -cJvf archive.tar.xz files/    # xz compression (best ratio)
tar -xvf archive.tar.gz -C /dest   # extract to a specific directory

# Flag meanings: c=create x=extract t=list v=verbose f=file
#                z=gzip j=bzip2 J=xz

# Standalone compression
gzip file                          # -> file.gz (removes original)
gunzip file.gz                     # decompress
bzip2 file                         # -> file.bz2
xz file                            # -> file.xz
zcat file.gz                       # view a .gz without extracting

# zip (cross-platform)
zip archive.zip file1 file2        # create a zip
zip -r archive.zip folder/         # recursive
unzip archive.zip                  # extract
unzip -l archive.zip               # list contents
```

Rule of thumb: `.tar.gz` is fast and universal; `.tar.xz` compresses smaller but slower; use `.zip` when sharing with Windows users.

### Troubleshooting

**"tar: Removing leading '/' from member names"** — this is just a warning; tar strips absolute paths so the archive extracts relative to the current directory (safer). Nothing to fix.

**Not sure which compression an archive uses** — run `file archive.ext`; it identifies gzip/bzip2/xz/zip. Modern `tar -xf archive.*` auto-detects compression, so you rarely need the `z`/`j`/`J` flag when extracting.

**Extraction dumps files all over the current directory** — the archive wasn't made with a top-level folder. Peek first with `tar -tf archive.tar | head`, then extract into a fresh dir with `-C`.

**"unexpected end of file" / corrupted archive** — the download or transfer was incomplete. Re-download and verify with a checksum (`sha256sum`). Partial recovery is sometimes possible with `gzip -d` ignoring errors, but data may be lost.

**Extraction fails with "No space left on device"** — the uncompressed data is far bigger than the archive. Check free space with `df -h` before extracting large archives.

**Permissions/owners wrong after extraction** — only root can restore original ownership. Extract as root, or accept that files become owned by you.

---

## B. Kernel Modules & Hardware

The kernel loads **modules** (drivers) on demand. Hardware is inspected with the `ls*` family of commands.

```bash
lspci                          # PCI devices (GPU, network cards)
lsusb                          # USB devices
lscpu                          # CPU details
lsblk                          # block/storage devices
lshw                           # comprehensive hardware listing (root)
dmidecode                      # BIOS/hardware info from firmware (root)

# Kernel modules
lsmod                          # currently loaded modules
modinfo module_name            # details about a module
sudo modprobe module_name      # load a module (and dependencies)
sudo modprobe -r module_name   # unload a module
sudo insmod file.ko            # insert a module file directly
sudo rmmod module_name         # remove a module
cat /proc/modules              # raw loaded-module list

# Kernel parameters (sysctl)
sysctl -a                      # all tunable kernel parameters
sudo sysctl net.ipv4.ip_forward=1   # set one at runtime
# make permanent in /etc/sysctl.conf or /etc/sysctl.d/
sudo sysctl -p                 # reload sysctl config
```

Device files live in `/dev`; kernel and process info are exposed as virtual files under `/proc` and `/sys` (e.g. `cat /proc/cpuinfo`, `cat /proc/meminfo`).

### Troubleshooting

**A device isn't detected** — check kernel messages as you plug it in: `dmesg -w` (or `sudo dmesg | tail`). Confirm it appears on the bus with `lspci`/`lsusb`. No driver message usually means a missing module or firmware.

**"modprobe: FATAL: Module not found"** — the driver isn't installed for your running kernel, or you upgraded the kernel without rebuilding out-of-tree modules (e.g. NVIDIA). Reinstall the matching kernel headers/driver package and reboot.

**Missing firmware warnings in dmesg** — install the `linux-firmware` package (name varies by distro).

**A module won't unload: "Module is in use"** — something depends on it. `lsmod` shows the "Used by" column; unload the dependents first, or stop the device/service using it.

**Hardware works after boot but a setting resets** — runtime `modprobe`/`sysctl` changes aren't persistent. Add module options to `/etc/modprobe.d/`, autoload via `/etc/modules-load.d/`, and kernel params to `/etc/sysctl.d/`.

**Wrong or flaky driver loading** — blacklist it in `/etc/modprobe.d/blacklist.conf` (`blacklist modulename`) and rebuild the initramfs (`update-initramfs -u` or `dracut -f`).

---