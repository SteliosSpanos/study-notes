# Section 5: Logs & Text Processing

## Where Logs Live
| Path | Contents |
|---|---|
| `/var/log/syslog` (Debian) / `/var/log/messages` (RHEL) | General system log |
| `/var/log/auth.log` / `/var/log/secure` | Logins, sudo, SSH |
| `/var/log/kern.log` | Kernel |
| `/var/log/nginx/`, `/var/log/apache2/` | Web server access/error |
| `/var/log/apt/`, `/var/log/dnf.log` | Package installs |
| `journalctl` | systemd journal (see Section 1) |
| `docker logs <ctr>` | Container stdout/stderr |

## Reading Logs
```bash
tail -n 100 app.log
tail -f app.log                    # follow
tail -F app.log                    # follow across log rotation
less +F app.log                    # follow; Ctrl-C to scroll, F to resume
less app.log                       # / search, n next, G end, g start, q quit
head -n 20 file
```

---

## grep
```bash
grep "error" app.log
grep -i "error" app.log                  # case-insensitive
grep -v "healthcheck" access.log         # invert: exclude lines
grep -c "500" access.log                 # count matches
grep -n "panic" app.log                  # line numbers
grep -r "DB_HOST" /etc/myapp/            # recursive
grep -rl "password" .                    # only filenames
grep -E "error|fatal|panic" app.log      # extended regex / OR
grep -w "fail" app.log                   # whole word only
grep -o "user=[a-z]*" app.log            # print only the match
grep -A 5 -B 2 "Exception" app.log       # 5 lines After, 2 Before
grep -C 3 "Exception" app.log            # 3 lines Context
```
`ripgrep` (`rg`) is a much faster alternative for code/large trees: `rg -i "timeout" /var/log`.

## cut / sort / uniq / wc
```bash
cut -d: -f1 /etc/passwd                  # field 1, ':' delimiter
cut -d' ' -f1 access.log                 # client IPs
sort file | uniq                         # dedupe
sort file | uniq -c | sort -rn           # count occurrences, most frequent first
sort -k2 -n file                         # sort by column 2 numerically
sort -h                                  # human sizes (1K 2M 3G)
wc -l file                               # line count
```

## awk — Column Processing
```bash
awk '{print $1}' access.log                      # first column
awk -F: '{print $1, $7}' /etc/passwd             # custom separator
awk '$9 == 500' access.log                       # rows where status = 500
awk '$9 >= 500 {print $7}' access.log            # URLs that returned 5xx
awk '{sum += $10} END {print sum}' access.log    # total bytes
awk 'NR==10,NR==20' file                         # lines 10–20
awk '{print $NF}' file                           # last column
```

## sed — Stream Editing
```bash
sed -n '10,20p' file                     # print lines 10–20
sed 's/foo/bar/' file                    # replace first per line (prints result)
sed 's/foo/bar/g' file                   # replace all
sed -i 's/8080/9090/g' config.ini        # edit IN PLACE
sed -i.bak 's/8080/9090/g' config.ini    # in place, keep .bak backup
sed '/^#/d' file                         # delete comment lines
sed '/^$/d' file                         # delete empty lines
sed -n '/10:00/,/10:05/p' app.log        # lines between two patterns (time window)
```

## tr / xargs / tee
```bash
echo "a,b,c" | tr ',' '\n'
tr -d '\r' < win.txt > unix.txt          # strip Windows line endings
find . -name "*.log" | xargs grep -l "ERROR"
find . -name "*.tmp" -print0 | xargs -0 rm   # safe with spaces in names
cat hosts.txt | xargs -I{} ssh {} uptime
./deploy.sh 2>&1 | tee deploy.log        # see output AND save it
echo "line" | sudo tee -a /etc/hosts     # append to root-owned file
```

## diff
```bash
diff old.conf new.conf
diff -u old.conf new.conf                 # unified format (like git)
diff <(ssh web1 cat /etc/nginx/nginx.conf) <(ssh web2 cat /etc/nginx/nginx.conf)   # config drift
```

## jq — JSON
```bash
curl -s https://api.github.com/repos/torvalds/linux | jq .
jq '.name' file.json
jq -r '.items[].metadata.name' pods.json          # -r raw strings (no quotes)
jq '.[] | select(.status == "failed")' jobs.json
jq '{name: .name, stars: .stargazers_count}' repo.json
kubectl get pods -o json | jq -r '.items[] | select(.status.phase!="Running") | .metadata.name'
journalctl -u myapp -o json | jq -r '.MESSAGE'
```
`yq` = same idea for YAML: `yq '.services | keys' docker-compose.yaml`.

---

## Real-world One-liners
```bash
# Top 10 client IPs hitting nginx
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Top requested URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Count HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 5xx per minute
awk '$9 ~ /^5/ {print substr($4, 2, 17)}' /var/log/nginx/access.log | uniq -c

# Failed SSH logins by IP
grep "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+" | sort | uniq -c | sort -rn | head

# Errors in the last hour from a service
journalctl -u myapp --since "1 hour ago" -p err --no-pager

# Find which config file sets a value
grep -rn "max_connections" /etc/ 2>/dev/null
```

## logrotate
```bash
cat /etc/logrotate.d/nginx
sudo logrotate -d /etc/logrotate.d/myapp   # dry run / debug
sudo logrotate -f /etc/logrotate.d/myapp   # force rotation now
```
```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate      # for apps that can't reopen log files
}
```
