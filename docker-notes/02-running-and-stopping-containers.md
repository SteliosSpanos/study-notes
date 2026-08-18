# Section 2: Running and Stopping Containers

## Running Ubuntu
```bash
docker run ubuntu
```
- Pulls and runs the image, but exits immediately — it tries to open a shell, but without the right flags there's nothing to interact with.

## Key Flags: `-i`, `-t`, `-d`
- `-t` → allocates a **tty** (terminal), lets you see a shell prompt.
- `-i` → passes **STDIN** to the container (interactive input).
- `-d` → runs **detached** (in the background).

```bash
docker run -it ubuntu
root@2eb70ecf5789:/# ls
bin  boot  dev  etc  home  lib  ...
```
- Combine `-i` and `-t` as `-it` to get a fully interactive shell.
- Exit the container shell with `exit`.

## Running a Named Background Container
```bash
docker run -d -it --name looper ubuntu sh -c 'while true; do date; sleep 1; done'
```
- `-d` — detached
- `-it` — interactive + tty
- `--name looper` — gives the container a friendly name to reference later
- Everything after the image name (`ubuntu`) is the command run inside the container

> **Windows CMD note:** use double quotes instead of single quotes around the script.

Check it's running:
```bash
docker container ls
```

## Viewing Logs
```bash
docker logs -f looper
```
- `-f` follows the log output live (like `tail -f`).

## Pause / Unpause
```bash
docker pause looper
docker unpause looper
```
- Pausing freezes the container's processes; log output pauses too.

## Attaching to a Running Container
```bash
docker attach looper
```
- Brings the container's process (STDOUT) to the foreground in that terminal.
- **Ctrl+C** while attached (with STDIN attached) will **stop the container**, since it kills the foreground process.

### Attaching without STDIN
```bash
docker start looper
docker attach --no-stdin looper
```
- `--no-stdin` means Ctrl+C only disconnects your terminal from STDOUT — the container **keeps running**.

## Running Commands Inside a Running Container: `docker exec`
```bash
docker exec looper ls -la
```
- Runs a one-off command inside the running container.

Interactive shell inside a running container:
```bash
docker exec -it looper bash
root@2a49df3ba735:/# ps aux
```
- Lets you poke around live, check processes (`ps aux`), etc.
- Exit the shell with `exit` (this does NOT stop the container, since the main process — `sh -c 'while true...'` — is still running).

## Stopping vs Killing
- `docker stop` sends **SIGTERM**, waits a grace period, then sends **SIGKILL** if needed.
- Some processes (like our `looper` loop) ignore SIGTERM, so `stop` is slow — `kill` is faster since it sends SIGKILL immediately.

```bash
docker kill looper && docker rm looper
# equivalent to:
docker rm --force looper
```

## Auto-Removing Containers with `--rm`
```bash
docker run -d --rm -it --name looper-it ubuntu sh -c 'while true; do date; sleep 1; done'
```
- `--rm` automatically deletes the container once it exits — keeps things clean, no leftover stopped containers.
- **Caveat:** since the container is removed on exit, `docker start` can't be used to restart it afterward.

## Detaching Without Stopping: Ctrl+P, Ctrl+Q
```bash
docker attach looper-it
# press Ctrl+P then Ctrl+Q
```
- This detaches from the container **without stopping it** (unlike Ctrl+C, which sends a kill signal and — combined with `--rm` — removes the container).

## Multi-Platform / Architecture Notes (M1/M2 Mac)
- Some images only support specific architectures (e.g., `linux/amd64`). Running them on ARM (Apple Silicon) triggers a warning:
```
WARNING: The requested image's platform (linux/amd64) does not match the detected
host platform (linux/arm64/v8) and no specific platform was requested
```
- Docker Desktop for Mac can still run these via **emulation**, but performance may be worse than native.
- Popular images (like `ubuntu`) are **multi-platform images** — they bundle variants for different architectures, so Docker auto-selects the right one and no warning appears.

## A Container Is Just Ubuntu (Or Whatever the Base Image Is)
```bash
docker run -it ubuntu
root@881a1d4ecff2:/# ls
root@881a1d4ecff2:/# ps
root@881a1d4ecff2:/# date
```
- Behaves like a normal minimal Ubuntu install.

### Installing Packages Inside a Container
```bash
apt-get update
apt-get -y install nano
```
- Works just like on a normal Ubuntu machine.
- **Important:** These changes are NOT permanent — if the container is removed, everything installed inside it is lost.
- To make custom setups permanent/reusable, you build a custom **image** (covered in a later section via Dockerfiles).

## Command Recap
| Command | Purpose |
|---|---|
| `docker run -it <image>` | Run interactively with a tty |
| `docker run -d -it --name <name> <image> <cmd>` | Run detached, named, interactive |
| `docker logs -f <container>` | Follow container logs |
| `docker pause` / `docker unpause` | Pause/resume a container's processes |
| `docker attach <container>` | Attach terminal to container's STDOUT |
| `docker attach --no-stdin <container>` | Attach without exposing STDIN (safe from Ctrl+C stop) |
| `docker exec <container> <cmd>` | Run a one-off command in a running container |
| `docker exec -it <container> bash` | Open interactive shell in a running container |
| `docker stop <container>` | Graceful stop (SIGTERM → SIGKILL after grace period) |
| `docker kill <container>` | Immediate stop (SIGKILL) |
| `docker rm <container>` / `docker rm --force <container>` | Remove container (force = stop + remove) |
| `docker run --rm ...` | Auto-remove container on exit |
| Ctrl+P, Ctrl+Q | Detach from attached container without stopping it |
| Ctrl+C (while attached w/ STDIN) | Stops the container (kills foreground process) |
