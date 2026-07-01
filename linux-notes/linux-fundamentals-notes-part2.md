# Linux Fundamentals (Part 2)

## Backup and Restore

**Tools:** Rsync, Duplicity (Rsync + encryption), Deja Dup (GUI front-end, uses Rsync, supports encryption).

| Tool | Description |
|---|---|
| `rsync` | Fast sync — only transfers changed portions of files; local or remote |
| `duplicity` | Rsync + encryption, can target FTP/cloud (e.g. S3) |
| `deja dup` | GUI wrapper, encrypted backups, beginner-friendly |

Additional encryption options for backups: GnuPG, eCryptfs, LUKS.

```bash
sudo apt install rsync -y
```

```bash
# Basic backup to remote host (-a archive: preserves perms/timestamps, -v verbose)
rsync -av /path/to/mydirectory user@backup_server:/path/to/backup/directory

# Compressed, incremental, with deletion sync
rsync -avz --backup --backup-dir=/path/to/backup/folder --delete /path/to/mydirectory user@backup_server:/path/to/backup/directory

# Restore
rsync -av user@remote_host:/path/to/backup/directory /path/to/mydirectory

# Force rsync over SSH (encrypted transfer)
rsync -avz -e ssh /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

**Flags:** `-a` archive (preserve attributes), `-v` verbose, `-z` compress, `--backup`/`--backup-dir` keep incremental copies, `--delete` remove files on dest no longer in source, `-e ssh` transport over SSH.

### Auto-sync with cron + rsync

```bash
ssh-keygen -t rsa -b 2048          # generate key pair (no password prompt later)
ssh-copy-id user@backup_server     # copy public key to remote
```

`RSYNC_Backup.sh`:
```bash
#!/bin/bash
rsync -avz -e ssh /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

```bash
chmod +x RSYNC_Backup.sh
crontab -e
```

Crontab entry (run hourly at minute 0):
```
0 * * * * /path/to/RSYNC_Backup.sh
```

---

## File System Management

**Common Linux file systems:**

| FS | Notes |
|---|---|
| ext2 | No journaling; low overhead (USB drives) |
| ext3/ext4 | Journaling; ext4 is the modern default (performance + reliability + large file support) |
| Btrfs | Snapshotting, built-in data integrity checks |
| XFS | High performance, excels with large files / high I/O |
| NTFS | Windows-native; useful for dual-boot/cross-compat |

### Inodes
An **inode** stores a file's metadata (permissions, owner, size, timestamps) and pointers to its data blocks — but not the filename or the data itself. The **inode table** is the kernel's master index of all files/dirs. A disk can run out of inodes before running out of raw storage space.

```bash
ls -il    # show inode numbers alongside listing
```

### File types
- **Regular files** — text/binary data, anywhere in the tree.
- **Directories** — containers for files/other directories; the containing dir is the "parent directory."
- **Symbolic links (symlinks)** — pointer/shortcut to another file or directory, avoids duplication.

### Disks & Partitioning

```bash
sudo fdisk -l       # list disks/partitions
```
Tools: `fdisk`, `gpart`, `GParted`.

### Mounting

Mounting = attaching a partition/drive to a directory (mount point) so its contents become accessible there.

```bash
mount                          # list currently mounted filesystems
sudo mount /dev/sdb1 /mnt/usb  # mount a device to a mount point
sudo umount /mnt/usb           # unmount
lsof | grep <user>             # check for open files/processes blocking unmount
```

**`/etc/fstab`** — defines filesystems to auto-mount at boot:
```
<file system>           <mount point>  <type>  <options>          <dump>  <pass>
/dev/sda1               /              ext4    defaults           0       1
/dev/sda2               /home          ext4    defaults           0       2
/dev/sdb1               /mnt/usb       ext4    rw,noauto,user      0       0
192.168.1.100:/nfs      /mnt/nfs       nfs     defaults           0       0
```
- `noauto` — prevents auto-mount at boot (still mountable manually).
- Use `blkid` to get a device's UUID for more robust fstab entries.
- All mounted filesystems (fstab or manual) are cleanly unmounted automatically on shutdown.

### Swap

Swap = disk space used as overflow when RAM is full; kernel moves inactive memory pages there.

```bash
mkswap /dev/sdXn    # prepare a device/file as swap space
swapon /dev/sdXn    # activate it
```

- Size depends on RAM and workload; less critical on systems with abundant RAM.
- Best practice: dedicated partition/file, separate from main filesystem (avoids fragmentation); consider encrypting swap (sensitive data can be paged there).
- Also used for **hibernation** — system state is saved to swap and restored on power-on.

---

## Containerization

Containers package an app + dependencies into a lightweight, isolated, portable unit. Unlike VMs, **containers share the host kernel** — much lighter weight than full virtualization. Tools: Docker, Docker Compose, LXC.

**Security note:** containers offer less isolation than VMs. Misconfigurations can lead to privilege escalation or container escape.

### Docker

**Install (Ubuntu):**
```bash
sudo apt update -y
sudo apt install ca-certificates curl gnupg lsb-release -y
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update -y
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker <user>     # add user to docker group (relogin required)
docker run hello-world             # test
```

- **Docker Hub** — public/private registry of pre-built images.
- **Dockerfile** — instructions to build an image (`FROM`, `RUN`, `EXPOSE`, `CMD`, etc).

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y apache2 openssh-server && rm -rf /var/lib/apt/lists/*
RUN useradd -m docker-user && echo "docker-user:password" | chpasswd
RUN chown -R docker-user:docker-user /var/www/html /var/run/apache2 /var/log/apache2 /var/lock/apache2 && \
    usermod -aG sudo docker-user && \
    echo "docker-user ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
EXPOSE 22 80
CMD service ssh start && /usr/sbin/apache2ctl -D FOREGROUND
```

```bash
docker build -t FS_docker .                       # build image from Dockerfile in cwd
docker run -p <host_port>:<container_port> -d <image>
docker run -p 8022:22 -p 8080:80 -d FS_docker      # example: map SSH+HTTP
```

**Management commands:**

| Command | Description |
|---|---|
| `docker ps` | List running containers |
| `docker stop <id>` | Stop a container |
| `docker start <id>` | Start a stopped container |
| `docker restart <id>` | Restart a container |
| `docker rm <id>` | Remove a container |
| `docker rmi <image>` | Remove an image |
| `docker logs <id>` | View container logs |

**Key facts:**
- Containers are **stateless** — changes inside a running container are lost when it stops/is removed unless you commit a new image or use **volumes** to persist data.
- To preserve changes, write a new Dockerfile (starting `FROM` the base) and rebuild with a new tag.
- At scale, use Docker Compose or Kubernetes for orchestration.

### LXC (Linux Containers)

System-level containerization using cgroups + namespaces; more manual setup than Docker, less portable, but powerful.

```bash
sudo apt install lxc -y
sudo lxc-create -n linuxcontainer -t ubuntu
```

| Command | Description |
|---|---|
| `lxc-ls` | List containers |
| `lxc-start -n <name>` | Start |
| `lxc-stop -n <name>` | Stop |
| `lxc-restart -n <name>` | Restart |
| `lxc-attach -n <name>` | Connect/attach to a container |
| `lxc-config -n <name> -s storage\|network\|security` | Manage container settings |

**Docker vs LXC:**

| Category | Docker | LXC |
|---|---|---|
| Approach | Application-focused | System/VM-like, traditional |
| Image building | Standardized image format | Manual rootfs setup |
| Portability | High (Docker Hub etc.) | Lower, tied to host config |
| Ease of use | Simple CLI, big community | More sysadmin knowledge needed |
| Security | More isolation by default (AppArmor/SELinux, read-only FS) | Needs more manual hardening |

**Resource limiting (cgroups) — example config** at `/usr/share/lxc/config/<name>.conf`:
```
lxc.cgroup.cpu.shares = 512
lxc.cgroup.memory.limit_in_bytes = 512M
```
- `cpu.shares` default 1024; 512 = half the CPU share relative to other containers.
- `memory.limit_in_bytes` accepts K/M/G/T suffixes.

```bash
sudo systemctl restart lxc.service   # apply config changes
```

**Namespaces** isolate: PID space, network interfaces/routing/firewall rules (`net`), and root filesystem (`mnt`) per-container from the host. Namespaces ≠ complete security — pair with other hardening.

---

## Network Configuration

### Key protocols to know
TCP/IP, DNS, DHCP, FTP.

### Network Access Control (NAC) models

| Model | Description |
|---|---|
| DAC (Discretionary) | Resource owner sets permissions |
| MAC (Mandatory) | OS enforces policy, not the owner — more secure, less flexible |
| RBAC (Role-Based) | Permissions tied to organizational roles |

Linux NAC enforcement tools: **SELinux**, **AppArmor**, **TCP wrappers**.

### Interface configuration

```bash
ifconfig            # legacy, still common — view interfaces (deprecated in favor of ip)
ip addr             # modern equivalent

sudo ifconfig eth0 up               # OR
sudo ip link set eth0 up            # activate interface

sudo ifconfig eth0 192.168.1.2                  # assign IP
sudo ifconfig eth0 netmask 255.255.255.0         # assign netmask
sudo route add default gw 192.168.1.1 eth0       # set default gateway
```

**DNS** — `/etc/resolv.conf`:
```
nameserver 8.8.8.8
nameserver 8.8.4.4
```
⚠️ Changes here are **not persistent** — may be overwritten by NetworkManager/systemd-resolved. Use the proper network manager config for permanent changes.

**Persistent interface config** — `/etc/network/interfaces`:
```
auto eth0
iface eth0 inet static
  address 192.168.1.2
  netmask 255.255.255.0
  gateway 192.168.1.1
  dns-nameservers 8.8.8.8 8.8.4.4
```

```bash
sudo systemctl restart networking   # apply changes
```

### NAC technologies (detail)
- **DAC** — owner-controlled permissions (r/w/x/delete).
- **MAC** — security labels on resources + clearances on users/processes; access only if clearance ≥ label. Used in high-security contexts (military, gov, finance, healthcare).
- **RBAC** — permissions tied to roles, not identity; scales well for large orgs.

### Monitoring tools
`syslog`, `rsyslog`, `ss` (socket stats), `lsof` (open files), ELK stack (Elasticsearch, Logstash, Kibana).

### Troubleshooting tools

```bash
ping <host>            # connectivity test (ICMP)
traceroute <host>      # path/hop tracing
netstat -a              # active connections + listening ports
```

Common issues: connectivity failures, DNS resolution failures, packet loss, performance degradation.
Common causes: firewall/router misconfig, bad cabling, wrong network settings, hardware failure, DNS misconfig/outage, congestion, outdated hardware, unpatched software.

Other relevant tools: Tcpdump, Wireshark, Nmap.

### Hardening mechanisms

| Tool | Type | Notes |
|---|---|---|
| SELinux | MAC, kernel-integrated | Fine-grained, complex to configure |
| AppArmor | MAC, LSM (profile-based) | Simpler, less granular than SELinux |
| TCP Wrappers | Host-based NAC | Filters service access by client IP; no broader resource control |

---

## Remote Desktop Protocols

| Protocol | Platform | Notes |
|---|---|---|
| RDP | Primarily Windows | Graphical remote desktop access |
| VNC | Cross-platform, common on Linux | Graphical remote access, RFB protocol |
| X11/XServer | Unix/Linux native | Network-transparent GUI protocol; rendering happens locally, not on the remote host |
| XDMCP | Unix/Linux | Manages remote X sessions over UDP/177; insecure |

### X11 / XServer
- Ports: TCP 6000 (display :0), range ~6001–6009 for additional displays.
- Renders locally (lower remote load/traffic vs VNC/RDP, which render remotely and stream pixels).
- **Unencrypted by default** — major security weakness. Secure via SSH tunneling (X11 forwarding).

```bash
# Enable on server: /etc/ssh/sshd_config
X11Forwarding yes
```
```bash
ssh -X htb-student@10.129.23.11 /usr/bin/firefox   # run remote GUI app via SSH tunnel
```

**Security risks:** unencrypted traffic over TCP 6000–6010 can be sniffed; tools like `xwd`/`xgrabsc` can capture window contents on an exposed X server without traditional sniffing. Historical CVEs: CVE-2017-2624/2625/2626 (XOrg Server, weak session keys → arbitrary code execution).

### XDMCP
- UDP port 177.
- Insecure — vulnerable to MITM attacks (impersonation, command execution, data theft).

### VNC
- Encrypts data in transit, requires auth — generally considered secure.
- Default port: TCP 5900 (display 0); additional displays use 590x (5901, 5902...).
- Common implementations: TigerVNC, TightVNC, RealVNC, UltraVNC.

**TigerVNC setup (with XFCE):**
```bash
sudo apt install xfce4 xfce4-goodies tigervnc-standalone-server -y
vncpasswd
```

```bash
touch ~/.vnc/xstartup ~/.vnc/config
cat <<EOT >> ~/.vnc/xstartup
#!/bin/bash
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
/usr/bin/startxfce4
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
x-window-manager &
EOT

cat <<EOT >> ~/.vnc/config
geometry=1920x1080
dpi=96
EOT

chmod +x ~/.vnc/xstartup
```

```bash
vncserver            # start VNC server
vncserver -list      # list sessions (display, port, PID)
```

**Secure VNC via SSH tunnel:**
```bash
ssh -L 5901:127.0.0.1:5901 -N -f -l htb-student 10.129.14.130
xtightvncviewer localhost:5901
```
