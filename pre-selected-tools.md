# Pre-selected Tools and Technologies

The default toolbox for a new repository. It covers several languages and
kinds of project on purpose. A new repo keeps the sections that apply to it and
drops the rest. Don't add a competing tool for something already listed here
without a reason worth writing down.

Each entry says what the tool is for and what it replaces. **Required** tools
go into every repo where they apply. **Suggested** tools are the ones our
current projects already use. Start with them unless the project needs
something else.

---

## 1. Every repository

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [git](https://git-scm.com) | Required | Version control. Use Conventional Commits (see `release-deploy.md`). | — |
| [mise](https://mise.jdx.dev) | Required | Pinning tool versions (`node`, `pnpm`, `python`, `uv`, `just`, …) in `mise.toml`. It manages versions only. Tasks go in the justfile. | nvm, pyenv, asdf, `.tool-versions`, mise tasks |
| [just](https://just.systems) | Required | The one place to run the repo's commands (`just dev`, `just test`, `just release`, `just deploy`). Recipes are thin wrappers over `package.json` scripts or `uv run`. | Makefiles, loose shell scripts, npm scripts as the entry point |
| [git-cliff](https://git-cliff.org) | Required | Generating `CHANGELOG.md` and release notes from conventional commits. CI regenerates the changelog. Nobody edits it by hand. | Hand-edited changelogs, custom `changelog.js` scripts, changesets |
| [rsync](https://rsync.samba.org) | Required | Syncing config, data and static builds to servers (`rsync -avz --delete`). Use `--inplace` for single files that a container bind-mounts. | scp, ad-hoc tarballs |
| [NetBird](https://netbird.io) | Required | Private mesh network between laptops, servers and CI. SSH and deploys go over it, so admin ports don't have to be open to the internet. | Public SSH, wg-easy / hand-managed WireGuard, Tailscale |
| Docker + Compose | Suggested | Local dependencies (Postgres, Mailpit) and production images. Use a `compose.yaml` per repo. | Installing services on the host |
| GitHub Actions + GHCR | Suggested | CI. On a tag, build the image, smoke-test it and push it to `ghcr.io/<org>/<app>`. | Building images on the server |
| [gitleaks](https://github.com/gitleaks/gitleaks) | Suggested | Scanning for secrets before commit and in CI. | — |

A typical `mise.toml`:

```toml
# mise manages tool versions only. Task running lives in the justfile.
[tools]
just = "latest"
node = "lts"
pnpm = "latest"
uv = "latest"
```

---

## 2. JavaScript / TypeScript

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [pnpm](https://pnpm.io) | Required | Package manager and workspaces (`pnpm-workspace.yaml`). | npm, yarn, bun |
| [Biome](https://biomejs.dev) | Required | Lint and format in one tool (`biome check`, `biome check --write`). | ESLint + Prettier, lint-staged setups |
| TypeScript | Suggested | All JS code, with `strict` on. | Plain JS |
| Node.js LTS | Suggested | Runtime, pinned by mise. Use `node:<lts>-alpine` in Dockerfiles. | — |
| [Vite](https://vite.dev) | Suggested | Dev server and bundler for SPAs. | webpack, CRA |
| [Vitest](https://vitest.dev) | Suggested | Unit and integration tests. | Jest |
| [Playwright](https://playwright.dev) | Suggested | End-to-end tests against a compose stack. | Cypress |
| [Testcontainers](https://testcontainers.com) | Suggested | Integration tests against a real, throwaway Postgres. | Mocked databases, shared test DBs |
| [Turborepo](https://turbo.build) | Suggested | Only for monorepos with several apps and libs. | nx, lerna |
| [Zod](https://zod.dev) | Suggested | Runtime validation and shared contracts between client and server. | joi, yup, class-validator for new code |

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
| [Drizzle ORM](https://orm.drizzle.team) + drizzle-kit | Suggested | Schema, queries and migrations. | Prisma, TypeORM, MikroORM |
| [Mailpit](https://mailpit.axllent.org) | Suggested | Catching email in local and e2e environments. | Real SMTP in dev |

---

## 5. Authentication

Pick **one** per project:

| Tool | Status | Use it when | Instead of |
|---|---|---|---|
| [Better Auth](https://better-auth.com) | Required (self-hosted) | The app owns its users in its own Postgres. Pairs with Drizzle, and `just db-generate` regenerates its schema. | Passport + JWT, Lucia, Auth.js |
| [Descope](https://descope.com) | Required (managed) | The client wants a hosted identity provider (flows, SSO, passkeys, OTP) and doesn't want to run it themselves. SDKs exist for JS and Python. | Auth0, Clerk, Cognito, Firebase Auth |

Better Auth and Descope are for the app's own users. Authelia (section 8)
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
| pydantic-settings | Suggested | Config from the environment. | Hand-rolled `os.environ` parsing |
| hatchling | Suggested | Build backend for workspace packages (`src/` layout). | setuptools |

---

## 8. Infrastructure and deployment

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [Caddy](https://caddyserver.com) | Required | Reverse proxy and automatic TLS on self-hosted servers. One shared Caddyfile per server. | nginx + certbot, Traefik |
| [Authelia](https://www.authelia.com) | Required | SSO and 2FA in front of internal tools and dashboards, through Caddy `forward_auth`. It protects things that have no login of their own. | Basic auth, exposing admin UIs |
| Docker Compose on the server | Suggested | Running published images: `docker compose pull && up -d --force-recreate --wait`. | Building on the server |
| [Pulumi](https://www.pulumi.com) | Required | Cloud infrastructure as code (Python or TypeScript), whenever a project uses AWS or another cloud. | Terraform, CDK, clicking in consoles |
| Kubernetes + Kustomize + ArgoCD | Only if required | Only for client projects that already run on it. A single server with Compose is the default. | — |

---

## 9. AI-assisted development

| Tool | Status | Use it for | Instead of |
|---|---|---|---|
| [Claude Code](https://claude.com/claude-code) | Required | The coding agent. Every repo has an `AGENTS.md`, with `CLAUDE.md` symlinked to it, that describes commands (`just …`), conventions and this tool list. | — |
| [Superpowers](https://github.com/obra/superpowers) | Required | Claude Code skills for the working process: brainstorm → spec → plan → TDD → review. Specs and plans go in `docs/superpowers/`. | Ad-hoc prompting |

---

## Using this in a new repository

1. Copy this file into the repo as `docs/tools.md` and link it from `AGENTS.md`.
2. Delete the sections the repo doesn't need.
3. Create `mise.toml`, a `justfile` with `_default: @just --list --unsorted`,
   `biome.json` or the Ruff config, and `cliff.toml`.
4. If the repo departs from one of these choices, note it under the entry and
   say why.
