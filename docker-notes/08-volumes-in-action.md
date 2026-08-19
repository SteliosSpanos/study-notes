# Section 8: Volumes in Action

## The Project
Setting up a real multi-service app: **Redmine** (project management), **PostgreSQL** (database), and **Adminer** (DB admin GUI). All have official Docker images.

- "Official" status isn't critical, just a signal that the image is well-supported.
- Same approach could be used for WordPress, MediaWiki, Sentry, etc.

## Setting Up Postgres

Stripped-down compose service based on the official Postgres docs:
```yaml
services:
  db:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: example
    container_name: db_redmine
```
- `restart: unless-stopped` — keeps the container running unless **explicitly stopped**, unlike `restart: always` which would also restart it after a reboot.
- Per Postgres docs (v18+), `/var/lib/postgresql` should be mounted separately so data persists.

## Anonymous Volumes (What Happens by Default)
Running `docker compose up` with no volume defined still creates a volume automatically (because the Postgres image itself declares one):

```bash
docker container inspect db_redmine | grep -A 5 Mounts
```
```
"Mounts": [
    {
        "Type": "volume",
        "Name": "b6151a106a0f8b8f...",
        "Source": "/var/lib/docker/volumes/b6151a106a0f8b8f.../_data",
        "Destination": "/var/lib/postgresql",
```
- This is an **anonymous volume** — random name, not explicitly managed.
```bash
docker volume ls
docker volume prune   # remove unused volumes
docker compose down   # tear down the stack
```

## Defining a Named Volume Explicitly
```yaml
services:
  db:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: example
    container_name: db_redmine
    volumes:
      - database:/var/lib/postgresql

volumes:
  database:
```
```bash
docker volume ls
# redmine_database
```
- Compose prefixes the volume name with the **project name** (derived from the directory containing the compose file) — e.g. `redmine_database`.
- Project name can be overridden with the `COMPOSE_PROJECT_NAME` env var.
- Named volumes are much more human-readable/manageable than anonymous ones.

## Adding Redmine
```yaml
redmine:
  image: redmine:6.1
  environment:
    - REDMINE_DB_POSTGRES=db
    - REDMINE_DB_PASSWORD=example
  ports:
    - 9999:3000
  depends_on:
    - db
```
- `depends_on` ensures `db` **starts before** `redmine` — but does **NOT** guarantee the database is actually ready to accept connections yet.
- Redmine connects to Postgres via the DNS name `db` (the service name) — standard Docker networking (see earlier section).

On `docker compose up`, Redmine runs DB migrations automatically, then starts its web server (Puma) on port 3000 (mapped to host 9999).

### Persisting Redmine's Own Files
- Redmine's Dockerfile also declares a volume (`/usr/src/redmine/files`) — without explicit config, this also becomes an unmanaged **anonymous volume**.
- Better to name it explicitly:

```yaml
services:
  db:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: example
    container_name: db_redmine
    volumes:
      - database:/var/lib/postgresql
  redmine:
    image: redmine:6.1
    environment:
      - REDMINE_DB_POSTGRES=db
      - REDMINE_DB_PASSWORD=example
    ports:
      - 9999:3000
    depends_on:
      - db
    volumes:
      - files:/usr/src/redmine/files

volumes:
  database:
  files:
```

App now accessible at `http://localhost:9999`.

## Verifying Volume Usage with `docker container diff`
```bash
docker container diff $(docker compose ps -q redmine)
```
- Shows filesystem changes inside the container.
- If all changes are confined to `/usr/src/redmine/...` (the mounted volume path), it confirms persistent data is correctly isolated to the volume — nothing meaningful is being written elsewhere in the container's ephemeral layer.

## Interacting with Postgres Directly
```bash
docker container exec -it db_redmine psql -U postgres
```
- Note: `docker compose exec` (Compose's version) allocates a TTY and enters interactive mode **by default**, without needing `-it` (unlike plain `docker exec`).

### Backups with `pg_dump`
```bash
docker container exec db_redmine pg_dump -U postgres > redmine.dump
```

## Adding Adminer (DB Admin GUI)
```yaml
adminer:
  image: adminer:5
  restart: always
  environment:
    - ADMINER_DESIGN=galkaev
  ports:
    - 8083:8080
```
- Accessible at `http://localhost:8083`.
- Adminer connects to Postgres over the Docker network automatically — **assumes** the DB's DNS name is `db` by default.
- If your DB service has a different name, tell Adminer explicitly:
```yaml
adminer:
  environment:
    - ADMINER_DEFAULT_SERVER=database_server
```

## Bind Mount vs Docker-Managed Volume
| | Bind Mount | Docker-Managed Volume |
|---|---|---|
| Location | You choose the exact host path | Docker controls the location (`/var/lib/docker/volumes/...`) |
| Backups | Easy — you know exactly where the data lives | Less trivial — location isn't directly controlled |
| Use case | When you want visibility/control over data location | Good default, simpler declaration |

Example bind mount for Postgres data:
```yaml
services:
  db:
    image: postgres:18
    volumes:
      - ./database:/var/lib/postgresql
```
- Deleting the local `./database` folder wipes the data — useful to understand for testing persistence behavior (stop container, delete folder, restart → data is gone).

## Reverse Proxy with Nginx
Goal: single entry point (`http://localhost:8000`) routing:
- `/api/*` → backend
- everything else → frontend

```yaml
services:
  frontend:
    # ...
  backend:
    # ...
  redis:
    # ...
  db:
    # ...
  nginx:
    image: nginx
    ports:
      - 8000:80
    # ...
```

`nginx.conf` (bind-mounted into the container at `/etc/nginx/nginx.conf`):
```nginx
events { worker_connections 1024; }

http {
  server {
    # this is the port that Nginx container is set up to listen
    listen 80;

    # configure here where requests to http://localhost:8000/...
    # are forwarded
    location / {
      proxy_pass _frontend-url_;
    }

    # configure here where requests to http://localhost:8000/api/...
    # are forwarded
    location /api/ {
      proxy_set_header Host $host;
      proxy_pass _backend-url_;
    }
  }
}
```
- `_frontend-url_` / `_backend-url_` should be replaced with the internal Docker network URLs (using service names), e.g. `http://frontend:3000` and `http://backend:4000/` — since Nginx, backend, and frontend all share the same Docker network.
- **Gotcha:** trailing slash matters on the `/api/` proxy_pass target, or requests won't route correctly (per Nginx `proxy_pass` semantics).
- Debug via container logs — e.g. `invalid URL prefix in /etc/nginx/nginx.conf` indicates a malformed `proxy_pass` value.

## Publishing Ports: What It Actually Does
Key networking lesson:

- **Container ports are always reachable by other containers on the same Docker network**, even if you never publish them.
- **Publishing a port (`-p host:container` or `ports:` in Compose)** is *only* needed to expose a service **outside** the Docker network — i.e., to your host machine / the internet.
- If a service (like `db` or `redis`) only ever needs to be reached by other containers, **don't publish its port at all** — no `ports:` entry needed.

This means a well-locked-down setup (per this section's Redmine/Postgres/backend examples) publishes **only** the reverse proxy's port (e.g. `8000:80`), keeping frontend, backend, redis, and db reachable only within the Docker network.

## Command Recap
| Command | Purpose |
|---|---|
| `docker container inspect <name> \| grep -A 5 Mounts` | Inspect a container's volume mounts |
| `docker volume ls` | List volumes |
| `docker volume prune` | Remove unused volumes |
| `docker volume rm <name>` | Remove a specific volume |
| `docker container diff <container>` | Show filesystem changes (A/D/C) |
| `docker compose exec <service> <cmd>` | Run interactive command in a Compose-managed container (TTY by default) |
| `docker container exec <name> pg_dump -U postgres > file.dump` | Backup a Postgres DB |
| `docker compose ps -q <service>` | Get container ID for a Compose service |

## Key Takeaways
| Concept | Summary |
|---|---|
| Anonymous volume | Auto-created if image declares a volume path but you don't name one; hard to track/manage |
| Named volume | Explicit `volumes:` block; Compose prefixes with project name |
| Bind mount | Host path is explicit and known — easier backups, but less portable |
| `depends_on` | Controls **start order** only, not readiness |
| `restart: unless-stopped` vs `always` | `unless-stopped` won't restart after reboot if manually stopped; `always` will |
| Port publishing | Only required for access from **outside** the Docker network; inter-container access works without it |
