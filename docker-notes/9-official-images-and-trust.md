# Section 9: Official Images and Trust

## Chapter Goal
Move from "just making things work" to **best practices**: understanding image size choices (e.g. Alpine vs Ubuntu) and the risks of running containers as `root` — both flagged earlier but deferred until now.

## What Makes an Image "Official"?
- "Official" status is decided by the maintainers of **Docker Official Images**.
- The official images repository holds a curated library, added via normal **pull request** processes, with an extended **verification process** documented in the repo's README.
- Many major projects (Postgres, Python) are maintained directly under the **docker-library** organization.
- Others (e.g. **Ubuntu**, **Node.js**) are included in the official library but actually **maintained by a separate organization** (Canonical, the Node.js Foundation, etc.) — the "official" tag doesn't mean Docker Inc. itself built it.

## Tracing the Ubuntu Image's Origin
From the Ubuntu Docker Hub description:
> This image is built from official rootfs tarballs provided by Canonical (see dist-* tags at the `ubuntu-base` Launchpad repo).

So the image traces back to **Canonical's own build artifacts**, not something Docker Inc. assembled independently.

### Verifying the Dockerfile
- Docker Hub's "Dockerfile" link for some images actually returns a **JSON digest file**, not the raw Dockerfile — a bit misleading at first glance.
- The digest shown corresponds to a specific architecture (e.g. `amd64`) — but pulling and checking locally may not match exactly if you're on a different architecture:
```bash
docker pull ubuntu:24.04
docker image ls --digests
```
- The actual Dockerfile can be found in Canonical's/Ubuntu's own repository.

### Cross-Checking with `docker image history`
```bash
docker image history --no-trunc ubuntu:24.04
```
- Shows the actual build steps/layers baked into the image.
- Comparing this output against the published Dockerfile lets you **verify they match** — increasing trust that what you're running is what's publicly documented.
- If still not convinced, you could **build the image yourself** from source.

### What the Ubuntu Dockerfile Actually Does
- Starts `FROM scratch` — a special, completely empty base image.
- Then **adds** a tarball (`ubuntu-*-oci-$LAUNCHPAD_BUILD_ARCH-root.tar.gz`) to the root filesystem using `ADD`.
- Note: the tarball is never manually "extracted" as a separate step — per the `ADD` instruction's documented behavior: if the source is a local tar archive in a recognized compression format (identity, gzip, bzip2, xz), Docker **automatically unpacks it** as a directory during the build.
- Checksums of the tarball could be manually verified if desired.
- For Ubuntu specifically, **Launchpad automation** creates the PRs into `docker-library`, and the official-images maintainers review/verify them before merging.

### Docker Hub Security Info
- Each image tag's Docker Hub page also shows its **layers** and flags potential **security issues** (vulnerabilities) found in that image — worth checking before choosing a base image.

## Key Takeaway
- The build process behind official images is **open and independently verifiable** — you're not forced to blindly trust a label.
- Being "official" doesn't inherently mean the image is more secure or specially vetted beyond what's publicly checkable — it mainly signals it went through Docker's review/PR process and is actively maintained.

> *"You can't trust code that you did not totally create yourself."*
> — Ken Thompson, *Reflections on Trusting Trust* (1984)

This quote underscores the deeper point: even with full transparency and verification tools, ultimate trust has limits — verification helps, but it's not absolute proof.

## Practical Verification Commands
| Command | Purpose |
|---|---|
| `docker pull <image>:<tag>` | Download the image |
| `docker image ls --digests` | Show image digests for comparison |
| `docker image history --no-trunc <image>:<tag>` | Show full build history/layers of an image |
