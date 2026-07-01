# Linux Fundamentals

## Navigation

| Command | Description |
|---|---|
| `pwd` | Print working directory |
| `ls` | List directory contents |
| `ls -l` | Long listing (perms, links, owner, group, size, date, name) |
| `ls -la` | Long listing including hidden files (dotfiles, e.g. `.bashrc`) |
| `ls -l /path` | List contents of a path without `cd`-ing into it |
| `cd <dir>` | Change directory |
| `cd /full/path` | Jump directly via absolute path |
| `cd -` | Return to previous directory |
| `cd ..` | Move to parent directory |
| `clear` / `Ctrl+L` | Clear terminal |

**`ls -l` column breakdown:**
`drwxr-xr-x 2 cry0l1t3 htbacademy 4096 Nov 13 17:37 Desktop`

| Field | Meaning |
|---|---|
| `drwxr-xr-x` | Type + permissions |
| `2` | Number of hard links |
| `cry0l1t3` | Owner |
| `htbacademy` | Group owner |
| `4096` | Size in bytes / blocks used |
| `Nov 13 17:37` | Last modified date/time |
| `Desktop` | Name |

**Notes:**
- `.` = current directory, `..` = parent directory.
- Total blocks shown at top of `ls -l` output = total size (blocks × 1024 bytes).
- Tab-complete: press `[TAB]` twice to list ambiguous completions.
- `Ctrl+R` — reverse search command history.
- `↑` / `↓` — scroll through command history.

---

## Working with Files and Directories

| Command | Description |
|---|---|
| `touch <name>` | Create empty file |
| `mkdir <name>` | Create directory |
| `mkdir -p a/b/c` | Create nested directories (parents as needed) |
| `tree .` | Show directory structure as a tree |
| `mv <src> <dst>` | Move or rename file/directory |
| `mv file1 file2 dir/` | Move multiple files into a directory |
| `cp <src> <dst>` | Copy file/directory |

**Examples:**
```bash
touch info.txt
mkdir Storage
mkdir -p Storage/local/user/documents
touch ./Storage/local/user/userinfo.txt
mv info.txt information.txt              # rename
mv information.txt readme.txt Storage/   # move multiple into dir
cp Storage/readme.txt Storage/local/      # copy
```

---

## Find Files and Directories

| Tool | Description |
|---|---|
| `which <prog>` | Path to executable (e.g. `which python` → `/usr/bin/python`); empty if not found |
| `find <location> <options>` | Search filesystem with filters |
| `locate <pattern>` | Fast search using a prebuilt database (fewer filter options than `find`) |
| `sudo updatedb` | Refresh the `locate` database |

**`find` options:**

| Option | Meaning |
|---|---|
| `-type f` | Files only (`d` for directories) |
| `-name *.conf` | Match by name/pattern |
| `-user root` | Owned by user |
| `-size +20k` | Larger than 20 KiB |
| `-newermt 2020-03-03` | Modified after date |
| `-exec ls -al {} \;` | Run a command on each result (`{}` = placeholder, `\;` escapes the semicolon) |
| `2>/dev/null` | Suppress errors (shell redirection, not a `find` option) |

```bash
find / -type f -name *.conf -user root -size +20k -newermt 2020-03-03 -exec ls -al {} \; 2>/dev/null
```

---

## File Descriptors and Redirections

**Default file descriptors:**

| FD | Stream |
|---|---|
| 0 | STDIN |
| 1 | STDOUT |
| 2 | STDERR |

| Syntax | Description |
|---|---|
| `cmd > file` | Redirect STDOUT to file (overwrite) |
| `cmd >> file` | Redirect STDOUT, append |
| `cmd 2> file` | Redirect STDERR to file |
| `cmd 2>/dev/null` | Discard STDERR |
| `cmd 2> err.txt 1> out.txt` | Separate STDOUT/STDERR to different files |
| `cmd < file` | Use file as STDIN |
| `cmd << EOF ... EOF` | Heredoc — STDIN stream terminated by EOF marker |
| `cmd1 \| cmd2` | Pipe — STDOUT of cmd1 becomes STDIN of cmd2 |

```bash
find /etc/ -name shadow 2>/dev/null                  # hide errors
find /etc/ -name shadow 2>/dev/null > results.txt    # STDOUT to file
find /etc/ -name shadow 2> stderr.txt 1> stdout.txt   # split streams
cat < stdout.txt                                      # file as STDIN
find /etc/ -name passwd >> stdout.txt 2>/dev/null     # append
cat << EOF > stream.txt                                # heredoc to file
find /etc/ -name *.conf 2>/dev/null | grep systemd | wc -l   # pipe chain
```

`/dev/null` = "null device," discards anything written to it.

---

## Filter Contents

| Tool | Description |
|---|---|
| `more <file>` | Pager; view one screen at a time. `Q` quits, output stays in terminal |
| `less <file>` | Pager with more features than `more`; `Q` quits, output does **not** stay in terminal |
| `head <file>` | First 10 lines (default) |
| `tail <file>` | Last 10 lines (default) |
| `sort` | Sort lines alphabetically/numerically |
| `grep "pattern"` | Filter lines matching pattern |
| `grep -v "pattern"` | Exclude lines matching pattern |
| `cut -d":" -f1` | Split on delimiter `:`, print field 1 |
| `tr ":" " "` | Translate/replace characters |
| `column -t` | Format input into aligned table columns |
| `awk '{print $1, $NF}'` | Print specific fields ($1 = first, $NF = last) |
| `sed 's/old/new/g'` | Stream editor — substitute text (regex), `g` = replace all matches |
| `wc -l` | Count lines |

**Chaining example (build-up):**
```bash
cat /etc/passwd | more
cat /etc/passwd | sort
cat /etc/passwd | grep "/bin/bash"
cat /etc/passwd | grep -v "false\|nologin"
cat /etc/passwd | grep -v "false\|nologin" | cut -d":" -f1
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " "
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | column -t
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}'
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}' | sed 's/bin/HTB/g'
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}' | wc -l
```

**Reference:** `man <tool>`, `<tool> -h` / `<tool> --help`

---

## Permission Management

**Permission types:** `r` (read), `w` (write), `x` (execute) — assigned per owner, group, others.

- **Execute (`x`) on a directory** = required to traverse/enter it (`cd` into it), independent of read/write. Without it → "Permission Denied," even if contents are listed.
- **Read (`r`) on a directory** = can list contents (`ls`).
- **Write (`w`) on a directory** = can create/delete/rename files inside it.
- **Execute (`x`) on a file** = required to run that file.

**`ls -l` permission string layout:**
```
-rwxrw-r--  1 root root 1641 May 4 23:42 /etc/passwd
 |||||||||
 |||||||+-- others: read
 ||||+----- group: read, write
 |+-------- owner: read, write, execute
 +--------- file type (- file, d dir, l link)
```

### chmod — change permissions

```bash
chmod a+r shell        # add read for all (u=owner, g=group, o=others, a=all; + add, - remove)
chmod 754 shell         # octal: owner=7(rwx) group=5(r-x) other=4(r--)
```

**Octal values:** r=4, w=2, x=1 — sum the bits you want per category (owner/group/other).

| Octal | Binary | Perms |
|---|---|---|
| 7 | 111 | rwx |
| 5 | 101 | r-x |
| 4 | 100 | r-- |

### chown — change owner/group
```bash
chown <user>:<group> <file/directory>
chown root:root shell
```

### Special permission bits

| Bit | Indicator | Effect |
|---|---|---|
| SUID | `s` instead of `x` in owner's slot | File runs with **owner's** privileges, regardless of who executes it |
| SGID | `s` instead of `x` in group's slot | File runs with **group's** privileges |
| Sticky bit | `t`/`T` instead of `x` in others' slot (on directories) | Only file owner, directory owner, or root can delete/rename files inside — others can still use the directory |

- Lowercase `t` = sticky bit set **and** execute permission set for others.
- Uppercase `T` = sticky bit set but **no** execute permission for others (others can't traverse/list contents).
- SUID/SGID misconfigurations are a major privilege-escalation vector (see GTFOBins) — e.g. SUID on a binary that can spawn a shell (like `journalctl`) lets any user get a root shell.

---

## Package Management

Packages bundle binaries, configs, dependency info, and update tracking. Common formats: `.deb` (Debian-based), `.rpm` (Red Hat-based).

| Tool | Description |
|---|---|
| `dpkg` | Low-level installer for `.deb` files |
| `apt` | High-level front-end for package management (search, install, remove, dependency resolution) |
| `aptitude` | Alternative high-level interface to the package manager |
| `snap` | Sandboxed app packages |
| `gem` | Ruby package manager |
| `pip` | Python package installer |
| `git` | Version control / repo cloning |

**Repository config:** `/etc/apt/sources.list` and `/etc/apt/sources.list.d/*.list` define repo sources (stable/testing/unstable).

```bash
apt-cache search impacket          # search package cache
apt-cache show impacket-scripts    # show package details
apt list --installed               # list installed packages
sudo apt install impacket-scripts -y   # install
```

**Git:**
```bash
mkdir ~/nishang/ && git clone https://github.com/samratashok/nishang.git ~/nishang
```

**Manual .deb install via dpkg:**
```bash
wget http://archive.ubuntu.com/ubuntu/pool/main/s/strace/strace_4.21-1ubuntu1_amd64.deb
sudo dpkg -i strace_4.21-1ubuntu1_amd64.deb
strace -h   # verify
```

---

## Service and Process Management

**Services (daemons)** run in the background; names often end in `d` (e.g. `sshd`). Two categories: **system services** (core startup/hardware tasks) and **user-installed services** (optional add-ons).

`systemd` is the init system on most modern distros — PID 1. Every process has a PID; child processes have a PPID. Process info lives under `/proc/`.

### systemctl
```bash
systemctl start ssh              # start a service
systemctl status ssh             # check status
systemctl enable ssh             # enable on boot
systemctl list-units --type=service   # list all services
journalctl -u ssh.service --no-pager  # view service logs
```

```bash
ps -aux | grep ssh   # list processes, filter
```

### Process states
Running, Waiting, Stopped, Zombie (terminated but still has a process-table entry).

### Signals
```bash
kill -l        # list all signals
kill -9 <PID>  # SIGKILL — force kill, no cleanup
```

| # | Signal | Description |
|---|---|---|
| 1 | SIGHUP | Sent when controlling terminal closes |
| 2 | SIGINT | `Ctrl+C` — interrupt |
| 3 | SIGQUIT | `Ctrl+D` — quit |
| 9 | SIGKILL | Immediate kill, no cleanup |
| 15 | SIGTERM | Normal termination request |
| 19 | SIGSTOP | Stop process (unhandleable) |
| 20 | SIGTSTP | `Ctrl+Z` — suspend (handleable) |

Process control tools: `kill`, `pkill`, `pgrep`, `killall`.

### Backgrounding / foregrounding
```bash
[Ctrl+Z]        # suspend current foreground process (SIGTSTP)
jobs             # list background/suspended jobs
bg               # resume most recent suspended job in background
fg <job_id>      # bring a job to the foreground
command &        # launch a command directly in the background
```

### Chaining commands

| Operator | Behavior |
|---|---|
| `;` | Run commands sequentially, regardless of success/failure of previous one |
| `&&` | Run next command only if previous succeeded |
| `\|` | Pipe — pass STDOUT of one command as STDIN to the next |

```bash
echo '1'; ls MISSING_FILE; echo '3'     # '3' still runs despite error
echo '1' && ls MISSING_FILE && echo '3'  # stops after the error
```

---

## Task Scheduling

Two main tools: **systemd timers** and **cron**.

### systemd timers
1. Create a `.timer` unit (schedule).
2. Create a matching `.service` unit (what to run).
3. Reload systemd and enable/start the timer.

```bash
sudo mkdir /etc/systemd/system/mytimer.timer.d
sudo vim /etc/systemd/system/mytimer.timer
```
```ini
[Unit]
Description=My Timer

[Timer]
OnBootSec=3min
OnUnitActiveSec=1hour

[Install]
WantedBy=timers.target
```
- `OnBootSec` — run once, N time after boot.
- `OnUnitActiveSec` — run repeatedly at this interval.

```bash
sudo vim /etc/systemd/system/mytimer.service
```
```ini
[Unit]
Description=My Service

[Service]
ExecStart=/full/path/to/my/script.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start mytimer.timer
sudo systemctl enable mytimer.timer
```

### Cron
Crontab fields: `minute(0-59) hour(0-23) day-of-month(1-31) month(1-12) day-of-week(0-7)`

```cron
0 */6 * * * /path/to/update_software.sh     # every 6 hours
0 0 1 * * /path/to/scripts/run_scripts.sh    # 1st of every month, midnight
0 0 * * 0 /path/to/scripts/clean_database.sh # every Sunday, midnight
```

**systemd vs cron:** systemd timers require a timer unit + service unit pair; cron uses a single crontab entry per task. Cron is simpler for basic schedules; systemd timers integrate better with the rest of the service ecosystem (logging, dependencies).

> Security note: cron jobs are a common persistence/backdoor mechanism — worth checking during audits/pentests.

---

## Network Services

### SSH
```bash
sudo apt install openssh-server -y
systemctl status ssh
ssh user@10.129.17.122
```
Config file: `/etc/ssh/sshd_config` (max connections, password vs key auth, host key checking, etc).

### NFS (Network File System)
Lets remote files be mounted and used as if local.

```bash
sudo apt install nfs-kernel-server -y
systemctl status nfs-kernel-server
```

Config file: `/etc/exports`.

| Option | Meaning |
|---|---|
| `rw` | Read/write access |
| `ro` | Read-only access |
| `no_root_squash` | Client root keeps root privileges on the share |
| `root_squash` | Client root is restricted to normal-user privileges |
| `sync` | Confirm writes to disk before acknowledging |
| `async` | Faster, but riskier (possible inconsistency) |

```bash
mkdir nfs_sharing
echo '/home/cry0l1t3/nfs_sharing hostname(rw,sync,no_root_squash)' >> /etc/exports

mkdir ~/target_nfs
mount 10.129.12.17:/home/john/dev_scripts ~/target_nfs   # mount remote share
```

> NFS misconfig (e.g. `no_root_squash`) can be abused for privilege escalation.

### Web Servers
Common Linux web servers: **Apache**, Nginx, Lighttpd, Caddy. Apache is modular — extended via modules:

| Module | Purpose |
|---|---|
| `mod_ssl` | Encrypts browser↔server communication |
| `mod_proxy` | Routes/forwards requests (reverse proxy) |
| `mod_headers` | Modify HTTP headers |
| `mod_rewrite` | Rewrite URLs on the fly |

```bash
sudo apt install apache2 -y
sudo systemctl start apache2
curl -I http://localhost:8080   # headers only, check status
```

Config: `/etc/apache2/apache2.conf` (global), `.htaccess` (per-directory overrides), `/etc/apache2/ports.conf` (listening port).

```apache
<Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

**curl vs wget:**
| Tool | Behavior |
|---|---|
| `curl <url>` | Prints page source to STDOUT |
| `wget <url>` | Downloads and saves content locally (e.g. `index.html`) |

**Quick file-serving with Python:**
```bash
python3 -m http.server                                   # serve cwd on port 8000
python3 -m http.server --directory /home/cry0l1t3/target_files
python3 -m http.server 443                                # custom port
```

### VPN
Encrypted tunnel into a remote network. Common Linux solutions: OpenVPN, L2TP/IPsec, PPTP, SSTP, SoftEther.

```bash
sudo apt install openvpn -y
sudo openvpn --config internal.ovpn
```
Config: `/etc/openvpn/server.conf`.

---

## Regular Expressions (RegEx)

A regex is a pattern made of literal characters + **metacharacters** that defines what to match in text. Used with tools like `grep`, `sed`.

| Operator | Description |
|---|---|
| `(a)` | Grouping — bundle sub-patterns together |
| `[a-z]` | Character class — match any one character in the set/range |
| `{1,10}` | Quantifier — repeat the preceding pattern N to M times |
| `\|` | OR — match if either side matches |
| `.*` | Wildcard span — acts like AND when used between two patterns (matches if both appear, in order) |

Use `grep -E` (extended regex) to enable these operators.

```bash
grep -E "(my|false)" /etc/passwd     # OR: lines containing "my" OR "false"
grep -E "(my.*false)" /etc/passwd    # AND: lines containing "my" THEN "false"
grep -E "my" /etc/passwd | grep -E "false"   # equivalent AND via piping
```

---

## User Management

Privileged files (e.g. `/etc/shadow`, which stores encrypted passwords) are root-only by default.

```bash
cat /etc/shadow        # Permission denied (as normal user)
sudo cat /etc/shadow   # works with elevated privileges
```

| Command | Description |
|---|---|
| `sudo` | Run a command as another user (typically root) |
| `su` | Switch user (default: superuser); authenticates via PAM, spawns a shell |
| `useradd` | Create a new user |
| `userdel` | Delete a user and related files |
| `usermod` | Modify an existing user account |
| `addgroup` | Create a new group |
| `delgroup` | Remove a group |
| `passwd` | Change a user's password |
