# Docker playbook

How a repository builds, stores and runs container images. The release flow
that publishes images, and the deploy recipe, are in `release-deploy.md`
(§5–§7). This file holds the Docker rules they rely on. Values that differ
per project (`IMAGE`, `PORT`, health path, `DEPLOY_HOST`…) come from
`release-deploy.md` §0.

The short version: **Alpine base, multi-stage, non-root, healthchecked.
Built by CI for amd64 + arm64, stored in GHCR under immutable version tags,
and deployed by pulling, never by building on the server.**

```
 local: just image ──▶ native arch, --load, smoke test          (never pushed)
 CI:    v* tag     ──▶ buildx amd64+arm64 ──▶ smoke ──▶ ghcr.io/<owner>/<repo>:X.Y.Z, X.Y, latest, sha-…
 server:            compose pull <tag> ──▶ up -d --wait ──▶ healthy        (no build: on the server)
```

---

## 1. Base images

- **Alpine by default**: `node:<lts>-alpine`, `postgres:<major>-alpine`,
  `redis:<major>-alpine`, `caddy:<major>-alpine`.
- **`-slim` (Debian) only where musl breaks things**: Python projects whose
  dependencies lack musl wheels (numpy, pandas, torch, anything compiled),
  or native Node modules without musl builds. Record the reason in a comment
  on the `FROM` line.
- **Pin the version in the tag.** Pin at least the major, and pin the
  distroless or Alpine minor when reproducibility matters
  (`node:24.8-alpine3.22`). Never use a bare `latest` or an untagged image.
- Build and runtime stages use the **same base family and version**, so
  native modules compiled in one run in the other.
- Base image updates come from Renovate or Dependabot PRs, not from manual
  bumps.

## 2. Dockerfile rules

- **Multi-stage.** At least `deps` → (`build`) → final. The final stage holds
  only the runtime, production dependencies and build output. It has no
  compilers, package managers, dev dependencies, source, tests or `.git`.
- **Dependencies before source.** Copy only the manifest and lockfile,
  install with the frozen lockfile, *then* copy the source. The dependency
  layer is cached until the lockfile changes.
- **Cache mounts** for package manager stores, and a cache-friendly layer
  order. See §3.
- **Secrets never in the image.** No secrets in `ARG`, `ENV` or copied files.
  Build-time secrets (a private registry token) use
  `RUN --mount=type=secret,id=…`. Runtime secrets come from the environment.
- **Non-root.** Alpine Node images: `USER node`. Anything else:
  `RUN addgroup -S app && adduser -S -G app -u 10001 app`, then `USER app`.
  Files the app writes to are `--chown`ed to that user.
- **`.dockerignore` is an allowlist**: `*`, then a `!` line for each path the
  build needs.
- **`HEALTHCHECK`** on the health path, probed with the runtime itself
  (Alpine has no `curl`): `node -e "fetch(…)"` / `python -c "urllib…"`, or
  `wget -qO- …` (BusyBox) where no runtime exists.
- **Exec-form `CMD`** (`["node", "dist/server.js"]`), so the process is PID 1
  and gets `SIGTERM`. The app shuts down gracefully on `SIGTERM` (closes the
  server, drains, exits). Add `init: true` in compose, or `tini`, if the app
  spawns children.
- **One process per container.** Migrations run as a separate one-off
  command or service, not inside the app's entrypoint.
- `EXPOSE <PORT>`, `ENV NODE_ENV=production` / `PYTHONUNBUFFERED=1`, and OCI labels (`org.opencontainers.image.source`,
  `.version`, `.revision`, `.title`, `.description`, `.licenses`), with
  version and revision passed in as build args.
- **Logs:** the readable console goes to stdout, and the JSON Lines files go
  to `LOG_DIR=/var/log/app`, which production mounts from the host
  (`observability.md` §1). The image creates that directory, owned by the
  app user, and never writes logs anywhere else.
- Lint the Dockerfile with **hadolint** in CI.

### Node (pnpm, TypeScript build)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS base
WORKDIR /app
RUN corepack enable

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

FROM deps AS build
COPY . .
RUN pnpm build && pnpm prune --prod

FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production PORT=3000
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --from=build --chown=node:node /app/package.json ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1),()=>process.exit(1))"
CMD ["node", "dist/server.js"]
```

In a pnpm monorepo, build one app with `pnpm deploy --filter <app> --prod /out`
in the build stage and copy `/out` into the final stage.

### Python (uv, Alpine)

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.14-alpine AS build
COPY --from=ghcr.io/astral-sh/uv:0.12 /uv /bin/uv
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-editable

FROM python:3.14-alpine
RUN addgroup -S app && adduser -S -G app -u 10001 app
WORKDIR /app
COPY --from=build --chown=app:app /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1 PORT=8000
USER app
EXPOSE 8000
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)"
CMD ["myapp"]
```

If a dependency has no musl wheel, switch both stages to `python:3.14-slim`
and use `useradd --system --uid 10001 app` in the final stage.

### Static frontend (SPA / Astro)

Build in `node:<lts>-alpine`, then serve `dist/` from `caddy:2-alpine` with a
`Caddyfile` (`file_server`, `try_files {path} /index.html` for SPAs). There's
no Node in the final image.

## 3. Build cache

Goal: a source-only change rebuilds in seconds, and dependencies are only
reinstalled when the lockfile changes, on every machine and in CI.

### 3.1 Layer order

BuildKit reuses a layer only if the layer and everything above it are
unchanged. So order a Dockerfile from what changes least to what changes
most:

- Base image and system packages (`apk add`).
- Manifest + lockfile → dependency install.
- Source → build.
- Build args that change every build (`VERSION`, `REVISION`), declared
  **last**, just before the `LABEL` that uses them. An `ARG` declared early
  invalidates every layer after it.

Rules that keep the cache hit:

- **Copy precisely.** `COPY package.json pnpm-lock.yaml ./`, not
  `COPY . .`, before installing. In monorepos, copy every workspace's
  `package.json` (or use `pnpm fetch`, which needs only the lockfile) so a
  source change in one package doesn't bust the install layer.
- **The `.dockerignore` allowlist is a cache tool too.** Anything outside it
  (`.git`, `node_modules`, `dist`, test output, editor files) can't change
  the build context, so it can't invalidate `COPY . .`.
- **No build-time non-determinism**: no `apt-get update` / `apk update` in a
  separate layer, no timestamps or `git describe` written into files during
  the build, no `curl` of an unpinned URL. Anything like that busts the cache
  or gives different images from the same commit.
- **One `RUN` per logical step**, combining install and cleanup
  (`apk add --no-cache …`). Don't chain unrelated steps: when one changes,
  the others rebuild too.

### 3.2 Cache mounts

A cache mount keeps a package manager's download store between builds
without it ever entering a layer. After a lockfile change, only new packages
are downloaded:

| Tool | Mount                                                      |
| ---- | ---------------------------------------------------------- |
| pnpm | `--mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store` |
| npm  | `--mount=type=cache,target=/root/.npm`                     |
| uv   | `--mount=type=cache,target=/root/.cache/uv` (with `UV_LINK_MODE=copy`) |
| pip  | `--mount=type=cache,target=/root/.cache/pip`               |
| apk  | `--mount=type=cache,target=/etc/apk/cache` (then `apk add` without `--no-cache`) |
| Build tools | Turborepo `.turbo`, Vite / Next caches: `--mount=type=cache,target=/app/.turbo` |

- Cache mounts are local to the builder. They speed up rebuilds on the same
  machine (and on CI with a persistent builder), but they aren't exported
  with `cache-to`. The layer cache in §3.3 is what travels.
- **Bind mounts** for files a step only reads, so they never become a layer:
  `RUN --mount=type=bind,source=pnpm-lock.yaml,target=pnpm-lock.yaml pnpm fetch`.

### 3.3 Sharing the cache between machines

| Where         | Cache backend                                   | Notes |
| ------------- | ----------------------------------------------- | ----- |
| GitHub Actions | `cache-from: type=gha,scope=<image>-<arch>` / `cache-to: type=gha,mode=max,scope=<image>-<arch>` | One scope per image and architecture, so amd64 and arm64 don't evict each other. `mode=max` also caches the intermediate stages (`deps`, `build`), which is where the time goes. |
| Across machines | `type=registry,ref=ghcr.io/<owner>/<repo>:buildcache,mode=max` | Use when the GHA cache (10 GB per repo, evicted after 7 days unused) is too small, or so developers can pull CI's cache. The `buildcache` tag is exempt from the cleanup workflow (§5). |
| Local          | The builder's own cache                          | Default. For a cold laptop, `--cache-from type=registry,ref=…:buildcache` reuses CI's layers (read-only; only CI writes the cache). |

- **Multi-arch in CI:** build each platform in its own matrix job on a
  native runner (`ubuntu-24.04` and `ubuntu-24.04-arm`) with its own cache
  scope, then merge the digests into one manifest list
  (`docker buildx imagetools create`). That's far faster than QEMU emulation,
  and each architecture's cache stays warm.
- **Smoke-test build and push build share the cache**, so the push step
  reuses the smoke-tested layers instead of rebuilding.

### 3.4 Checking and maintaining it

- **Check that the cache hits:** run `just image` twice with no changes. The
  second run should show `CACHED` for every step. Then change one source
  file: only the steps from `COPY . .` down should rerun. If the install
  step reruns, the layer order or `.dockerignore` is wrong.
- `docker buildx du` shows the builder's cache size. Run
  `docker buildx prune --keep-storage 20GB` (or `--filter until=168h`) when
  local disks fill up. Never prune in CI with a persistent builder.
- On servers, the cache doesn't matter (they only pull). Prune images there,
  not the build cache (§7).

## 4. Building on different machines

Developer machines are mixed (x86_64 Linux, Apple Silicon), and servers may
be amd64 or arm64. So:

- **Published images are always multi-arch** (`linux/amd64,linux/arm64`),
  built by CI with `docker buildx` (see `release-deploy.md` §5). Nobody pushes
  a release image from a laptop.
- **Local builds are native-arch and never pushed.** `just image` builds for
  the host platform with `--load`, and `just image-smoke` runs it and checks
  `/health`. That's the same check CI does.
- **Cross-arch locally only for debugging**, e.g. reproducing an arm64-only
  bug on x86:
  `docker buildx build --platform linux/arm64 --load .` (needs QEMU:
  `docker run --privileged --rm tonistiigi/binfmt --install arm64`). It's
  slow; don't make it the normal path.
- **Architecture-specific downloads** in a Dockerfile use BuildKit's
  `ARG TARGETARCH`, never a hardcoded `amd64`.
- **Same inputs everywhere.** The Dockerfile only reads lockfiles and pinned
  base images, so a build on any machine gives the same result. Local caches
  are an optimisation, never a dependency. How the cache is shared between
  machines and CI is in §3.3.
- **Never build on the server.** Compose files on servers have `image:` and
  no `build:`.

```just
# Build the image for this machine's architecture (not pushed)
image:
    docker buildx build --load -t {{IMAGE}}:dev \
      --build-arg VERSION=dev --build-arg REVISION=$(git rev-parse HEAD) .

# Run the local image and wait for /health
image-smoke: image
    #!/usr/bin/env bash
    set -euo pipefail
    id=$(docker run -d --rm -p {{PORT}}:{{PORT}} {{IMAGE}}:dev)
    trap 'docker logs "$id"; docker stop "$id" >/dev/null' EXIT
    for _ in $(seq 30); do curl -fsS localhost:{{PORT}}/health && exit 0; sleep 1; done
    exit 1
```

## 5. Storing images (GHCR)

- **Registry:** `ghcr.io/<owner>/<repo>`, all lowercase.
- **Tags:**
  - Version tags follow Semantic Versioning (`X.Y.Z`, `X.Y.Z-rc.N`) and are
    **immutable**: never re-pushed
    or overwritten.
  - `X.Y`, `latest` and `next` are moving pointers.
  - `sha-<short>` ties an image to its commit.
  - `dev` and local tags never reach the registry.
- **Only CI pushes**, using `GITHUB_TOKEN` with `packages: write`.
- **Pulling:**
  - Public packages pull anonymously.
  - For private packages, servers log in once with a **read-only**
    fine-grained PAT (`read:packages`), stored in the host's Docker config
    and not in the repo.
  - Developers log in with `gh auth token | docker login ghcr.io -u <user> --password-stdin`.
- **Visibility:** GHCR creates packages private. Decide public or private on
  the first release, and link the package to the repo (the
  `org.opencontainers.image.source` label does this).
- **Retention:** a scheduled workflow
  (`actions/delete-package-versions` or `dataaxiom/ghcr-cleanup-action`)
  deletes untagged manifests and old `sha-*` tags. Version tags are always
  kept, because they're the rollback targets. So is `buildcache`, if the
  registry cache is used (§3.3).
- **Scanning:** Trivy (`aquasecurity/trivy-action`) scans the smoke-tested
  image in CI. It fails the build on fixable `CRITICAL` vulnerabilities and
  reports the rest.

## 6. Running: compose

The repo has one `compose.yaml` for **local development**. The server has
its own compose file for **production**, synced by `just deploy`
(`release-deploy.md` §7).

**Local (`compose.yaml`):**

- Holds only the dependencies (Postgres, Redis, Mailpit…). The app itself
  runs on the host through `just dev`, unless it has to run in a container.
- Every service has a `healthcheck`. Dependents use
  `depends_on: { db: { condition: service_healthy } }`, and
  `docker compose up -d --wait` blocks until everything is healthy.
- Ports bind to `127.0.0.1` (`"127.0.0.1:5432:5432"`), never `0.0.0.0`.
- Data lives in named volumes, and dev credentials are in plain text: they
  only ever work locally.

**Production (server compose):**

- `image: ${IMAGE}:${IMAGE_TAG:-latest}`, with no `build:`.
- No published ports. The service joins the proxy network, and Caddy is the
  only way in.
- `restart: unless-stopped`, `mem_limit`, `init: true`,
  `read_only: true` with `tmpfs: [/tmp]` where the app allows it, and
  `security_opt: [no-new-privileges:true]`.
- Config and data are bind-mounted from a host directory (easy to back up).
  Secrets live in a `chmod 600` `.env` on the host.
- Logging, both outputs capped so logs can't fill the disk:
  - Console (stdout): the `json-file` driver with rotation
    (`options: { max-size: "10m", max-file: "3" }`).
  - Files: `./logs:/var/log/app` bind-mounted, rotated by the app itself
    (`LOG_MAX_SIZE`, `LOG_MAX_FILES`), and shipped by the collector
    (`observability.md` §4).
- Migrations run as a one-off before `up`:
  `docker compose run --rm <svc> <migrate-cmd>`.
- Persistent data is backed up off-site with Restic on a schedule:
  - Databases are dumped first (`pg_dump`); never copy live database files.
  - Backups go to S3-compatible storage, encrypted and incremental.
  - A restore has been tested at least once. A backup nobody has restored
    from is a hope, not a backup.

## 7. Deploying

The full recipe is in `release-deploy.md` §7. The Docker parts:

- **Deploy a version, not a moving tag, in production.** Set
  `IMAGE_TAG=X.Y.Z`. `latest` is fine for personal projects. `next` is for
  staging.
- `docker compose pull -q <svc>`, then run migrations, then
  `docker compose up -d --wait <svc>`. `--wait` makes a green deploy mean a
  healthy container.
- **Roll back** by redeploying the previous version tag:
  `IMAGE_TAG=<previous> just deploy`. Every version tag stays in the
  registry for this reason. Migrations must be backwards compatible with the
  previous version (expand, then contract), so a rollback never needs a
  down-migration.
- **Record what runs.** After a deploy, log the running image digest:
  `docker compose images` / `docker inspect --format '{{.Image}}'`.
- **Clean up** old images on the host after a successful deploy:
  `docker image prune -f`. Only dangling images go, so the previous version
  stays cached for a fast rollback until the next prune.

## 8. Bootstrapping a new repo: checklist

- [ ] `Dockerfile`: Alpine base (or slim with a reason), multi-stage, cache
      mounts, non-root, `HEALTHCHECK`, exec-form `CMD`, OCI labels
- [ ] Layer order checked: a second `just image` is fully `CACHED`, and a
      source change doesn't rerun the dependency install (§3.4)
- [ ] CI cache: `type=gha,mode=max` scoped per architecture, or a registry
      `buildcache` tag excluded from cleanup (§3.3)
- [ ] `.dockerignore`: allowlist
- [ ] App handles `SIGTERM` gracefully and exposes `/health`
- [ ] `compose.yaml` for local dependencies: healthchecks, `127.0.0.1` ports
- [ ] `justfile`: `image`, `image-smoke`
- [ ] CI: hadolint, build + smoke on PRs, multi-arch push on `v*` tags,
      Trivy scan (`release-deploy.md` §5)
- [ ] GHCR: visibility decided, package linked to the repo, cleanup workflow
- [ ] Renovate or Dependabot watching base images
- [ ] Server compose: `image:` only, proxy network, limits, stdout log rotation, `logs/` bind mount
