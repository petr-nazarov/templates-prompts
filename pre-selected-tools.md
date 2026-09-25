# Pre-selected Tools and Technologies

The default toolbox for a new repository. It covers several languages and
kinds of project on purpose. A new repo keeps the sections that apply to it and
drops the rest. Don't add a competing tool for something already listed here
without a reason worth writing down.

Each entry says what the tool is for and what it replaces. **Required** tools
go into every repo where they apply. **Suggested** tools are the ones our
current projects already use. Start with them unless the project needs
something else.

The short version: **mise pins the tools, just runs them. pnpm + Biome +
TypeScript, uv + Ruff for Python. React Router, Tailwind and shadcn/ui on the
web, with searchable drop-downs. Postgres + Drizzle, Docker images in GHCR,
Caddy in front, NetBird between machines, Claude Code with Superpowers.**

---

## 1. Every repository

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [git](https://git-scm.com) | Required | Version control. Use Conventional Commits for every commit and Semantic Versioning for every release (see `release-deploy.md`). Commits are authored by the user, never with AI attribution. | — |
| [mise](https://mise.jdx.dev) | Required | Pinning tool versions (`node`, `pnpm`, `python`, `uv`, `just`, …) in `mise.toml`. It manages versions only. Tasks go in the justfile. | nvm, pyenv, asdf, `.tool-versions`, mise tasks |
| [just](https://just.systems) | Required | The one place to run the repo's commands (`just dev`, `just test`, `just release`, `just deploy`). Recipes are thin wrappers over `package.json` scripts or `uv run`. | Makefiles, loose shell scripts, npm scripts as the entry point |
| [git-cliff](https://git-cliff.org) | Required | Generating `CHANGELOG.md` and release notes from conventional commits. CI regenerates the changelog. Nobody edits it by hand. | Hand-edited changelogs, custom `changelog.js` scripts, changesets |
| [rsync](https://rsync.samba.org) | Required | Syncing config, data and static builds to servers (`rsync -avz --delete`). Use `--inplace` for single files that a container bind-mounts. | scp, ad-hoc tarballs |
| [NetBird](https://netbird.io) | Required | Private mesh network between laptops, servers and CI. SSH and deploys go over it, so admin ports don't have to be open to the internet. | Public SSH, wg-easy / hand-managed WireGuard, Tailscale |
| Docker + Compose | Suggested | Local dependencies (Postgres, Mailpit) and production images. Use a `compose.yaml` per repo. | Installing services on the host |
| GitHub Actions + GHCR | Suggested | CI. On a tag, build the image, smoke-test it and push it to `ghcr.io/<org>/<app>`. | Building images on the server |
| [gitleaks](https://github.com/gitleaks/gitleaks) | Suggested | Scanning for secrets before commit and in CI. | — |
| [SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age) | Required | Secrets that must be versioned (server `.env` files, infra variables): committed as `*.sops.*` files, with values encrypted and keys readable. The age private key never enters the repo. Local dev `.env` files stay uncommitted. | Plaintext secrets in git, secrets passed around in chat |
| [Dev Containers](https://containers.dev) (`@devcontainers/cli`) | Suggested | A reproducible dev environment for repos with awkward system dependencies, or for running agents in isolation. Commit `.devcontainer/`. Forward the host's `ssh-agent` socket; never mount private keys. Use docker-outside-of-docker. | "Works on my machine" setup docs |

A typical `mise.toml`:

```toml
# mise manages tool versions only. Task running lives in the justfile.
[tools]
just = "latest"
node = "lts"
# pnpm from its GitHub release: the aqua registry entry had no linux asset for
# pnpm 12 in mise 2026.3, and "npm:pnpm" breaks in CI (pnpm 12's npm package
# replaces itself with a native binary in a postinstall script mise skips).
"github:pnpm/pnpm" = "latest"
uv = "latest"
gitleaks = "latest"  # the pre-commit hook and CI use it
```

---

## 2. JavaScript / TypeScript

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [pnpm](https://pnpm.io) | Required | Package manager and workspaces (`pnpm-workspace.yaml`). pnpm 12 runs no dependency build scripts until they're approved: list them under `allowBuilds` in `pnpm-workspace.yaml` (`esbuild: true` for tsx and Vite, `false` for optional native add-ons). It also holds back packages published in the last day, and `pnpm add` of one writes a `minimumReleaseAgeExclude` entry; review those. | npm, yarn, bun |
| [Biome](https://biomejs.dev) | Required | Lint and format in one tool (`biome check`, `biome check --write`). | ESLint + Prettier, lint-staged setups |
| [husky](https://typicode.github.io/husky) + [commitlint](https://commitlint.js.org) | Required | Git hooks, committed in `.husky/`: `commit-msg` runs `commitlint --edit "$1"` (`@commitlint/config-conventional`), so every commit is a Conventional Commit, and `pre-commit` runs `biome check --staged` and `gitleaks git --staged`. Installed by the `prepare` script on `pnpm install`. CI runs commitlint on PR commits too, since hooks can be skipped. See the hook notes below the table. | Unchecked commit messages, lint-staged, pre-commit (Python framework) |
| TypeScript | Suggested | All JS code, with `strict` on. | Plain JS |
| Node.js LTS | Suggested | Runtime, pinned by mise. Use `node:<lts>-alpine` in Dockerfiles. | — |
| [Vite](https://vite.dev) | Suggested | Dev server and bundler for SPAs. | webpack, CRA |
| [Vitest](https://vitest.dev) | Suggested | Unit and integration tests. | Jest |
| [Playwright](https://playwright.dev) | Suggested | End-to-end tests against a compose stack. | Cypress |
| [Testcontainers](https://testcontainers.com) | Suggested | Integration tests against a real, throwaway Postgres. | Mocked databases, shared test DBs |
| [WireMock](https://wiremock.org) | Suggested | Faking third-party HTTP APIs in integration tests (via its Testcontainers module). | Hitting real vendor sandboxes from tests |
| [Stryker](https://stryker-mutator.io) | Suggested | Periodic mutation testing of critical modules (money, auth). See `testing.md`. | Coverage percentage targets |
| [Turborepo](https://turbo.build) | Suggested | Only for monorepos with several apps and libs. | nx, lerna |
| [Zod](https://zod.dev) | Suggested | Runtime validation and shared contracts between client and server. | joi, yup, class-validator for new code |

**Git hook notes** (seen 2026-09-25):

- Git runs hooks in a shell that hasn't activated mise, so `gitleaks` or the
  pinned pnpm may be missing. Run tools through
  `run() { if command -v mise >/dev/null 2>&1; then mise exec -- "$@"; else "$@"; fi; }`.
- husky sets a *local* `core.hooksPath`, which hides a global one (for
  example a global `commit-msg` that strips AI trailers). Chain it at the top
  of `.husky/commit-msg`:
  `g=$(git config --global --path core.hooksPath); [ -x "$g/commit-msg" ] && "$g/commit-msg" "$1"`.

---

## 3. Web frontend

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| React | Suggested | UI. | — |
| [React Router](https://reactrouter.com) | Required | Routing, and framework mode (`@react-router/dev`) for full-stack or SSR apps. Import from `react-router`, not `react-router-dom`. | Next.js, TanStack Router, `react-router-dom` |
| [Tailwind CSS](https://tailwindcss.com) v4 | Required | Styling through `@tailwindcss/vite`. | Sass, CSS modules, styled-components |
| [shadcn/ui](https://ui.shadcn.com) | Suggested | Components copied into the repo, built on Radix / Base UI, with `clsx`, `tailwind-merge`, `class-variance-authority` and `tw-animate-css`. | MUI, Chakra, Ant |
| lucide-react | Suggested | Icons. | Font Awesome, heroicons |
| Geist (`@fontsource-variable/geist`) | Suggested | Default UI font, self-hosted. | Google Fonts CDN |
| sonner, cmdk, react-day-picker | Suggested | Toasts, command palette, date picker. | — |
| shadcn/ui Combobox (Popover + `cmdk` Command) | Required | Every drop-down. It must be searchable: the user types to filter the options. | Native `<select>`, non-searchable `Select` components |
| date-fns (+ `@date-fns/tz`) | Suggested | Dates and time zones. | moment, dayjs |
| react-i18next | Suggested | Only when the app needs translations. | — |
| vite-plugin-pwa | Suggested | Only for installable or offline apps. | Hand-written service workers |
| [Astro](https://astro.build) | Suggested | Static and content sites (websites, blogs, catalogs). | Hugo, Gatsby, Next static export |

**Drop-downs are always searchable.** When a spec, ticket or prompt says
"drop-down", "select" or "picker", build a searchable combobox, even when there
are only a few options. This is not optional. The same rule applies to mobile
apps: use a searchable picker, not the platform's plain picker.

---

## 4. Backend (TypeScript)

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [Hono](https://hono.dev) | Suggested | Small APIs and a single process that serves `dist/` plus `/api`. | Express for new code |
| [NestJS](https://nestjs.com) | Suggested | Larger APIs that need modules, DI and many domains. Use SWC (`unplugin-swc`) for tests. | — |
| PostgreSQL | Suggested | Default database. Use `postgres:<major>-alpine` in compose. | MySQL, MongoDB unless required |
| [pino](https://getpino.io) (`nestjs-pino`) | Required | Logging: `pino-pretty` to the console and `pino-roll` to rotating ECS JSON Lines files (`@elastic/ecs-pino-format`), with redaction and a request ID (see `observability.md`). | `console.log`, winston |
| [Drizzle ORM](https://orm.drizzle.team) + drizzle-kit | Suggested | Schema, queries and migrations. | Prisma, TypeORM, MikroORM |
| [Mailpit](https://mailpit.axllent.org) | Suggested | Catching email in local and e2e environments. | Real SMTP in dev |

---

## 5. Authentication

Pick **one** per project:

| Tool | Status | Use it when | Instead of |
|---|---|---|---|
| [Better Auth](https://better-auth.com) | Required (self-hosted) | The app owns its users in its own Postgres. Pairs with Drizzle, and `just db-generate` regenerates its schema. | Passport + JWT, Lucia, Auth.js |
| [Descope](https://descope.com) | Required (managed) | The client wants a hosted identity provider (flows, SSO, passkeys, OTP) and doesn't want to run it themselves. SDKs exist for JS and Python. | Auth0, Clerk, Cognito, Firebase Auth |

**Better Auth as an MCP server's OAuth server** (seen 2026-09-25, better-auth
1.7): MCP clients (Claude Code, claude.ai connectors, the Claude apps) need
dynamic client registration, PKCE and tokens bound to the `/mcp` resource.

- Use `@better-auth/mcp` with the `jwt()` plugin. better-auth 1.7 no longer
  ships the old built-in `mcp` plugin. `requireMcpAuth` guards `/mcp` and
  sends the `WWW-Authenticate` challenge. Route `/.well-known/*` to
  `auth.handler`.
- Claude Code registers `http://localhost:<port>/callback` without
  `application_type`, which the provider rejects as a web client. A `before`
  hook on `/oauth2/register` sets `application_type: "native"` for loopback
  redirect URIs.
- The schema CLI is the `auth` package (`better-auth generate --adapter
  drizzle --dialect postgresql`), not `@better-auth/cli`. Point it at a config
  built without a database.
- The plugin writes to the database when it starts, so the app needs its
  database even to answer `/health`. Image smoke tests run a Postgres next to
  it (`docker.md` §4).
- Restrict sign-in with `user.validateUserInfo`. It sees the provider profile
  (the GitHub `login`) and runs on every sign-in, not only sign-up.

Better Auth and Descope are for the app's own users. Authelia (§8)
guards internal tools. It isn't an app auth library.

---

## 6. Mobile

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [React Native](https://reactnative.dev) | Required | Mobile apps. It shares TypeScript, Zod contracts and logic with the web apps. | Flutter, native-only Swift/Kotlin |
| [Expo](https://expo.dev) | Required | Toolchain for React Native: Expo Router, EAS Build/Submit, OTA updates. | Bare React Native CLI projects |

---

## 7. Python

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [uv](https://docs.astral.sh/uv) | Required | Python versions, venvs, dependencies, lockfile (`uv.lock`) and workspaces (`[tool.uv.workspace]`). Run everything through `uv run`. | pip, pip-tools, poetry, pipenv, pyenv |
| [Ruff](https://docs.astral.sh/ruff) | Suggested | Lint and format (`select = ["E","F","I","UP","B","SIM"]`). | black, isort, flake8, pylint |
| pytest (+ pytest-asyncio) | Suggested | Tests. Use Testcontainers for real databases. | unittest |
| [FastAPI](https://fastapi.tiangolo.com) + uvicorn | Suggested | HTTP APIs. | Flask, Django REST for new services |
| stdlib `logging` + [ecs-logging](https://github.com/elastic/ecs-logging-python) | Required | Logging: a coloured `StreamHandler` and a `RotatingFileHandler` with the ECS formatter, behind a `QueueHandler` (see `observability.md`). | `print`, `basicConfig` plain text |
| pydantic-settings | Suggested | Config from the environment. | Hand-rolled `os.environ` parsing |
| hatchling | Suggested | Build backend for workspace packages (`src/` layout). | setuptools |

---

## 8. Infrastructure and deployment

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [Caddy](https://caddyserver.com) | Required | Reverse proxy and automatic TLS on self-hosted servers. One shared Caddyfile per server. | nginx + certbot, Traefik |
| [Authelia](https://www.authelia.com) | Required | SSO and 2FA in front of internal tools and dashboards, through Caddy `forward_auth`. It protects things that have no login of their own. | Basic auth, exposing admin UIs, Authentik |
| Docker Compose on the server | Suggested | Running published images: `docker compose pull && up -d --force-recreate --wait`. The default for one server. | Building on the server |
| [Docker Swarm](https://docs.docker.com/engine/swarm/) | Suggested | 2–10 servers: the same Compose syntax, rolling updates, service discovery. Bind-mount directories must exist before deploying. | Kubernetes for small clusters |
| [Pulumi](https://www.pulumi.com) | Required | Cloud infrastructure as code (Python or TypeScript), whenever a project uses AWS or another cloud. It runs from CI. Nobody changes resources by hand in a console. | Terraform, CDK, clicking in consoles |
| [Ansible](https://docs.ansible.com) | Suggested | Configuring servers after provisioning: users, packages, Docker, and SSH hardening (key-only login, password auth off). It's declarative, so re-runs are safe. Secrets via `community.sops`. | Hand-run setup scripts, SSH-ing in to fix things |
| [Cloudflare](https://www.cloudflare.com) | Suggested | DNS and the proxy in front of public hostnames: hides the origin IP, WAF, bot filtering. The origin accepts web traffic only through the proxy, and admin access goes over NetBird. | Exposing the origin IP directly |
| [Restic](https://restic.net) | Required (where data persists) | Encrypted, incremental, off-site backups of databases (dump first) and bind-mounted data to S3-compatible storage, on a schedule, with a restore tested at least once. | No backups, same-disk copies |
| [Dozzle](https://dozzle.dev) + [Beszel](https://beszel.dev) + [Gatus](https://github.com/TwiN/gatus) | Suggested | Small-setup observability: container logs, host/container metrics, uptime checks with alerts. Grafana + Loki + Prometheus only when the project needs more (see `observability.md`). | Uptime Kuma, no monitoring |
| Kubernetes + Kustomize + ArgoCD | Only if required | Only for client projects that already run on it. Compose for one server and Swarm for a few are the defaults. | — |

---

## 9. AI-assisted development

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [Claude Code](https://claude.com/claude-code) | Required | The coding agent. Every repo has an `AGENTS.md`, with `CLAUDE.md` symlinked to it, that describes commands (`just …`), conventions and this tool list. | — |
| [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) + `@hono/mcp` | Suggested | Remote MCP servers: Streamable HTTP on `/mcp`, stateless (a new server and transport per request), OAuth through Better Auth (§5). | Hand-rolled JSON-RPC, SSE-only servers |
| [Superpowers](https://github.com/obra/superpowers) | Required | Claude Code skills for the working process: brainstorm → spec → plan → TDD → review. Specs and plans go in `docs/superpowers/`. | Ad-hoc prompting |

---

## 10. Bootstrapping a new repo: checklist

- [ ] This file copied to `docs/tools.md`, linked from `AGENTS.md`, with the sections the repo doesn't need deleted
- [ ] `mise.toml` pinning the tools the repo uses
- [ ] `justfile` with `_default: @just --list --unsorted`
- [ ] `biome.json` or the Ruff config
- [ ] husky hooks (`commit-msg` with commitlint, `pre-commit` with Biome and gitleaks) and `commitlint.config.js`
- [ ] Any departure from these choices noted under its entry, with the reason
