# Setup-machine skill

Instructions for an agent to **write** a repo's `setup-machine` skill. Every
repo has one at `.claude/skills/setup-machine/SKILL.md`. What it does is
different in every repo, so this file isn't the skill. It explains how to
build the skill for a given repo.

The short version: **the skill takes a bare `git clone` on a machine with
nothing logged in to a repo where `just dev` and `just test` work. The agent
does every step it can. The user does every login. Nothing is skipped
silently.**

```
 bare clone ──▶ check tools ──▶ install (mise) ──▶ logins (user runs them) ──▶ env files
                                                                                  │
            report ◀── verify (just lint / just test)  ◀── services, migrations, seed ◀── dependencies
```

---

## 1. What the skill must do

- **Check before acting.** Every step first checks whether it's already done
  (tool present, logged in, file exists, container running), and skips it if
  so. Running the skill twice is safe and the second run changes nothing.
- **Install tools through mise.** Run `mise install` for everything in
  `mise.toml`. Anything mise can't install (Docker, NetBird, a system
  library) gets the install command for the user's OS, which the user
  approves.
- **Hand every login to the user.** The agent never types, pastes or asks for
  a password or token in chat. For each login it:
  - checks the status first (`gh auth status`, `docker info`, `netbird status`,
    `aws sts get-caller-identity`, …);
  - if the user isn't logged in, tells them the exact command to run with the
    `!` prefix (for example `! gh auth login`) and why it's needed;
  - waits, then runs the status check again before moving on.
- **Create every env file.** For each `.env.example` (or similar) in the repo,
  create the real file next to it. Each variable is one of three kinds:
  - **Local default**: copy the example value (`localhost` URLs, the dev
    database from `compose.yaml`).
  - **Generated**: create a random value locally (`openssl rand -hex 32`) for
    session secrets, signing keys and local passwords. A script does this
    (`scripts/setup-env.sh`, run by `just setup`): the `secrets.sh` hook
    stops the agent from writing `.env`, and the value should never pass
    through the agent anyway. The script copies `.env.example`, fills the
    empty generated values, never overwrites one, and exits non-zero listing
    what the user still has to provide.
  - **From the user**: an external secret (API key, OAuth client). Tell the
    user where to get it and ask them to paste it into the file themselves.
    Never print a secret's value in chat or logs.
- **Install dependencies and start services.** `pnpm install` / `uv sync`,
  `docker compose up -d --wait`, migrations and seed data.
- **Verify.** Run `just lint` and `just test` (or the fastest test set that
  proves the setup works), and open `just dev` once if the repo has a server.
- **Report.** End with a list of every step and its result (done, already
  done, skipped because the user declined, failed with the error). Don't say
  the setup is complete if any step failed or was skipped.

The mechanical, non-interactive parts belong in a `just setup` recipe so a
person can run them without an agent. The skill calls `just setup` and adds
what a script can't do: logins, secrets, judgment and the report.

---

## 2. How to write it for a repo

Survey the repo first. Every one of these can add a setup step:

| Look at | For |
|---|---|
| `mise.toml` | Tools and versions |
| `justfile` | Existing `setup`, `dev`, `db-*` recipes to reuse |
| Every `.env.example`, `.env.sample`, `*.env.template` | Env files and their variables |
| `compose.yaml` | Local services, ports, dev credentials |
| `package.json`, `pnpm-workspace.yaml`, `pyproject.toml` | Dependencies, postinstall steps, codegen |
| `.mcp.json`, `.claude/mcp-env.sh` | Env vars the MCP servers need |
| `.github/workflows/` | Secrets and logins CI needs, which the dev machine may need too |
| `AGENTS.md`, `README.md`, `docs/` | Setup steps written down by hand |
| `release-deploy.md` settings | Registry (GHCR), servers over NetBird, deploy keys |

Then:

- List every step, and every login with its status command.
- Sort every env variable into local default, generated or from the user.
  Write down where each "from the user" value comes from (which dashboard,
  which person).
- Write the skill in the format of §3. Put long checks in
  `.claude/skills/setup-machine/scripts/` and call them from the skill.
- **Test it on a fresh clone.** Clone the repo into a scratch folder and run
  the skill there from start to finish. Setup that has only been run on a
  machine already set up proves nothing.
- Add the skill to the Skills section of `AGENTS.md`.

Keep it current: any commit that adds an env variable, a service, a tool or a
login also updates the skill.

---

## 3. Skill format

````markdown
---
name: setup-machine
description: Set up this repo on a new machine after a bare git clone, installing tools, walking the user through logins, creating env files, starting services and verifying the result. Use when the user has just cloned the repo, is on a new machine, or `just dev` fails because something isn't set up.
allowed-tools: [Bash, Read, Write, Edit, AskUserQuestion]
---

# Setup machine

Takes a bare clone of <project> to a working `just dev` and `just test`.
Every step checks first and skips work that's already done. Never ask for,
type or print a secret. The user runs every login.

## 1. Tools
- Check: `mise --version`, `docker info`, <others>.
- Install: `mise install`. <OS-specific install commands for the rest.>

## 2. Logins (the user runs these)
| Service | Check | User runs | Why |
|---|---|---|---|
| GitHub | `gh auth status` | `! gh auth login` | <clone private deps, tickets> |
| <GHCR> | `docker login ghcr.io` succeeds | `! gh auth token \| docker login ghcr.io -u <user> --password-stdin` | <pull images> |
| <NetBird> | `netbird status` | `! netbird up` | <reach the servers> |

## 3. Env files
| File | From | Variable | Kind | Source |
|---|---|---|---|---|
| `.env` | `.env.example` | `DATABASE_URL` | local default | `compose.yaml` |
| `.env` | `.env.example` | `SESSION_SECRET` | generated | `openssl rand -hex 32` |
| `.env` | `.env.example` | `<API_KEY>` | from the user | <dashboard URL or person> |

## 4. Install, services, data
- `just setup` (<what it does>)
- <Migrations, seed data, codegen.>

## 5. Verify
- `just lint`, `just test`, `just dev` (then open <URL>).

## 6. Report
List every step: done, already done, skipped (why) or failed (error).
````

---

## 4. Bootstrapping a new repo: checklist

- [ ] Survey done (§2) and every env variable sorted into its kind
- [ ] `just setup` recipe for the mechanical steps
- [ ] `.claude/skills/setup-machine/SKILL.md` in the §3 format, plus any `scripts/`
- [ ] Run end to end on a fresh clone in a scratch folder, with every step reported as done
- [ ] Listed under Skills in `AGENTS.md`
