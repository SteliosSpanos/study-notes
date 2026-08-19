# Section 11: Using a Non-Root User

## Why This Matters
- By default, processes inside a container run as **root**.
- Security risk: a bug in Docker or the Linux kernel could theoretically let a process **escape the container**, in which case running as root inside the container is far more dangerous than running as an unprivileged user.
- Two mitigation strategies:
  1. **Add and use a non-root user inside the container** (covered here).
  2. Use [user namespace remapping](https://docs.docker.com/engine/security/userns-remap/) — maps the container's root user to a high, non-existent UID on the host, for cases where you must run as root *inside* the container.

## Starting Point: The yt-dlp Dockerfile (from Chapter 3)
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```

## Adding a Non-Root User
```dockerfile
RUN useradd -m appuser
```
- Creates a new user `appuser` (`-m` creates a home directory for them).

### Switching to That User with `USER`
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

RUN useradd -m appuser
USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
- **Everything after `USER appuser`** — including `CMD`/`ENTRYPOINT` — runs as that user, not root.

## The Catch: Permission Errors
Build and run without a bind mount:
```bash
docker build -t yt-dlp .
docker run yt-dlp https://www.youtube.com/watch?v=XsqlHHTGQrw
```
Result:
```
[download] Unable to open file: [Errno 13] Permission denied: '...f137.mp4.part'. Retrying...
ERROR: unable to open for writing: [Errno 13] Permission denied: ...
```
- `appuser` doesn't have write permission to the container's filesystem (e.g. `/mydir`), so file downloads fail.

### Two Ways to Handle This
1. **Just don't fix it** — if the intended usage is to always run with `/mydir` bind-mounted from the host (as covered in the volumes section), the mounted directory's permissions from the host apply, and it may work fine without any in-container permission changes.
2. **Fix it explicitly with `chown`** — grant `appuser` ownership of the working directory.

### Granting Write Permission Properly
**Important ordering rule:** permission changes (like `chown`) must happen **while still running as root** — i.e., **before** the `USER` directive switches the active user.

```dockerfile
FROM ubuntu:24.04

# ...

WORKDIR /mydir

# create the appuser
RUN useradd -m appuser

# change the owner of current dir to appuser
RUN chown appuser .

# now we can change the user
USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
- `chown appuser .` changes ownership of `/mydir` (the current `WORKDIR`) to `appuser`, executed while the build process is still root.
- Only **after** ownership is set does `USER appuser` switch the active user for all subsequent instructions and the running container.

## Key Takeaways
| Concept | Summary |
|---|---|
| Default container user | `root`, unless explicitly changed — a security risk if the container is ever compromised |
| `RUN useradd -m <name>` | Creates a new non-root user (with home directory) inside the image |
| `USER <name>` | Switches the active user for all subsequent Dockerfile instructions and the running container |
| Permission changes (`chown`) | Must be done **before** `USER` switches away from root |
| Bind-mounted directories | Inherit host permissions — may sidestep in-container permission issues depending on setup |
| Alternative: userns-remap | Maps container root to an unprivileged host UID, for cases requiring root inside the container |
