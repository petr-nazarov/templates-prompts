# Architectural decisions

The architecture patterns every project follows.

---

## 1. Controller → Service → Repository

| Layer          | Owns                                                               | Never does                       |
| -------------- | ------------------------------------------------------------------ | -------------------------------- |
| **Controller** | The edge: routing, input validation, auth, dispatching to services | Business rules, DB access        |
| **Service**    | All business logic: calculations, orchestration, domain rules      | Know about HTTP or the DB driver |
| **Repository** | External access: databases, third-party APIs                       | Make business decisions          |

- One service can be exposed through several controller sets (HTTP, CLI, jobs).
- Layer boundaries are enforced by lint rules: no ORM imports outside
  `repositories/`, no controller imports a repository. The lint rules have
  their own tests.

## 2. Dependency injection

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

## 3. Strategy

- An abstract base defines the contract.
- One concrete class per variant.
- A registry maps a key to a strategy.
- The strategy is selected at runtime, in the constructor or a factory.
- Multi-dimensional variants use a composite key, e.g.
  `` `${emailType}:${channel}` ``.

## 4. Base repository, base service, base controller

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

## 5. Factory, Singleton, Adapter

- **Factory:** used when the concrete class is only known at runtime. It
  returns an instance and has no side effects.
- **Singleton:** one instance per runtime (e.g. a DB pool), held in a static
  field or owned by the DI container.
- **Adapter:** translates an external API's vocabulary into ours at the
  boundary.

## 6. Ports & adapters at external boundaries

- An abstract class is the port and DI token (e.g. `RateProvider`).
- A concrete adapter (e.g. `OpenExchangeRateProvider`) is bound in the module.

## 7. Schemas as the single contract

- Request and response shapes are Zod schemas in a shared `contracts` package.
- Controllers `parse()` input.
- A serializer interceptor validates every response against its schema.
- The same schemas generate OpenAPI and frontend types.

## 8. Monorepo with enforced package boundaries

- pnpm + Turborepo.
- `libs/{contracts,shared,db,auth,ui}` expose explicit `exports`, including a
  `./testing` subpath.
- Conventional commits via commitlint, husky hooks, and `just` / `mise` as the
  task runners.

## 9. Feature-module folders

- `modules/<feature>/{controllers,repositories}/`, with `*.service.ts` and
  `*.module.ts` alongside.
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
- Request context (user, tenant) is carried by AsyncLocalStorage.
- Services never pass `tx` or `tenantId` through their signatures.

## 12. Typed, validated env config

- A Zod `envSchema` is validated once at startup, so the app fails fast.
- The `.env` loader never overrides real environment variables and is skipped
  in production.

## 13. Declarative auth

- Decorators declare intent on the handler: `@Public`, `@RequiresTenant`,
  `@RequiresKind`, `@CurrentUser`, `@CurrentTenant`.
- A guard chain enforces it. The guard order is covered by a test.

## 14. Value objects & exact arithmetic

- Money is stored as integer minor units and parsed from strings, never
  through floats.
- Pure domain functions live in `libs/shared`.

## 15. Migrations

- drizzle-kit generates SQL migrations and snapshots.
- A dedicated migrate entrypoint applies them. The app never auto-syncs the
  schema.

## 16. Testing

- Repository and integration tests run against real infrastructure (a real
  Postgres, real temp files), not mocks.
- Service unit tests use hand-written stubs from a test kit.
- `test` and `test:integration` are separate scripts. Playwright covers e2e.

## 17. Offline-first outbox (clients)

- Writes are queued locally, merged, and synced with version-based conflict
  handling.

## 18. Typed i18n

- Translations are typed catalogues, so a missing key is a compile error.

## 19. ADRs and "why" comments

- Decisions are recorded as numbered ADRs and cited from code comments and
  lint messages.
- Comments explain why, not what.
