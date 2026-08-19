# Section 7: Docker Networking

## Connecting Services
- Connecting two services (e.g. a server + database) is done via a **Docker network**.
- **Docker Compose automatically** creates a network and joins all services in a `docker-compose.yaml` file to it, complete with internal **DNS**.
- Containers can reference each other by their **service name** (as defined in the compose file) — this is different from the container name.

### Example: `webapp` + `webapp-helper`
- `webapp-helper` listens on port 3000.
- `webapp` can reach it simply via `webapp-helper:3000` — Compose's internal DNS resolves it.
- No need to publish ports outside the network for internal service-to-service communication.

> **Security reminder:** Plan your infrastructure deliberately. E.g., don't add a `ports:` config to a Redis service if it should only be reachable internally — services in the same Compose network can already talk to each other without exposing ports to the outside world.

## Manual Network Definition
You can define networks explicitly instead of relying on the default one. Useful when containers from **different** Compose files need to share a network.

### Defining a Named Network
```yaml
services:
  db:
    image: postgres:18.1-alpine
    networks:
      - database-network # Name in this Docker Compose file

networks:
  database-network: # Name in this Docker Compose file
    name: the-database-network # Actual network name
```
- Created by `docker compose up`, removed by `docker compose down`.
- Services join a network by listing it under `networks:` in the service definition.

### Connecting to an External Network
```yaml
services:
  db:
    image: backend-image
    networks:
      - database-network

networks:
  database-network:
    external:
      name: the-database-network # Must match the actual existing network name
```

### Making the Default Network External
```yaml
services:
  db:
    image: backend-image

networks:
  default:
    external:
      name: the-database-network
```
- By default, all services join a network called `default` — this can be reconfigured to point at an existing external network.

## Scaling Services
```bash
docker compose up --scale whoami=3
```
- Runs multiple instances of a service.
- **Problem:** if the service has a fixed host port (e.g. `8000:8000`), scaling fails — all instances try to bind the same host port → clash.

### Fix: Only Specify the Container Port
```yaml
ports:
  - 8000
```
- Leaving the host port unspecified lets Docker assign a free one automatically per instance.
```bash
docker compose up --scale whoami=3
```
Find which host port an instance is bound to:
```bash
docker compose port --index 1 whoami 8000
docker compose port --index 2 whoami 8000
docker compose port --index 3 whoami 8000
```
Then access each instance directly:
```bash
curl 127.0.0.1:<assigned-port>
```

## Load Balancing with nginx-proxy
- In production, a **load balancer** typically sits in front of scaled services.
- For local/single-server setups: [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy) is a simple solution.

### Basic Setup
```yaml
services:
  whoami:
    image: jwilder/whoami
  proxy:
    image: jwilder/nginx-proxy
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
    ports:
      - 80:80
```
- Mounts the host's **Docker socket** (`docker.sock`) into the proxy container in **read-only** mode (`:ro`) — lets nginx-proxy discover running containers via the Docker daemon.
- Without routing info, requests get a `503 Service Temporarily Unavailable` — nginx doesn't know which backend service to route to.

> **Note for M1/M2 Mac users:** `jwilder/nginx-proxy` may throw `runtime: failed to create new OS thread`. Use `ninanung/nginx-proxy` as a temporary fix.

### Routing with `VIRTUAL_HOST`
- nginx-proxy uses two env vars: `VIRTUAL_HOST` and `VIRTUAL_PORT`.
- `VIRTUAL_PORT` isn't needed if the image already declares `EXPOSE` (e.g. `jwilder/whoami` does this).

```yaml
services:
  whoami:
    image: jwilder/whoami
    environment:
      - VIRTUAL_HOST=whoami.colasloth.com
  proxy:
    image: jwilder/nginx-proxy
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
    ports:
      - 80:80
```
```bash
docker compose up -d --scale whoami=3
curl whoami.colasloth.com
# I'm f6f85f4848a8
curl whoami.colasloth.com
# I'm 740dc0de1954  -> requests get load balanced across instances
```
- **`colasloth.com`** (and similar: `localtest.me`, `lvh.me`, `vcap.me`) is a DNS trick where all subdomains resolve to `127.0.0.1` — handy for local testing with real-looking hostnames.

### Adding More Services Behind the Same Proxy
Serve simple static pages using the official `nginx` image, mounting content directly (no custom build needed):
```bash
echo "hello" > hello.html
echo "world" > world.html
```
```yaml
hello:
  image: nginx:1.19-alpine
  volumes:
    - ./hello.html:/usr/share/nginx/html/index.html:ro
  environment:
    - VIRTUAL_HOST=hello.colasloth.com
world:
  image: nginx:1.19-alpine
  volumes:
    - ./world.html:/usr/share/nginx/html/index.html:ro
  environment:
    - VIRTUAL_HOST=world.colasloth.com
```
```bash
docker compose up -d --scale whoami=3
curl hello.colasloth.com   # hello
curl world.colasloth.com   # world
curl whoami.colasloth.com  # I'm <container-id> (load balanced)
```
- Because `hello.html` is **bind-mounted** (not baked into the image), editing it on the host updates the served content live — **no container restart needed**.

## Key Takeaways
| Concept | Summary |
|---|---|
| Compose default network | Auto-created; services reachable by service name via internal DNS |
| Manual network | Explicitly named/shared network, can be marked `external` to connect across Compose files |
| Scaling | `docker compose up --scale <service>=<n>`; avoid fixed host ports to prevent clashes |
| `docker compose port` | Find the host port assigned to a scaled instance |
| nginx-proxy | Simple local load balancer using Docker socket + `VIRTUAL_HOST` env var |
| Bind-mounted content | Updates reflect live in the container without rebuilding/restarting |
