# Architectural decisions

The architecture patterns projects follow. Three of them are **chosen per
project at setup** (§0), because a small project doesn't need them. The rest
apply to every project with application code. A repo that departs from a
pattern records why in an ADR (§20).

The short version: **ask first whether the project gets the layered
architecture, dependency injection and base classes. Always: contracts are
shared Zod schemas, input is validated at the edge, errors are codes, request
context travels implicitly, and money is never a float.**

---

## 0. Choices made at setup

The agent setting up a repo asks the user these questions (`new-repo.md` §2)
and never assumes the answers. The answers go in `AGENTS.md` under
Conventions. In the copied `docs/architecture.md`, a section answered "no"
keeps its heading (so § references still work) and is cut down to its
"Without…" rule, so the repo's docs only describe what it uses.

| Choice | Section | Usually yes | Usually no | With "no" |
| ------ | ------- | ----------- | ---------- | --------- |
| **Layered architecture** (controller → service → repository) | §1 | APIs with several domains, business rules worth testing on their own, more than one entry point (HTTP, CLI, jobs) | Small services, scripts, a static site's API, one-domain CRUD | Route handlers call a small data-access module directly (§1, "Without layers") |
| **Dependency injection** | §2 | Layered apps, anything with swappable providers (payments, email, rates), NestJS | Small apps and scripts | Plain modules that import what they use (§2, "Without DI") |
| **Base classes** (generic base repository, service, controller) | §4 | Only asked if layered is yes. Many entities with the same CRUD | Few entities, or entities that each behave differently | Each class is written out in full (§4, "Without base classes") |

- Choosing **NestJS** answers the first two: it is layered and built on DI.
- The choices can be revisited later. Adding one to a running project is a
  refactor, recorded in an ADR (§20).

## 1. Controller → Service → Repository

*Chosen at setup (§0). Also known as a layered or three-layer architecture;
the bottom layer is the Repository pattern.*

| Layer          | Owns                                                               | Never does                       |
| -------------- | ------------------------------------------------------------------ | -------------------------------- |
| **Controller** | The edge: routing, input validation, auth, dispatching to services | Business rules, DB access        |
| **Service**    | All business logic: calculations, orchestration, domain rules      | Know about HTTP or the DB driver |
| **Repository** | External access: databases, third-party APIs                       | Make business decisions          |

- One service can be exposed through several controller sets (HTTP, CLI, jobs).
- Layer boundaries are enforced by lint rules: no ORM imports outside
  `repositories/`, no controller imports a repository. The lint rules have
  their own tests.

**Without layers** (when the answer is "no"):

- Route handlers validate input (§7), do the work, and return the response.
- All database and third-party access still goes through one module per
  concern (`db/orders.ts`, `clients/stripe.ts`), never inline SQL or `fetch`
  in a handler. That keeps a later move to layers a small refactor.
- When a handler's logic grows past what one screen can hold, or a second
  entry point needs the same logic, that's the signal to add a service layer.

## 2. Dependency injection

*Chosen at setup (§0). The container is also called an IoC container.*

```ts
abstract class CatsRepository {
  abstract findMany(filter: CatFilter): Promise<Cat[]>;
}

class CatsService {
  constructor(private readonly repository: CatsRepository) {}
}
```

- Dependencies arrive through the constructor, typed as an abstraction.
- Inject constructed instances, not classes.
- Inject stateless collaborators (services, repositories, clients), never
  domain data.
- Constructors only wire things up. They hold no logic.
- An abstract class is the DI token.
- A container does the wiring: NestJS modules in apps, `tsyringe` without a
  framework.
- Tests register stubs in the container.
- `tsyringe` won't load without `import "reflect-metadata"` at the entry
  point, even when every registration is a factory and no decorator is used.

**Without DI** (when the answer is "no"):

- Modules import what they use. Anything that differs by environment (a
  provider, a client) is picked once, in a factory that reads the config
  (§6).
- Functions that need a collaborator a test must replace take it as an
  argument with a default, instead of importing it.
- Tests fake the outside world at its boundary (Testcontainers, WireMock),
  not by patching imports.

## 3. Strategy

- An abstract base defines the contract.
- One concrete class per variant.
- A registry maps a key to a strategy.
- The strategy is selected at runtime, in the constructor or a factory.
- Multi-dimensional variants use a composite key, e.g.
  `` `${emailType}:${channel}` ``.

## 4. Base repository, base service, base controller

*Chosen at setup (§0), only with the layered architecture. Also known as the
Layer Supertype pattern, or a generic repository.*

```ts
abstract class BaseRepository<TInsert, TRead> {
  constructor(protected readonly db: Db, protected readonly table: Table) {}
  createOne(data: TInsert): Promise<TRead> { /* ... */ }
  findById(id: string): Promise<TRead | null> { /* ... */ }
  // updateOne, deleteOne, findMany ...
}
```

- CRUD logic lives once, in generic base classes.
- Each entity adds only its schemas, thin subclasses that call `super(...)`,
  and overrides where it differs.
- Cross-cutting rules go into an intermediate base, e.g.
  `TenantRepository extends BaseRepository` scopes every query via
  `tenantScope()`.

**Without base classes** (when the answer is "no"):

- Each repository, service and controller is written out in full.
- Cross-cutting rules such as tenant scoping live in one shared helper
  (`tenantScope(query)`) that every query calls, and a test checks that no
  query skips it.

## 5. Factory, Singleton, Adapter

- **Factory:** used when the concrete class is only known at runtime. It
  returns an instance and has no side effects.
- **Singleton:** one instance per runtime (e.g. a DB pool), held in a static
  field or owned by the DI container.
- **Adapter:** translates an external API's vocabulary into ours at the
  boundary.

## 6. Ports & adapters at external boundaries

- An abstract class (or an interface) is the port (e.g. `RateProvider`).
- A concrete adapter (e.g. `OpenExchangeRateProvider`) implements it.
- With DI, the port is the DI token and the adapter is bound in the module.
  Without DI, one factory picks the adapter from config, and callers import
  the factory's result.

## 7. Schemas as the single contract

- Request and response shapes are Zod schemas in a shared `contracts` package.
- Controllers `parse()` input.
- A serializer interceptor validates every response against its schema.
- The same schemas generate OpenAPI and frontend types.

## 8. Monorepo with enforced package boundaries

- pnpm + Turborepo.
- `libs/{contracts,shared,db,auth,ui}` expose explicit `exports`, including a
  `./testing` subpath.
- Conventional Commits enforced by commitlint in a husky `commit-msg` hook
  (see `pre-selected-tools.md` §2), Semantic Versioning for releases, and
  `just` / `mise` as the task runners.

## 9. Feature-module folders

- Layered: `modules/<feature>/{controllers,repositories}/`, with
  `*.service.ts` and `*.module.ts` alongside.
- Without layers: `modules/<feature>/` holds its routes and its data-access
  module.
- Shared plumbing lives in `common/`.

## 10. Error codes

- An append-only enum of error codes.
- Small factories, such as `conflict(code, params)`, `badRequest(...)` and
  `notFound(...)`, map them to HTTP.
- The response body is `{ code, message, params }`, and the client translates
  it.
- Domain `Error` subclasses are kept to a minimum.

## 11. Unit of Work + request context

- `UnitOfWork.run()` opens a transaction, or joins the one already open.
- Request context (user, tenant, request ID) is carried by AsyncLocalStorage
  (`contextvars` in Python). The logger reads the request ID from it (see
  `observability.md`).
- Services never pass `tx` or `tenantId` through their signatures.

## 12. Typed, validated env config

- A Zod `envSchema` is validated once at startup, so the app fails fast.
- The `.env` loader never overrides real environment variables and is skipped
  in production.

## 13. Declarative auth

- Decorators declare intent on the handler: `@Public`, `@RequiresTenant`,
  `@RequiresKind`, `@CurrentUser`, `@CurrentTenant`.
- A guard chain enforces it. The guard order is covered by a test.

## 14. Auth at the edge and in the app

- Internal tools with no login of their own sit behind the reverse proxy's
  forward auth (SSO at the proxy). The tool itself stays unaware of auth.
- Application roles live in the application. Proxy auth only answers "in or
  out".
- With an external IdP (OIDC): the backend verifies JWT signatures locally
  against the IdP's cached JWKS, and never calls the IdP per request.
- Access tokens are short-lived (≤ 15 min) and refreshed.
- JWT payloads are readable by anyone: no secrets, no credentials, no large
  permission lists.
- The app's own frontend never collects credentials on behalf of an IdP.
  Use the IdP's hosted page or its embeddable flow component.

## 15. Value objects & exact arithmetic

- Money is stored as integer minor units and parsed from strings, never
  through floats.
- Pure domain functions live in `libs/shared`.

## 16. Migrations

- drizzle-kit generates SQL migrations and snapshots.
- A dedicated migrate entrypoint applies them. The app never auto-syncs the
  schema.

## 17. Testing

- Repository and integration tests run against real infrastructure (a real
  Postgres, real temp files), not mocks.
- Service unit tests use hand-written stubs from a test kit, registered in
  the container with DI or passed in as arguments without it.
- `test` and `test:integration` are separate scripts. Playwright covers e2e.
- The full rules are in `testing.md`.

## 18. Offline-first outbox (clients)

- Writes are queued locally, merged, and synced with version-based conflict
  handling.

## 19. Typed i18n

- Translations are typed catalogues, so a missing key is a compile error.

## 20. ADRs and "why" comments

- Decisions are recorded as numbered ADRs in `docs/adr/NNNN-<slug>.md`
  (context, decision, consequences), and cited from code comments and lint
  messages (`// ADR-0008`).
- An ADR is never rewritten after it's accepted. A change of mind is a new
  ADR that supersedes it, or a dated amendment at the end.
- Comments explain why, not what.

---

## 21. Bootstrapping a new repo: checklist

- [ ] The three §0 choices asked, answered by the user, and recorded in `AGENTS.md`; each "no" section cut down to its "Without…" rule
- [ ] Folder layout per §8 and §9 (`libs/*` with explicit `exports`, `modules/<feature>/`)
- [ ] Layered: lint rules for the layer boundaries (§1), with their own tests. Not layered: one data-access module per concern
- [ ] DI (if chosen): container wired (§2), NestJS modules or `tsyringe` without a framework
- [ ] Base classes (if chosen): `BaseRepository` and friends (§4), with the tenant-scoped intermediate base if the app is multi-tenant
- [ ] `contracts` package with Zod schemas, and the response serializer (§7)
- [ ] Error-code enum and the error factories (§10)
- [ ] Request context and `UnitOfWork` (§11), read by the logger
- [ ] Env schema validated at startup (§12)
- [ ] Migrations generated by drizzle-kit, with a separate migrate entrypoint (§16)
- [ ] `docs/adr/` created; the first real decision becomes `0001-<slug>.md`
