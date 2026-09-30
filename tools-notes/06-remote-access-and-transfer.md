# Section 6: Remote Access & File Transfer

## ssh
```bash
ssh user@host
ssh -p 2222 user@host
ssh -i ~/.ssh/id_ed25519 user@host
ssh user@host 'df -h && uptime'          # run remote command
ssh -v user@host                         # debug connection/auth (-vvv for more)
ssh -J bastion user@private-host         # jump through a bastion host
ssh -A user@host                         # forward agent (only to trusted hosts)
```

### Keys
```bash
ssh-keygen -t ed25519 -C "you@laptop"
ssh-copy-id user@host                    # install pubkey in remote authorized_keys
ssh-add ~/.ssh/id_ed25519                # load key into agent
ssh-keygen -R host                       # remove stale host key ("REMOTE HOST IDENTIFICATION HAS CHANGED")
```
> Only remove a changed host key when you know **why** it changed (server rebuilt, IP reused). Otherwise it can indicate a man-in-the-middle.

### Tunnels / Port Forwarding
```bash
ssh -L 5432:localhost:5432 user@db-host      # local: localhost:5432 → db-host's 5432
ssh -L 8080:internal-app:80 user@bastion     # reach internal service through bastion
ssh -R 9000:localhost:3000 user@public-host  # remote: expose local 3000 on public-host:9000
ssh -D 1080 user@host                        # SOCKS proxy
ssh -fNL 5432:localhost:5432 user@db-host    # -f background, -N no shell
```

### ~/.ssh/config
```
Host bastion
    HostName 203.0.113.10
    User admin
    IdentityFile ~/.ssh/id_ed25519

Host app-*
    User deploy
    ProxyJump bastion

Host *
    ServerAliveInterval 60
```
Then just: `ssh app-1`.

### SSH Troubleshooting
| Error | Check |
|---|---|
| `Connection refused` | sshd not running / wrong port → `systemctl status ssh`, `ss -tlpn \| grep 22` |
| `Connection timed out` | Firewall / security group / wrong IP |
| `Permission denied (publickey)` | Wrong key, wrong user, perms: `~/.ssh` = `700`, `authorized_keys` = `600` |
| `Host key verification failed` | Host key changed → see warning above |

Server side: `journalctl -u ssh -f` (Debian) / `journalctl -u sshd -f` (RHEL) while you connect.

---

## scp
```bash
scp file.txt user@host:/tmp/
scp user@host:/var/log/app.log .
scp -r ./dist user@host:/var/www/
scp -P 2222 file user@host:~       # capital P for port
```

## rsync (preferred for anything big or repeated)
```bash
rsync -avz ./dist/ user@host:/var/www/site/           # -a archive, -v verbose, -z compress
rsync -avz --delete ./dist/ user@host:/var/www/site/  # mirror: delete extra files on dest
rsync -avzn --delete ./dist/ user@host:/var/www/site/ # -n dry run — ALWAYS try first with --delete
rsync -avP bigfile user@host:/data/                   # -P progress + resume partial
rsync -av --exclude 'node_modules' --exclude '.git' ./ user@host:/app/
rsync -av -e "ssh -p 2222" ./ user@host:/app/
```
> **Trailing slash matters:** `src/` copies the *contents* of src; `src` copies the *directory itself* into dest.

## sftp
```bash
sftp user@host      # ls, cd, get, put, lcd, lls, exit
```

---

## tmux — Keep Sessions Alive over SSH
```bash
tmux new -s deploy
tmux ls
tmux attach -t deploy
tmux kill-session -t deploy
```
| Keys (prefix `Ctrl-b`) | Action |
|---|---|
| `d` | Detach |
| `c` | New window |
| `n` / `p` | Next / previous window |
| `%` / `"` | Split vertical / horizontal |
| arrow | Move between panes |
| `[` | Scroll mode (q to exit) |

---

## Users, Sudo & Who's Logged In
```bash
whoami; id                      # current user, UID, groups
who; w                          # who is logged in + what they run
last -n 20                      # recent logins
lastb                           # failed logins (root)
sudo -l                         # what can I run with sudo
sudo visudo                     # edit sudoers safely (syntax-checked)
sudo usermod -aG docker $USER   # add to group (re-login to apply)
```
