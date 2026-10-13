# Notifications playbook

How a repository pings a person: Pushover for push notifications to a phone,
and a Telegram bot posting into a group. It covers the choice of channels,
routing, the notifier script, each provider's setup and limits, testing, and
catching a notifier that has gone quiet. Uptime checks that alert from
outside are in `observability.md` §5.

**Optional.** A repo gets this doc only when the user chose at least one
channel during setup (`new-repo.md` §2). It then keeps only the sections for
the channels it uses, with **Project settings** (§0) filled in. Pushover and
Telegram are the channels described today. Another one is added the way §7
says, without changing the callers.

The short version: **one notifier script that every caller goes through, with
a level per message. Alerts go to the alert channel, routine pings only to a
channel nobody gets a push from. Messages are content-free, the notifier
rate-limits itself before the provider does, it never fails or blocks its
caller, a dead-man's switch catches a silent notifier, and tests run against a
fake.**

---

## 0. Project settings

Fill these in once per repository; everything below refers to them by name.

| Setting | Default | Used by |
| ------- | ------- | ------- |
| Channels | the ones chosen at setup: Pushover, Telegram, or both | notifier, env file, checklist |
| Alert channel | Pushover if chosen, otherwise Telegram | §1, exit status |
| What pings | breakage always; waiting and routine as chosen at setup | §1 routing |
| `NOTIFY_WAITING` | `1` if "waiting for a person" was chosen, else `0` | §1 |
| `NOTIFY_ROUTINE` | `1` if routine pings were chosen and a Telegram group exists, else `0` | §1 |
| `NOTIFY_PREFIX` | `[<project>]` | every message |
| `NOTIFY_STATE_DIR` | `$XDG_STATE_HOME/<project>/notify` (`~/.local/state/…`) | §2 |
| Heartbeat | the dead-man's switch service and its check URL, or "none yet" | §6 |

Record who chose each value and when in the copied doc, next to the table.

## 1. Routing: who gets what

Every message has a **level**. The routing table maps a level to channels,
depending on which channels the repo has:

| Level | Example | Pushover + Telegram | Pushover only | Telegram only |
| ----- | ------- | ------------------- | ------------- | ------------- |
| `alert` | crash, failed job, failed push, disk low, a dependency down | Pushover, plus a copy in the group | Pushover | group, with sound |
| `waiting` | a prompt or approval is pending | group; Pushover only if `NOTIFY_WAITING=1` | Pushover if `NOTIFY_WAITING=1`, else nothing | group, with sound |
| `routine` | started, finished, pushed, deployed | group, silent | nothing (events log only) | group, silent |
| notifier trouble | the Telegram channel is muted or disabled | Pushover, once | n/a | events log, and the heartbeat is withheld (§6) |

- **Pushover is for things that need a person.** A phone that buzzes for
  every routine event gets muted, and then the real alarm is missed. Keep it
  to alerts, and decide each new ping's level on purpose. Routine pings never
  go to Pushover, even quietly.
- **The group is the timeline.** It can be noisy because routine messages go
  out with `disable_notification=true` (§4.2), so nobody gets a push for
  them. People read it when they want to know what happened.
- **With Telegram only,** the group is both timeline and alarm. Alerts are
  sent with sound, everything else silently, and the people who need alerts
  keep the group's notifications on.
- **A resolution pings once too** ("resolved: …") at the same level as the
  problem, so a person knows they can stop looking.
- **Repeats are reminders, not floods:** one ping per problem, then a reminder
  at a fixed interval (for example every 30 minutes) while it lasts.
- `NOTIFY_WAITING` and `NOTIFY_ROUTINE` are settings with defaults (§0), so the
  routing can change without code changes.
- Health checks such as Gatus can alert through the same Pushover app or
  Telegram bot directly. They do not go through the notifier, so keep their
  volume low.

## 2. The notifier script

### 2.1 Interface

- **One entry point**, for example `scripts/notify.sh [--level alert|waiting|routine] <text>`,
  with `alert` as the default level. Every caller (scripts, hooks, health
  checks, CI) goes through it, so routing, limits and logging live in one
  place. In app code, a thin wrapper calls the same logic or the script.
- **Exit status:** non-zero only when an `alert` reached none of the channels
  it was routed to. A failed copy, a `waiting` or a `routine` message never
  changes the exit code.
- **Never block or break the caller.** Use short timeouts (`curl -m 15`),
  no retry loops, and call it as `notify … || true` from code that must
  continue.
- **Not configured is not an error.** Empty or missing credentials turn a
  channel off. With no channel configured, the notifier logs and exits 0, so
  a fresh clone and CI work without secrets.

### 2.2 Messages and secrets

- **Content-free messages:** `NOTIFY_PREFIX` plus the component and a state
  (`[shop worker] nightly import failed`). Never put personal data, customer
  content, secrets, tokens or full URLs with query strings in a message. The
  providers store messages on their servers, and phones show them on the lock
  screen.
- **Log every attempt** to a local events log (time, level, channel, outcome,
  HTTP code). Don't log the text if it could ever hold anything sensitive.
- **Pass credentials through config, not argv.** A token on the command line
  shows up in `ps` and shell history. Feed them to curl through its config on
  stdin (`curl -K -`); §3.2 and §4.2 show how for each provider.
- **Configure through env**, from the gitignored env file, with every
  variable listed in `.env.example` with an empty value.

### 2.3 Limit yourself first

A loop or a test against the real API is the usual way to burn a quota or
get a bot restricted, so the notifier limits itself per channel before
anything is sent:

- **Minimum gap** between two messages (for example 3 s), waited out under a
  file lock (`flock`), so parallel callers queue up.
- **Hourly and daily caps** (for example 40 and 200 for Telegram, 20 and 100
  for Pushover). Over a cap, the message is dropped and counted. The next
  message that gets through carries "(+N earlier pings not sent here)".
- **Dedupe:** the same text again within a window (for example 10 minutes) is
  skipped. Keep the window shorter than the reminder interval (§1).
- **Failed attempts count toward the caps,** so a broken channel can't turn
  into a retry loop.
- The limits are settings with these defaults. Never loosen them to make a
  test pass (§5).

### 2.4 State

- **Keep the state in one directory per channel**
  (`$NOTIFY_STATE_DIR/<channel>/`), with mode `700`: the lock, sent
  timestamps, the last-text hashes, the counters and the muted/disabled
  markers.
- **Muted** means "wait, then try again" (rate limited). **Disabled** means
  "the config is wrong, don't try until it changes". The marker holds a hash
  of the channel's credentials, and the channel turns itself back on when
  they change.

## 3. Pushover

Only when the repo uses Pushover.

### 3.1 Setup

- Create **one application per project** at pushover.net, with its own name
  and icon. The app token (`PUSHOVER_TOKEN`) identifies the sender, so a ping
  shows where it came from, and each app has its own monthly quota.
- The **user key** (`PUSHOVER_USER`) is the receiver. To reach several
  people, use a **delivery group** key instead of sending once per person.
- Both values are secrets: they live in the env file only, never in a commit,
  a log or a chat. The user creates the app and pastes the values into the
  env file; the agent never sees them (`setup-machine-skill.md` §1).
- Check the keys once with `POST /1/users/validate.json` before relying on
  them.

### 3.2 Sending

- Send `POST https://api.pushover.net/1/messages.json` with `token`, `user`,
  `message` and optionally `title`, `priority`, `url`, `url_title`.
- **Use `--form-string`, not `-F`, for every field.** With `-F`, a value
  starting with `@` or `<` makes curl read a local file and upload it. Put the
  `token` and `user` fields in the curl config on stdin
  (`form-string = "token=…"`), not on the command line.
- Limits: the message is at most 1024 characters, the title 250. Truncate
  before sending.
- Priority:
  - `-1` arrives quietly, `0` is normal, `1` bypasses quiet hours.
  - `2` (emergency) repeats until someone acknowledges it, and needs `retry`
    (at least 30 s) and `expire` (at most 10800 s). Use it only for real
    outages, and only if the user asked for it at setup.
- A daily summary or report is a normal-priority ping with a fixed title, not
  one ping per item.

### 3.3 Limits and failures

- The free monthly quota is per application (10,000 messages at the time of
  writing). The `X-Limit-App-Remaining` response header shows what is left.
  Warn once at 10 % remaining.
- `429`: the quota is used up. Mute the channel until the reset time in
  `X-Limit-App-Reset`.
- Any other `4xx`: the request or the config is wrong (bad token, bad user).
  Don't retry; mark the channel disabled (§2.4) and let the dead-man's switch
  surface it (§6).
- `5xx` or a network error: at most one retry, after 5 s or more. Never send
  more than two requests at once.

## 4. Telegram bots

Only when the repo uses a Telegram group.

### 4.1 Setup

- Create the bot with **BotFather** `/newbot`, one bot per project and
  purpose. The token (`TELEGRAM_BOT_TOKEN`) is a secret, so treat it like a
  password. If it leaks, `/revoke` it in BotFather straight away.
- **Leave privacy mode on** (the default) for a bot that only sends. The group
  then shows it as having "no access to messages". That is correct, not an
  error.
- **Add the bot to a private group as a member.** An invite link doesn't work
  for bots. Then run BotFather `/setjoingroups` → **Disable**, so nobody else
  can add the bot to their groups.
- **Get the chat id once** (`TELEGRAM_CHAT_ID`): send a message in the group,
  call `getUpdates` **once** and read `chat.id`. Group ids are negative;
  supergroup ids start with `-100`. `getUpdates` returns nothing while a
  webhook is set, and two processes polling it get `409 Conflict`.
- If the group uses topics (a forum), also store the topic's
  `message_thread_id` (`TELEGRAM_THREAD_ID`).

### 4.2 Sending

- Send `POST /bot<token>/sendMessage` with `chat_id` and `text`. The token
  is in the URL path, so pass the URL through curl's config on stdin:
  `printf 'url = "https://api.telegram.org/bot%s/sendMessage"\n' "$TELEGRAM_BOT_TOKEN" | curl -K - …`.
- Add `link_preview_options={"is_disabled":true}` so links don't render a
  preview.
- Set `disable_notification=true` on every message except the ones §1 sends
  "with sound", so the group stays quiet even for people who left its sound
  on.
- **Plain text by default.** `MarkdownV2` needs nearly every punctuation mark
  escaped, and one missed character makes the whole send fail with `400`. If
  you need formatting, use `parse_mode=HTML` and escape `<`, `>` and `&`.
- The text is at most 4096 characters. Truncate below that, leaving room for
  any suffix the notifier adds.

### 4.3 Rate limits

Telegram allows roughly one message per second per chat and about 20 per
minute in a group. A bot that keeps going over gets `429` and, if it goes on,
can be restricted. The limiter in §2.3 keeps well under that. On `429`, read
`parameters.retry_after`, mute the channel for at least that long (minimum
60 s), and say so once (§1, notifier trouble).

### 4.4 Errors that turn the channel off

- `400`, `401`, `403` or `404` means the config is dead (wrong token, bot
  kicked, chat gone). Mark the channel disabled (§2.4), with a hash of token +
  chat id, and say once what to fix (§1, notifier trouble).
- **Supergroup migration:** changing some group settings (history visible to
  new members, some admin changes) turns a group into a supergroup with a new
  `-100…` id. Sending to the old id fails with `400` and
  `parameters.migrate_to_chat_id`. Have the notifier name the new id in its
  notifier-trouble message and the events log. After the migration, check
  that the bot is still a member: `getChat` on the new id answers "chat not
  found" if it isn't. With `/setjoingroups` disabled, it may not be carried
  over. Allow groups for a moment, re-add the bot, then disable again.
- After fixing a config, remove the `disabled` marker if the hash still
  matches (for example the bot was re-added but the id didn't change), then
  send **one** test ping.

### 4.5 Bots that also read

Only when the bot receives commands or messages, not just sends:

- Use a **webhook** in production, with a `secret_token`. Reject every
  request whose `X-Telegram-Bot-Api-Secret-Token` header doesn't match.
  Long polling (`getUpdates`) is fine for local development, from one process
  only.
- **Allow-list chat ids and user ids.** Anyone can find a bot and message it.
  Ignore everyone else, without replying.
- A command that changes something needs the same approval and audit
  logging as any other write.
- Treat incoming text as untrusted input: never pass it to a shell, SQL or
  an LLM prompt as instructions.

## 5. Testing

- **Test against a fake.** Put a fake `curl` (or an HTTP stub) first in
  `PATH`, point each API base URL at an `.invalid` host, and give the state
  directory a temp path. The fake records requests and answers with a chosen
  HTTP code and body.
- Cover, at least, for each channel the repo has:
  - not configured → nothing sent, exit 0
  - configured → one request, with the credentials only in the curl config
  - each level routed as §1 says for this repo's channels
  - dedupe, minimum gap, hourly cap, daily cap, and the dropped-count suffix
  - `429` mutes; a config error disables, and the channel turns back on
    after the config changes
  - Telegram: migration names the new id
  - with both channels: a failing Pushover still sends the copy and returns
    non-zero; a failing Telegram never changes the exit code
- **Never write a test loop against the real API.** After setup or a config
  change, send one manual test ping per channel by hand.

## 6. A notifier that fails silently

A broken notifier looks exactly like a quiet day. Cover it from outside:

- A **dead-man's switch** (Healthchecks.io, Checkly heartbeat, Gatus external
  endpoint) expects a ping from the machine every N minutes and alerts on its
  own channel (email, its app) when the pings stop.
- The health check or a cron job sends the heartbeat, and **withholds it
  while the alert channel is muted or disabled.** The external check then
  raises the alarm the notifier can't.
- When the alert channel works again, send one "notifications restored"
  ping.
- With both channels, Telegram failures never withhold the heartbeat. They
  are reported on Pushover instead (§1).
- A repo with nothing running continuously (only CI or one-off scripts) can
  skip the heartbeat. Write "none yet" in §0 and say why.

## 7. Adding another channel

The notifier is built so a channel can be added later (ntfy, email, Slack, a
second Telegram group) without touching the callers. A new channel needs:

- Its env variables, empty in `.env.example`, with "empty = off".
- A send function: credentials through curl's config on stdin, a short
  timeout, truncation to the provider's limits, plain text.
- Its own limiter settings (§2.3) and state directory (§2.4).
- Its errors sorted into three kinds: rate limited (mute), config wrong
  (disable until the config changes), transient (at most one retry).
- A column in the §1 routing table, saying which levels it gets and whether
  it can be the alert channel.
- Fake-based tests (§5), and a section in this doc with its setup and limits.

## 8. Bootstrapping a new repo: checklist

Only the items for the channels this repo uses.

- [ ] **Project settings** (§0) filled in: channels, alert channel, levels the user chose, heartbeat
- [ ] One notifier script that every caller uses, with `--level`; exit code as in §2.1; exit 0 when nothing is configured
- [ ] Routing table (§1) cut down to this repo's channels and written in `docs/notifications.md`
- [ ] Credentials in the gitignored env file, listed in `.env.example` with empty values (`PUSHOVER_TOKEN`, `PUSHOVER_USER`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, as they apply), and sorted as "from the user" in the `setup-machine` skill
- [ ] Credentials passed to curl through stdin config, never argv; state directory mode `700`
- [ ] Local rate limiter per channel: gap, hourly and daily caps, dedupe, dropped counter; mute on `429`; disable on config errors until the config changes
- [ ] Pushover: one app per project; `--form-string` for every field; message and title truncated; no retry on `4xx`
- [ ] Telegram: privacy mode on, `/setjoingroups` disabled after adding, chat id read with a single `getUpdates`; migration id surfaced
- [ ] Messages content-free; a test or grep guard where callers build texts from data
- [ ] Tests against a fake `curl`; one manual real ping per channel after setup
- [ ] Dead-man's switch that is withheld while the alert channel fails, or "none yet" with the reason
