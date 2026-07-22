# Database Design Patterns — `@effect/sql`

Verified against `@effect/sql` 0.52.0 / `@effect/sql-pg` 0.53.0 (effect 3.22.0 line, July 2026). Canonical docs are the package READMEs (github.com/Effect-TS/effect, `v3` branch, `packages/sql`) — effect.website has no SQL section. Packages are 0.x: pin and upgrade with `effect`.

## Client basics

```ts
import { SqlClient } from "@effect/sql"

const sql = yield* SqlClient.SqlClient

// The template tag IS the query effect: Statement<A> extends Effect<ReadonlyArray<A>, SqlError>
const rows = yield* sql<{ id: number; name: string }>`SELECT id, name FROM people WHERE id = ${id}`
```

- Interpolations are **always bound parameters** — injection-safe by construction. `sql.unsafe(...)` is banned outside migrations.
- `sql("table_name")` escapes an *identifier*; `sql.in("id", ids)` for IN-lists (empty array handled); `sql.insert(record | records)` with `.returning("*")`; `sql.update(record)`; `sql.and([...])`/`sql.or([...])` for where-clauses; `.stream` on any statement gives a `Stream<A, SqlError>` for large results (constant memory).
- Dialect branching when needed: `sql.onDialectOrElse({ pg: ..., orElse: ... })`.
- Case mapping: `transformQueryNames: String.camelToSnake` + `transformResultNames: String.snakeToCamel` in the client config — DB stays snake_case, domain stays camelCase.

## Layer setup (Postgres)

```ts
import { PgClient } from "@effect/sql-pg"
import { Config } from "effect"

export const SqlLive = PgClient.layerConfig({
  host: Config.string("PGHOST"),
  database: Config.string("PGDATABASE"),
  username: Config.string("PGUSER"),
  password: Config.redacted("PGPASSWORD"),
  maxConnections: Config.integer("PG_POOL_MAX").pipe(Config.withDefault(10)),
  transformQueryNames: String.camelToSnake,
  transformResultNames: String.snakeToCamel,
})
// provides both SqlClient.SqlClient and PgClient.PgClient (adds .json, .listen/.notify)
```

Pool lifecycle is scoped to the layer (closed on shutdown). Pool knobs: `maxConnections`, `minConnections`, `connectionTTL`, `idleTimeout`, `connectTimeout`. No first-class statement timeout — set it in the connection options or wrap hot queries in `Effect.timeout` (interruption issues `pg_cancel_backend`). Health check: `sql\`SELECT 1\`` in the readiness probe. Every statement automatically gets a tracing span with `db.*` attributes.

## Transactions — boundary at the workflow, plumbing-free

`sql.withTransaction(effect)` semantics (verified in source):
- Rollback on **any non-success exit** — typed failure, defect, or interruption. Commit/rollback themselves are uninterruptible.
- **Nesting = savepoints**: inner `withTransaction` calls degrade to `SAVEPOINT`, so services can declare their own atomicity and still compose.
- The transaction connection propagates through the **fiber context**: every `sql\`...\`` anywhere inside the wrapped effect — including inside repositories and other services — automatically reuses it. No connection passing, no R-channel plumbing.

Therefore the rule: **repositories never call `withTransaction`; workflows own the boundary.**

```ts
const transferFunds = (from: AccountId, to: AccountId, amount: Cents) =>
  Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    return yield* sql.withTransaction(
      Effect.gen(function* () {
        yield* AccountRepo.debit(from, amount)    // both repos silently share
        yield* AccountRepo.credit(to, amount)     // the transaction connection
        yield* LedgerRepo.record({ from, to, amount })
      })
    )
  }).pipe(Effect.withSpan("Accounts.transferFunds"))
```

## Decoding rows — schemas at the DB boundary

Never trust row shapes; the "parse, don't validate" boundary applies to your own database too.

```ts
import { SqlSchema } from "@effect/sql"

// Request AND Result schemas; returns a plain function
const findByEmail = SqlSchema.findOne({
  Request: Email,
  Result: User,                        // decodes rows into the domain type
  execute: (email) => sql`SELECT * FROM users WHERE email = ${email}`,
})
// variants: findAll → ReadonlyArray<A>; findOne → Option<A>; single → A (fails NoSuchElementException); void
```

## Repositories: `Model.Class` + `makeRepository`

`Model.Class` extends `Schema.Class` with per-operation variants — one definition yields `select` / `insert` / `update` / `json` schemas, encoding which fields the DB generates vs the app supplies:

```ts
import { Model } from "@effect/sql"

export class User extends Model.Class<User>("User")({
  id: Model.Generated(UserId),               // in select/update, NOT in insert (DB generates)
  email: Email,
  passwordHash: Model.Sensitive(Schema.String),   // excluded from json variants — cannot leak into API responses
  createdAt: Model.DateTimeInsertFromDate,   // stamped on insert
  updatedAt: Model.DateTimeUpdateFromDate,   // stamped on insert + update
}) {}

export class UserRepo extends Effect.Service<UserRepo>()("app/UserRepo", {
  effect: Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    const base = yield* Model.makeRepository(User, {
      tableName: "users", spanPrefix: "UserRepo", idColumn: "id",
    }) // insert / update / findById (Option) / delete — spanned, RETURNING-based
    const findByEmail = SqlSchema.findOne({
      Request: Email, Result: User,
      execute: (email) => sql`SELECT * FROM users WHERE email = ${email}`,
    })
    return { ...base, findByEmail } as const
  }),
}) {}
```

Note `makeRepository` treats parse/SQL errors as defects (`orDie`) — it assumes schema and DDL agree; migrations are what keep that true. Wrap in domain-error translation where callers should handle failures.

## Batching — kill N+1 with `SqlResolver`

```ts
import { SqlResolver } from "@effect/sql"

const GetById = yield* SqlResolver.findById("GetUserById", {
  Id: UserId, Result: User,
  ResultId: (row) => row.id,
  execute: (ids) => sql`SELECT * FROM users WHERE ${sql.in("id", ids)}`,  // ONE query for N concurrent lookups
})
const getById = (id: UserId) => Effect.withRequestCaching("on")(GetById.execute(id))
// also: SqlResolver.ordered (batched INSERT..RETURNING, order-matched), .grouped (1→many), .void
```

Concurrent `getById` calls across a request are automatically batched into one query and deduped.

## Migrations

Forward-only, numbered files; runs in a transaction with an exclusive lock (concurrent deploys: loser no-ops).

```ts
// src/migrations/0001_create_users.ts — default-export an Effect requiring SqlClient
export default Effect.flatMap(SqlClient.SqlClient, (sql) => sql`
  CREATE TABLE users (id uuid PRIMARY KEY, email varchar(255) NOT NULL UNIQUE, ...)`)

// wiring
const MigratorLive = PgMigrator.layer({
  loader: PgMigrator.fromFileSystem(fileURLToPath(new URL("migrations", import.meta.url))),
}).pipe(Layer.provide(SqlLive), Layer.provide(NodeContext.layer))
```

## Testing DB code

Two tiers, both by layer swap (the repo code never changes):

1. **Fast tier — in-memory SQLite**: `SqliteClient.layer({ filename: ":memory:" })` from `@effect/sql-sqlite-node` provides the same `SqlClient.SqlClient` tag. Works when SQL is dialect-neutral (or branched with `onDialectOrElse`).
2. **Real tier — testcontainers** (the pattern Effect's own pg tests use):

```ts
export class PgContainer extends Effect.Service<PgContainer>()("test/PgContainer", {
  scoped: Effect.acquireRelease(
    Effect.tryPromise({ try: () => new PostgreSqlContainer("postgres:alpine").start(),
                        catch: (cause) => new ContainerError({ cause }) }),
    (c) => Effect.promise(() => c.stop())
  ),
}) {
  static ClientLive = Layer.unwrapEffect(
    Effect.gen(function* () {
      const c = yield* PgContainer
      return PgClient.layer({ url: Redacted.make(c.getConnectionUri()) })
    })
  ).pipe(Layer.provide(this.Default))
}
```

Run migrations in the test layer so tests exercise the real DDL.

Query builders, if wanted: `@effect/sql-drizzle` (joins `withTransaction` automatically); `@effect/sql-kysely` (pin Kysely — compat is version-sensitive). Raw `sql` + `SqlSchema` is the default; add a builder only for genuinely dynamic query construction.

## Checklist

- [ ] All queries via the `sql` tag (bound params); `sql.unsafe` only in migrations
- [ ] Every row decoded by Schema (`SqlSchema.*` / `Model`) — no `as` casts on rows
- [ ] Repositories: `Effect.Service` wrapping `SqlClient`; infra errors translated; no `withTransaction` inside
- [ ] Transaction boundaries declared in workflows; multi-write invariants covered by a transaction test (failure → nothing persisted)
- [ ] Hot lookups batched with `SqlResolver` where N+1 is possible
- [ ] Migrations numbered, forward-only, run by `Migrator` layer in dev/test/prod alike
- [ ] Sensitive columns marked `Model.Sensitive`; case transforms configured once on the client
- [ ] Large result sets streamed (`.stream`), not loaded whole
