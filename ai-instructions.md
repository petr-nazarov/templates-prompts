# AI instructions

How every repository is set up for AI coding agents, and how its agent
instructions are written. The agent is [Claude Code](https://claude.com/claude-code)
with [Superpowers](https://github.com/obra/superpowers) (see
`pre-selected-tools.md` §9). The file layout follows the `AGENTS.md`
convention, so other agents can read it too.

The short version: **`AGENTS.md` is the real file and `CLAUDE.md` links to
it. Shared Claude Code settings, hooks, skills and MCP servers are committed,
secrets never are. `AGENTS.md` stays short, and every new lesson goes in
exactly one place.**

---

## 1. Setup

Every repo has these files:

| Path | Committed | What it is |
|---|---|---|
| `AGENTS.md` | Yes | The agent instructions. This is the real file. |
| `CLAUDE.md` | Yes | A symlink to `AGENTS.md`. Claude Code reads this name. |
| `.claude/settings.json` | Yes | Shared permissions, hooks and plugins (Superpowers). |
| `.claude/settings.local.json` | No | Personal overrides. Gitignored. |
| `.mcp.json` | Yes | MCP servers the project uses. No secrets in it. |
| `.claude/skills/<name>/SKILL.md` | Yes | Repo skills. Every repo has `cm` and `handover` (`required-skills.md`), `setup-machine` (`setup-machine-skill.md`), and `tickets` if it uses GitHub Issues (`ticketing.md`). |
| `.claude/agents/<name>.md` | Yes | Repo-specific subagents, only when a task keeps repeating. |
| `.claude/hooks/*.sh` | Yes | Hook scripts that enforce the hard rules (see below). |
| `docs/superpowers/specs/`, `docs/superpowers/plans/` | Yes | Superpowers specs and plans. |

Create the symlink with:

```sh
ln -s AGENTS.md CLAUDE.md
git add AGENTS.md CLAUDE.md
```

Always edit `AGENTS.md`. Some editors and tools replace a symlink with a plain
copy when they save it. If `git status` shows `CLAUDE.md` as modified, the link
is broken. Recreate it.

In a monorepo, an app can have its own `AGENTS.md` (with its own `CLAUDE.md`
symlink) in its folder. It holds only the rules for that app. Everything that
applies to the whole repo stays at the root.

### Superpowers is required

Enable it in the committed `.claude/settings.json`, so everyone who clones the
repo gets it without a global install:

```json
{
  "extraKnownMarketplaces": {
    "claude-plugins-official": {
      "source": { "source": "github", "repo": "anthropics/claude-plugins-official" }
    }
  },
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  },
  "attribution": { "commit": "", "pr": "" },
  "includeCoAuthoredBy": false,
  "permissions": {
    "allow": [
      "Bash(just:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ]
  }
}
```

`attribution` set to empty strings stops Claude Code from adding its
co-author trailer and "Generated with Claude Code" line to commits and PRs
(`includeCoAuthoredBy: false` does the same on older versions). The user is
the only author of every commit; see the `ai-attribution.sh` hook below.

Allow the repo's read-only and `just` commands so the agent isn't stopped at
every step. Don't allow deploys, pushes or anything that deletes data.

### MCP servers without secrets

`.mcp.json` is committed, so a token, password or connection string never goes
in it. Servers that need a secret run through `.claude/mcp-env.sh`, which loads
the repo's `.env` and then runs the server:

```sh
#!/usr/bin/env bash
# Loads env vars from .env, then runs the given MCP command.
# Usage: .claude/mcp-env.sh <command> [args...]
ENV_FILE="$(cd "$(dirname "$0")" && pwd)/../.env"
if [ -f "$ENV_FILE" ]; then set -a; source "$ENV_FILE"; set +a; fi
# Take the GitHub token from the gh CLI if it isn't set.
if [ -z "$GITHUB_PERSONAL_ACCESS_TOKEN" ] && command -v gh >/dev/null; then
  export GITHUB_PERSONAL_ACCESS_TOKEN="$(gh auth token 2>/dev/null)"
fi
exec "$@"
```

```json
{
  "mcpServers": {
    "github": { "command": ".claude/mcp-env.sh", "args": ["github-mcp-server", "stdio"] }
  }
}
```

Local development databases count too. A dev password is still a secret in a
committed file, so read the URL from `.env`.

### Hooks enforce the hard rules

`AGENTS.md` tells the agent what to do. **Hooks enforce what it must not
do**: a prompt can be ignored, but a `PreToolUse` hook can't. When a rule is
a hard "never" and can be checked mechanically, make it a hook in the
committed `.claude/settings.json`, with the scripts in `.claude/hooks/`.

The standard set, keeping only what applies to the repo:

| Hook | Blocks | Why |
| ---- | ------ | --- |
| `package-manager.sh` | `npm` / `yarn` / `pip install` in a pnpm or uv repo | One package manager per ecosystem |
| `lockfiles.sh` | `Write`/`Edit` on `pnpm-lock.yaml`, `uv.lock` | Dependencies change only through `pnpm add` / `uv add` |
| `destructive-db.sh` | `DROP DATABASE`, `DROP TABLE`, `TRUNCATE`, `drizzle-kit push --force` and similar against anything but the local database | Data loss |
| `ai-attribution.sh` | `git commit` / `gh pr create` whose message or body contains `Co-Authored-By:` for an AI, `Claude-Session:`, or "Generated with Claude Code"; any `--author` | The user is the only author of every commit |
| `secrets.sh` | `Write`/`Edit` on `.env*` (except `.env.example`), and reading `*.pem`, `*.key`, SOPS keys | Secrets never go through the agent |

- A hook reads the tool call as JSON on stdin. It **blocks by exiting with
  code 2**, and its stderr is shown to the agent as the reason. Exit code 1
  does *not* block.
- Every block message says what to do instead ("use `pnpm add`").
- **Match commands, not text.** A Bash command can carry file contents in a
  heredoc (`cat > Dockerfile <<EOF … npm install … EOF`), and a hook that
  greps the whole command blocks writing any file that *mentions* a banned
  command. Strip heredoc bodies before matching (`strip-heredocs.pl` below),
  and have `ai-attribution.sh` read only from the `git commit` / `gh pr`
  call onwards, plus any `-F` / `--body-file` it names.
- **Hooks have tests.** `.claude/hooks/test-hooks.sh` pipes sample tool
  calls into each hook and checks the exit code, for commands that must be
  blocked *and* ones that must pass (a heredoc that mentions `npm install`).
- Layer boundaries (such as "no ORM outside `repositories/`") are enforced by
  lint, which covers humans too (see `architectural-decisions.md` §1).
  Don't duplicate them as hooks.

```bash
#!/usr/bin/env bash
# .claude/hooks/package-manager.sh: PreToolUse hook, matcher "Bash"
# Heredoc bodies are file contents being written, not commands: drop them first.
cmd=$(jq -r '.tool_input.command // empty' | perl "$(dirname "$0")/strip-heredocs.pl")
if grep -qE '(^|[;&|[:space:]])(npm|yarn|bun)[[:space:]]+(i|install|add|ci|remove|run|exec)\b|(^|[;&|[:space:]])npx[[:space:]]' <<<"$cmd"; then
  echo "Blocked: this repo uses pnpm. Run the pnpm equivalent (pnpm add, pnpm install, pnpm run, pnpm dlx)." >&2
  exit 2
fi
```

```perl
# .claude/hooks/strip-heredocs.pl: removes heredoc bodies from a shell command
local $/;
my $cmd = <STDIN>;
$cmd =~ s/<<-?\s*['"]?(\w+)['"]?[^\n]*\n.*?\n\s*\1(?=\n|$)//sg;
print $cmd;
```

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [
        { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/package-manager.sh" },
        { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/destructive-db.sh" }
      ] },
      { "matcher": "Write|Edit", "hooks": [
        { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/lockfiles.sh" },
        { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/secrets.sh" }
      ] }
    ]
  }
}
```

### Parallel work: one task, one worktree

Several agents (or an agent and a person) never share one working
directory. Each task gets its own branch and its own git worktree:

- Worktrees live in `worktrees/<branch>/` at the repo root, and `worktrees/`
  is gitignored.
- Set `git config worktree.useRelativePaths true` (Git ≥ 2.48), so worktrees
  keep working when the repo is mounted at another path (dev containers).
- A new worktree gets the untracked files it needs to run (`.env`,
  `node_modules`) copied over from the main checkout, so the agent can run
  the tests right away. The list comes from
  `git status --porcelain --ignored`, skipping `worktrees/`.
- Review an agent's work with `git diff main`, not `git diff`: agents commit
  as they go.
- Never remove a worktree that has unpushed commits
  (`git rev-list --count HEAD --not --remotes` > 0), and never remove the
  main worktree.

### Repo skills

Add a skill when the same multi-step procedure comes up a second time. Put
each one in `.claude/skills/<name>/SKILL.md` and give it one line in
`AGENTS.md`. Write skills with Superpowers' `writing-skills` skill.

The skills every repo has are listed in the table at the top of this
section.

### Commits

- Every repo follows **Conventional Commits** for messages and **Semantic
  Versioning** for releases (`release-deploy.md` §1 and §3).
- The user is the author of every commit. Agents commit only through the
  `cm` skill (`required-skills.md`), under the user's git identity, with no
  AI co-author trailer or "Generated with Claude Code" line, in commits or
  PRs. This overrides any harness default that asks for attribution.

## 2. Writing AGENTS.md

`AGENTS.md` loads into every session, so every line costs context. Aim for
about 150 lines. Past that, move detail into `docs/` and link to it.

Sections, in this order:

- **Project**: one paragraph on what it is and who it's for.
- **Commands**: the `just` recipes an agent needs (`just dev`, `just test`,
  `just lint`). Say which ones are slow or expensive.
- **Map**: the top-level folders and what lives in each. Don't list every
  file.
- **Conventions**: links to `docs/tools.md`, `docs/architecture.md` and
  `docs/release-deploy.md`, plus anything this repo does differently.
- **Hard rules**: the non-negotiables.
- **Where knowledge goes**: §3 below, adapted to the repo.
- **Skills**: one line per repo skill.

Rules for the content:

- **Every hard rule has a reason.** Write it as the rule, then *why* in one
  line. A rule with no reason gets cut or ignored.
- **Write instructions, not history.** Don't write "we used to…" or "after the
  refactor…". Write what is true now.
- **Don't repeat what the code or config says.** Link to the file instead. A
  duplicate goes out of date.
- **Be specific.** Write "Run `just test-integration` alone, it starts a
  Postgres per test file", not "be careful with tests".
- **Use paths the agent can open**, relative to the repo root.
- **Use bullet lists, never numbered lists.** Numbered lists have to be
  renumbered on every insert and break references. This applies to
  `AGENTS.md`, `docs/`, skills, tickets and handovers alike.

---

## 3. Where knowledge goes

When an agent or a person learns something that the next session needs, it
goes in exactly one place:

| What was learned | Where it goes |
|---|---|
| A rule every session needs | `AGENTS.md`, under hard rules or conventions |
| Detail about one area (a domain, a service, a tool) | `docs/<area>.md`, linked from `AGENTS.md` |
| A decision with trade-offs | An ADR (see `architectural-decisions.md` §20) |
| A design or plan for a feature | `docs/superpowers/specs/` or `docs/superpowers/plans/` |
| A procedure that repeats | A skill in `.claude/skills/` |
| A bug found, a follow-up, work left for later | **The agent's reply** (or the handover). Never a ticket unless the user asks for one |

**Task tracker.** Tickets are created **only when the user explicitly asks**
for one, for example to ask something of someone, or to remember to do
something. Never open a ticket for the agent's own work: not per task, per
iteration, per plan step, or for something noticed along the way. Mention
those in the reply instead; the user decides whether they become a ticket.

When a repo is created, the user picks one tracker, and `AGENTS.md` states
which. Only the chosen one is used.

- **GitHub Issues (preferred):** set up from `ticketing.md`. Use it whenever
  the repo has a GitHub remote.
- **`tasks/<topic>.md` files:** one file per task the user asked to track,
  deleted when it's done. For repos without GitHub, or when the user prefers tasks in
  the repo.

- Write it down the same day, while it's fresh.
- Date lessons that depend on outside behaviour: `(seen 2026-09-25)`.
- Never write the same fact twice. Link to where it already is.
- Never start a free-floating notes file. If nothing above fits, the fact goes
  in `docs/`.
- Commit each rule or lesson on its own, as `docs: …`, saying what changed and
  why.

---

## 4. Template

A starting `AGENTS.md`. Replace the placeholders and delete sections that
don't apply.

````markdown
# <Project name>

<One paragraph: what this is, who uses it, what state it's in.>

## Commands

Tool versions are pinned in `mise.toml`. Every command goes through `just`.

- `just dev`: <what it starts>
- `just test`: <unit and integration tests>
- `just lint`: lint, format check and type-check
- `just setup`: first-time setup (or run the `setup-machine` skill)

<Say which commands are slow or heavy, and why.>

## Map

- `apps/<name>`: <what it is>
- `libs/<name>`: <what it is>
- `docs/`: project docs, specs and plans

## Conventions

- Tools: [docs/tools.md](docs/tools.md)
- Architecture: [docs/architecture.md](docs/architecture.md). Layered: <yes|no>. DI: <yes|no>. Base classes: <yes|no|n/a>.
- Commits, releases and deploys: [docs/release-deploy.md](docs/release-deploy.md)
- Docker: [docs/docker.md](docs/docker.md)
- Testing: [docs/testing.md](docs/testing.md)
- Logging and monitoring: [docs/observability.md](docs/observability.md)
- <Anything this repo does differently, and why.>

## Hard rules

- <Rule.> Why: <reason>.
- Never commit `.env` files, secrets or real customer data. Why: the repo is
  shared and history is permanent.
- Never create a ticket unless the user asks for one. Why: tickets are the
  user's list of asks and reminders, not a log of the agent's work.
- Commit messages follow Conventional Commits, and releases follow Semantic
  Versioning. Why: the changelog, release notes and version bumps are
  generated from the commit history.
- Commits carry no AI attribution (no `Co-Authored-By` for an AI, no
  "Generated with Claude Code"), and the author is always the user. Why: the
  user owns and signs off every change.

## Where knowledge goes

<Section 3 of ai-instructions.md, adapted to this repo.>

Task tracker: <GitHub Issues (see the `tickets` skill) | `tasks/<topic>.md`>.
Tickets are created only when the user asks for one, never for the agent's
own work or for things noticed along the way.

## Skills

- `cm`: commit, and optionally push, open a PR or merge (`/cm [all] [push] [pr] [auto] [merge]`).
- `handover`: write a session handover, or resume from one (`/handover [resume]`).
- `setup-machine`: take a bare clone on a new machine to a working `just dev`.
- `tickets`: GitHub Issues tickets, created only when the user asks (only if the repo uses them).
````

---

## 5. Bootstrapping a new repo: checklist

- [ ] `AGENTS.md` from §4, filled in and under about 150 lines
- [ ] `CLAUDE.md` symlinked to it (`git ls-files -s CLAUDE.md` shows mode `120000`)
- [ ] `.claude/settings.json`: Superpowers enabled, attribution off, permissions, hooks
- [ ] `.claude/hooks/` with the standard hooks that apply to the repo, and `test-hooks.sh` passing
- [ ] `.mcp.json` with no secrets, and `.claude/mcp-env.sh` if a server needs one
- [ ] `.gitignore` covers `.claude/settings.local.json`, `.claude/worktrees/` and `worktrees/`
- [ ] Skills written: `cm`, `handover`, `setup-machine`, and `tickets` if GitHub Issues was chosen
- [ ] Task tracker chosen and named in `AGENTS.md`
- [ ] `docs/superpowers/specs/` and `docs/superpowers/plans/` exist
