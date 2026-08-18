# Section 5: Interacting with the Container via Volumes and Ports

## The Problem
The yt-dlp container works, but downloaded videos get stuck inside the container's ephemeral storage — we had to manually `docker cp` them out. There's a better way: **volumes / bind mounts**.

## Bind Mounts with `-v`
- A **bind mount** maps a file or directory from the **host machine** into the container.
- Prior to Docker Engine v23, `-v` required an absolute host path. **Since v23**, relative paths are supported too.

```bash
docker run -v "$(pwd):/mydir" yt-dlp https://www.youtube.com/watch?v=saEpkcVi1d4
```
- Mounts the current host directory as `/mydir` inside the container — this **overwrites** whatever was placed at `/mydir` by the Dockerfile.
- Downloaded videos now land directly in the host's working directory.

### What a Volume Actually Is
- A **shared directory/file** between host and container.
- Changes made inside the container (to a mounted path) persist on the **host filesystem**, surviving container removal.
- Without volumes: any files created/modified inside a container are lost once the container is removed and recreated from the image.
- Volumes also enable **file sharing between containers**.

### Mounting a Single File
```bash
docker run -v "$(pwd)/material.md:/mydir/material.md" ...
```
- Edits to `material.md` sync both ways (host ↔ container).
- **Note:** if the specified host file doesn't exist, `-v` will create a **directory** at that path instead (a common gotcha) — make sure the file exists first.

## Networking Basics

### Sending Messages
- Programs communicate via URLs like `http://127.0.0.1:3000`:
  - `http` = protocol
  - `127.0.0.1` = IP address (can be a hostname instead)
  - `3000` = port
- `127.0.0.1` / `localhost` = refers to **the machine or container itself**.
  - From inside a container → `localhost` = that same container.
  - From the host → `localhost` = the host machine.

### Receiving Messages
- A program can **listen** on a specific port; messages sent to that port get delivered to it.

### Port Mapping
- You can map a host port to a container port. E.g., host port 1000 → container port 2000: sending to `http://localhost:1000` on the host reaches the container's port 2000.

## Exposing vs Publishing Ports
Two separate steps to let outside traffic reach a container:

1. **Exposing a port** — declares (informationally) that the container listens on a port. Mostly documentation/config hint for humans; doesn't actually open anything.
   ```dockerfile
   EXPOSE <port>
   ```
2. **Publishing a port** — actually maps a host port to the container port, making it reachable.
   ```bash
   docker run -p <host-port>:<container-port>
   ```

- If you omit the host port, Docker picks a free one automatically:
```bash
docker run -p 4567 app-in-port
```

### Limiting to a Protocol (e.g. UDP)
```dockerfile
EXPOSE <port>/udp
```
```bash
docker run -p <host-port>:<container-port>/udp
```

## Security Considerations
> **Reminder:** Publishing a port can expose your application to anyone on the internet.

- Avoid opening ports carelessly — an attacker could exploit an insecure service behind an open port.
- **Restrict to localhost only** by binding the host IP explicitly:
```bash
docker run -p 127.0.0.1:3456:3000
```
  - Only accepts connections from your own machine through host port 3456 → container port 3000.
- **Shorthand risk:** `-p 3456:3000` is equivalent to `-p 0.0.0.0:3456:3000` — this opens the port to **everyone**, not just localhost.
- Not always risky, but worth considering depending on the sensitivity of the application.

## Command Recap
| Command | Purpose |
|---|---|
| `docker run -v "$(pwd):/path/in/container" <image>` | Bind mount host directory into container |
| `docker run -v "$(pwd)/file:/path/in/container/file" <image>` | Bind mount a single host file |
| `EXPOSE <port>` (Dockerfile) | Document that container listens on a port |
| `docker run -p <host-port>:<container-port>` | Publish/map a port from host to container |
| `docker run -p <container-port>` | Publish to an auto-assigned free host port |
| `docker run -p 127.0.0.1:<host-port>:<container-port>` | Publish but restrict access to localhost only |
| `EXPOSE <port>/udp` + `-p host:container/udp` | Restrict exposure/publishing to UDP protocol |
