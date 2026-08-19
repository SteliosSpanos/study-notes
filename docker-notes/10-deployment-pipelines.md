# Section 10: Deployment Pipelines

## CI/CD Basics
- **CI/CD pipeline** (deployment pipeline) is a cornerstone of DevOps.
- Per GitLab's definition: CI/CD automates the manual steps traditionally needed to get code from a commit into production — changes are automatically tested, delivered, and deployed, minimizing downtime and speeding up releases.

## Goal of This Section
Build a pipeline so that **every commit automatically**:
1. Builds a Docker image
2. Pushes it to Docker Hub
3. Deploys/restarts it on a target machine

Works the same whether the target is a local machine, a cloud VM (e.g. Hetzner), or even a Raspberry Pi.

**Tools used:**
- **GitHub Actions** — builds the image and pushes it to Docker Hub (first half of the pipeline).
- **Watchtower** — pulls and restarts the updated image on the target machine (second half of the pipeline).

Example project: [mluukkai/beermapping](https://github.com/mluukkai/beermapping) — a simple containerized Node.js app.

## Part 1: GitHub Actions (Build & Push)

GitHub Actions overview: a CI/CD platform for automating build/test/deploy workflows — e.g. build & test on every push/PR, or deploy merged PRs to production.

### Workflow File: `.github/workflows/main.yml`
```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build:  # name of the job
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v5

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/beermapping:latest
```

### Workflow Structure
- A **workflow** consists of one or more **jobs**. Here, just one job: `build`.
- A **job** consists of a series of **steps**, each a small discrete operation:
  1. `actions/checkout@v5` — checks out the repo code (GitHub's own action)
  2. `docker/setup-buildx-action@v3` — sets up the Docker Buildx build environment
  3. `docker/login-action@v3` — logs in to Docker Hub
  4. `docker/build-push-action@v6` — builds the image and pushes it to Docker Hub
- Steps 2–4 are **official Docker-provided GitHub Actions**.

### Required Secrets
Before this workflow will run, add two repository secrets:
- `DOCKER_USERNAME`
- `DOCKER_PASSWORD`

Setup steps:
1. GitHub repo → **Settings** → **Secrets and variables** → add both secrets.
2. `DOCKER_PASSWORD` (or an access token) is generated in **Docker Hub → Account Settings → Security**.

> GitHub Actions only handles the **first half** — building and pushing the image to Docker Hub whenever code is pushed to `main`.

## Part 2: Watchtower (Auto-Deploy)

**Watchtower** is an open-source tool that automates updating running containers:
- Periodically polls the image's source (e.g. Docker Hub) for changes.
- If a new version of the image is pushed, Watchtower automatically **pulls it and restarts** the corresponding running container.
- Respects tags — e.g., a container running `ubuntu:24.04` won't be updated unless a *new* version of that specific tag is published.

> **Note:** Watchtower is not recommended for commercial/production environments — better tooling exists for that (not covered in this course), but it's a good simple example of the concept.

### Security Warning
> Anyone with access to your Docker Hub account effectively has access to your machine via Watchtower. If they push a malicious image update, Watchtower will pull and run it automatically without question.

### Running Watchtower via Compose
```yaml
services:
  watchtower:
    image: nickfedor/watchtower
    environment:
      -  WATCHTOWER_POLL_INTERVAL=60 # Poll every 60 seconds
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    container_name: watchtower
```
- Mounts the **Docker socket** so Watchtower can monitor/control other containers on the host.
- `WATCHTOWER_POLL_INTERVAL` — how often (in seconds) it checks for new image versions.

**Caution:** By default, Watchtower will try to update **every** container running on the machine, not just your intended app. The Watchtower docs describe how to scope it to specific containers only (e.g. via labels).

## Full Pipeline Summary
| Stage | Tool | What it does |
|---|---|---|
| 1. Commit & push | Git / GitHub | Developer pushes code to `main` |
| 2. Build & push image | GitHub Actions | Automatically builds Docker image, pushes to Docker Hub |
| 3. Detect & deploy update | Watchtower | Polls Docker Hub, pulls new image, restarts the container on the target machine |

## Key Takeaways
| Concept | Summary |
|---|---|
| GitHub Actions | Handles CI — build/test/push automation triggered by repo events |
| Docker Hub secrets | Required for GitHub Actions to authenticate and push images |
| Watchtower | Handles CD — polls for new images and auto-updates running containers |
| Docker socket mount | Gives Watchtower control over the host's containers — a real security consideration |
| Tag-awareness | Watchtower only updates when the **same tag** gets a new underlying image |
