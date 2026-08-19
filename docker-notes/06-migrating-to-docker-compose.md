# Section 6: Migrating to Docker Compose

## Why Docker Compose?
- Working with plain `docker` CLI involves lots of options (build, push, run flags, volumes, ports, etc.).
- **Docker Compose** streamlines multi-step / multi-container Docker workflows into a single config file + simple commands.
- Compose used to be a standalone tool, now it's fully integrated into Docker (`docker compose ...`).

### File Naming
- Preferred extension: `.yaml` (though `.yml` also works — both are common in the wild).
- Filename options: `docker-compose.yaml` (older/traditional) or `compose.yaml` (newer standard).

General command form:
```bash
docker compose [-f <arg>...] [options] [COMMAND] [ARGS...]
```

## Basic Compose File (Building from a Dockerfile)
Given this Dockerfile (yt-dlp example from before):
```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3 ffmpeg
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```

`docker-compose.yaml`:
```yaml
services:
  yt-dlp-ubuntu:
    image: <username>/<repositoryname>
    build: .
```
- Defines one **service** named `yt-dlp-ubuntu`.
- `build:` can be a path (e.g. `.` for current directory) or an object with `context` and `dockerfile` keys for more control.

Build & push:
```bash
docker compose build
docker compose push
```

## Volumes in Docker Compose
Syntax: `location-in-host:location-in-container` (relative paths OK, no need for absolute path).

```yaml
services:
  yt-dlp-ubuntu:
    image: <username>/<repositoryname>
    build: .
    volumes:
      - .:/mydir
    container_name: yt-dlp
```
- `container_name` sets a fixed name for the running container.
- Run using the **service name**:
```bash
docker compose run yt-dlp-ubuntu https://www.youtube.com/watch?v=saEpkcVi1d4
```

## Using Pre-Built Images (No `build` Needed)
If you're just running existing images, omit `build`:
```yaml
services:
  nginx:
    image: nginx:1.29
  database:
    image: postgres:18
```
Start both:
```bash
docker compose up
```
- Note: in the example, Postgres fails to start because `POSTGRES_PASSWORD` wasn't set — a reminder that some images need required environment variables/config to run properly.

## Key Compose Commands
| Command | Purpose |
|---|---|
| `docker compose up` | Start all services defined in the compose file |
| `docker compose up -d` | Start services in detached mode |
| `docker compose down` | Stop and remove running services |
| `docker compose logs` | View logs of running services |
| `docker compose ps` | List services and their status |
| `docker compose build` | Build images defined with `build:` |
| `docker compose push` | Push built images to a registry |
| `docker compose run <service> [args]` | Run a specific service (like `docker run`) |

## Web Services with Docker Compose
Example: [jwilder/whoami](https://github.com/jwilder/whoami) — a simple service that returns its container ID/hostname.

Plain docker (for comparison):
```bash
docker container run -d -p 8000:8000 jwilder/whoami
curl http://localhost:8000
docker container stop <id>
docker container rm <id>
```

Equivalent as Compose (`whoami/docker-compose.yaml`):
```yaml
services:
  whoami:
    image: jwilder/whoami
    ports:
      - 8000:8000
```
```bash
docker compose up -d
curl localhost:8000
docker compose down
```

## Environment Variables in Compose
```yaml
services:
  backend:
    image: <some-image>
    environment:
      - VARIABLE=VALUE
      - VARIABLE2=VALUE2
```
- Other, possibly more elegant methods exist for defining env vars in Compose (e.g. `.env` files) — worth exploring further as needed.

## Summary: Compose File Anatomy So Far
| Key | Purpose |
|---|---|
| `services:` | Top-level key listing each container/service |
| `<service-name>:` | Name used to reference the service (e.g. in `docker compose run <name>`) |
| `image:` | Image to use (pulled if no `build`, or tagged name if built) |
| `build:` | Path (or context/dockerfile object) to build the image from a Dockerfile |
| `volumes:` | List of `host:container` bind mounts |
| `ports:` | List of `host:container` port mappings |
| `container_name:` | Fixed name for the resulting container |
| `environment:` | List of environment variables passed into the container |
