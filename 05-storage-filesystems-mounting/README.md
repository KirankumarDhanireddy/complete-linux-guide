# 5. Storage, Filesystems & Mounting

## A. Disks, Partitions, Filesystems & Volumes (incl. LVM)

**Disks** appear as block devices under `/dev` (e.g. `/dev/sda`, `/dev/nvme0n1`). A disk is divided into **partitions** (`/dev/sda1`), each formatted with a **filesystem** (ext4, xfs, btrfs, vfat).

```bash
lsblk                          # tree of disks and partitions (great overview)
lsblk -f                       # include filesystem type and UUID
fdisk -l                       # list disks and partitions (MBR/GPT)
blkid                          # UUIDs and filesystem types
df -h                          # disk space usage by mounted filesystem
df -i                          # inode usage
du -sh /path                   # size of a directory
du -h --max-depth=1 /var       # per-subdirectory sizes
```

Partitioning tools:

```bash
sudo fdisk /dev/sdb            # interactive partitioning (MBR/GPT)
sudo parted /dev/sdb           # scriptable partitioning, GPT-friendly
sudo cfdisk /dev/sdb           # menu-driven partitioning
```

Creating filesystems (formatting):

```bash
sudo mkfs.ext4 /dev/sdb1       # format as ext4
sudo mkfs.xfs /dev/sdb1        # format as xfs
sudo mkfs.vfat /dev/sdb1       # format as FAT (USB drives)
sudo mkswap /dev/sdb2          # make a swap partition
sudo swapon /dev/sdb2          # enable swap
sudo swapoff /dev/sdb2         # disable swap
```

Checking and repairing:

```bash
sudo fsck /dev/sdb1            # check/repair a filesystem (unmounted)
sudo e2fsck -f /dev/sdb1       # force-check an ext filesystem
sudo tune2fs -l /dev/sdb1      # show ext filesystem parameters
sudo resize2fs /dev/sdb1       # resize an ext filesystem
```

### LVM (Logical Volume Manager)

LVM adds flexibility: physical disks/partitions become **Physical Volumes (PV)**, grouped into a **Volume Group (VG)**, which is sliced into resizable **Logical Volumes (LV)** that you format and mount. You can grow volumes across disks without repartitioning.

```bash
# Physical Volumes
sudo pvcreate /dev/sdb /dev/sdc    # mark disks as PVs
sudo pvs                           # summary of PVs
sudo pvdisplay                     # detailed PV info

# Volume Groups
sudo vgcreate myvg /dev/sdb /dev/sdc   # create VG from PVs
sudo vgextend myvg /dev/sdd            # add a disk to the VG
sudo vgs                               # summary of VGs
sudo vgdisplay                         # detailed VG info

# Logical Volumes
sudo lvcreate -L 20G -n data myvg      # create a 20G LV named 'data'
sudo lvextend -L +10G /dev/myvg/data   # grow the LV by 10G
sudo lvextend -r -L +10G /dev/myvg/data   # grow LV + resize FS together
sudo lvs                               # summary of LVs
sudo lvdisplay                         # detailed LV info

# Format and use the LV
sudo mkfs.ext4 /dev/myvg/data
sudo mount /dev/myvg/data /mnt/data
```

RAID basics (software RAID with `mdadm`):

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat               # RAID status
sudo mdadm --detail /dev/md0   # array details
```

### Troubleshooting

**New disk doesn't show up** — rescan without rebooting: `sudo partprobe`, or `echo 1 | sudo tee /sys/class/block/sdX/device/rescan`. Confirm with `lsblk` and `dmesg | tail`.

**"No space left on device"** — find what's eating it: `df -h` to spot the full filesystem, then `du -sh /path/* | sort -rh | head` to drill in. Common culprits: `/var/log`, old kernels in `/boot`, Docker images, `/tmp`. Remember to also check inodes with `df -i`.

**A deleted large file didn't free space** — a process still holds it open. `lsof | grep deleted` finds it; restart that process to release the space.

**Filesystem is read-only unexpectedly** — the kernel remounted it read-only after detecting corruption. Check `dmesg` for I/O errors, unmount, and run `fsck`. Never `fsck` a mounted filesystem.

**`fsck` won't run: "device is busy"** — the filesystem is mounted. Unmount first; for the root filesystem you must do this from rescue mode or a live USB.

**LVM: "Insufficient free extents" when extending** — the volume group is full. Add a disk (`pvcreate` + `vgextend`) before `lvextend`.

**LVM volume grew but the filesystem is still the old size** — `lvextend` only resizes the container. Grow the filesystem too: `resize2fs /dev/vg/lv` (ext) or `xfs_growfs /mountpoint` (xfs). Using `lvextend -r` does both at once.

**Disk shows SMART warnings / suspected failing drive** — check health with `sudo smartctl -a /dev/sdX` (from `smartmontools`). Back up immediately if reallocated sectors are climbing.

---

## B. Mounting & the `/etc/fstab` File

**Mounting** attaches a filesystem to a directory (the mount point) in the tree.

```bash
sudo mount /dev/sdb1 /mnt/data     # mount a device to a directory
sudo umount /mnt/data              # unmount (by mount point)
sudo umount /dev/sdb1              # unmount (by device)
mount                              # list all current mounts
findmnt                            # tree view of mounts
mount -o remount,rw /              # remount read-write
sudo mount -o ro /dev/sdb1 /mnt    # mount read-only
```

To mount automatically at boot, add an entry to `/etc/fstab`. Each line: `device  mountpoint  fstype  options  dump  pass`.

```
# Prefer UUIDs over /dev names (they don't change)
UUID=1234-abcd  /mnt/data  ext4  defaults  0  2
/dev/myvg/data  /srv       ext4  defaults  0  2
tmpfs           /tmp       tmpfs defaults  0  0
```

```bash
blkid /dev/sdb1                # get the UUID to put in fstab
sudo mount -a                 # mount everything in fstab (test your entries)
```

Common mount options: `defaults`, `noatime` (faster, skip access-time writes), `ro`/`rw`, `nofail` (don't block boot if missing), `user` (allow non-root mount). The last two fields: `dump` (backup flag, usually 0) and `pass` (fsck order — `1` for root, `2` for others, `0` to skip).

### Troubleshooting

**System won't boot after editing `/etc/fstab`** — this is the classic fstab mistake. A bad entry drops you into emergency mode. Remount root writable (`mount -o remount,rw /`), fix or comment out the offending line, then reboot. **Always test new entries with `sudo mount -a` before rebooting** — if it errors, your fstab is wrong.

**Prevent fstab from blocking boot** — add the `nofail` option so a missing device doesn't halt startup (essential for removable/network drives).

**"mount: wrong fs type, bad option, bad superblock"** — usually the filesystem type is wrong or the superblock is damaged. Detect the real type with `blkid` or `lsblk -f`; for a bad superblock, `fsck` may repair it or you can point to a backup superblock.

**"mount point does not exist"** — create the directory first: `sudo mkdir -p /mnt/data`.

**"target is busy" on unmount** — something is using the mount. Find it with `lsof +D /mnt/data` or `fuser -vm /mnt/data`. As a last resort `umount -l` (lazy) detaches it once free.

**Mount works manually but not from fstab** — the `/dev/sdX` name changed between boots. Use `UUID=` (from `blkid`) instead of device names in fstab.

**USB drive mounts read-only** — often an unclean NTFS/exFAT filesystem. Run a filesystem check from another OS, or mount with the appropriate driver (`ntfs-3g`).

---