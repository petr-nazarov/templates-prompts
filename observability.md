# Observability playbook

How a repository logs, traces requests, and is monitored in production.

The short version: **every service logs twice: a readable, coloured console
on stdout, and rotating JSON Lines files in ECS format. Every line carries
the request ID. No secrets or personal data in logs. Logs are collected off
the machine, and metrics sit next to them on one dashboard.**

---

## 1. Application logs

### 1.1 Two outputs, always

Every service logs to **both** outputs, in every environment (development,
CI, staging, production):

| Output  | Where                  | Format                                  | For                  |
| ------- | ---------------------- | --------------------------------------- | -------------------- |
| Console | stdout                 | Coloured, one line per event, readable  | People watching live |
| File    | `$LOG_DIR/app.jsonl`   | JSON Lines, ECS fields, one object per line | Machines: search, shipping, retention |

- **One logger, two handlers.** Use the language's handler/formatter (or
  transport) mechanism: a single logger with a console handler and a file
  handler, each with its own formatter and level. Code calls the logger and
  never writes to stdout or to a file directly (`console.log`, `print`).
- **Never JSON on the console, never colours in the file.** The console is
  for reading, the file is for parsing.
- **Errors get their own file too:** `$LOG_DIR/error.jsonl` receives `warn`
  and above, so a failure is found without searching everything.
- **Files rotate.** Rotate by size (`LOG_MAX_SIZE`, default `10MB`) and keep a
  fixed number of files (`LOG_MAX_FILES`, default `10`). A log that can fill
  the disk is a bug.
- **In containers, `LOG_DIR` is a bind-mounted host directory**
  (`./logs:/var/log/app`), never the container's own filesystem. Files then
  survive a recreate, and the collector reads them from the host. The
  console output still goes to stdout, which Dozzle and `docker logs` show.
- **Libraries:**
  - Node: pino with two transport targets, `pino-pretty` to stdout and
    `pino-roll` to the file, and `@elastic/ecs-pino-format` for the ECS
    fields (`nestjs-pino` in Nest).
  - Python: stdlib `logging` with a `StreamHandler` (a coloured formatter)
    and a `RotatingFileHandler` using `ecs-logging`'s `StdlibFormatter`, both
    behind a `QueueHandler` (structlog can render both the same way).

### 1.2 Configuration

All settings come from the environment and are validated at startup with the
rest of the config (see typed env config in `architectural-decisions.md`).

| Variable                      | Default          | Meaning                                         |
| ----------------------------- | ---------------- | ----------------------------------------------- |
| `LOG_LEVEL`                   | `info`           | Level for both outputs                          |
| `LOG_LEVEL_CONSOLE`           | `LOG_LEVEL`      | Overrides the console level (e.g. `debug` locally) |
| `LOG_LEVEL_FILE`              | `LOG_LEVEL`      | Overrides the file level                        |
| `LOG_DIR`                     | `./logs`         | Directory for the JSON Lines files (gitignored) |
| `LOG_MAX_SIZE`                | `10MB`           | Rotate a file at this size                      |
| `LOG_MAX_FILES`               | `10`             | Rotated files kept per log                      |
| `SERVICE_NAME`                | package name     | `service.name` on every line                    |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | unset            | Tracing is off unless this is set               |

### 1.3 What a log line contains

The file output follows the [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html)
(ECS), so any collector (Loki, Elastic, Datadog) can parse it without custom
mapping.

- **Required on every line:** `@timestamp` (ISO 8601, UTC), `log.level`,
  `message`, `service.name`, `service.version` (the SemVer release, or the
  commit SHA for untagged builds), `service.environment`, `process.pid`,
  `host.name`.
- **From the request context, when known:** `http.request.id` (the request
  ID, §2), `trace.id` and `span.id` (OpenTelemetry), `user.id`, and the
  tenant as `organization.id`.
- **Per event:** `event.action` for business events (`order.placed`),
  `event.duration` (nanoseconds) for timings, the `http.*` / `url.*` fields
  for requests, and `error.type`, `error.message` and `error.stack_trace` for
  errors.
- The console shows the same event in one line: time, level, service,
  request ID, message, then the fields.

```text
10:42:07.113 INFO  api  req=5f0c…  order placed  orderId=o_812 amount=4200
```

```json
{"@timestamp":"2026-09-25T10:42:07.113Z","log.level":"info","message":"order placed","service.name":"api","service.version":"1.4.2","service.environment":"production","process.pid":1,"host.name":"web-1","http.request.id":"5f0c9a1e-…","trace.id":"4bf92f35…","user.id":"u_19","event.action":"order.placed","order":{"id":"o_812","amount":4200}}
```

### 1.4 Writing log calls

- **Log with fields, not interpolated strings:**
  `log.info({ orderId, userId, amount }, "order placed")`, not
  `` log.info(`Order ${orderId} placed by ${userId}`) ``. The message is a
  constant you can search for, and the context goes in fields you can
  filter on.
- **Messages are short, lower-case English, with no values in them** and no
  line breaks. A stack trace goes in `error.stack_trace`, not in the message.
- **Levels:**
  - `debug`: step-by-step detail and timings. Off in production unless
    turned on for an investigation.
  - `info`: one line per meaningful business event (a sign-up, an order, a
    job finished), plus one access line per request. Not one line per
    function call.
  - `warn`: something went wrong and was handled (a retry that succeeded, a
    fallback used, bad input rejected).
  - `error`: something failed and someone needs to look at it.
- **Log an error once, where it's handled**, with the error object attached
  so the stack trace is kept. Don't log and rethrow: that logs the same
  failure at every layer.
- **Hiding an error from the client never means hiding it from the logs.**
  The client gets a generic message and the error code; the log gets the
  real error.
- **Log decisions and outcomes, not noise.** A log line should help answer
  "what happened to this request?" If nobody would ever search for it, it's
  `debug` or nothing.

### 1.5 Never log

- Passwords, tokens, API keys, session IDs, `Authorization` and `Cookie`
  headers, or any secret.
- Personal data beyond IDs: names, emails, phone numbers, addresses, card or
  bank numbers, government IDs. Mask them where they must appear
  (`+972 •••• 1234`).
- Whole request or response bodies. Log the fields you need.

Enforce it in the logger, not at each call site:

- **Redaction is configured in the logger** (pino `redact` paths, a
  `logging.Filter` in Python) and matches keys by name (`password`, `token`,
  `secret`, `authorization`, `cookie`, `apiKey`…) at any depth, so it covers
  every call.
- **A `redact()` helper** is used whenever an object is serialized into a log
  field, so nested secrets are masked too.
- A test logs an object full of secrets and asserts that none of them reach
  either output.

### 1.6 Performance and reliability

- **Logging never blocks request handling.** Writes are asynchronous and
  buffered (pino transports run in a worker thread; Python uses a
  `QueueHandler` with a `QueueListener`).
- **Flush on shutdown.** On `SIGTERM` the app flushes the logger before it
  exits, so the last lines, usually the interesting ones, aren't lost.
- **A logging failure never crashes the app.** If the file can't be written
  (disk full, missing permissions), the console keeps working and one warning
  says so.
- Under extreme load, dropping log lines is preferable to stalling the app.

## 2. Request IDs

- The frontend generates a UUID per user action and sends it as
  `X-Request-ID`.
- Backend middleware reads the header, or generates an ID if it's missing,
  and echoes it on the response.
- The field is `http.request.id` in the JSON file (§1.3) and `req` on the
  console. When OpenTelemetry is on, `trace.id` and `span.id` sit next to it;
  the request ID stays, because it also covers the frontend and jobs that
  have no trace.
- The ID is stored in the request context (AsyncLocalStorage in Node,
  `contextvars` in Python). This is the same context that carries the user
  and tenant (see Unit of Work + request context in
  `architectural-decisions.md`).
- The logger reads the ID from the context, so every line has it without
  passing it through function signatures.
- Outgoing HTTP calls and queued jobs forward the ID, so one search shows
  a whole action across frontend, API and workers.

## 3. Access logs

- The reverse proxy (Caddy) writes JSON access logs, collected as their own
  stream, separate from application logs.
- **Retention:**
  - Application logs: about 3 months.
  - Access logs: about 2 years, or whatever the client's compliance
    requires.
- **Compliance projects** (audits, SLAs, regulated data) ship access and
  audit logs to WORM storage: S3 Object Lock in compliance mode, with a
  retention period. Logs then can't be changed or deleted, even by us.

## 4. Collection and dashboards

Pick the smallest setup that answers the questions:

| Setup | Tools | When |
| ----- | ----- | ---- |
| Small (one server, personal or early project) | **Dozzle** (container logs), **Beszel** (host and container metrics) | Default. One compose file, near-zero config. |
| Full | **Grafana + Loki + Prometheus** (Grafana Alloy or Vector as the collector), Loki storing to S3 | Several services or servers, alerting, clients who need retention, or correlating logs with metrics |

- Logs leave the machine. If a server dies, its logs must still exist. The
  collector (Alloy or Vector) tails the JSON Lines files in each service's
  `LOG_DIR` on the host and ships them. Dozzle shows the console output.
- Put metrics and logs on the same Grafana dashboard: select a spike on a
  metric, and the logs panel shows what the app was doing then.
- Business metrics (sign-ups, orders) can be Prometheus metrics or SQL
  panels. Keep them next to the infrastructure metrics.

## 5. Health and uptime

- `/health` returns `200` with `{ status, version }` and needs no auth. The
  Docker `HEALTHCHECK` and deploy `--wait` use it (see `docker.md`).
- If readiness depends on dependencies, add a separate `/health/ready` that
  checks the database and cache. `/health` itself stays cheap, so a slow
  database doesn't make the orchestrator restart healthy containers.
- **Gatus** (lightweight, YAML-configured) checks every public endpoint
  from outside and alerts through ntfy / email / Slack. Synthetic flow
  checks are described in `testing.md` §7.

## 6. Bootstrapping a new repo: checklist

- [ ] Logger configured with two handlers: coloured console on stdout, rotating ECS JSON Lines in `LOG_DIR` (`app.jsonl`, `error.jsonl`)
- [ ] `LOG_*` variables in the env schema and `.env.example`; `logs/` gitignored
- [ ] Redaction paths and the `redact()` helper, with a test that no secret reaches either output
- [ ] Logger flushed on `SIGTERM`
- [ ] In production compose, `LOG_DIR` bind-mounted from the host
- [ ] Request-ID middleware, stored in the request context and included in every log line
- [ ] `version` injected at build time (build arg → env) and logged
- [ ] `/health` returning the version; `/health/ready` if needed
- [ ] Caddy access logs in JSON
- [ ] Dozzle + Beszel on the server, or the Grafana stack if the project needs it
- [ ] Gatus check for each public endpoint
