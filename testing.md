# Testing playbook

How a repository tests its code. Tools come from `pre-selected-tools.md`
(Vitest, Playwright, Testcontainers, pytest).

The short version: **tests exist to shorten the feedback loop. CI enforces
them. Prefer component and integration tests against real dependencies,
mock only what we don't own, and always test the unhappy paths.**

---

## 1. Test types

| Type        | Scope                                            | Real                          | Faked                                  |
| ----------- | ------------------------------------------------ | ----------------------------- | -------------------------------------- |
| Unit        | One function                                     | The function                  | Its collaborators (stubs)              |
| Component   | One module or service, as a black box            | Everything inside it          | What's outside it                      |
| Integration | Several components together                      | Database, cache, queues (Testcontainers) | Third-party APIs (WireMock or stubs) |
| E2E         | The whole system through the UI                  | Browser, backend, database    | Third-party APIs only                  |

- **When time is short, write component tests before unit tests.** A test
  through the public API proves the helpers work *and* that the contract
  held, and it survives refactoring of the internals.
- **Unit tests** are for logic worth pinning on its own: calculations,
  parsers, domain rules (e.g. money arithmetic).
- **Repositories are tested against a real database**, never a mocked one.
  Only a real database tests queries, constraints and migrations.
- **Don't test what we didn't write.** Postgres, Redis and Caddy have their
  own test suites.

## 2. Writing tests

- **One test, one behaviour.** When it fails, the name alone says what broke.
- **Names start with "should"**: `it("should return null when no provider is given")`.
  A failure then reads as the expectation that wasn't met.
- **Arrange, Act, Assert**, in that order, visibly separated.
- **Atomic.** Same result alone, in sequence or in random order. Setup and
  teardown go in `beforeEach` / `afterEach`, not in the test body.
- **Unhappy paths are mandatory.** For every happy path, test the bad inputs
  (`undefined`, empty, out of range), the failures of dependencies, and that
  the *right* error is thrown, with the right code (see error codes in
  `architectural-decisions.md`).
- **Readable over DRY.** Repeat setup across tests rather than hiding it
  three helpers deep. Shared builders and fixtures (a test kit with
  `makeUser()`, `stubRates()`) are fine. Clever abstractions that turn a
  test into an unreadable one-liner are not.
- **Stubs, mocks and spies are different things.** A stub returns canned
  data. A mock verifies an interaction. A spy watches a real call. Prefer
  stubs and asserting on outputs. Verify interactions only when the
  interaction *is* the behaviour (an email was sent).
- **UI selectors**: Playwright's `getByRole` / `getByLabel` first. Where
  there's no accessible handle, use `data-testid`. Never use CSS classes or
  DOM structure.

## 3. Real dependencies

- **Testcontainers** starts real Postgres, Redis, etc. on random ports for
  each run, and Ryuk cleans them up even if the suite crashes. Use one
  container per suite and reset state between tests (truncate or a
  transaction rollback). Use a fresh container per test only when isolation
  demands it.
- **Migrations run against the test database** before the suite, so every
  integration run also tests the migrations.
- **Third-party HTTP APIs** (payments, email, exchange rates) are never hit
  from tests. Adapters are tested against **WireMock** (Testcontainers
  module), or with hand-written stubs behind the port (see ports & adapters
  in `architectural-decisions.md`).
- **APIs with an OpenAPI spec** also run **Schemathesis** against a local
  instance in CI. It generates valid, malformed and hostile requests and
  checks that responses match the schema.

## 4. TDD

Write the failing test first, then just enough code to make it pass, then
refactor. With a coding agent this is the default: the test is the
specification the agent's code must satisfy (Superpowers'
`test-driven-development` skill enforces it).

## 5. CI: tests that aren't enforced don't exist

- Every PR runs `just lint` and `just test`. A red check blocks the merge.
  `main` is protected.
- `test` (unit + component, fast, no Docker) and `test:integration`
  (Testcontainers) are separate scripts, so the fast suite can run in watch
  mode.
- In monorepos, PRs may run only affected packages (Turborepo's
  `--filter=...[origin/main]`). Before a release, the full suite runs.
- E2E (Playwright) runs against the compose stack on PRs that touch the UI
  or API, and always before a release.

## 6. Who tests the tests

- **Coverage** finds blind spots. It isn't a quality target, so never gate
  a merge on a percentage.
- **Mutation testing** (Stryker for JS/TS, mutmut for Python) shows whether
  tests actually assert anything. Run it on critical modules (money, auth,
  permissions) periodically, not on every PR. Fix the surviving mutants that
  matter, and don't chase 100%.

## 7. After deploy

Synthetic checks (Gatus for uptime, or Checkly / Playwright on a schedule
for key flows such as login) run against production. A broken flow is found
through an alert, not a user complaint.

## 8. Bootstrapping a new repo: checklist

- [ ] Vitest (or pytest) configured, with `test` and `test:integration` scripts and `just` recipes
- [ ] Testcontainers setup for the database, running migrations first
- [ ] A test kit with builders and stubs for the domain
- [ ] Playwright against the compose stack, if there's a UI
- [ ] CI runs lint + tests on every PR, and `main` requires them to pass
- [ ] Schemathesis in CI, if the API publishes OpenAPI
- [ ] Uptime / synthetic checks on production after the first deploy
