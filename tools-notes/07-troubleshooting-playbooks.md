# Section 7: Troubleshooting Playbooks

Step-by-step checklists tying the previous sections together.

## General Method
1. **Define** the problem precisely — what fails, for whom, since when?
2. **What changed?** Deploys, config, package updates, certs, DNS.
   ```bash
   last reboot; grep -h " install \| upgrade " /var/log/dpkg.log | tail
   ```
3. **Check the obvious** — disk, memory, service state, logs.
4. **Bottom-up** through layers (network → port → app).
5. **Change one thing at a time**, verify, write down what you did.

---

## "The Service Is Down"
```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
systemctl --failed
ss -tlpn | grep 8080                 # is it listening? on which address?
curl -v http://127.0.0.1:8080/health # works locally?
df -h; free -h                       # disk full / OOM?
dmesg -T | tail -30
```
- Works locally, not remotely → bind address (`127.0.0.1` vs `0.0.0.0`), firewall, security group, reverse proxy.

## "The Server Is Slow"
```bash
uptime                        # load vs nproc
top                           # %us high = app, %wa high = disk, %st high = VM host
vmstat 1 5                    # r > cores? si/so > 0?
free -h                       # available memory, swap use
iostat -xz 1 3                # %util, await
ss -s                         # connection explosion?
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```
| Finding | Next step |
|---|---|
| One process pegging CPU | `top -H -p PID` (threads), `strace -c -p PID`, app profiler |
| High `wa` | `iotop -o`, check disk health, log spam |
| Swapping | Find memory hog, add RAM/limits, check for leaks |
| Many `TIME-WAIT` / `CLOSE-WAIT` | Connection pooling issues, app not closing sockets (`CLOSE-WAIT`) |

## "Disk Is Full"
```bash
df -h                                   # which filesystem
df -i                                   # or inodes?
sudo du -xh / --max-depth=2 2>/dev/null | sort -h | tail -20
sudo journalctl --disk-usage
sudo lsof +L1                           # deleted-but-open files
docker system df                        # Docker images/volumes/build cache
```
Quick wins:
```bash
sudo journalctl --vacuum-size=200M
sudo apt clean                          # or: sudo dnf clean all
docker system prune                     # removes stopped containers, dangling images, unused networks (asks first)
sudo find /var/log -name "*.gz" -mtime +14 -delete
```

## "Can't Connect to Host / Port"
```bash
ping -c 3 host                          # reachable? (ICMP may be blocked)
dig +short host                         # resolves to the right IP?
nc -zv host 443                         # port open?
traceroute -T -p 443 host               # where does it stop?
curl -v https://host                    # TLS / HTTP level
```
On the target:
```bash
ss -tlpn | grep 443
sudo ufw status                         # Ubuntu
sudo firewall-cmd --list-all            # RHEL
sudo iptables -L -n                     # raw rules
sudo tcpdump -i any -nn port 443        # do packets even arrive?
```
| `nc` / `curl` result | Meaning |
|---|---|
| `Connection refused` | Host reachable, nothing listening (or REJECT rule) |
| Timeout | Packets dropped — firewall, routing, wrong IP, host down |
| `Could not resolve host` | DNS |
| `SSL certificate problem` | Expired / wrong hostname / missing intermediate CA |

## "DNS Is Weird"
```bash
getent hosts example.com                # what apps see
dig +short example.com                  # what the resolver says
dig @8.8.8.8 +short example.com         # what the internet says
grep example.com /etc/hosts
resolvectl status
```
- Differences between resolvers after a change = **TTL / propagation**; check `dig example.com` → TTL column.

## "Certificate Expired / TLS Errors"
```bash
echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -dates -subject
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

## "Container Keeps Restarting"
```bash
docker ps -a
docker logs --tail 100 <ctr>
docker inspect <ctr> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
docker stats --no-stream
docker exec -it <ctr> sh
```
- Exit code `137` = killed by SIGKILL (often OOM). `143` = SIGTERM. `1` = app error → read logs.

---

## Tool Cheat-sheet by Symptom
| Symptom | First tools |
|---|---|
| Service failed | `systemctl status`, `journalctl -u` |
| High CPU | `top`, `htop`, `ps --sort`, `strace` |
| High memory / OOM | `free -h`, `ps --sort=-%mem`, `dmesg` |
| Slow disk | `iostat -x`, `iotop`, `dmesg` |
| Disk full | `df -h`, `df -i`, `du`, `ncdu`, `lsof +L1` |
| Port not reachable | `ss -tlpn`, `nc -zv`, firewall, `tcpdump` |
| DNS | `dig`, `getent hosts`, `resolvectl` |
| HTTP errors | `curl -v`, access/error logs, `awk` one-liners |
| Weird app behaviour | `strace`, `lsof -p`, app logs |
| Who changed what | `journalctl`, `/var/log/auth.log`, `last`, package logs |
