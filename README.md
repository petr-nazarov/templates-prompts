# Templates & prompts

My defaults for every repository: the tools, architecture, conventions and
agent setup a new project starts from. An agent reads these files and builds
a repo that follows them. People read them to know what the defaults are and
why.

## Start a repo with one prompt

Open Claude Code in the new (empty) folder, or in an existing repo that
should follow these templates, and send:

```text
Set up this repository from my templates at
https://github.com/petr-nazarov/templates-prompts (use
~/Projects/Personal/templates-prompts if it exists). Read new-repo.md there
and follow it end to end.

The project: <one or two sentences on what it is. Optional: you'll be asked.>
```

The agent gets the templates, asks all its questions in one round, then
builds the foundation, verifies it against every checklist, and reports
what's left for you (secrets, logins, GitHub settings).

## Rules in every repo

- **Semantic Versioning** for releases and **Conventional Commits** for
  commit messages.
- The user is the **only author** of every commit: no AI co-author trailers
  and no "Generated with Claude Code" lines, in commits or PRs.
- **Bullet lists, never numbered lists**, in docs, skills, tickets and
  handovers.
- **Claude Code with Superpowers** is the coding agent. `AGENTS.md` is the
  instructions file, and `CLAUDE.md` is a symlink to it.
- The templates are the defaults. A repo that departs from one writes down
  why.

## The files

**Entry point**

| File | What it is |
|---|---|
| [`new-repo.md`](new-repo.md) | The bootstrap procedure the prompt runs: get the templates, ask once, decide which files apply, build in order, verify and report. Works for new and existing repos. |

**Copied into the new repo as docs** (sections that don't apply deleted, Project settings filled in)

| File | What it covers | Becomes |
|---|---|---|
| [`pre-selected-tools.md`](pre-selected-tools.md) | The default toolbox by area (every repo, JS/TS, frontend, backend, auth, mobile, Python, infra, AI), with required vs. suggested tools and what each replaces. | `docs/tools.md` |
| [`architectural-decisions.md`](architectural-decisions.md) | Architecture patterns: the layered architecture (controller → service → repository), DI and base classes, each chosen at setup; plus schemas as contracts, error codes, request context and ADRs for every project. | `docs/architecture.md` |
| [`testing.md`](testing.md) | Test types, writing rules ("should" names, AAA, unhappy paths), real dependencies via Testcontainers, CI enforcement, mutation testing. | `docs/testing.md` |
| [`observability.md`](observability.md) | Logging to both a readable console and rotating ECS JSON Lines files, request IDs, redaction, access logs and retention, monitoring, health and uptime checks. | `docs/observability.md` |
| [`release-deploy.md`](release-deploy.md) | Commits, the changelog, releases and prereleases, CI/CD and deployment. | `docs/release-deploy.md` |
| [`docker.md`](docker.md) | Alpine multi-stage Dockerfiles, build cache, multi-arch builds, GHCR, compose, deploying. | `docs/docker.md` |

**Used to build the agent setup** (read by the agent, not copied)

| File | What it covers | Produces |
|---|---|---|
| [`ai-instructions.md`](ai-instructions.md) | Claude Code setup: `AGENTS.md` + `CLAUDE.md` symlink, committed settings with Superpowers, hooks, `.mcp.json` without secrets, worktrees, where new knowledge goes, and an `AGENTS.md` template. | `AGENTS.md`, `CLAUDE.md`, `.claude/`, `.mcp.json` |
| [`required-skills.md`](required-skills.md) | What the `cm` (commit, push, PR, merge) and `handover` skills must do. | `.claude/skills/cm/`, `.claude/skills/handover/` |
| [`setup-machine-skill.md`](setup-machine-skill.md) | How to write the repo's `setup-machine` skill: bare clone to a working `just dev`, with the user doing every login. | `.claude/skills/setup-machine/` |
| [`ticketing.md`](ticketing.md) | GitHub Issues ticketing: the `tickets` skill, issue template, account-check script, labels and status flow. Tickets are created only when the user asks for one. Only when GitHub Issues is the chosen tracker. | `.claude/skills/tickets/` |

**Copied as is**

| File | What it is |
|---|---|
| [`.gitignore`](.gitignore) | Broad `.gitignore`: temp files, logs, secrets, Node, Python, OSes, editors and agent files. |

## Changing the templates

- Every template has the same shape: a title, what it covers, **The short
  version** in bold, then numbered `## N.` sections, ending with
  **Bootstrapping a new repo: checklist**. `new-repo.md` verifies a repo
  against those checklists.
- Files point to each other by section (`docker.md` §3.3), so renumbering a
  section means updating its references.
- A new template gets a row in the right table above and in the plan table
  in `new-repo.md` §3.
