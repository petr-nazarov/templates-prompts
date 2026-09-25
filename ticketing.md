# Ticketing playbook (GitHub Issues)

How a repository manages tickets. Tickets are **GitHub Issues** on the
project's own repo, driven by the `gh` CLI through a Claude Code skill
(`.claude/skills/tickets/`). Where a value differs per project, it comes from
**Project settings** below.

This is the preferred task tracker. The alternative, `tasks/<topic>.md`
files, is chosen when the repo is created (see `ai-instructions.md`, "Where
knowledge goes"). Set this up only if GitHub Issues was chosen.

The short version: **an issue's body is its current truth and its comments are
its history. Status lives in labels, and commits close issues with
`Fixes #N`.**

```
 "we need a ticket"  ──▶  duplicate check ──▶  facts + screenshots ──▶  gh issue create (status: todo | blocked)
                                                                               │
 start work ──▶ status: in progress + assignee ──▶ commits "Refs #N" ──▶ PR ──▶ status: in review
                                                                               │
                                          "Fixes #N" reaches main  ──▶  closed (completed), status label removed
```

---

## 0. Project settings

Fill these in once per repository. Everything below refers to them by name.

| Setting    | Example                      | Used by                          |
| ---------- | ---------------------------- | -------------------------------- |
| `REPO`     | `<owner>/<repo>`             | skill, account script, labels    |
| `PROJECT`  | `CardBook`                   | skill wording                    |
| Areas      | `mobile`, `api`, `web`, `infra` | `area:` labels                |
| Assets dir | `docs/tickets/assets/`       | screenshots committed for issues |

---

## 1. Files to create

```
.claude/skills/tickets/
├── SKILL.md                     # §2
├── template.md                  # §3
└── scripts/ensure-gh-account.sh # §4 (chmod +x)
```

Replace `<owner>/<repo>`, `<Project>` and the area list in all three files.

---

## 2. `.claude/skills/tickets/SKILL.md`

````markdown
---
name: tickets
description: Use when the user says we need a ticket (including a design ticket asking for a design decision or confirmation), or asks to file, open, create, update, re-status, block, or close a ticket or issue for <Project>, or when work just finished belongs to an existing ticket. Tickets are GitHub Issues on <owner>/<repo>, managed with the gh CLI.
---

# Tickets (GitHub Issues)

<Project>'s tickets are **GitHub Issues** on `<owner>/<repo>`, driven through
the `gh` CLI. This skill covers creating them, keeping them true as work
happens, moving their status, and closing them.

## 0. Always first: act as the right account

This machine may have several GitHub accounts. Before **any** `gh` call in
this skill, run:

```bash
bash .claude/skills/tickets/scripts/ensure-gh-account.sh
```

It finds the logged-in account that can push to `<owner>/<repo>` and switches
`gh` to it (`gh auth switch`) if another account is active.

- **Exit 0**: go on. If a switch happened, mention it in your reply:
  `gh auth switch` is global, so other projects' `gh` calls now also run as
  this account until the user switches back.
- **Exit 2**: not logged in, or no logged-in account has access. Stop, and ask
  the user to run `! gh auth login` with the right account. Don't create
  tickets anywhere else in the meantime.
- **Exit 3**: `GH_TOKEN` is set to the wrong account and overrides any switch.
  Stop and tell the user. Don't unset it yourself.

## 1. Labels: type, area, status, priority

Status lives in labels, because GitHub only has open and closed. Create the
label set once, and again if any is missing (`--force` makes it idempotent):

```bash
R=<owner>/<repo>
gh label create "status: todo"        -R $R --force --color EDEDED --description "Ready to pick up"
gh label create "status: in progress" -R $R --force --color 1D76DB --description "Being worked on"
gh label create "status: blocked"     -R $R --force --color B60205 --description "Waiting on a blocker or decision"
gh label create "status: in review"   -R $R --force --color 5319E7 --description "PR open or awaiting verification"
gh label create "type: bug"           -R $R --force --color D73A4A
gh label create "type: feature"       -R $R --force --color A2EEEF
gh label create "type: chore"         -R $R --force --color FEF2C0
gh label create "type: design"        -R $R --force --color F9D0C4 --description "Needs a design decision or confirmation"
gh label create "priority: high"      -R $R --force --color B60205
gh label create "priority: medium"    -R $R --force --color FBCA04
gh label create "priority: low"       -R $R --force --color 0E8A16
gh label create "area: design"        -R $R --force --color F9D0C4
# one per project area:
gh label create "area: <area>"        -R $R --force --color C5DEF5
```

Every open ticket has exactly **one** `status:` label, one `type:` label and
at least one `area:` label. Add a `priority:` label when the user states one,
or when the impact makes it obvious (a crash, data loss).

| State       | Labels / issue state                                         |
| ----------- | ------------------------------------------------------------ |
| Todo        | open, `status: todo`                                         |
| In progress | open, `status: in progress`, assigned (`--add-assignee @me`) |
| Blocked     | open, `status: blocked`, and the Blockers section says on what |
| In review   | open, `status: in review`, and the PR is linked              |
| Done        | closed as completed; remove the `status:` label              |
| Won't do    | closed as not planned, with a comment saying why             |

Changing status always swaps the label in one call, so an issue never has
zero or two:

```bash
gh issue edit N -R $R --remove-label "status: todo" --add-label "status: in progress"
```

## 2. Creating a ticket

- **Check for a duplicate first:**
  `gh issue list -R $R --state all --search "<key words>" --limit 20`.
  If one matches, update it instead and say so.
- **Gather facts before writing.** Reproduce the problem or read the code.
  A ticket states what was observed. Mark a guess as a guess.
- **Take screenshots whenever the ticket involves a screen.** Every screen
  the ticket names gets a real screenshot from the running app: the broken
  state, and for a design ticket each screen involved. See "Screenshots".
  Skip only when nothing visible is involved (an API or infra ticket) or the
  screen really can't be reached, and then say why under Evidence.
- **Write the body from `.claude/skills/tickets/template.md`.** Keep every
  section. Write "None" rather than deleting one, so readers can tell it was
  considered. What each section must carry:
  - **Current state**: the observed behaviour, where it happens (app,
    platform, OS, screen, endpoint), and how it was found.
  - **Desired state**: the outcome, not the implementation.
  - **Decisions to be made**: every open question the user or team must
    settle, each with options, trade-offs and a recommendation. Never settle
    one silently yourself.
  - **Blockers**: concrete, each with a link to what unblocks it.
  - **Acceptance criteria**: checkboxes that someone who didn't write the
    code can verify.
- **Title**: an imperative or a problem statement, specific, ≤ 70 chars, no
  ticket jargon. Write `Android 15+ cuts the last word off single-line texts`,
  not `Bug: text issue`.
- **Create** it with the body read from a file, so the Markdown survives the
  shell:

  ```bash
  gh issue create -R $R --title "..." --body-file /path/to/body.md \
    --label "type: bug" --label "area: <area>" --label "status: todo"
  ```

  The initial status is `status: todo`, or `status: blocked` when a blocker
  or an open decision stops work from starting.
- **Report** the issue number, title, labels and URL, with the URL on the
  last line.

When the user says "we need a ticket", that is the go-ahead to create it.
Don't ask for confirmation again. Ask only when the ticket depends on a fact
that is unknown and can't be found in the code or the app.

### Design tickets

A design ticket asks the designer to decide or confirm something before it's
built. Examples: a screen that's missing information, a flow with two
reasonable answers, copy nobody has written, or a state the designs don't
cover. Open one when the answer is the designer's to give, not the
developer's.

- **Title**: starts with `[Design]`, then the question or problem.
- **Labels**: `type: design`, `area: design`, the product area it's in, and
  `status: todo`.
- **Body**: the same template, written for a designer rather than a
  developer:
  - **Summary / Current state**: the situation in product terms: who sees
    what, where, and why it's a problem. Put a screenshot of every screen
    involved inline, next to the sentence it illustrates. Mention code facts
    only where they limit the options.
  - **Decisions to be made**: the heart of the ticket. Each question has its
    options, the trade-off of each, and a recommendation. Say what the
    designer should deliver: a yes or no, a frame, or copy.
  - **Desired state**: "design has decided X, and the frames are linked",
    not the implementation.
  - **Acceptance criteria**: the decision is recorded in the ticket, and any
    frames are linked under Links.
- Once it's decided, record the decision in the body (see §3). If the
  decision needs building, open a separate dev ticket that links back to it.
  Don't turn the design ticket into a dev ticket.

### Screenshots

Take them from the running app, never from memory or a mock-up:

- **Web**: a Playwright screenshot (`page.screenshot({ path, fullPage: true })`).
- **Android emulator**: `adb exec-out screencap -p > shot.png`.
- **iOS simulator**: `xcrun simctl io booted screenshot shot.png`.
- **Getting to the screen**: drive the app with the project's e2e tooling,
  logged in as a dev seed user. Use seed data only, never a real user's.
- **Design side**: when the ticket compares the app with the design, add the
  frame itself (e.g. via the Figma tools) and link the node under Links.
- Look at each screenshot before attaching it, and drop any that don't show
  the point. Name each file for what it shows
  (`incoming-request-no-cardhook.png`) and caption it in the body.

### Images

`gh` can't upload attachments to issues. For screenshots:

- Commit them under `docs/tickets/assets/<issue-number>-<slug>/` and push.
- Reference each one by commit SHA, so the link survives later moves:
  `![caption](https://github.com/<owner>/<repo>/blob/<sha>/docs/tickets/assets/<n>-<slug>/file.png?raw=true)`.
- If the current branch has commits that shouldn't be pushed yet, don't
  push it just for the images. Commit them on their own branch from
  `origin/main` in a temporary worktree, and push only that branch:

  ```bash
  git fetch origin main
  git worktree add -b tickets/<n>-assets <scratch>/assets-wt origin/main
  # copy images into <scratch>/assets-wt/docs/tickets/assets/<n>-<slug>/, commit, then:
  git -C <scratch>/assets-wt push -u origin tickets/<n>-assets
  git worktree remove <scratch>/assets-wt
  ```

- If the images can't be pushed at all, still create the ticket, list the
  local paths under Evidence, and tell the user the images still need
  uploading. They can also drag them into the issue in the browser.

The folder name needs the issue number, so create the issue first, then
commit the images and add them with `gh issue edit N --body-file`.

## 3. Keeping a ticket true as work happens

A ticket's body is its **current truth**. Its comments are its **history**.

- **Starting work**: set status to `in progress` and `--add-assignee @me`.
- **Progress worth recording** (a finding, a partial fix, a changed plan):
  comment with `gh issue comment N -R $R --body-file ...`. Say what was done,
  with commit SHAs or PR links, and what's next.
- **The facts changed** (new findings, a settled decision, a new blocker):
  edit the body so it's true again (`gh issue view N --json body -q .body`,
  edit, then `gh issue edit N --body-file`), then comment on what changed.
  - A settled decision: tick it and add
    `→ Decided YYYY-MM-DD by <who>: <outcome>`.
  - A new blocker: add it to Blockers and set status to `blocked`. When it
    clears, strike it through (`~~…~~`) with the date and set the right
    status.
  - A verified acceptance criterion: tick it.
- **Commits** follow Conventional Commits (the `cm` skill writes them).
  Put `Refs #N` in the footer of related commits, and
  `Fixes #N` in the one that finishes the ticket. `Fixes` closes the issue
  when it reaches `main`.
- **PR opened**: set status to `in review` and link the PR under Links.

When a session finishes work that belongs to an open ticket, update that
ticket before reporting done, whether or not the user asked for it.

## 4. Closing

Close a ticket only when every acceptance criterion is ticked, or when the
user says to.

```bash
gh issue edit N -R $R --remove-label "status: in review"   # whichever status it had
gh issue close N -R $R --reason completed --comment "Done in <sha/PR>. Verified: <how>."
```

For won't-do or a duplicate, use `--reason "not planned"` with a comment
giving the reason or the duplicate's number. To reopen, run
`gh issue reopen N` and set a status.

## Common mistakes

| Mistake | Fix |
|---|---|
| Running `gh issue` before the account check | Always run `ensure-gh-account.sh` first |
| Two `status:` labels, or none, on an open issue | Swap with `--remove-label` and `--add-label` in one `gh issue edit` |
| Deleting an empty section | Write "None" |
| Deciding an open question in the body | List it under Decisions with options and a recommendation |
| Progress only in comments; the body still describes the old state | Edit the body to match, then comment |
| Closing with unchecked acceptance criteria | Verify and tick them, or ask the user |
| `--body "..."` with multi-line Markdown | Use `--body-file` |
| Linking screenshots by branch name | Link by commit SHA |
| A ticket about a screen with no screenshot of it | Take one from the running app, or say under Evidence why you couldn't |
| A design ticket that decides the design itself | Put the options and a recommendation under Decisions, and let design choose |
````

---

## 3. `.claude/skills/tickets/template.md`

````markdown
## Summary

<!-- Two or three sentences: what is wrong or missing, who it affects, why it matters now. -->

## Current state

<!-- What happens today, as observed: facts, not guesses. Include where (app,
platform, OS version, screen, endpoint) and how it was found. -->

## Desired state

<!-- What "done" looks like from the user's side. Describe the outcome, not the
implementation. -->

## Evidence

<!-- Screenshots, logs, error text, repro steps. For images, see "Images" in
the skill. Write repro steps as bullets, in order, that a stranger could follow. -->

## Decisions to be made

<!-- One checkbox per open question. Each gets the options and a
recommendation with its reason. Once it is settled, tick it and add the
outcome, the date and who decided. Write "None" when there are none. -->

- [ ] **Question?**
  - Option A — trade-off.
  - Option B — trade-off.
  - Recommendation: A, because …

## Blockers

<!-- What stops progress and who or what can unblock it: another issue (#123),
an upstream fix (link), access, a decision above. Write "None" when there are
none. -->

## Acceptance criteria

<!-- Checkable statements, each verifiable by someone who did not write the
code. Tick them as they are verified. -->

- [ ] …

## Out of scope

<!-- What this ticket deliberately does not cover, so it does not grow. -->

## Links

<!-- Related issues and PRs, commits, upstream issues, docs, design nodes. -->
````

---

## 4. `.claude/skills/tickets/scripts/ensure-gh-account.sh`

```bash
#!/usr/bin/env bash
# Make the gh CLI act as the project's GitHub account before touching issues.
#
# The right account is whichever logged-in github.com account can push to
# $REPO. No login is hardcoded, so this keeps working if the account is
# renamed or a teammate runs it.
#
# Exit codes:
#   0  gh now acts as an account with push access (its login is printed)
#   2  nothing to switch to: not logged in, or no logged-in account has access
#   3  GH_TOKEN is set to an account without access (it overrides any switch)
set -euo pipefail

REPO="${TICKETS_REPO:-<owner>/<repo>}"
HOST="github.com"

can_push() { # [token] -> exit 0 when that identity can push to $REPO
  local out
  if [ -n "${1:-}" ]; then
    out=$(GH_TOKEN="$1" gh api "repos/$REPO" --jq '.permissions.push' 2>/dev/null || true)
  else
    out=$(gh api "repos/$REPO" --jq '.permissions.push' 2>/dev/null || true)
  fi
  [ "$out" = "true" ]
}

login_of_env_token() { # prints the login GH_TOKEN belongs to, or nothing
  local login
  if login=$(gh api user --jq .login 2>/dev/null); then
    echo "$login"
  fi
}

# GH_TOKEN (e.g. from a mise.local.toml) beats the active account, so a switch
# would change nothing. Accept it when it is right, refuse loudly when not.
if [ -n "${GH_TOKEN:-}" ]; then
  if can_push "$GH_TOKEN"; then
    echo "gh: using GH_TOKEN ($(login_of_env_token)), which has access to $REPO"
    exit 0
  fi
  who=$(login_of_env_token)
  echo "gh: GH_TOKEN is set to ${who:-an invalid token}, which cannot push to $REPO." >&2
  echo "    GH_TOKEN overrides gh auth switch. Unset it or point it at an account with access." >&2
  exit 3
fi

accounts=$(gh auth status --hostname "$HOST" --json hosts \
  --jq ".hosts[\"$HOST\"][]? | select(.state == \"success\") | \"\(.login) \(.active)\"" 2>/dev/null || true)

if [ -z "$accounts" ]; then
  echo "gh: not logged in to $HOST. Log in with an account that can push to $REPO: gh auth login" >&2
  exit 2
fi

active=$(awk '$2 == "true" { print $1 }' <<<"$accounts")

if [ -n "$active" ] && can_push; then
  echo "gh: active account $active already has access to $REPO"
  exit 0
fi

while read -r login _; do
  [ "$login" = "$active" ] && continue
  if can_push "$(gh auth token --hostname "$HOST" --user "$login")"; then
    gh auth switch --hostname "$HOST" --user "$login" >/dev/null
    echo "gh: switched active account from ${active:-none} to $login (access to $REPO)"
    exit 0
  fi
done <<<"$accounts"

echo "gh: none of the logged-in accounts ($(awk '{print $1}' <<<"$accounts" | paste -sd, -)) can push to $REPO." >&2
echo "    Log in with an account that has access: gh auth login" >&2
exit 2
```

---

## 5. Bootstrapping a new repo: checklist

- [ ] Fill in **Project settings** (§0).
- [ ] Create the three files from §2–§4 with the placeholders replaced, then
  `chmod +x .claude/skills/tickets/scripts/ensure-gh-account.sh`.
- [ ] Run `ensure-gh-account.sh`. It must exit 0.
- [ ] Create the label set (skill §1), with one `area:` label per project area.
- [ ] Create `docs/tickets/assets/` with a `.gitkeep`.
- [ ] Commit: `chore(tickets): add GitHub Issues ticketing skill`.
