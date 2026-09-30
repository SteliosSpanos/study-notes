# Section 1: systemctl & journalctl

## systemd in One Paragraph
- **systemd** is the init system (PID 1) on most modern distros (Debian, Ubuntu, RHEL, Fedora, Arch).
- Everything it manages is a **unit**: `.service`, `.socket`, `.timer`, `.mount`, `.target`, `.path`, ...
- `systemctl` controls units, `journalctl` reads their logs (the **journal**).

| Unit file location | Purpose |
|---|---|
| `/lib/systemd/system/` (or `/usr/lib/systemd/system/`) | Installed by packages — don't edit |
| `/etc/systemd/system/` | Admin overrides and custom units — **wins** over the above |
| `~/.config/systemd/user/` | Per-user units (`systemctl --user`) |

---

## systemctl

### Service Lifecycle
```bash
sudo systemctl start nginx        # start now
sudo systemctl stop nginx         # stop now
sudo systemctl restart nginx      # stop + start (drops connections)
sudo systemctl reload nginx       # re-read config without restart (if supported)
sudo systemctl reload-or-restart nginx
sudo systemctl kill -s SIGKILL nginx   # send signal to all processes of the unit
```

### Boot Behaviour
```bash
sudo systemctl enable nginx         # start at boot (creates symlink)
sudo systemctl enable --now nginx   # enable + start in one go
sudo systemctl disable --now nginx  # disable + stop
sudo systemctl mask nginx           # make it impossible to start (links to /dev/null)
sudo systemctl unmask nginx
```
- `enable` ≠ `start`. A freshly installed service may be enabled but not running, or running but not enabled.

### Inspecting State
```bash
systemctl status nginx              # state, PID, memory, cgroup, last log lines
systemctl is-active nginx           # active / inactive / failed  (good for scripts)
systemctl is-enabled nginx          # enabled / disabled / masked / static
systemctl is-failed nginx
systemctl cat nginx                 # show the unit file + drop-ins actually used
systemctl show nginx -p MainPID,Restart,ExecStart   # raw properties
systemctl list-dependencies nginx
```

### Listing Units
```bash
systemctl list-units --type=service               # loaded + active services
systemctl list-units --type=service --state=running
systemctl list-units --all                        # include inactive
systemctl list-unit-files --type=service          # enabled/disabled state of every unit
systemctl --failed                                # anything that crashed  ← check first
systemctl list-timers                             # scheduled timers + next run
```

### Editing Units Safely
```bash
sudo systemctl edit nginx           # create drop-in override (/etc/systemd/system/nginx.service.d/override.conf)
sudo systemctl edit --full nginx    # copy full unit to /etc and edit it
sudo systemctl daemon-reload        # REQUIRED after changing unit files by hand
sudo systemctl revert nginx         # drop all local overrides
```
Example override — auto-restart a flaky service:
```ini
[Service]
Restart=on-failure
RestartSec=5s
```

> **Common mistake:** editing a unit file and forgetting `daemon-reload` — systemd keeps using the old cached version and warns `Warning: The unit file ... changed on disk`.

### Minimal Custom Service
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --port 8080
Restart=on-failure
Environment=APP_ENV=production

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

### System-wide Actions
```bash
systemctl reboot
systemctl poweroff
systemctl get-default                     # current boot target
sudo systemctl set-default multi-user.target   # boot to CLI (graphical.target = GUI)
systemctl list-jobs                       # stuck start/stop jobs
systemd-analyze blame                     # which units slowed down boot
systemd-analyze critical-chain
```

### Unit States Cheat-sheet
| State | Meaning |
|---|---|
| `active (running)` | Running normally |
| `active (exited)` | Oneshot finished successfully |
| `inactive (dead)` | Stopped |
| `failed` | Crashed / non-zero exit / start timed out |
| `activating (auto-restart)` | Crash loop — waiting to restart |

---

## journalctl

### Basics
```bash
journalctl                     # everything, oldest first (opens in pager)
journalctl -e                  # jump to end
journalctl -f                  # follow live (like tail -f)
journalctl -n 100              # last 100 lines
journalctl -r                  # newest first
journalctl --no-pager          # for piping into grep etc.
```

### Filtering
| Option | Example | Meaning |
|---|---|---|
| `-u` | `-u nginx` | By unit (repeatable: `-u nginx -u php-fpm`) |
| `-b` | `-b`, `-b -1` | Current boot / previous boot |
| `--since` / `--until` | `--since "1 hour ago"` | Time window |
| `-p` | `-p err` | Priority: `emerg alert crit err warning notice info debug` (shows that level and above) |
| `-k` | `-k` | Kernel messages only (like `dmesg`) |
| `-g` | `-g "timeout"` | Grep by regex in message |
| `_PID=` | `_PID=1234` | By process ID |
| `_COMM=` | `_COMM=sshd` | By executable name |
| `-t` | `-t CRON` | By syslog identifier |
| `--user` | `--user -u myunit` | User units |

### Everyday Examples
```bash
journalctl -u nginx -f                                  # tail one service
journalctl -u nginx -b                                  # this boot only
journalctl -u nginx --since "2026-09-30 10:00" --until "2026-09-30 10:30"
journalctl -u nginx --since today -p warning
journalctl -p err -b                                    # all errors since boot
journalctl -b -1 -e                                     # why did last boot die/reboot?
journalctl --list-boots
journalctl -k -p warning                                # kernel warnings (disk, OOM, NIC)
journalctl _COMM=sshd -g "Failed password"              # SSH brute-force attempts
journalctl -u myapp -o cat                              # message only, no metadata
journalctl -u myapp -o json-pretty -n 1                 # all fields of an entry
```

### Output Formats (`-o`)
| Format | Use |
|---|---|
| `short` (default) | Syslog-like |
| `short-iso` | ISO timestamps |
| `cat` | Just the message |
| `json` / `json-pretty` | Machine parsing / see all fields |
| `verbose` | Every field |

### Disk Usage & Cleanup
```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=500M      # shrink to 500 MB
sudo journalctl --vacuum-time=2weeks    # delete older than 2 weeks
sudo journalctl --rotate
```
- Permanent limit: `SystemMaxUse=500M` in `/etc/systemd/journald.conf`, then `systemctl restart systemd-journald`.
- If `/var/log/journal/` doesn't exist, logs are **volatile** (lost on reboot). Create it to persist them.

> **Tip:** Non-root users only see their own logs. Add yourself to the `systemd-journal` (or `adm`) group to read everything without sudo.

---

## Debugging a Service That Won't Start
```bash
systemctl status myapp                 # 1. what state + last lines + exit code
journalctl -u myapp -n 50 --no-pager   # 2. full recent log
systemctl cat myapp                    # 3. what ExecStart/User/WorkingDirectory really are
sudo -u myapp /opt/myapp/bin/server    # 4. run the ExecStart by hand as the same user
systemd-analyze verify /etc/systemd/system/myapp.service   # 5. syntax check
```
| Symptom in `status` | Likely cause |
|---|---|
| `status=203/EXEC` | Binary path wrong / not executable / missing shebang |
| `status=217/USER` | `User=` doesn't exist |
| `status=200/CHDIR` | `WorkingDirectory=` doesn't exist |
| `status=1/FAILURE` | App itself exited with error → read its log |
| `start request repeated too quickly` | Crash loop hit `StartLimitBurst` → fix, then `systemctl reset-failed myapp` |
| `Timeout` | App didn't signal readiness — check `Type=` (`simple` vs `forking` vs `notify`) |
