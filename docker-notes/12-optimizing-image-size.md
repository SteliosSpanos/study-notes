# Section 13: Optimizing the Image Size

## Why Image Size Matters
1. **Speed** — smaller images pull from the registry much faster.
2. **Security** — larger images have a bigger attack surface (more installed software = more potential vulnerabilities).

## What Is a Layer?
- Each instruction in a Dockerfile creates a **layer**.
- Rebuilding after a Dockerfile change only rebuilds the **changed layers** — this is what makes images lightweight and fast compared to other virtualization.
- The final image = combination of all layers.
- Fewer layers → generally smaller, cleaner images (though the real savings usually come from **what** each layer adds, not just the layer count).

## Baseline: The yt-dlp Image
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

RUN useradd -m appuser
RUN chown appuser .

USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
**Size: 869 MB**

## Step 1: Combine RUN Commands into One Layer
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg && \
    curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp && \
    chmod a+x /usr/local/bin/yt-dlp && \
    useradd -m appuser && \
    chown appuser .

USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
**Size: 866 MB** — only 3 MB smaller. Layer count alone isn't the big lever; what's actually installed matters more.

**Sidenote:** you can pin package versions (e.g. `curl=1.2.3`) to guarantee reproducible builds later — trade-off is that pinned old versions may accumulate security issues over time.

### Inspecting Layer Sizes
```bash
docker image history yt-dlp
```
```
IMAGE          CREATED          CREATED BY                                      SIZE
731a086e8f41   52 seconds ago   ENTRYPOINT ["/usr/local/bin/yt-dlp"]            0B
<missing>      52 seconds ago   USER appuser                                    0B
<missing>      52 seconds ago   RUN /bin/sh -c apt-get update && apt-get ins…   788MB
```
The single `RUN` layer is responsible for **788 MB** — this is where the real bloat is.

## Step 2: Clean Up After Installing
Remove apt's cached package lists (no longer needed after install):
```dockerfile
.. && \
rm -rf /var/lib/apt/lists/*
```
**Size: 807 MB**

Also remove `curl` and its dependencies once it's no longer needed at runtime:
```dockerfile
.. && \
apt-get purge -y --auto-remove curl && \
rm -rf /var/lib/apt/lists/*
```
**Size: 803 MB**

**General principle:** remove build-time-only tools/artifacts (source caches, package manager caches, unused dependencies) within the **same layer** that installed them — if removed in a later layer, the earlier layer still retains the original size in the image history.

## Alpine Linux Variant
- Alpine's base image is much smaller (~8 MB) — built on **musl** (alternative libc) and **busybox**, so not all software is 100% compatible, but works fine for many use cases including yt-dlp.

`Dockerfile.alpine`:
```dockerfile
FROM alpine:3.21

WORKDIR /mydir

RUN apk add --no-cache curl ffmpeg python3 ca-certificates && \
    curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp && \
    chmod a+x /usr/local/bin/yt-dlp && \
    adduser -D appuser && \
    chown appuser . && \
    apk del curl 

USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```

**Notes on Alpine specifics:**
- Package manager is **`apk`**, not `apt-get`.
- `--no-cache` avoids downloading/retaining package source caches (Alpine equivalent of cleaning up afterward).
- No `useradd` — use **`adduser -D`** instead.
- Most package names are similar; browse available packages at pkgs.alpinelinux.org.

Build with a distinguishing tag:
```bash
docker build -t yt-dlp:alpine-3.21 -f Dockerfile.alpine .
docker run -v "$(pwd):/mydir" yt-dlp:alpine-3.21 https://www.youtube.com/watch?v=bNw2i-mRT4I
```

**Size: 176 MB total** (single RUN layer = 168 MB) — dramatically smaller than the Ubuntu-based image (803 MB).

## Using a Preinstalled Language Environment
- Many programming languages have official preinstalled images on Docker Hub — often better than manually installing the runtime yourself.
- Example: use the official **Python** image instead of installing `python3` manually:

```dockerfile
# we are using a new base image
FROM python:3.12-alpine

WORKDIR /mydir

# no need to install python3 anymore
RUN apk add --no-cache curl ffmpeg ca-certificates && \
    curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp && \
    chmod a+x /usr/local/bin/yt-dlp && \
    adduser -D appuser && \
    chown appuser . && \
    apk del curl 

USER appuser

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```
**Size: 181 MB** — slightly *larger* than the manual-install Alpine version (176 MB), since the preinstalled Python image includes more than strictly needed. Convenience vs. minimal size is a trade-off.

## Publishing Multiple Image Variants
Tag distinctly to publish variants without overwriting each other:
```bash
docker image tag yt-dlp:alpine-3.21 <username>/yt-dlp:alpine-3.21
docker image push <username>/yt-dlp:alpine-3.21

docker image tag yt-dlp:python-alpine <username>/yt-dlp:python-alpine
docker image push <username>/yt-dlp:python-alpine
```

Or replace `latest` entirely with a new base (risky if consumers depend on the old base):
```bash
docker image tag yt-dlp:python-alpine <username>/yt-dlp
docker image push <username>/yt-dlp
```
**Important:** the `:latest` tag just means "the most recently pushed image without an explicit tag" — it says nothing about content/stability; anyone depending on it could be surprised by underlying changes (e.g. Ubuntu → Alpine base).

## Multi-Stage Builds
- Useful when you need tools **only for building**, not for running the final app (e.g., a compiler/toolchain that shouldn't ship in the runtime image).

### Example: Jekyll Static Site + Nginx
Single-stage version (Ruby + Jekyll, for testing):
```dockerfile
FROM ruby:3

WORKDIR /usr/app

RUN gem install jekyll
RUN jekyll new .
RUN jekyll build
```
Add a CMD to test locally:
```dockerfile
CMD ["bundle", "exec", "jekyll", "serve", "--host", "0.0.0.0"]
```
```bash
docker build -t jekyll .
docker run -p 4000:4000 jekyll
# visit http://localhost:4000
```

### Converting to Multi-Stage
```dockerfile
# the first stage needs to be given a name
FROM ruby:3 AS build-stage
WORKDIR /usr/app

RUN gem install jekyll
RUN jekyll new .
RUN jekyll build

# we will now add a new stage
FROM nginx:1.19-alpine

COPY --from=build-stage /usr/app/_site/ /usr/share/nginx/html
```
- First stage (`build-stage`) — has Ruby + Jekyll, builds the static site.
- Second stage — starts fresh from a lightweight `nginx:1.19-alpine` image, and **only copies the built static output** (`_site/`) from the first stage using `COPY --from=build-stage`.
- The final image contains **no Ruby, no gems, no build tools** — just Nginx + static HTML/CSS/JS.
- You can also reference an external image as a build stage, e.g. `--from=python:3.12`.

### Result
```bash
docker build -t jekyll:nginx .
docker image ls
```
```
REPOSITORY    TAG     IMAGE ID         CREATED           SIZE
jekyll        nginx   9e2f597ad99e     8 seconds ago     35.3MB
jekyll        ruby    5dae3d9f8dfb     26 minutes ago    1.57GB
```
- Final image: **35.3 MB** vs. the single-stage Ruby-based build at **1.57 GB** — massive reduction.
```bash
docker run -it -p 8080:80 jekyll:nginx
```

**Best practice tip:** for maximum minimalism/security, consider using `FROM scratch` as the final stage's base when possible — it contains absolutely nothing except what you explicitly add.

## Summary Table of Size Reductions (yt-dlp example)
| Version | Size |
|---|---|
| Ubuntu, multiple RUN layers | 869 MB |
| Ubuntu, combined RUN layers | 866 MB |
| + removed apt lists | 807 MB |
| + purged curl | 803 MB |
| Alpine (manual Python install) | 176 MB |
| Alpine + preinstalled Python image | 181 MB |

## Key Takeaways
| Technique | Effect |
|---|---|
| Combine RUN commands | Minor size reduction, but improves build cache efficiency |
| Clean up caches/temp files in the same layer | Prevents bloat from persisting in image history |
| Remove build-only dependencies (`apt-get purge`, `apk del`) | Shrinks final image significantly |
| Use Alpine base images | Much smaller footprint than Ubuntu/Debian-based images |
| Use official preinstalled language images | Saves manual setup, sometimes at a small size cost |
| Multi-stage builds | Keep build tools out of the final runtime image entirely — often the biggest win |
| `FROM scratch` | Minimal possible base — best for security/size when feasible |
