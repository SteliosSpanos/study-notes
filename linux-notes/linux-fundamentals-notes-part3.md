# Linux Fundamentals (Part 3)

## Linux Security

**Core hardening practices:**

```bash
apt update && apt dist-upgrade   # keep OS and packages current
```

| Practice | Why |
|---|---|
| Keep OS/packages updated | Patches known vulnerabilities |
| Use a host firewall (iptables / nftables / ufw / firewalld) | Restrict traffic if network-level rules are insufficient |
| Disable SSH password login; disable root SSH login | Reduces brute-force / credential-theft risk |
| Avoid administering as root | Limits blast radius of compromise |
| Principle of least privilege | Grant specific `sudo` commands via sudoers instead of full root access |
| `fail2ban` | Tracks failed logins; blocks/handles hosts that exceed a threshold |
| Periodic audits | Catch outdated kernels, bad permissions, world-writable files, misconfigured cron/services |
| SELinux / AppArmor | Kernel-level MAC — label every process/file/object, enforce policy rules on access (e.g. who can append to or move a file) |

**Additional security tools:** Snort, chkrootkit, rkhunter, Lynis.

**Other recommended settings:**
- Remove/disable unnecessary services and software.
- Remove services relying on unencrypted authentication.
- Ensure NTP and Syslog are running.
- One account per user (no shared logins).
- Enforce strong passwords; configure password aging and history (restrict reuse).
- Lock accounts after repeated login failures.
- Disable unwanted SUID/SGID binaries.

> Security is a continuous process, not a one-time setup.

### TCP Wrappers

Host-based access control restricting service access by hostname/IP. Checked **before** granting a connection.

Config files:
- `/etc/hosts.allow` — explicit allow rules
- `/etc/hosts.deny` — explicit deny rules

```bash
# /etc/hosts.allow
sshd : 10.129.14.0/24              # allow SSH from local subnet
ftpd : 10.129.14.10                # allow FTP from a specific host
telnetd : .inlanefreight.local      # allow Telnet from a domain
```

```bash
# /etc/hosts.deny
ALL : .inlanefreight.com           # deny all services from a domain
sshd : 10.129.22.22                # deny SSH from a specific host
ftpd : 10.129.22.0/24              # deny FTP from an IP range
```

**Key facts:**
- Rule order matters — first match wins.
- TCP wrappers control access to **services**, not ports — not a substitute for a real firewall.

---

## Firewall Setup (iptables)

Linux firewalling is implemented via the **Netfilter** kernel framework. `iptables` is the classic configuration tool (succeeded `ipchains`/`ipfwadm`, introduced in kernel 2.4).

**Alternatives:**

| Tool | Notes |
|---|---|
| `iptables` | Classic, de facto standard, complex syntax |
| `nftables` | Modern syntax, better performance, not iptables-compatible |
| `ufw` | "Uncomplicated Firewall" — simple front-end, built on iptables |
| `firewalld` | Dynamic zones/services, more flexible management |

### Core components

| Component | Role |
|---|---|
| Tables | Categorize rules by purpose |
| Chains | Group rules applied to a traffic type |
| Rules | Match criteria + action |
| Matches | Specific filter criteria (IP, port, protocol, state...) |
| Targets | Action taken on a match (ACCEPT, DROP, etc.) |

### Tables

| Table | Purpose | Built-in Chains |
|---|---|---|
| `filter` | Filter traffic by IP/port/protocol | INPUT, OUTPUT, FORWARD |
| `nat` | Modify source/destination IPs | PREROUTING, POSTROUTING |
| `mangle` | Modify packet header fields | PREROUTING, OUTPUT, INPUT, FORWARD, POSTROUTING |
| `raw` | Special packet-processing config | PREROUTING, OUTPUT |

### Chains
- **Built-in** — auto-created per table (e.g. `filter`'s INPUT/OUTPUT/FORWARD).
  - `PREROUTING` — modifies destination IP before routing.
  - `POSTROUTING` — modifies source IP after routing.
- **User-defined** — custom groupings (e.g. all rules for a set of web servers, or all rules for port 80) added to any table.

### Rules & Targets

Rules are added with `-A <chain>` plus match criteria and a `-j <target>`.

| Target | Effect |
|---|---|
| `ACCEPT` | Allow packet through |
| `DROP` | Silently block packet |
| `REJECT` | Block + send error response to sender |
| `LOG` | Log packet info |
| `SNAT` | Rewrite source IP (NAT, e.g. private→public) |
| `DNAT` | Rewrite destination IP (NAT, e.g. forwarding) |
| `MASQUERADE` | Like SNAT but for dynamic source IPs |
| `REDIRECT` | Send packet to a different port/IP |
| `MARK` | Tag packet with a Netfilter mark for later use |

```bash
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # allow inbound SSH
```

### Matches

| Match | Meaning |
|---|---|
| `-p`/`--protocol` | Protocol (tcp, udp, icmp) |
| `--dport` | Destination port |
| `--sport` | Source port |
| `-s`/`--source` | Source IP |
| `-d`/`--destination` | Destination IP |
| `-m state` | Connection state (NEW, ESTABLISHED, RELATED) |
| `-m multiport` | Multiple ports/ranges |
| `-m tcp` / `-m udp` | Protocol-specific extra options |
| `-m string` | Match packet content string |
| `-m limit` | Rate limiting |
| `-m conntrack` | Connection-tracking state |
| `-m mark` | Match by Netfilter mark |
| `-m mac` | Match by MAC address |
| `-m iprange` | Match an IP range |

```bash
sudo iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT   # allow inbound HTTP
```

---

## System Logs

Logs reveal system behavior, app activity, and security events — useful both for defense and for pentesters validating whether their activity triggered alerts.

**Hardening basics:** correct log levels, log rotation (avoid unbounded growth), secure storage/permissions, regular review.

### Log categories

| Type | Location | Contains |
|---|---|---|
| Kernel logs | `/var/log/kern.log` | Driver/hardware events, system calls, kernel events — crashes, resource limits, suspicious syscalls |
| System logs | `/var/log/syslog` | Service start/stop, login attempts, reboots |
| Authentication logs | `/var/log/auth.log` | Auth attempts (success/failure) — more focused than syslog for this purpose |
| Application logs | e.g. `/var/log/apache2/error.log`, `/var/log/mysql/error.log` | App-specific activity/errors |
| Security logs | e.g. `/var/log/fail2ban.log`, `/var/log/ufw.log`, or general syslog/auth.log | Security tool events (banned IPs, firewall activity) |

**Example syslog entries:**
```
Feb 28 2023 15:04:22 server sshd[3010]: Failed password for htb-student from 10.14.15.2 port 50223 ssh2
Feb 28 2023 15:07:19 server sshd[3010]: Accepted password for htb-student from 10.14.15.2 port 50223 ssh2
```

**Example auth.log entries:**
```
Feb 28 2023 18:15:01 sshd[5678]: Accepted publickey for admin from 10.14.15.2 port 43210 ssh2
Feb 28 2023 18:15:03 sudo: admin : TTY=pts/1 ; PWD=/home/admin ; USER=root ; COMMAND=/bin/bash
Feb 28 2023 18:15:12 kernel: firewall: unexpected traffic allowed on port 22
```
- Shows pubkey auth success, sudo command execution (admin running as root), and a flagged anomaly (unexpected allowed traffic) — worth investigating as a possible breach indicator.

### Access logs — default locations

| Service | Path |
|---|---|
| Apache | `/var/log/apache2/access.log` |
| Nginx | `/var/log/nginx/access.log` |
| OpenSSH | `/var/log/auth.log` (Ubuntu) / `/var/log/secure` (CentOS/RHEL) |
| MySQL | `/var/log/mysql/mysql.log` |
| PostgreSQL | `/var/log/postgresql/postgresql-<version>-main.log` |
| Systemd | `/var/log/journal/` |

**Example access log entry:**
```
2023-03-07T10:15:23+00:00 servername privileged.sh: htb-student accessed /root/hidden/api-keys.txt
```

**Analysis tools:** `tail`, `grep`, `sed`, plus GUI log viewers built into most desktop environments.

**Audit/access logs** specifically track: login attempts, file access, network connections, config changes — key for spotting unauthorized access, data exfiltration, or other suspicious activity.
