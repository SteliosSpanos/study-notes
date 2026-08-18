# Section 3: In-Depth Dive into Images

## Where Images Come From
- When you run `docker run <image>`, if the image isn't found locally, Docker automatically searches **Docker Hub**.
- Any public image can be pulled/run directly, e.g. `docker run postgres`.

### Searching Docker Hub
```bash
docker search hello-world
```
Output columns: `NAME`, `DESCRIPTION`, `STARS`, `OFFICIAL`.

- **Official images** — curated/reviewed by Docker Inc., built from the `docker-library` repos.
  - Recognizable by `[OK]` in the `OFFICIAL` column and **no prefix** in the name (no `organisation/` part).
- Non-official images may be unmaintained or poorly documented — check carefully before use.

### Other Registries
- Docker Hub isn't the only registry — e.g. **Quay** (`quay.io`).
- `docker search` only searches Docker Hub by default; prefix with a registry to search elsewhere: `docker search quay.io/hello`.
- Pull from a non-Docker-Hub registry by specifying the host:
```bash
docker pull quay.io/podman/hello
```
- If no host is given, Docker Hub is assumed by default.

## Anatomy of an Image Name & Tags
- Full image reference format:
```
registry/organisation/image:tag
```
- Shorthand `ubuntu` defaults to: registry = Docker Hub, organisation = `library`, tag = `latest`.
- **Tags** version different builds of the same image:
```bash
docker pull ubuntu           # defaults to :latest
docker pull ubuntu:25.10     # specific version
```
- Note: `ubuntu:latest` doesn't always mean "newest build" — per Ubuntu's docs, it points to the latest **LTS** release.

### Local Tagging (aka "renaming")
```bash
docker tag ubuntu:25.10 ubuntu:questing_quokka
docker tag ubuntu:25.10 fav_distro:questing_quokka
```
- Creates a new local reference/alias to an existing image — check effects with `docker image ls`.

### Layers
- Images are composed of **layers**, downloaded in parallel for speed.
- More on layers/optimization in a later chapter.

## Building Your Own Image

### Example App
`hello.sh`:
```sh
#!/bin/sh

echo "Hello, docker!"
```
Test locally first:
```bash
chmod +x hello.sh
./hello.sh
```
(Windows users can skip `chmod` and instead do it inside the Dockerfile.)

### Writing a Dockerfile
```dockerfile
# Start from the alpine image that is smaller but no fancy tools
FROM alpine:3.21

# Use /usr/src/app as our workdir. The following instructions will be executed in this location.
WORKDIR /usr/src/app

# Copy the hello.sh file from this directory to /usr/src/app/ creating /usr/src/app/hello.sh
COPY hello.sh .

# Alternatively, if we skipped chmod earlier, we can add execution permissions during the build.
# RUN chmod +x hello.sh

# When running docker run the command will be ./hello.sh
CMD ["./hello.sh"]
```
- **Alpine Linux** is a popular small base image (used for smaller image sizes — more on this in ch.4). Ubuntu is fine too and has more debug tooling.
- **Pin specific versions** (e.g. `alpine:3.21`) rather than `latest` — avoids unexpected breaking changes and makes it clear when updates are needed for security fixes.

### Building the Image
```bash
docker build -t hello-docker .
```
- `docker build` looks for a file named `Dockerfile` by default.
- `.` = build context (current directory)
- `-t <name>` = tag/name the resulting image

Verify:
```bash
docker image ls
```

**Common errors:**
- `Permission denied` running `./hello.sh` → forgot `chmod +x hello.sh`; uncomment the `RUN chmod +x hello.sh` line in the Dockerfile.
- `not found` on Windows → file has CRLF line endings instead of LF; convert `hello.sh` to LF before building.

Run it:
```bash
docker run hello-docker
```

### Build Steps = Layers
- Build output shows steps like `[1/3]`, `[2/3]`, `[3/3]` — each corresponds to a new **layer** stacked on the base image.
- Fewer layers → less storage.
- Layers are **cached** — if only the later Dockerfile instructions change (e.g. editing `hello.sh`), Docker can reuse cached earlier layers and only rebuild from the changed step onward (`COPY` auto-detects file changes). Speeds up build pipelines. More on this in ch.4.

## Manually Creating Layers (`docker cp`, `docker diff`, `docker commit`)

### Copying a file into a running container
Terminal 1 — start container with a shell instead of the default CMD:
```bash
docker run -it hello-docker sh
```
Terminal 2 — find the container name/ID and copy a file in:
```bash
docker ps
touch additional.txt
docker cp ./additional.txt zen_rosalind:/usr/src/app/
```
Back in Terminal 1, confirm:
```bash
ls
# additional.txt  hello.sh
```

### Checking Changes: `docker diff`
```bash
docker diff zen_rosalind
```
Output prefixes:
- `A` = added
- `D` = deleted
- `C` = changed

### Saving Changes as a New Image: `docker commit`
```bash
docker commit zen_rosalind hello-docker-additional
docker image ls
```
- This adds a new layer on top of the existing image and saves it under a new name.
- **Not recommended for regular use** — better to track changes via the Dockerfile itself (version-controllable, reproducible, no "magic" manual steps).

## Adding Files via Dockerfile Instead (Preferred Approach)
```dockerfile
# Start from the alpine image
FROM alpine:3.21

# Use /usr/src/app as our workdir. The following instructions will be executed in this location.
WORKDIR /usr/src/app

# Copy the hello.sh file from this location to /usr/src/app/ creating /usr/src/app/hello.sh.
COPY hello.sh .

# Execute a command with `/bin/sh -c` prefix.
RUN touch additional.txt

# When running Docker run the command will be ./hello.sh
CMD ["./hello.sh"]
```
Build a versioned tag:
```bash
docker build -t hello-docker:v2 .
```
- `RUN` executes commands **during build time**, baking the result into the image — equivalent effect to the manual `docker commit` approach but reproducible and version-controlled.
- **Key distinction:** every Dockerfile instruction (`FROM`, `WORKDIR`, `COPY`, `RUN`, etc.) runs at **build time**, except `CMD` (and one more instruction covered later), which runs at **container start time** (`docker run`) — unless overridden.

## Multiple Dockerfiles in One Project
- Name alternate Dockerfiles as `Dockerfile.<something>` (e.g. `Dockerfile.testing`).
- Specify which one to use with `--file` / `-f`:
```bash
docker build -t tester -f Dockerfile.testing .
```

## Command Recap
| Command | Purpose |
|---|---|
| `docker search <term>` | Search Docker Hub for images |
| `docker pull <image>[:tag]` | Download an image without running it |
| `docker pull <registry>/<image>` | Pull from a non-default registry |
| `docker tag <src> <new>` | Create a local alias/tag for an image |
| `docker build -t <name> .` | Build an image from a Dockerfile in current dir |
| `docker build -t <name> -f <file> .` | Build using a specific Dockerfile |
| `docker cp <src> <container>:<path>` | Copy a file into a running container |
| `docker diff <container>` | Show filesystem changes in a container (A/D/C) |
| `docker commit <container> <new-image>` | Save a container's changes as a new image (not recommended vs. Dockerfile) |

## Dockerfile Instructions Covered So Far
| Instruction | Purpose | When it runs |
|---|---|---|
| `FROM` | Sets the base image | Build time |
| `WORKDIR` | Sets working directory for subsequent instructions | Build time |
| `COPY` | Copies files from host into the image | Build time |
| `RUN` | Executes a command, baking result into the image | Build time |
| `CMD` | Default command run when container starts | **Run time** (`docker run`), overridable |
