# Section 2: Processes & Performance

## Quick Health Check (first 60 seconds on a box)
```bash
uptime                 # load averages 1/5/15 min
dmesg -T | tail        # kernel errors: OOM killer, disk I/O, NIC resets
vmstat 1 5             # run queue, swap, CPU split
free -h                # memory
df -h                  # full disks?
top / htop             # who is eating CPU/RAM
iostat -xz 1 3         # disk saturation
ss -s                  # socket summary
journalctl -p err -b   # recent errors
```
> **Load average** = processes running or waiting (CPU **and** uninterruptible I/O). Compare against core count (`nproc`): load 8 on 8 cores = fully busy, load 8 on 2 cores = overloaded.

---

## ps — Process Snapshot
```bash
ps aux                              # every process, BSD style
ps -ef                              # every process, full format with PPID
ps -ef --forest                     # tree view
ps aux --sort=-%mem | head          # top memory users
ps aux --sort=-%cpu | head          # top CPU users
ps -o pid,ppid,user,%cpu,%mem,etime,cmd -p 1234   # custom columns for one PID
ps -u www-data                      # by user
ps -C nginx                         # by command name
```
| Column | Meaning |
|---|---|
| `VSZ` | Virtual memory (mostly irrelevant) |
| `RSS` | Resident memory actually in RAM |
| `STAT` | `R` running, `S` sleeping, `D` uninterruptible (I/O), `Z` zombie, `T` stopped |
| `etime` | Elapsed time since start |

## pgrep / pkill / pidof
```bash
pgrep -a nginx          # PIDs + command line
pgrep -u root sshd
pidof nginx
pkill -f "python app.py"   # match full command line
pkill -HUP nginx           # send SIGHUP (reload)
```

## kill — Signals
```bash
kill 1234            # SIGTERM (15) — ask politely, allow cleanup
kill -9 1234         # SIGKILL — immediate, no cleanup (last resort)
kill -HUP 1234       # SIGHUP — many daemons reload config
kill -l              # list signals
killall nginx
```

---

## top / htop
```bash
top
top -o %MEM          # sort by memory
top -u www-data      # only one user
top -b -n 1 | head -20   # batch mode — for scripts/logs
htop                 # nicer: tree (F5), search (F3), kill (F9), sort (F6)
```
**Useful `top` keys:** `P` sort CPU, `M` sort memory, `1` per-core view, `c` full command, `k` kill, `H` threads.

**Header CPU fields:**
| Field | High value means |
|---|---|
| `us` | User-space CPU — app is busy |
| `sy` | Kernel CPU — syscalls, context switching |
| `wa` | Waiting on I/O — **disk is the bottleneck** |
| `st` | Stolen by hypervisor — noisy neighbour on a VM |
| `si`/`hi` | Interrupt handling — often network heavy |

---

## Memory: free & vmstat
```bash
free -h
free -h -s 2          # refresh every 2s
```
- Look at **`available`**, not `free`. Linux uses spare RAM for cache (`buff/cache`) and releases it on demand.
- Swap `used` growing + `si/so` non-zero in `vmstat` = real memory pressure.

```bash
vmstat 1              # every second
vmstat -w 1 10        # wide, 10 samples
```
| Column | Watch for |
|---|---|
| `r` | Runnable procs > CPU cores → CPU saturated |
| `b` | Blocked on I/O |
| `si` / `so` | Swap in/out — should be ~0 |
| `wa` | I/O wait % |

### OOM Killer
```bash
dmesg -T | grep -i -E "killed process|out of memory"
journalctl -k -g "Out of memory"
```

---

## Disk I/O: iostat, iotop
```bash
iostat -xz 1          # extended stats, skip idle devices (package: sysstat)
sudo iotop -o         # only processes doing I/O right now
```
| iostat column | Meaning |
|---|---|
| `%util` | Near 100% → device saturated |
| `r_await` / `w_await` | Avg latency (ms) per read/write |
| `aqu-sz` | Queue length |

## sar — Historical Metrics
```bash
sar -u 1 5            # CPU now
sar -r                # memory today
sar -q                # load/run queue history
sar -n DEV 1          # network per interface
sar -u -f /var/log/sysstat/sa15   # CPU on the 15th (path differs on RHEL: /var/log/sa/)
```
- Needs `sysstat` installed and enabled — answers "what happened at 3 AM?".

---

## lsof — Open Files (everything is a file)
```bash
sudo lsof -p 1234                # files opened by a process
sudo lsof -u www-data
sudo lsof /var/log/syslog        # who has this file open
sudo lsof +D /mnt/data           # who uses this directory (why umount fails)
sudo lsof -i :8080               # who listens/connects on port 8080
sudo lsof -i TCP -s TCP:LISTEN
sudo lsof | grep deleted         # deleted files still held open → "disk full but du says no"
```

## fuser
```bash
sudo fuser -v 8080/tcp           # PID using port
sudo fuser -vm /mnt/data         # processes using a mount
sudo fuser -k 8080/tcp           # kill it
```

---

## strace — Trace System Calls
```bash
sudo strace -p 1234                    # attach to running process
sudo strace -f -p 1234                 # include child threads/processes
strace -e trace=open,openat ./app      # which files does it try to open?
strace -e trace=network curl -s example.com
strace -c ./app                        # summary: count & time per syscall
strace -f -tt -o /tmp/trace.log ./app  # timestamps, to file
```
- Great for "it fails silently": look for `ENOENT` (missing file), `EACCES` (permission), `ECONNREFUSED`.
- Adds big overhead — careful on production.

---

## nice / renice / ionice
```bash
nice -n 10 tar czf backup.tgz /data     # start with lower priority (-20 high … 19 low)
sudo renice -n 5 -p 1234
ionice -c3 rsync -a /src /dst           # idle I/O class — don't hurt the app
```

## Running Things That Survive Logout
```bash
nohup ./long-job.sh > job.log 2>&1 &
tmux new -s work        # detach: Ctrl-b d   reattach: tmux attach -t work
screen -S work          # detach: Ctrl-a d   reattach: screen -r work
```

## timeout & watch
```bash
timeout 10s curl http://slow-service       # kill if it takes > 10s
watch -n 2 'ss -tn state established | wc -l'   # re-run every 2s
watch -d free -h                           # -d highlights changes
```
