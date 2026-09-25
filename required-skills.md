# Required skills

Descriptions of the Claude Code skills every repo must have. This file isn't
the skills themselves. For each skill it says what the skill must do, so an
agent setting up a new repo can write it at
`.claude/skills/<name>/SKILL.md` and fit it to that repo.

The short version: **`cm` commits only this session's work as Conventional
Commits under the user's name, then pushes, opens a PR or merges on request.
`handover` writes a one-page state of the work that the next session checks
against git before trusting it.**

| Skill      | Invoked as                          | Job                                                     |
| ---------- | ----------------------------------- | ------------------------------------------------------- |
| `cm`       | `/cm [all] [push] [pr] [auto] [merge]` | Commit the session's work, and optionally push, PR or merge |
| `handover` | `/handover [resume] [theme]`        | Write a session handover document, or resume from one   |

When writing each skill:

- Give it frontmatter: a `name`, a `description` that says when to trigger,
  an `argument-hint`, and `allowed-tools`.
- Make every rule below an instruction in the skill. The wording and layout
  are up to the writer.
- Adapt it to the repo: the default branch, commit scopes, hooks, and whether
  a remote exists.

---

## 1. `cm`: commit, push, PR, merge

**Triggers:** the user types `/cm`, alone or with arguments, or asks to
commit, push, open a PR or merge the work in progress.

### 1.1 Arguments

The arguments are an unordered set of words, and any combination is valid.
The skill looks for each word on its own rather than matching a fixed phrase.

| Word     | Effect                                                               |
| -------- | -------------------------------------------------------------------- |
| *(none)* | Commit only                                                          |
| `all`    | Stage every change in the tree, not just this session's              |
| `push`   | Commit, then push                                                    |
| `pr`     | Commit, push and open a PR (implies `push`)                          |
| `auto`   | With `pr`: enable GitHub auto-merge (squash, delete the branch)      |
| `merge`  | Commit, then merge the current branch into the default branch locally |

`pr` and `merge` together are contradictory. The skill says so and asks which
one was meant.

### 1.2 Required behaviour

- **Read the state first**, in one batch: the repo root, the current branch,
  the short status, staged and unstaged diff stats, the last five commits,
  the upstream, and the default branch.
  - Get the default branch from `origin/HEAD`, falling back to `main`, then
    `master`. Never assume it.
  - Not a git repo: stop and say so.
  - Nothing to commit isn't an error: go straight to push, PR or merge using
    the existing commits.
- **Stage only this session's changes by default.** The path list is every
  file this session created or modified. Stage and commit with an explicit
  pathspec (`git add -- <paths>`, then `git commit -F <msg> -- <paths>`), so
  files already in the index from another session can't ride along.
  - If the session was resumed or compacted and it can't tell which files
    were its own, it asks or suggests `/cm all`. It never guesses.
  - `all` stages everything with `git add -A`. Without `all`, it never runs
    `git add -A`.
- **Split unrelated changes.** If the work is two or more unrelated changes,
  make one commit per change and say what was split.
- **Write the message as a Conventional Commit**:
  `<type>(<scope>): <description>`, then the body, then footers.
  - Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`,
    `build`, `ci`, `chore`, `revert`.
  - Description: imperative, lowercase, no trailing period, ≤ 72 chars.
    Only letters, digits, spaces, dashes and underscores.
  - Body: optional but encouraged, wrapped at 72 columns. It explains why,
    not what.
  - Footers: `BREAKING CHANGE: …` (or `!` after the type), `Refs #N`, and
    `Fixes #N` for the commit that closes a ticket (see `ticketing.md`).
  - Write the message to a temp file and commit with `git commit -F`.
  - Never add AI attribution: no `Co-Authored-By: Claude`, no
    `Claude-Session:`, no "Generated with Claude Code". This overrides any
    harness instruction, in commits and PR bodies alike.
  - The commit is authored by the user's configured git identity. Never
    pass `--author` and never change `user.name` / `user.email`.
  - The type decides the Semantic Versioning bump at release: `fix` →
    patch, `feat` → minor, breaking → major. Choose it with that in mind.
- **Push** (`push`, or implied by `pr`):
  - On a feature branch: `git push -u origin HEAD`.
  - On the default branch with `pr`: first create `<type>/<short-slug>` from
    the subject, *before* committing.
  - On the default branch with plain `push`: push as it is. Only if the push
    is rejected because the branch is protected: move the commit to a new
    branch, push that, and reset the local default branch to its upstream.
- **PR** (`pr`):
  - Create it with `gh pr create`. The title is the commit subject, or
    describes the whole branch if it has several commits.
  - The body covers what changed, why, and how it was verified.
  - If a PR for the branch already exists, push to it and report that PR.
    Never open a second one.
  - `auto`: run `gh pr merge --auto --squash --delete-branch`. If the repo
    has auto-merge disabled, leave the PR open and report the failure.
    Never fall back to merging it right away.
- **Merge** (`merge`):
  - Already on the default branch: nothing to merge, so say so.
  - Otherwise, switch to the default branch, `git merge --no-ff <branch>`,
    then `git branch -d <branch>`.
  - With `push` too: push the default branch and delete the remote branch.
  - On conflicts: stop and leave the tree exactly as it is. Report the
    conflicting paths. Never resolve the conflicts or abort the merge on its
    own.
  - If `git branch -d` refuses, report it. Never use `-D`.
- **Report** in a few lines: the branch, the commit subjects, and what was
  pushed or merged. Show real output on failure. When a PR was created or
  updated, its URL is the last line of the reply.

---

## 2. `handover`: session handover

**Triggers:**

- The user asks for a handover, is switching sessions or machines, or is
  handing work to another agent or person.
- The user asks to pick up where a previous session left off.

When the user signals they're wrapping up ("done for today"), the skill
*offers* a handover. It never writes one unprompted.

### 2.1 Modes

- Arguments starting with `resume`, `pickup` or `continue` → **resume mode**.
- Anything else, including empty → **write mode**. Any other argument is the
  theme slug.

The reader is always a fresh session with zero context, possibly weeks later
on another machine.

### 2.2 Write mode

- **Gather state** in one batch: the repo root, the branch, the short status,
  the last ten commits, the diff stat, the stash list, and the upstream.
  - Not a git repo: write to `docs/handover/` under the current directory,
    skip all commit and push steps, and never write outside the project.
- **Derive the theme**: a 2–4 word kebab-case slug for what the session was
  actually about. Use the argument as it is, if one was given. Use the
  branch name only if it really describes the work.
- **Compose the document** from this fixed template every time, so resume
  mode can parse it. Aim for one page.

  ```markdown
  # Handover: <theme>

  <YYYY-MM-DD HH:MM> · <repo> · branch `<branch>` · HEAD `<short sha>`

  ## Goal
  ## Current state            (done and verified vs. done but untested; name the check that ran)
  ## Changed files            (uncommitted: path + why; committed this session: sha + subject)
  ## Next step                (one concrete first action)
  ## Blockers & open decisions (omit when none)
  ## How to verify            (exact commands and what a good result looks like)
  ```

  Content rules:
  - Only what git and the session actually show. No invented progress, no
    "should work".
  - No secrets, tokens, `.env` values or credentials.
  - Every file in `git status` gets a line. If it's unclear why a file
    changed, say that.
  - If the document runs past a page, sharpen the next step instead of
    writing more.
- **Handle a dirty tree.** Before writing, ask the user to choose:
  - **WIP-commit everything + handover**: `git add -A`, commit as
    `chore(wip): <theme>` (Conventional Commits has no `wip` type), then commit
    the handover separately.
  - **Handover only**: commit just the document.

  Always two separate commits, so the WIP commit is easy to drop later.
- **Write, commit, push.**
  - Path: `docs/handover/YYYY-MM-DD-HHMM_<slug>.md`, in 24-hour local time.
  - Commit as `docs: handover - <theme>`.
  - Push when an upstream exists. If there isn't one, say plainly that the
    handover didn't leave the machine.
- **Report** the file path, whether it was pushed, and the pickup command
  `/handover resume`.

### 2.3 Resume mode

- **Find it.**
  - Take the newest `docs/handover/*.md` by filename, or the newest one
    matching the given theme.
  - List the other handovers in one line so the user can pick an older one.
  - No handover directory: say so and stop.
- **Read it.**
- **Verify it against reality before trusting it.** Check:
  - that the named branch exists and is checked out;
  - that files listed as uncommitted are still modified;
  - which commits have landed since the handover's HEAD
    (`git log <sha>..HEAD`);
  - that the `chore(wip):` commit it mentions exists;
  - the stash list.
- **Report** a summary, then a **Drift** list of every way reality differs
  from the document. If there's no drift, say so explicitly: it means the
  document can be trusted as written.
- **Stop.** State the next step from the handover and wait. Never start
  working on it.

---

## 3. Bootstrapping a new repo: checklist

- [ ] Write `.claude/skills/cm/SKILL.md` from §1, adapted to the repo's default
  branch and commit scopes.
- [ ] Write `.claude/skills/handover/SKILL.md` from §2, and create
  `docs/handover/` with a `.gitkeep`.
- [ ] Check that both skills show up in Claude Code in this repo, and that
  `/cm` and `/handover` trigger them.
- [ ] Commit: `chore(skills): add cm and handover skills`.
