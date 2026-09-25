# Release & deploy playbook

How this repository handles commits, changelogs, releases, CI/CD and
deployment. Follow it as written; where a value differs per project it comes
from **Project settings** below.

The short version: **commits are the source of truth, a tag is the only thing
that publishes, CI turns the tag into an image + changelog + GitHub release,
and production only ever runs a published image.**

```
 commit (conventional)  ──push──▶  main  ──CI──▶  CHANGELOG.md regenerated (git-cliff), committed back
                                    │
 just release [bump] [pre]          │  bump version file, commit "chore(release): vX.Y.Z[-rc.N]",
                                    ▼  annotated tag, ONE atomic push of branch + tag
                            vX.Y.Z  or  vX.Y.Z-rc.N
                                    │
              ┌─────────────────────┼──────────────────────────┐
              ▼                     ▼                          ▼
     image built, smoke-tested,   CHANGELOG.md             GitHub release, notes from
     pushed:                      regenerated              `just release-notes <tag>`
       stable: X.Y.Z, X.Y, latest                          (marked prerelease if tag has "-")
       pre:    X.Y.Z-rc.N, next
                                    │
 just deploy  ◀─────────────────────┘
   sync config → pull image → recreate container → wait for healthcheck
```

---

## 0. Project settings and toolchain

Fill these in once per repository; everything below refers to them by name.

| Setting          | Default                                  | Used by                    |
| ---------------- | ---------------------------------------- | -------------------------- |
| `DEFAULT_BRANCH` | `main`                                   | release recipe, CI         |
| Version file     | `package.json` / `pyproject.toml`        | release recipe             |
| `IMAGE`          | `ghcr.io/<owner>/<repo>` (lowercase)     | image workflow, compose    |
| `IMAGE_TAG`      | `latest`                                 | deploy                     |
| `PORT`           | `3000` (Node) / `8000` (Python)          | Dockerfile, compose        |
| Health path      | `/health`                                | Dockerfile, CI smoke test  |
| `DEPLOY_HOST`    | ssh host alias                           | deploy                     |
| `DEPLOY_DIR`     | directory holding `compose.yaml` on host | deploy                     |
| `SERVICE`        | compose service name                     | deploy                     |

**Fixed tool choices** (don't substitute without a reason):

| Concern                | Tool                                              |
| ---------------------- | ------------------------------------------------- |
| Tool versions          | **mise** (`mise.toml` pins every tool below)      |
| Task runner            | **just** (`justfile`)                             |
| Changelog & notes      | **git-cliff** (`cliff.toml`)                      |
| Python env & packaging | **uv** (`pyproject.toml` + `uv.lock`)             |
| Python lint & format   | **ruff**                                          |
| Node package manager   | **pnpm** (`pnpm-lock.yaml`); corepack in Dockerfiles |
| JS/TS lint & format    | **biome** (`biome.json`)                          |
| CI/CD                  | GitHub Actions (`jdx/mise-action` to install tools) |
| Registry               | GHCR                                              |
| Runtime                | Docker Compose behind a Caddy reverse proxy       |

```toml
# mise.toml: keep only what the project uses; pin exact versions
[tools]
just = "1"
git-cliff = "2"
node = "24"          # Node projects
"github:pnpm/pnpm" = "12.6.0"  # not aqua or npm:, see pre-selected-tools.md §1
gitleaks = "8"
biome = "2"
python = "3.14"      # Python projects
uv = "0.12"
ruff = "0.16"
```

Everyone (and CI) runs `mise install`; nothing is installed globally by hand.

## 1. Commits

- **Conventional Commits, always.** `type(scope): description` with types
  `feat fix perf refactor docs style test build ci chore revert`. The changelog,
  release notes and automatic version bumps are all derived from these subjects,
  so they are the *only* place release documentation is written.
- **Description:** imperative, lowercase, no trailing period, ≤ 72 chars.
- **Semantic Versioning.** Commit types drive the version: `fix` → patch,
  `feat` → minor, breaking change → major (see §3).
- **The user is the only author.** Commits are made under the user's git
  identity, with no `Co-Authored-By:` trailer for an AI, no
  `Claude-Session:` line and no "Generated with Claude Code" text, in commits
  and PR bodies alike. Agents never pass `--author`.
- **Body explains why**, wrapped at 72. The diff already shows what.
- **Breaking changes:** `!` after the type/scope or a `BREAKING CHANGE:` footer.
  git-cliff promotes those to a "Breaking changes" group.
- **One logical change per commit.** If the subject needs "and", split it.
  A feature plus its tests, docs and CI assertions is still one change.
- **Commit only what you changed.** Stage with an explicit pathspec
  (`git commit -- <paths>`), never a blanket `git add -A`, so unrelated work in
  the tree doesn't ride along.
- **Expect main to move under you.** CI commits the changelog back to main after
  every push, so a local main is routinely one commit behind. Before pushing:
  `git fetch && git rebase origin/main` (the bot commit only touches
  `CHANGELOG.md`, so this never conflicts).

## 2. Changelog (git-cliff)

- **Generated, never hand-edited.** git-cliff walks the history and tags,
  parses conventional commits, and groups them per release in Keep-a-Changelog
  layout. Anything past the newest stable tag is **Unreleased**.
- **Nothing is lost silently:** non-conventional subjects land under "Other".
- **Release commits are skipped** (they describe the release, not the project),
  but the tag on them still opens the section.
- **Prerelease tags don't get their own section** (`ignore_tags`): while a
  release candidate is in flight its changes sit under Unreleased, and the final
  `vX.Y.Z` section lists everything since the previous stable release. The
  per-candidate notes live on the GitHub prereleases instead.
- Each section gets a **compare link**; entries link to their commit. The repo
  URL is derived from `git remote get-url origin` (SSH or HTTPS form).
- **Release notes come from the same generator, not by scraping the rendered
  markdown back out:** `just release-notes <tag>` prints one release's body,
  covering everything since the previous *stable* tag.

`cliff.toml`:

```toml
[changelog]
header = """
# Changelog

All notable changes to this project are documented here. Generated by
[git-cliff](https://git-cliff.org) from Conventional Commits; do not edit.

"""
body = """
{% if version %}\
## [{{ version }}] - {{ timestamp | date(format="%Y-%m-%d") }}
{% if previous.version %}
[Compare](<REPO>/compare/{{ previous.version }}...{{ version }})
{% endif %}\
{% else %}\
## [Unreleased]
{% endif %}\
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | striptags | trim | upper_first }}
{% for commit in commits %}
- {% if commit.scope %}**{{ commit.scope }}:** {% endif %}\
{{ commit.message | split(pat="\n") | first | upper_first }} \
([{{ commit.id | truncate(length=7, end="") }}](<REPO>/commit/{{ commit.id }}))\
{% endfor %}
{% endfor %}\n
"""
trim = true
postprocessors = [
  # <REPO> -> https URL of `origin` (handles git@host:owner/repo.git and https://)
  { pattern = "<REPO>", replace_command = '''sed "s#<REPO>#$(git remote get-url origin | sed -E 's#^git@([^:]+):#https://\1/#; s#\.git$##')#g"''' },
]

[git]
conventional_commits = true
filter_unconventional = false
protect_breaking_commits = true
filter_commits = false
ignore_tags = "-(alpha|beta|rc)\\."
tag_pattern = "^v[0-9]+\\.[0-9]+\\.[0-9]+"
sort_commits = "oldest"
commit_parsers = [
  { message = "^chore\\(release\\)", skip = true },
  { message = "^docs\\(changelog\\)", skip = true },  # CI's own changelog commits
  { field = "breaking", pattern = "true", group = "<!-- 0 -->Breaking changes" },
  { message = "^feat", group = "<!-- 1 -->Features" },
  { message = "^fix", group = "<!-- 2 -->Bug fixes" },
  { message = "^perf", group = "<!-- 3 -->Performance" },
  { message = "^refactor", group = "<!-- 4 -->Refactoring" },
  { message = "^docs", group = "<!-- 5 -->Documentation" },
  { message = "^(build|ci)", group = "<!-- 6 -->Build & CI" },
  { message = "^(style|test|chore|revert)", group = "<!-- 7 -->Maintenance" },
  { message = ".*", group = "<!-- 9 -->Other" },
]
```

Recipes (`justfile`):

```just
# Regenerate CHANGELOG.md from the git history
changelog:
    git cliff -o CHANGELOG.md

# Fail if CHANGELOG.md is stale
changelog-check:
    @git cliff | diff -u CHANGELOG.md - >/dev/null || { echo "CHANGELOG.md is stale: run 'just changelog'" >&2; exit 1; }

# Print the release notes for one tag (everything since the previous stable release)
release-notes tag:
    #!/usr/bin/env bash
    set -euo pipefail
    prev="$(git describe --tags --abbrev=0 --exclude '*-*' "{{ tag }}^" 2>/dev/null || true)"
    if [[ -n "$prev" ]]; then git cliff --tag "{{ tag }}" --strip all "$prev..{{ tag }}"
    else git cliff --tag "{{ tag }}" --strip all --latest; fi  # first release: it is the newest tag
```

## 3. Releasing

`just release [bump] [pre]` is the entire ceremony. It:

- **Refuses** anything but `DEFAULT_BRANCH` and a clean working tree.
- **Catches up first:** `git fetch --tags`, fast-forwards to the remote branch,
  bails if diverged. (CI's changelog commit otherwise makes the final push fail
  every time.)
- **Computes the next version** from the newest tag (see the table in §4),
  `auto` asks git-cliff to derive the bump from the commits since the last
  stable release.
- **Refuses an existing tag.**
- Writes the version file(s), commits `chore(release): vX.Y.Z`, creates an
  **annotated** tag.
- **One atomic push:** `git push --atomic origin main vX.Y.Z`. A rejected
  branch must not leave a tag on the remote pointing at a commit nobody has.
- Prints what happens next.

Then **pull afterwards**: CI will have committed the changelog on top.

**Version files.** The git tag is always SemVer (`v1.3.0-rc.1`) and is the
source of truth for the current version.

- Node: `npm pkg set version=1.3.0-rc.1` writes `package.json` verbatim.
- Python: `uv version 1.3.0-rc.1` writes the PEP 440 form (`1.3.0rc1`) to
  `pyproject.toml` and updates `uv.lock`. Never hand-write the version there.

**Version bumps** follow SemVer: `feat` → minor, `fix` → patch, breaking →
major (`auto` does exactly this; on `0.x` a breaking change goes to `1.0.0`,
tune `[bump]` in `cliff.toml` if you want otherwise). Whatever you pick, be
consistent, because a pushed tag can't be quietly renumbered.

```just
default_branch := env("DEFAULT_BRANCH", "main")

# Cut a release. bump: patch|minor|major|auto|pre|final; pre: alpha|beta|rc
release bump="patch" pre="":
    #!/usr/bin/env bash
    set -euo pipefail
    die() { echo "release: $*" >&2; exit 1; }
    [[ "$(git branch --show-current)" == "{{ default_branch }}" ]] || die "not on {{ default_branch }}"
    [[ -z "$(git status --porcelain)" ]] || die "working tree not clean"
    git fetch -q --tags origin
    git merge -q --ff-only "origin/{{ default_branch }}" || die "diverged from origin/{{ default_branch }}"

    # Current version = newest reachable tag (the version file may hold a PEP 440 spelling).
    cur="$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null || echo v0.0.0)"
    cur="${cur#v}"; base="${cur%%-*}"; cur_pre=""; [[ "$cur" == *-* ]] && cur_pre="${cur#*-}"
    IFS=. read -r maj min pat <<<"$base"
    bump="{{ bump }}"; pre="{{ pre }}"
    [[ -z "$pre" || "$pre" =~ ^(alpha|beta|rc)$ ]] || die "pre must be alpha, beta or rc"

    if [[ -n "$cur_pre" ]]; then
      id="${cur_pre%.*}"; n="${cur_pre##*.}"
      case "$bump" in
        pre)   if [[ -z "$pre" || "$pre" == "$id" ]]; then next="$base-$id.$((n + 1))"
               elif [[ "$pre" > "$id" ]]; then next="$base-$pre.1"
               else die "can't go from $id back to $pre"; fi ;;
        final) [[ -z "$pre" ]] || die "final takes no pre id"; next="$base" ;;
        *)     die "v$cur is a prerelease: use 'just release pre [id]' or 'just release final'" ;;
      esac
    else
      case "$bump" in
        patch) next="$maj.$min.$((pat + 1))" ;;
        minor) next="$maj.$((min + 1)).0" ;;
        major) next="$((maj + 1)).0.0" ;;
        auto)  next="$(git cliff --bumped-version 2>/dev/null)"; next="${next#v}"
               [[ -n "$next" && "$next" != "$base" ]] || die "git-cliff found nothing to release" ;;
        *)     die "v$cur is not a prerelease: use patch|minor|major|auto [alpha|beta|rc]" ;;
      esac
      [[ -z "$pre" ]] || next="$next-$pre.1"
    fi
    tag="v$next"
    git rev-parse -q --verify "refs/tags/$tag" >/dev/null && die "$tag already exists"

    # Write the version file(s). uv stores the PEP 440 form (1.3.0-rc.1 -> 1.3.0rc1).
    files=()
    if [[ -f pyproject.toml ]]; then uv version -q --no-sync "$next"; files+=(pyproject.toml); [[ -f uv.lock ]] && files+=(uv.lock); fi
    if [[ -f package.json ]]; then npm pkg set version="$next"; files+=(package.json); fi
    (( ${#files[@]} )) || die "no pyproject.toml or package.json"

    git add -- "${files[@]}"
    git commit -q -m "chore(release): $tag" -- "${files[@]}"
    git tag -a "$tag" -m "$tag"
    git push -q --atomic origin "{{ default_branch }}" "$tag"
    echo "Released $tag (from v$cur). CI now publishes the changelog, GitHub release and image."
    echo "Run 'git pull' once CI has committed CHANGELOG.md."
```

## 4. Prereleases

Use a prerelease when a version needs to run somewhere real (staging, a few
users) before it becomes `latest`. Identifiers go `alpha` → `beta` → `rc`, only
forwards.

| Current       | Command                   | Next           | What it's for                           |
| ------------- | ------------------------- | -------------- | --------------------------------------- |
| `1.2.4`       | `just release`            | `1.2.5`        | normal patch release                    |
| `1.2.4`       | `just release minor`      | `1.3.0`        | normal minor release                    |
| `1.2.4`       | `just release auto`       | from commits   | let git-cliff pick patch/minor/major    |
| `1.2.4`       | `just release minor rc`   | `1.3.0-rc.1`   | **start** a prerelease of the next minor |
| `1.2.4`       | `just release major beta` | `2.0.0-beta.1` | start further back in the ladder        |
| `1.3.0-rc.1`  | `just release pre`        | `1.3.0-rc.2`   | **iterate**: fixes on the candidate     |
| `2.0.0-beta.3`| `just release pre rc`     | `2.0.0-rc.1`   | **advance** the identifier              |
| `1.3.0-rc.2`  | `just release final`      | `1.3.0`        | **promote** the candidate to stable     |
| `1.3.0-rc.2`  | `just release minor`      | refused        | finish or iterate the prerelease first  |

The flow:

- **Start:** `just release minor rc` → `v1.3.0-rc.1`.
- **CI treats any tag containing `-` as a prerelease:**
  - GitHub release is created with `--prerelease` (never shown as "Latest").
  - Image gets `1.3.0-rc.1` and the moving `next` tag, but **not** `X.Y` or
    `latest`, so production (which follows `latest`) is untouched.
  - Release notes: everything since the last stable tag.
  - `CHANGELOG.md` keeps those changes under Unreleased.
- **Try it:** deploy to staging with `IMAGE_TAG=next just deploy` (or the
  exact `IMAGE_TAG=1.3.0-rc.1`).
- **Fix and iterate:** land `fix:` commits on main, `just release pre` →
  `v1.3.0-rc.2`, redeploy staging. Normal work can continue on main meanwhile;
  everything merged goes into the next candidate.
- **Promote:** `just release final` → `v1.3.0`. CI publishes `1.3.0`, `1.3`,
  `latest`, a stable GitHub release whose notes cover the whole cycle, and a
  `## [v1.3.0]` changelog section. Then `just deploy` to production as usual.
- **Abandon:** just don't promote. Leave the tags (never delete or move a
  pushed tag); the next `just release pre` or `final` continues from them.

## 5. CI/CD (GitHub Actions)

Two workflows, each with one job and a clear trigger contract. Both start with
`actions/checkout` and `jdx/mise-action` so CI runs the exact tool versions from
`mise.toml`, and call the same `just` recipes developers run.

### Changelog & release workflow (`.github/workflows/changelog.yml`)

- **Triggers:** push to `main`, push of `v*` tags, manual dispatch.
  `paths-ignore: [CHANGELOG.md]` so its own commit doesn't loop.
- `permissions: contents: write`; `concurrency` group with
  `cancel-in-progress: false` (never abort a half-done changelog commit).
- Always check out **`ref: main`** with `fetch-depth: 0`, even on a tag push, so
  the tag run still updates the branch and git-cliff sees every tag.
- `just changelog` → commit only if changed (test with
  `git status --porcelain -- CHANGELOG.md`, not `git diff --quiet`, which
  ignores the first, still untracked, `CHANGELOG.md`), as
  `docs(changelog): update changelog` by `github-actions[bot]`
  (`41898282+github-actions[bot]@users.noreply.github.com`) → `git push origin HEAD:main`.
- **On a tag, publish the GitHub release.** A pushed tag is *not* a release:
  GitHub shows it with an empty body until something creates one.

  ```bash
  TAG="${GITHUB_REF_NAME}"
  notes="$(just release-notes "$TAG")"
  flags=(); [[ "$TAG" == *-* ]] && flags+=(--prerelease)
  if gh release view "$TAG" >/dev/null 2>&1; then
    gh release edit "$TAG" --notes "$notes" "${flags[@]}"      # idempotent reruns
  else
    gh release create "$TAG" --title "$TAG" --notes "$notes" --verify-tag "${flags[@]}"
  fi
  ```

### Image workflow (`.github/workflows/docker.yml`)

- **Only a `v*` tag publishes.** Pull requests (path-filtered to the files that
  go into the image, including the dependency manifest and lockfile) and manual
  runs build and smoke-test *without pushing*. Pushes to main leave the registry
  alone, so `latest` always means "newest stable release", never "whatever is
  on main".
- **Tags pushed** via `docker/metadata-action`. Lowercase the image name, GHCR
  rejects capitals.

  ```yaml
  flavor: latest=false
  tags: |
    type=semver,pattern={{version}}
    type=semver,pattern={{major}}.{{minor}}
    type=raw,value=latest,enable=${{ !contains(github.ref_name, '-') }}
    type=raw,value=next,enable=${{ contains(github.ref_name, '-') }}
    type=sha
  ```

  `{{major}}.{{minor}}` is skipped for prerelease versions automatically, so a
  stable tag yields `X.Y.Z, X.Y, latest, sha-…` and a prerelease yields
  `X.Y.Z-rc.N, next, sha-…`.
- **Smoke test the actual image before pushing it:** build single-arch with
  `load: true`, `docker run` it (with the services it needs at startup, such
  as a Postgres, and its migrations run from the image, in one
  `scripts/image-smoke.sh` shared with `just image-smoke`), poll the health
  path, then `curl` the key
  behaviours users depend on (status codes, content types, a 404). Add a
  negative assertion when you remove something. Dump `docker logs` in an
  `if: always()` step.
- Then build **multi-arch** (`linux/amd64`, `linux/arm64`) and push. Each
  platform builds in its own matrix job on a native runner (`ubuntu-24.04`,
  `ubuntu-24.04-arm`), not under QEMU, with its own cache scope
  (`type=gha,mode=max,scope=<image>-<arch>`). It pushes by digest, and a
  final job merges the digests into one tagged manifest list with
  `docker buildx imagetools create`. Details in `docker.md` §3.3. Write the
  pushed tags to `$GITHUB_STEP_SUMMARY`.
- **GHCR creates new packages private.** Flip to public once in package
  settings if hosts should pull anonymously.

### Checks on every push and PR

Run `just lint` and `just test` (and `just changelog-check` if you want a
pre-push guard). Lint is `biome ci .` for JS/TS and
`ruff check . && ruff format --check .` for Python.

## 6. The image

The Dockerfile rules and templates live in `docker.md`, which is the source of
truth for them:

- **Alpine base images** (`node:<lts>-alpine`, `python:<ver>-alpine`). Use
  `-slim` only where musl breaks a dependency, with the reason in a comment
  on the `FROM` line (`docker.md` §1).
- **Multi-stage, minimal final stage**, with dependencies installed from the
  lockfile before the source is copied (`docker.md` §2, templates for Node,
  Python and static sites).
- **Cache-friendly layer order and cache mounts** (`docker.md` §3).
- **`.dockerignore` as an allowlist:** `*`, then `!` for each file the image
  needs. Nothing personal or local can leak into a published image by
  accident.
- **Non-root user**, `EXPOSE`, and a `HEALTHCHECK` on `/health` (a dedicated
  endpoint that doesn't depend on content or auth), probed with the runtime
  itself because Alpine has no `curl`.
- **Config comes from the environment and mounts, not the image.** The same
  image runs in staging and production.
- OCI labels (`org.opencontainers.image.title/description/source/version/revision/licenses`).

## 7. Deployment

**Production runs the published image, never a local build.** That is the
point of the whole flow: what's deployed is what CI smoke-tested.

### Server side (compose behind Caddy)

- Service uses `image: ${IMAGE}:${IMAGE_TAG:-latest}`. No `build:` on the
  server. `IMAGE_TAG` lets the same compose file run a pinned version, a
  prerelease (`next`) on staging, or an older version to roll back.
- Joins the proxy's network; **no published ports**, only the proxy talks to it.
- Config and persistent data from a host directory (not a named volume), so
  they're easy to inspect and back up. Mount read-only whatever the app only
  reads. Secrets in an `.env` file on the host (`chmod 600`), never in the repo.
- `restart: unless-stopped` and a **`mem_limit`** (on a small box the OOM killer
  otherwise picks the biggest process, which is rarely the culprit).
- Caddy: `reverse_proxy <service>:<port>`. Validate then **reload** the config
  (`caddy validate` → `caddy reload`) instead of restarting the proxy: zero
  downtime for every other site.

### Deploy recipe (`just deploy`)

- **Pre-flight locally:** validate the config you're about to ship (e.g.
  `docker compose config -q`, app config lint) so a broken file fails on your
  machine, not in production.
- **Sync config** to the server (`rsync --delete`, with include/exclude
  filters so only the intended files go).
- `IMAGE_TAG=… docker compose pull -q <svc> && IMAGE_TAG=… docker compose up -d --wait <svc>`.
  `--wait` blocks until the healthcheck passes, so a green deploy means a
  healthy container.
- **One SSH connection for all of it.** Several back-to-back logins trip
  per-source throttling (OpenSSH ≥ 9.8 `PerSourcePenalties`, fail2ban,
  firewall rate limits) and get reset mid-handshake. Open a multiplexed master
  and route everything through it:

  ```bash
  socket="$(mktemp -u "${TMPDIR:-/tmp}/deploy.XXXXXX")"
  ssh -fNM -o ControlPath="$socket" "$DEPLOY_HOST"
  trap 'ssh -o ControlPath="$socket" -O exit "$DEPLOY_HOST" 2>/dev/null' EXIT
  remote="ssh -o ControlPath=$socket"
  rsync -avz --delete -e "$remote" deploy/ "$DEPLOY_HOST:$DEPLOY_DIR/"
  $remote "$DEPLOY_HOST" "cd '$DEPLOY_DIR' && export IMAGE_TAG='${IMAGE_TAG:-latest}' \
    && docker compose pull -q '$SERVICE' && docker compose up -d --wait '$SERVICE'"
  ```

- Every setting from §0 is overridable via env vars with sensible defaults
  (`DEPLOY_HOST := env("DEPLOY_HOST", "…")` in the justfile).
- `just` echoes `#` lines inside a (non-shebang) recipe body: put explanatory
  comments *above* the recipe; the last comment line becomes its
  `just --list` doc.

### Changing live infrastructure safely

- **Back up before touching:** timestamped copies of the compose file and proxy
  config, and a tarball of any directory you're replacing.
- **Bring the new thing up alongside the old**, verify it from inside the proxy
  container (`docker exec caddy wget -qO- http://<service>:<port>/health`),
  *then* switch the proxy, *then* verify from the public URL (status,
  content type, something only the new version has).
- Leave the old files in place as a rollback until confirmed, and say so.

## 8. The day-to-day loop

```bash
# work
git commit -- <paths>             # conventional message
git fetch && git rebase origin/main && git push

# release
just release                      # or: minor | major | auto
git pull                          # pick up CI's changelog commit
gh run watch                      # image + changelog + GitHub release

# prerelease
just release minor rc             # start:   vX.Y.0-rc.1
IMAGE_TAG=next just deploy        # staging
just release pre                  # iterate: rc.2, rc.3 …
just release final                # promote: vX.Y.0

# ship
just deploy                       # sync, pull newest release, recreate, wait healthy
```

## 9. Bootstrapping a new repo: checklist

- [ ] `mise.toml` pinning just, git-cliff and the language toolchain (node + pnpm + biome, or python + uv + ruff); `mise install`
- [ ] Version file at `0.0.0` (`package.json` or `pyproject.toml`); the first `just release minor` makes `v0.1.0`
- [ ] Lockfile committed (`pnpm-lock.yaml` / `uv.lock`)
- [ ] Linter config: `biome.json` (`biome init`) or `[tool.ruff]` in `pyproject.toml`
- [ ] husky + commitlint hooks enforcing Conventional Commits (`pre-selected-tools.md` §2), and a commitlint check in CI
- [ ] `cliff.toml` from §2
- [ ] `justfile`: `lint`, `test`, `build`, `changelog`, `changelog-check`, `release-notes`, `release`, `deploy`
- [ ] `.github/workflows/changelog.yml`: regen on main + tags, commit back, GitHub (pre)release on tags
- [ ] `.github/workflows/docker.yml`: PR/manual: build + smoke; tag: build + smoke + per-arch native builds merged into a multi-arch manifest
- [ ] `Dockerfile` per `docker.md`: Alpine, multi-stage, deps stage, cache mounts, non-root, `HEALTHCHECK` on `/health`
- [ ] App exposes `/health`
- [ ] `.dockerignore`: allowlist
- [ ] `compose.yaml` in the repo showing the intended `image: ${IMAGE}:${IMAGE_TAG:-latest}` usage
- [ ] `.gitignore`: build output, deps, local data, `tmp/`
- [ ] README sections: quick start (image), deploying, releases & changelog
- [ ] Make the registry package public (GHCR defaults to private)
- [ ] First `just release minor` → confirm image tags, changelog section and GitHub release all appear
