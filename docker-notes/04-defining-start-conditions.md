# Section 4: Defining Start Conditions for the Container

## Workflow: Explore Interactively, Then Write the Dockerfile
When building a more complex image, a good approach is to first explore inside an interactive container to figure out exactly what's needed, then encode those steps into the Dockerfile.

Example: containerizing **yt-dlp** (a YouTube/Imgur video downloader).

```bash
docker run -it ubuntu:24.04
```

Inside the container, work through the install step by step:
```bash
apt-get update && apt-get install -y curl
curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
chmod a+rx /usr/local/bin/yt-dlp
yt-dlp
# /usr/bin/env: 'python3': No such file or directory
apt-get install -y python3 ffmpeg
yt-dlp
# now works, just needs a URL argument
```
- No `sudo` needed — you're `root` inside the container by default.
- `python3` and `ffmpeg` are required dependencies for yt-dlp.

## Writing the Dockerfile from What You Learned
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+rx /usr/local/bin/yt-dlp

CMD ["/usr/local/bin/yt-dlp"]
```
- **Best practice:** put the most likely-to-change instructions **near the bottom** of the Dockerfile. This preserves cached layers for the earlier (more stable) steps and speeds up rebuilds.
- `WORKDIR /mydir` ensures downloaded videos land in a predictable location.

Build & run:
```bash
docker build -t yt-dlp .
docker run yt-dlp
```

## The Problem: `CMD` Gets Replaced, Not Appended To
```bash
docker run yt-dlp https://www.youtube.com/watch?v=uTZSILGTskA
# Error: tries to execute the URL itself as a command — fails
```
- Any argument given after the image name **replaces** the `CMD` entirely, it doesn't append to it.
```bash
docker run -it yt-dlp ps      # runs `ps` instead of yt-dlp
docker run -it yt-dlp pwd     # runs `pwd` instead of yt-dlp
```

## Solution: `ENTRYPOINT`
`ENTRYPOINT` defines the **main executable**; anything passed on `docker run` is **appended as arguments** to it (rather than replacing it).

```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

# Replacing CMD with ENTRYPOINT
ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
```bash
docker build -t yt-dlp .
docker run yt-dlp https://www.youtube.com/watch?v=uTZSILGTskA
# now correctly runs: /usr/local/bin/yt-dlp https://www.youtube.com/watch?v=uTZSILGTskA
```

## ENTRYPOINT vs CMD

- **Default entrypoint** (if none set) is `/bin/sh -c`, which is why plain `CMD ["./script.sh"]` works — it's passed as an argument to `/bin/sh -c`.
- If **both** `ENTRYPOINT` and `CMD` are defined, `CMD` provides **default arguments** to the `ENTRYPOINT`, and these defaults can be overridden on the command line.

```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

ENTRYPOINT ["/usr/local/bin/yt-dlp"]

# define a default argument
CMD ["https://www.youtube.com/watch?v=Aa55RKWZxxI"]
```
```bash
docker run yt-dlp
# downloads the default URL from CMD

docker run yt-dlp https://www.youtube.com/watch?v=DptFY_MszQs
# overrides CMD's default with this URL instead
```

## Exec Form vs Shell Form
- **Exec form** (preferred): `["executable", "arg1", "arg2"]` — runs the command directly, no shell wrapping.
- **Shell form**: `command arg1 arg2` (no brackets) — gets wrapped in `/bin/sh -c '...'`, which can cause unexpected behavior but is useful when you need shell features like environment variable expansion (e.g. `$MYSQL_PASSWORD`).

| Dockerfile | Resulting command |
|---|---|
| `ENTRYPOINT /bin/ping -c 3` <br> `CMD localhost` | `/bin/sh -c '/bin/ping -c 3'` (CMD ignored — only one shell-form command runs) |
| `ENTRYPOINT ["/bin/ping","-c","3"]` <br> `CMD localhost` | `/bin/ping -c 3` (shell-form CMD ignored due to exec-form ENTRYPOINT) |
| `ENTRYPOINT /bin/ping -c 3` <br> `CMD ["localhost"]` | `/bin/sh -c '/bin/ping -c 3'` (exec-form CMD ignored due to shell-form ENTRYPOINT) |
| `ENTRYPOINT ["/bin/ping","-c","3"]` <br> `CMD ["localhost"]` | `/bin/ping -c 3 localhost` (both exec form → combines correctly) |

**Rule of thumb:** use exec form (with brackets) for both `ENTRYPOINT` and `CMD` so they combine as expected.

## Practical Guidance
- Most of the time, you can **ignore ENTRYPOINT and just use CMD** — many base images (like Ubuntu) already default `ENTRYPOINT` to `sh`, letting you freely override `CMD` (e.g. to `bash` for interactive debugging).
- Example — Python's official image:
```bash
docker run -it python:3.11              # drops into Python REPL (CMD = python3)
docker run -it python:3.11 --version    # FAILS — tries to execute "--version" as a binary
docker run -it python:3.11 bash         # overrides CMD with bash shell instead
```
This tells us Python's image has an `ENTRYPOINT` that is NOT `python3` — otherwise `--version` would work as an argument.

Custom example — making Python itself the entrypoint:
```dockerfile
FROM python:3.11
ENTRYPOINT ["python3"]
CMD ["--help"]
```
Now arguments like `--version` get appended to `python3` correctly, and omitting arguments shows the help text (from CMD default).

## Remaining Problems with yt-dlp Image
1. **Major:** Downloaded files stay trapped inside the container filesystem.
2. **Minor:** Too many layers → larger image size (addressed in a later chapter on optimization).

### Workaround (not the proper fix): `docker cp`
Find the most recent container:
```bash
docker container ls -a --last 3
```
Check what changed:
```bash
docker diff determined_elion
# A /mydir/Welcome to Kumpula campus! ｜ University of Helsinki [DptFY_MszQs].mkv
```
Copy the file out to the host (quote paths with spaces):
```bash
docker cp "determined_elion://mydir/Welcome to Kumpula campus! ｜ University of Helsinki [DptFY_MszQs].mkv" .
```
- This works but isn't a proper solution — a better fix (likely **volumes**) is coming in a later section.

## Applied Example: "Improved Curler" with ENTRYPOINT
Script (`script.sh`):
```bash
#!/bin/bash

echo "Searching..";
sleep 1;
curl http://$1;
```
Dockerfile change: use `ENTRYPOINT ["./script.sh"]` instead of `CMD`.

```bash
docker build -t curler-v2 .
docker run curler-v2 helsinki.fi
```
- The argument `helsinki.fi` is passed as `$1` into the script via `ENTRYPOINT`, making the image flexible/reusable for different inputs.

## Key Takeaways
| Concept | Summary |
|---|---|
| `CMD` alone | Fully replaced by any `docker run` arguments |
| `ENTRYPOINT` alone | `docker run` arguments are appended to it |
| `ENTRYPOINT` + `CMD` | `CMD` = default args to `ENTRYPOINT`, overridable at runtime |
| Exec form `["cmd","arg"]` | Preferred; runs directly, no shell wrapping |
| Shell form `cmd arg` | Wrapped in `/bin/sh -c`; useful for env var expansion but only one shell-form directive takes effect |
| Layer ordering | Put frequently-changing instructions last to maximize build cache reuse |
