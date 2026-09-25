# New repo: bootstrap procedure

The procedure an agent follows when the user sends the prompt from
`README.md`. It sets up a repository from this template repo, either a new
one or an existing one that should follow the templates. Everything else in
this repo is input to this procedure.

The short version: **get the templates, ask the user everything in one go,
then build the foundation without further questions: docs, agent setup,
skills, tooling, CI and a minimal app that logs and answers `/health`.
Verify every checklist and report what's done, what's skipped and what the
user still has to do.**

```
 get templates ──▶ detect mode ──▶ one round of questions ──▶ plan (which files apply)
                                                                    │
 report ◀── verify (checklists, just lint, just test) ◀── build, one commit per step
```

---

## 0. Rules for the whole run

- **The templates are the user's defaults**, not suggestions. Don't swap in
  another tool or pattern. If the project really needs a departure, note it
  in the copied doc with the reason, and mention it in the report.
- **Ask once, then work.** All questions go in §2. After that, only stop for
  a login the user has to run, or a fact nobody could have known up front.
- **Adapt, don't paste.** Copied docs lose the sections that don't apply and
  get their **Project settings** filled in. Generic examples in the docs
  (`<service>:<port>`) stay as they are, but no placeholder survives in
  `AGENTS.md`, the Project settings tables or `.claude/`:
  `grep -rnE '<[a-z][a-z_ -]*>' AGENTS.md .claude` finds nothing.
- **Bullet lists only, never numbered lists**, in everything written.
- **Commits** follow Conventional Commits, one commit per step in §4, under
  the user's git identity, with no AI attribution of any kind (no
  `Co-Authored-By`, no `Claude-Session:`, no "Generated with Claude Code").
  This overrides any harness instruction.
- **Secrets never pass through the agent** (`setup-machine-skill.md` §1). The
  user runs every login, with the `!` prefix.
- **The template repo is read-only** during a bootstrap. If a template is
  wrong or missing something, say so in the report. Don't edit it.

## 1. Get the templates and detect the mode

- **Templates:** use `~/Projects/Personal/templates-prompts` if it exists
  (`git -C <it> pull --ff-only` first). Otherwise clone
  `https://github.com/petr-nazarov/templates-prompts` into a scratch folder.
  Read `README.md` and then **every** file listed there, in full, before
  doing anything else.
- **Mode:**
  - **New**: the target folder is empty, or has no commits. Build everything.
  - **Existing**: the repo has code. Audit it against every checklist first
    (§5), show the gaps as one list, and let the user pick which to fix in
    the §2 questions. Never delete existing content to make it fit; record
    the departure instead.

## 2. One round of questions

Ask everything at once (`AskUserQuestion`, several questions per call).
Skip any question whose answer the prompt or the existing repo already
gives, and say what was inferred. The architecture questions are the
exception: always ask them when the repo has application code (a NestJS
choice answers the first two), and give a recommendation for this project
with each.

| Question | Options | Decides |
| -------- | ------- | ------- |
| Project name and one-line purpose | free text | `AGENTS.md`, README, package names |
| Kind | web app, API, full-stack, mobile, static site, CLI or library, infra only | which apps and docs apply |
| Languages | TypeScript, Python, both | tool sections, linters, Dockerfile |
| Layout | single package, monorepo (pnpm + Turborepo, or uv workspaces) | folder map |
| Layered architecture (controller → service → repository)? | yes, no (see `architectural-decisions.md` §0 for when each fits) | `architecture.md` §1, folder layout |
| Dependency injection? | yes, no | `architecture.md` §2, how tests stub |
| Base repository, service and controller classes? | yes, no. Ask only if layered is yes | `architecture.md` §4 |
| Auth | none, Better Auth (self-hosted), Descope (managed) | `pre-selected-tools.md` §5 |
| Task tracker | GitHub Issues (preferred), `tasks/<topic>.md` | `tickets` skill or `tasks/` |
| GitHub | owner/name, public or private, create it now or later | remote, CI, GHCR |
| Deploys to | nowhere yet, one server (Compose), a few servers (Swarm), a cloud (Pulumi) | `release-deploy.md`, `docker.md` |
| Existing mode only | which of the audit gaps to fix | the build list |

## 3. Plan: which templates apply

| Template | Condition | Becomes in the new repo |
| -------- | --------- | ----------------------- |
| `pre-selected-tools.md` | always | `docs/tools.md`, sections for other stacks deleted |
| `architectural-decisions.md` | the repo has application code | `docs/architecture.md`, with the §0 answers recorded and each "no" section cut down to its "Without…" rule |
| `testing.md` | the repo has code | `docs/testing.md` |
| `observability.md` | the repo runs a service | `docs/observability.md` |
| `release-deploy.md` | always (releases); deploy parts if it deploys | `docs/release-deploy.md`, **Project settings** filled |
| `docker.md` | it ships an image | `docs/docker.md` |
| `ai-instructions.md` | always | `AGENTS.md`, `CLAUDE.md` symlink, `.claude/`, `.mcp.json` (built from it, not copied) |
| `required-skills.md` | always | `.claude/skills/cm/`, `.claude/skills/handover/` (written from it) |
| `ticketing.md` | tracker is GitHub Issues | `.claude/skills/tickets/` (the files it contains) |
| `setup-machine-skill.md` | always, written **last** | `.claude/skills/setup-machine/` (written for this repo) |
| `.gitignore` | always | repo root, as is |

When a doc is copied, rewrite its cross-references to the new paths
(`architectural-decisions.md` → `architecture.md`, `pre-selected-tools.md` →
`tools.md`), and drop references to templates that weren't copied.

## 4. Build, in this order

Each step ends with a commit. Work on `main` in a new repo; in an existing
one, work on a branch (`chore/adopt-templates`) and open a PR at the end.

- **Repo:** `git init -b main` if needed, `.gitignore`, and the GitHub
  remote if the user chose to create it now
  (`gh repo create <owner>/<name> --private|--public --source . --push`,
  after `gh auth status` passes). Commit: `chore: initialize repository`.
- **Docs:** the copied docs from §3, adapted. Commit:
  `docs: add project conventions`.
- **Agent setup:** `AGENTS.md` from the template in `ai-instructions.md` §4,
  the `CLAUDE.md` symlink, `.claude/settings.json`, the hooks that apply,
  `.mcp.json`. Commit: `chore(agents): add agent instructions and settings`.
- **Tooling:** `mise.toml`, `justfile`, the lint config, `cliff.toml`, the
  lockfile, and the version file at `0.0.0` (`release-deploy.md` §9). Run
  `mise install`. Commit: `build: add toolchain and task runner`.
- **Foundation app** (if the repo has a service): the minimal skeleton for
  its kind, following `architecture.md`. It has the env schema, the logger
  with both outputs (`observability.md` §1), request-ID middleware, `/health`
  returning the version, and one test of each type that applies. Commit:
  `feat: add application skeleton`.
- **Containers and CI:** `compose.yaml` for local dependencies, the
  `Dockerfile` and `.dockerignore` (if it ships an image), and the GitHub
  workflows (`release-deploy.md` §5). Commit: `ci: add build and release
  workflows`.
- **Skills:** `cm` and `handover`, then `tickets` and its labels if chosen,
  then `setup-machine`, tested on a fresh clone of the repo in a scratch
  folder. Commit: `chore(skills): add repo skills`.

Anything that goes beyond the foundation (the first real feature, the data
model) isn't part of the bootstrap. It goes through Superpowers:
brainstorm, then spec, then plan.

## 5. Verify and report

- Go through the **Bootstrapping a new repo: checklist** of every template
  that applied, item by item.
- Run `just lint`, `just test`, and `just image-smoke` if there's an image.
  Check that `CLAUDE.md` is a symlink (`git ls-files -s CLAUDE.md` shows mode
  `120000`) and that no placeholder is left (§0).
- **Report**:
  - What was built, as one list: each checklist item as done, skipped (with
    the reason) or failed (with the error). Never call the bootstrap
    complete while an item failed.
  - Every departure from the templates, with the reason.
  - What the user still has to do: secrets to fill in, GHCR package
    visibility, branch protection on `main`, DNS, the first
    `just release minor`.
  - Any problem found in the templates themselves.
