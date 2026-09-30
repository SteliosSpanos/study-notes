# Section 3: Networking Tools

## Troubleshoot Bottom-Up
| Layer | Question | Tool |
|---|---|---|
| Link | Is the interface up? | `ip link`, `ethtool` |
| IP | Do I have an address and a route? | `ip addr`, `ip route` |
| Reachability | Can I reach the host? | `ping`, `traceroute`, `mtr` |
| DNS | Does the name resolve correctly? | `dig`, `nslookup`, `getent hosts` |
| Port | Is the port open / listening? | `ss`, `nc`, `nmap` |
| Firewall | Is traffic blocked? | `iptables`, `nft`, `ufw` |
| Application | Does it answer correctly? | `curl`, `openssl s_client` |
| Packets | What is actually on the wire? | `tcpdump` |

---

## ip — Interfaces, Addresses, Routes
Replaces old `ifconfig`, `route`, `arp` (net-tools).
```bash
ip a                              # all addresses (ip addr show)
ip -br a                          # brief one-line-per-interface view
ip link show                      # interfaces + state (UP/DOWN) + MAC
sudo ip link set eth0 up
sudo ip addr add 192.168.1.50/24 dev eth0     # temporary (lost on reboot)
sudo ip addr del 192.168.1.50/24 dev eth0

ip route                          # routing table
ip route get 8.8.8.8              # which interface/gateway would be used
sudo ip route add 10.10.0.0/16 via 192.168.1.1
sudo ip route add default via 192.168.1.1

ip neigh                          # ARP table
ip -s link show eth0              # RX/TX counters, errors, drops
```
> Changes with `ip` are **not persistent**. Persist via netplan (`/etc/netplan/*.yaml`, Ubuntu), `nmcli` (NetworkManager), or `/etc/network/interfaces` (Debian).

## nmcli (NetworkManager)
```bash
nmcli device status
nmcli connection show
nmcli connection up "Wired connection 1"
nmcli con mod eth0 ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 1.1.1.1 ipv4.method manual
```

---

## ping
```bash
ping -c 4 8.8.8.8          # 4 packets then stop
ping -i 0.2 host           # interval 0.2s
ping -W 2 host             # 2s timeout per reply
ping -s 1472 -M do host    # test MTU (1472 + 28 header = 1500, don't fragment)
ping -4 / -6 host          # force IPv4/IPv6
```
- `ping 8.8.8.8` works but `ping google.com` fails → **DNS problem**, not network.
- No reply ≠ host down — many hosts/firewalls drop ICMP.

## traceroute / tracepath / mtr
```bash
traceroute example.com
traceroute -T -p 443 example.com   # TCP instead of UDP (passes more firewalls)
traceroute -n example.com          # no reverse DNS (faster)
tracepath example.com              # no root needed, also shows MTU
mtr example.com                    # live traceroute + ping per hop
mtr -rwc 100 example.com           # report mode, 100 cycles — paste into tickets
```
- Loss at an intermediate hop but **not** at the final one = that router just deprioritises ICMP, not a real issue.

---

## DNS: dig, nslookup, host, getent
```bash
dig example.com                    # A record, full output
dig +short example.com
dig example.com MX
dig example.com AAAA
dig example.com TXT
dig -x 93.184.216.34               # reverse lookup (PTR)
dig @1.1.1.1 example.com           # ask a specific resolver
dig +trace example.com             # follow delegation from root servers
dig example.com NS +short

nslookup example.com
host example.com
getent hosts example.com           # resolves the way apps do (includes /etc/hosts)
resolvectl status                  # systemd-resolved: which DNS servers per link
resolvectl flush-caches
```
| File | Role |
|---|---|
| `/etc/hosts` | Static name → IP overrides |
| `/etc/resolv.conf` | Resolver(s) used |
| `/etc/nsswitch.conf` | Order: `files dns` = check /etc/hosts first |

> `dig` bypasses `/etc/hosts`; `getent` doesn't. If they disagree, check `/etc/hosts`.

---

## ss — Sockets (replaces netstat)
```bash
ss -tulpn                  # TCP+UDP listening sockets with process — the classic
ss -tlpn                   # TCP listening only
ss -tn                     # established TCP connections, numeric
ss -tan state established
ss -tan state time-wait | wc -l
ss -tnp dst 10.0.0.5       # connections to a host
ss -tn sport = :443        # by source port
ss -s                      # summary counts
```
| Flag | Meaning |
|---|---|
| `-t` / `-u` | TCP / UDP |
| `-l` | Listening only |
| `-a` | All states |
| `-n` | Numeric (no DNS/service lookups) |
| `-p` | Show process (needs sudo for others' procs) |

> Listening on `127.0.0.1:8080` = only local access. `0.0.0.0:8080` or `*:8080` = all interfaces. Common reason "works on the server, not from outside".

Old equivalent: `netstat -tulpn` (package `net-tools`).

---

## nc (netcat) — Swiss Army Knife
```bash
nc -zv db.internal 5432            # is TCP port open?
nc -zv host 20-25                  # scan small port range
nc -zvu host 53                    # UDP (unreliable result)
nc -l 9000                         # listen (test firewall: listen on A, connect from B)
nc host 9000                       # connect & type
echo -e "GET / HTTP/1.0\r\n\r\n" | nc example.com 80
```
Alternative without nc: `timeout 3 bash -c '</dev/tcp/host/5432' && echo open`.

## telnet
```bash
telnet host 25                     # manual SMTP/any plaintext protocol test
```

## nmap — Port Scanning
```bash
nmap host                          # top 1000 TCP ports
nmap -p 22,80,443 host
nmap -p- host                      # all 65535 ports
nmap -sV host                      # detect service versions
nmap -sn 192.168.1.0/24            # ping sweep — which hosts are up
sudo nmap -sU -p 53,161 host       # UDP
```
> **Only scan hosts you own or are authorised to test.**

---

## curl — HTTP(S) Testing
```bash
curl https://example.com
curl -I https://example.com                    # headers only (HEAD)
curl -i https://example.com                    # headers + body
curl -v https://example.com                    # verbose: DNS, TLS handshake, headers
curl -L http://example.com                     # follow redirects
curl -sS -o /dev/null -w "%{http_code}\n" https://example.com   # status code only
curl -k https://self-signed.local              # ignore TLS errors (testing only)
curl -X POST -H "Content-Type: application/json" -d '{"a":1}' https://api/x
curl -u user:pass https://api/x
curl -H "Authorization: Bearer $TOKEN" https://api/x
curl --resolve example.com:443:10.0.0.5 https://example.com   # hit a specific backend, keep SNI/Host
curl -H "Host: example.com" http://10.0.0.5/                  # same idea over HTTP
curl --connect-timeout 5 -m 10 https://example.com
curl -O https://example.com/file.tar.gz        # save with remote name
```
**Timing breakdown** — where is the latency?
```bash
curl -o /dev/null -s -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" https://example.com
```

## wget
```bash
wget https://example.com/file.iso
wget -c https://example.com/file.iso     # resume
wget -qO- https://example.com            # print to stdout
```

## openssl — TLS / Certificates
```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer
openssl x509 -in cert.pem -noout -text        # inspect a cert file
openssl x509 -in cert.pem -noout -enddate      # expiry
```

---

## tcpdump — Packet Capture
```bash
sudo tcpdump -i any                          # all interfaces
sudo tcpdump -i eth0 -nn port 443            # -nn: no DNS, no port names
sudo tcpdump -i eth0 host 10.0.0.5
sudo tcpdump -i eth0 'tcp port 80 and src 10.0.0.5'
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0'   # only SYNs
sudo tcpdump -i eth0 -A port 80              # print payload ASCII
sudo tcpdump -i eth0 -c 100 -w cap.pcap      # save 100 packets → open in Wireshark
tcpdump -r cap.pcap
```
- SYN sent, no SYN-ACK back → firewall drop / host down.
- SYN sent, RST back → nothing listening on that port.

---

## Firewall Quick Checks
```bash
sudo ufw status verbose
sudo ufw allow 443/tcp
sudo firewall-cmd --list-all                 # firewalld (RHEL)
sudo firewall-cmd --add-port=8080/tcp --permanent && sudo firewall-cmd --reload
sudo iptables -L -n -v --line-numbers
sudo nft list ruleset
```
> Also check the **cloud** layer: security groups / NACLs (AWS), NSGs (Azure), VPC firewall rules (GCP). Host firewall open + cloud firewall closed = still blocked.

## Misc
```bash
hostname -I                  # all IPs of this host
curl -s ifconfig.me          # public IP
ethtool eth0                 # link speed, duplex, link detected
sudo ethtool -S eth0         # NIC error counters
iftop -i eth0                # live bandwidth per connection
nload                        # live bandwidth per interface
arp -n / ip neigh            # layer-2 neighbours
```
