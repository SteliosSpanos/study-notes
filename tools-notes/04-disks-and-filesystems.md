# Section 4: Disks & Filesystems

## df — Free Space per Filesystem
```bash
df -h                  # human readable
df -hT                 # + filesystem type
df -i                  # inodes — "No space left" with free GB = inodes exhausted
df -h /var             # which FS holds /var and how full
df -x tmpfs -x devtmpfs -h   # hide virtual filesystems
```

## du — What Uses the Space
```bash
du -sh /var/log                        # total of one directory
du -sh /* 2>/dev/null | sort -h        # biggest top-level dirs
du -h --max-depth=1 /var | sort -h     # one level deep
du -xh / --max-depth=2 | sort -h | tail -20   # -x: stay on one filesystem
du -ah /var/log | sort -rh | head -20  # biggest files+dirs
ncdu /                                 # interactive, much nicer
```

## find — Big / Old Files
```bash
find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null
find /var/log -name "*.gz" -mtime +30             # older than 30 days
find /var/log -name "*.gz" -mtime +30 -delete     # ...and delete them (check first!)
find /tmp -type f -atime +7
```

### "Disk full but du doesn't add up"
A deleted file still held open by a process keeps using space.
```bash
sudo lsof +L1                 # open files with link count 0 (deleted)
sudo lsof | grep '(deleted)'
```
Fix: restart the process holding it, or truncate via `/proc`:
```bash
sudo truncate -s 0 /proc/<PID>/fd/<FD>
```
> **Safe log emptying:** use `truncate -s 0 app.log` or `> app.log`, **not** `rm app.log` while the app is writing to it.

---

## lsblk / blkid / fdisk — Block Devices
```bash
lsblk                      # tree of disks, partitions, mountpoints
lsblk -f                   # + filesystem type, UUID, label
sudo blkid                 # UUIDs for /etc/fstab
sudo fdisk -l              # partition tables
sudo parted -l
```

## mount / findmnt / umount
```bash
findmnt                         # mount tree
findmnt /data                   # what is mounted here + options
mount | column -t
sudo mount /dev/sdb1 /mnt/data
sudo mount -o remount,rw /      # root went read-only after errors
sudo umount /mnt/data
sudo umount -l /mnt/data        # lazy unmount if "target is busy"
sudo mount -a                   # mount everything in fstab — TEST after editing fstab
```
`/etc/fstab` line:
```
UUID=1234-abcd  /data  ext4  defaults,nofail  0  2
```
> **Always run `sudo mount -a` after editing `/etc/fstab`.** A broken entry can drop the server into emergency mode at next boot. `nofail` prevents boot from hanging if the disk is missing.

## Creating a Filesystem (new disk)
```bash
sudo mkfs.ext4 /dev/sdb1       # DESTROYS existing data on sdb1
sudo mkfs.xfs /dev/sdb1
sudo mkdir /data && sudo mount /dev/sdb1 /data
```

## Growing a Disk (cloud VM after resizing the volume)
```bash
lsblk                              # see new size
sudo growpart /dev/xvda 1          # grow partition 1 (package cloud-guest-utils)
sudo resize2fs /dev/xvda1          # ext4
sudo xfs_growfs /                  # xfs (takes mountpoint)
```

## LVM Basics
```bash
sudo pvs; sudo vgs; sudo lvs              # physical volumes / volume groups / logical volumes
sudo lvextend -r -L +10G /dev/vg0/data    # -r also resizes the filesystem
sudo lvextend -r -l +100%FREE /dev/vg0/data
```

## Filesystem Health
```bash
dmesg -T | grep -i -E "i/o error|ext4|xfs|nvme"
sudo smartctl -a /dev/sda          # SMART disk health (smartmontools)
sudo fsck -n /dev/sdb1             # check only (-n = no changes). NEVER on a mounted FS
```

---

## Permissions & Ownership Troubleshooting
```bash
ls -la /path
stat file                          # perms, owner, timestamps, inode
namei -l /var/www/html/index.html  # permissions of every dir along the path
sudo -u www-data cat /var/www/html/index.html   # test access as the service user
getfacl file                       # ACLs (a '+' at the end of ls -l perms)
lsattr file                        # immutable flag 'i' blocks even root
```
- Web server "403 / permission denied" is often a **parent directory** missing `x` — `namei -l` shows it immediately.
- SELinux (RHEL): `ls -Z`, `getenforce`, `sudo ausearch -m avc -ts recent`, `restorecon -Rv /path`.

## Archiving & Compression
```bash
tar czf backup.tar.gz /etc          # create gzip
tar cJf backup.tar.xz /etc          # xz (smaller, slower)
tar tzf backup.tar.gz               # list contents
tar xzf backup.tar.gz -C /restore   # extract to dir
zip -r site.zip site/ && unzip site.zip
gzip -k file / gunzip file.gz
zcat / zgrep "error" app.log.1.gz   # read compressed logs directly
```
