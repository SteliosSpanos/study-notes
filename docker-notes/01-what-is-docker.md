# Section 1: What is Docker?

## Definition
- Docker is a set of tools (PaaS products) that use OS-level virtualization to deliver software in **containers**.
- **Containers** = packages of software (app + its dependencies/libraries).
- Containers are **isolated** so they don't interfere with each other or the host system, but Docker provides tools to let them interact when needed.

## Benefits of Containers

### 1. "Works on my machine" problem
- Without containers: app works on dev machine, passes tests, but fails on the server due to environment differences.
- Containers solve this by packaging the app **with all its dependencies**, so it runs the same everywhere.
- Note: "works in my container" issues can still happen, but they're usually just usage errors.

### 2. Isolated environments
- Example: 5 Python apps need to run alongside an app requiring Python 2.7.
- Containers let each app bundle its own dependency versions, avoiding conflicts on the same machine.

### 3. Development
- Complex apps depend on services like Postgres, MongoDB, Redis, etc.
- Instead of manually installing/managing each on your machine, Docker lets you spin up isolated instances with a single command.

### 4. Scaling
- Starting/stopping containers has low overhead.
- Enables spinning up multiple containers to meet demand and load-balance traffic (container orchestration — covered in later parts of the course).
- Orchestration example: if a container dies, the system detects it, reroutes traffic to healthy replicas, and spins up a replacement.

## Virtual Machines vs Containers
- **VMs**: run on a hypervisor, each VM has its **own full OS** + binaries/libraries → heavier, more resource-intensive, stronger isolation, slower startup.
- **Containers**: share the **host OS kernel**, only package the app + dependencies → lightweight, faster startup, better resource use, more portable, but weaker (process-level) isolation.
- Docker relies on the **Linux kernel**. macOS/Windows can't run Docker natively — e.g., Docker for Mac runs a Linux VM under the hood.

## Running Your First Container
```bash
docker container run hello-world
# shorthand:
docker run hello-world
```
- First run: Docker pulls (downloads) the image, then creates and runs a container from it.
- Subsequent runs: uses the local image, skips downloading.
- **Security note:** Always be cautious about what images you download/run from the internet.

## Images vs Containers (Cooking Metaphor)
- **Image** = recipe + ingredients (blueprint/template). Immutable — can't be edited once built. New images are made by adding layers on top of a base image.
- **Container** = the ready-to-eat meal — a running **instance** of an image.
- One image can be used to create multiple separate containers.

### Dockerfile
- A file (default name `Dockerfile`) with instructions to build an image, parsed by `docker image build`.
```dockerfile
FROM <image>:<tag>

RUN <install some dependencies>

CMD <command that is executed on `docker container run`>
```
- Relationship: **Dockerfile** (written by you) → builds **Image** (written by Docker) → runs as **Container**.

## Listing Images & Containers
```bash
docker image ls          # list images
docker container ls      # list running containers
docker container ls -a   # list ALL containers (including stopped)
# shorthand:
docker ps -a
```

## Docker Engine Architecture
Three parts:
1. **CLI client** — what you type commands into
2. **REST API**
3. **Docker daemon** — does the actual work (manages images, containers, etc.)

The CLI sends requests to the daemon via the REST API.

## Removing Images & Containers
- Can't remove an image while a container still references it:
```bash
docker image rm hello-world
# Error: conflict - container is using the image (must force)
```
- Must remove the dependent container(s) first:
```bash
docker container rm <container-name-or-id>
```
  - Can use just the first few characters of the ID (as long as it's unambiguous).
  - Multiple IDs can be passed at once: `docker container rm id1 id2 id3`
- Filter container list with `grep`:
```bash
docker container ls -a | grep hello-world
```
- Bulk cleanup:
```bash
docker container prune   # remove all stopped containers
docker image prune       # remove dangling (unnamed/unused) images
docker system prune      # clear almost everything
```
- Pull an image without running it:
```bash
docker image pull hello-world
```

## Running Containers in the Background
```bash
docker run -d nginx
```
- `-d` = detached mode, runs in background (otherwise the terminal blocks/freezes since the container runs in the foreground).
- Check running containers: `docker container ls`

## Stopping & Force-Removing Containers
```bash
docker container stop <name>          # stop first
docker container rm <name>            # then remove
# or force remove directly:
docker container rm --force <name>
```
- Name/ID/partial-ID are all interchangeable in these commands.

## Most Used Commands Cheat Sheet

| Command | Explanation | Shorthand |
|---|---|---|
| `docker image ls` | Lists all images | `docker images` |
| `docker image rm <image>` | Removes an image | `docker rmi` |
| `docker image pull <image>` | Pulls image from a registry | `docker pull` |
| `docker container ls -a` | Lists all containers | `docker ps -a` |
| `docker container run <image>` | Runs a container from an image | `docker run` |
| `docker container rm <container>` | Removes a container | `docker rm` |
| `docker container stop <container>` | Stops a container | `docker stop` |
| `docker container exec <container>` | Executes a command inside the container | `docker exec` |
