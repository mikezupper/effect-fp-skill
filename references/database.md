# Database Design Patterns — `effect/sql`

Verified against effect 4.0.1 / `@effect/sql-pg` 4.0.1 / `@effect/sql-sqlite-node` 4.0.1 (October 2026). In v4 the SQL core moved into `effect` itself: `effect/sql` (`SqlClient`, `SqlSchema`, `SqlModel`, `SqlResolver`, `Migrator`, `SqlError`, `Statement`) and `effect/schema` (`Model`). `@effect/sql` is v3-only — never install it with effect 4. Driver packages (`@effect/sql-pg`, `@effect/sql-sqlite-node`, `@effect/sql-mysql2`, ...) are versioned in lockstep with `effect`; upgrade them together. There is no v4 `@effect/sql-drizzle` / `@effect/sql-kysely` (both still 0.x on effect 3). All `effect/sql` modules are tagged `@stability unstable`. Canonical docs ship in the package: `node_modules/effect/ai-docs/src/40_sql` plus the JSDoc in the `.d.ts` files; source is github.com/Effect-TS/effect, `main` branch — effect.website has no SQL section.

## Client basics

```ts
import { Effect } from "effect"
import { SqlClient } from "effect/sql"

const program = Effect.gen(function* () {
  const sql = yield* SqlClient.SqlClient

  // The template tag IS the query effect: Statement<A> extends Effect<ReadonlyArray<A>, SqlError>
  const rows = yield* sql<{ id: number; name: string }>`SELECT id, name FROM people WHERE id = ${id}`
  return rows
})
```

- Interpolations are **always bound parameters** — injection-safe by construction. `sql.unsafe(...)` is banned outside migrations.
- `sql("table_name")` escapes an *identifier*; `sql.in("id", ids)` for IN-lists (empty array handled); `sql.insert(record | records)` with `.returning("*")`; `sql.update(record, omit?)`; `sql.and([...])`/`sql.or([...])` for where-clauses; `.stream` on any statement gives a `Stream<A, SqlError>` for large results (constant memory).
- Dialect branching when needed: `sql.onDialectOrElse({ pg: () => ..., orElse: () => ... })`.
- Case mapping: `transformQueryNames` (camel → snake) + `transformResultNames` (snake → camel) in the client config — DB stays snake_case, domain stays camelCase.
- Errors: every statement fails with `SqlError` (`import { SqlError } from "effect/sql"`), which carries a classified `reason` (constraint violation, connection, ...) — translate the ones callers can act on into domain errors at the repository.

## Layer setup (Postgres)

```ts
import { PgClient } from "@effect/sql-pg"
import { Config, String as Str } from "effect"

export const SqlLive = PgClient.layerConfig({
  host: Config.String("PGHOST"),
  database: Config.String("PGDATABASE"),
  username: Config.String("PGUSER"),
  password: Config.Redacted("PGPASSWORD"),
  maxConnections: Config.Int("PG_POOL_MAX").pipe(Config.withDefault(10)),
  transformQueryNames: Config.succeed(Str.camelToSnake),
  transformResultNames: Config.succeed(Str.snakeToCamel),
})
// provides both SqlClient.SqlClient and PgClient.PgClient (adds .json, .listen/.notify)
```

`layerConfig` takes a `Config` for **every** field — wrap non-config values (the transform functions) in `Config.succeed`; `PgClient.layer({...})` takes plain values. Or pass a single `url: Config.Redacted("DATABASE_URL")`. Pool lifecycle is scoped to the layer (closed on shutdown). Pool knobs: `maxConnections`, `minConnections`, `connectionTTL`, `idleTimeout`, `connectTimeout`. Statement timeout: `startupParameters: { statement_timeout: "5s" }` in the config, or wrap hot queries in `Effect.timeout` (interruption sends a Postgres `CancelRequest` for the running query). Health check: `sql\`SELECT 1\`` in the readiness probe. Every statement automatically gets a tracing span with `db.*` attributes (`db.query.text`, `db.operation.name`).

## Transactions — boundary at the workflow, plumbing-free

`sql.withTransaction(effect)` semantics (verified in source):
- Rollback on **any non-success exit** — typed failure, defect, or interruption. Commit/rollback themselves are uninterruptible.
- **Nesting = savepoints**: inner `withTransaction` calls degrade to `SAVEPOINT`, so services can declare their own atomicity and still compose (a failed inner block rolls back to its savepoint; the outer transaction can continue).
- The transaction connection propagates through the **fiber context**: every `sql\`...\`` anywhere inside the wrapped effect — including inside repositories and other services — automatically reuses it. No connection passing, no R-channel plumbing.
- It widens the error channel with `SqlError` (BEGIN/COMMIT can fail). With an `Effect.fn.Return<A, E>` annotation, handle it right there (`Effect.catchTag("SqlError", Effect.die)` or translate) so it doesn't leak into `E`.

Therefore the rule: **repositories never call `withTransaction`; workflows own the boundary.**

```ts
import { Effect } from "effect"
import { SqlClient } from "effect/sql"

export const transferFunds = Effect.fn("Accounts.transferFunds")(
  function* (from: AccountId, to: AccountId, amount: Cents) {
    const sql = yield* SqlClient.SqlClient
    const accounts = yield* AccountRepo
    const ledger = yield* LedgerRepo
    yield* sql.withTransaction(
      Effect.gen(function* () {
        yield* accounts.debit(from, amount)    // all three repos silently share
        yield* accounts.credit(to, amount)     // the transaction connection
        yield* ledger.record({ from, to, amount })
      })
    )
  }
)
```

## Decoding rows — schemas at the DB boundary

Never trust row shapes; the "parse, don't validate" boundary applies to your own database too.

```ts
import { Effect } from "effect"
import { SqlClient, SqlSchema } from "effect/sql"

export const program = Effect.gen(function* () {
  const sql = yield* SqlClient.SqlClient

  // Request AND Result schemas; returns a plain function
  const findByEmail = SqlSchema.findOneOption({
    Request: Email,
    Result: User,                        // decodes rows into the domain type
    execute: (email) => sql`SELECT * FROM users WHERE email = ${email}`,
  })
  // variants: findAll → Array<A>; findOneOption → Option<A>; findOne → A (fails NoSuchElementError);
  //           findNonEmpty → NonEmptyArray<A> (fails NoSuchElementError); void → discards rows
  return findByEmail
})
```

v3 → v4 trap: `SqlSchema.findOne` used to return `Option`; it now **fails** with `NoSuchElementError`. Use `findOneOption` for the old behavior; `single` is gone. Decode failures surface as `Schema.SchemaError`.

Column types: decode timestamps with the codec that matches what the driver returns — `Schema.DateTimeUtcFromDate` for pg `timestamptz` (a JS `Date`), `Schema.DateTimeUtcFromString` for SQLite text; plain `Schema.DateTimeUtc` expects an already-decoded `DateTime.Utc` and will reject rows.

## Repositories: `Model.Class` + `SqlModel.makeRepository`

`Model.Class` (from `effect/schema`) extends `Schema.Class` with per-operation variants — one definition yields `select` / `insert` / `update` / `json` / `jsonCreate` / `jsonUpdate` schemas, encoding which fields the DB or app generates vs the client supplies:

```ts
import { Context, Effect, Layer, Option, Schema } from "effect"
import { Model } from "effect/schema"
import { SqlClient, SqlModel, SqlSchema } from "effect/sql"

export class User extends Model.Class<User>("User")({
  id: Model.UuidV4Insert(UserId),                   // app-generated on insert; never client-settable
  email: Email,
  passwordHash: Model.Sensitive(Schema.String),     // excluded from json variants — cannot leak into API responses
  createdAt: Model.DateTimeInsertFromDate,          // stamped on insert
  updatedAt: Model.DateTimeUpdateFromDate,          // stamped on insert + update
}) {}

export class UserNotFound extends Schema.TaggedError<UserNotFound>()("UserNotFound", { id: UserId }) {}

export class UserRepo extends Context.Service<UserRepo, {
  readonly insert: (user: typeof User.insert.Type) => Effect.Effect<User>
  readonly findById: (id: UserId) => Effect.Effect<User, UserNotFound>
  readonly findByEmail: (email: Email) => Effect.Effect<Option.Option<User>>
}>()("app/infra/UserRepo") {
  static readonly layer = Layer.effect(UserRepo, Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    const repo = yield* SqlModel.makeRepository(User, {
      tableName: "users", spanPrefix: "UserRepo", idColumn: "id",
    }) // insert / update / findById / delete (+ insertVoid / updateVoid) — spanned, RETURNING-based
    const findByEmail = SqlSchema.findOneOption({
      Request: Email, Result: User,
      execute: (email) => sql`SELECT * FROM users WHERE email = ${email}`,
    })
    return UserRepo.of({
      insert: (user) => repo.insert(user).pipe(Effect.orDie),
      findById: (id) => repo.findById(id).pipe(
        Effect.catchTags({
          NoSuchElementError: () => new UserNotFound({ id }),   // absence → domain error
          SchemaError: Effect.die,                              // schema/DDL drift → defect
          SqlError: Effect.die,
        })
      ),
      findByEmail: (email) => findByEmail(email).pipe(Effect.orDie),
    })
  }))
}
```

- Field helpers: `Model.GeneratedByDb(S)` (select/json only — serial/identity columns not used as the update key), `Model.GeneratedByApp(S)`, `Model.UuidV4Insert` / `UuidV7Insert` (app-generated ids, defaulted by `User.insert.makeEffect`), `Model.Sensitive`, `Model.FieldOption` (nullable column ↔ `Option`), `Model.FieldExcept([...])` / `Model.FieldOnly([...])`, `Model.Field({ select, update, json, ... })` for full control (e.g. a DB-generated primary key that must appear in `update`). Timestamps: `DateTimeInsert`/`DateTimeUpdate` (string columns, SQLite), `...FromDate` (pg `Date`), `...FromNumber` (epoch millis).
- `makeRepository` in v4 **no longer `orDie`s**: `findById` fails `NoSuchElementError | SchemaError | SqlError`, the writes fail `SchemaError | SqlError`. Decide at the repository which become domain errors and which are defects (schema/DDL drift — migrations are what keep schema and DDL in agreement).
- Build insert payloads with `User.insert.makeEffect({...})` — it fills generated ids and timestamps from the Effect `Clock`, so `TestClock` controls them. Use `User.json` / `User.jsonCreate` / `User.jsonUpdate` as the HttpApi `success` / `payload` schemas.

## Batching — kill N+1 with `SqlResolver`

```ts
import { Effect } from "effect"
import { SqlClient, SqlResolver } from "effect/sql"

export const program = Effect.gen(function* () {
  const sql = yield* SqlClient.SqlClient

  const UserById = SqlResolver.findById({
    Id: UserId,
    Result: User,
    ResultId: (user) => user.id,
    execute: (ids) => sql`SELECT * FROM users WHERE ${sql.in("id", ids)}`,  // ONE query for N concurrent lookups
  })
  const getById = (id: UserId) => SqlResolver.request(id, UserById)
  // missing ids fail with NoSuchElementError; also: SqlResolver.ordered (batched INSERT..RETURNING,
  // order-matched), .grouped (1→many), .void; SqlModel.makeResolvers derives them from a Model

  return yield* Effect.forEach(ids, getById, { concurrency: "unbounded" })
})
```

Concurrent `getById` calls are automatically batched into one query, and equal ids within a batch are deduplicated; batches never cross transaction boundaries. In v4 resolvers are plain `RequestResolver` values (no `yield*` to build them) and `Effect.withRequestCaching` is gone — for cross-batch caching wrap the resolver with `RequestResolver.withCache({ capacity })` (or `RequestResolver.asCache` for TTL/invalidation).

## Migrations

Forward-only, numbered files; runs in a transaction with an exclusive lock (concurrent deploys: loser no-ops). The `Migrator` module is `@stability unstable`.

```ts
import { Effect } from "effect"
import { SqlClient } from "effect/sql"

// src/migrations/0001_create_users.ts — default-export an Effect requiring SqlClient
export default Effect.gen(function* () {
  const sql = yield* SqlClient.SqlClient
  yield* sql`
    CREATE TABLE users (id uuid PRIMARY KEY, email varchar(255) NOT NULL UNIQUE)`
})
```

```ts
import { NodeServices } from "@effect/platform-node"
import { PgMigrator } from "@effect/sql-pg"
import { Layer } from "effect"
import { fileURLToPath } from "node:url"

export const MigratorLive = PgMigrator.layer({
  loader: PgMigrator.fromFileSystem(fileURLToPath(new URL("migrations", import.meta.url))),
}).pipe(Layer.provide([SqlLive, NodeServices.layer]))
```

`PgMigrator.layer` needs `FileSystem`, `Path` and `ChildProcessSpawner` (for the optional `schemaDirectory` dump) — `NodeServices.layer` provides all three. Other loaders: `Migrator.fromRecord({ "0001_create_users": effect })` (inline, handy in tests), `fromGlob` (bundlers). Compose with the client so dependents see a migrated database: `MigratorLive.pipe(Layer.provideMerge(SqlLive))`.

## Testing DB code

Two tiers, both by layer swap (the repo code never changes):

1. **Fast tier — in-memory SQLite**: `SqliteClient.layer({ filename: ":memory:" })` from `@effect/sql-sqlite-node` provides the same `SqlClient.SqlClient` service (4.x is built on `node:sqlite` → Node ≥ 22.5). Run migrations with `SqliteMigrator.layer({ loader: SqliteMigrator.fromRecord({...}) })`. Works when SQL is dialect-neutral (or branched with `onDialectOrElse`) and column codecs match — SQLite can't bind a JS `Date`, so `...FromDate` model fields need the pg tier.
2. **Real tier — testcontainers** (the pattern Effect's own pg tests use):

```ts
import { PostgreSqlContainer } from "@testcontainers/postgresql"
import { PgClient } from "@effect/sql-pg"
import { Context, Effect, Layer, Redacted, Schema } from "effect"

class ContainerError extends Schema.TaggedError<ContainerError>()("ContainerError", { cause: Schema.Unknown }) {}

export class PgContainer extends Context.Service<PgContainer, {
  readonly connectionUri: string
}>()("test/PgContainer") {
  static readonly layer = Layer.effect(PgContainer, Effect.gen(function* () {
    const container = yield* Effect.acquireRelease(
      Effect.tryPromise({
        try: () => new PostgreSqlContainer("postgres:alpine").start(),
        catch: (cause) => new ContainerError({ cause }),
      }),
      (c) => Effect.promise(() => c.stop())
    )
    return PgContainer.of({ connectionUri: container.getConnectionUri() })
  }))

  static readonly ClientLive = Layer.unwrap(
    Effect.gen(function* () {
      const { connectionUri } = yield* PgContainer
      return PgClient.layer({ url: Redacted.make(connectionUri) })
    })
  ).pipe(Layer.provide(PgContainer.layer))
}
```

`Layer.effect` scopes `acquireRelease` to the layer (v4 has no separate `Layer.scoped`). Run migrations in the test layer so tests exercise the real DDL.

Query builders: there is no effect-4 `@effect/sql-drizzle` / `@effect/sql-kysely` yet. Raw `sql` + `SqlSchema` is the default; if a builder is genuinely needed for dynamic query construction, compile it to SQL text + params and run it through `sql.unsafe(text, params)` inside the repository — the one sanctioned non-migration use, still bound parameters.

## Checklist

- [ ] SQL imported from `effect/sql` / `effect/schema` (`Model`), drivers on the same version as `effect` — no `@effect/sql` in an effect 4 project
- [ ] All queries via the `sql` tag (bound params); `sql.unsafe` only in migrations (or builder output with bound params, inside a repo)
- [ ] Every row decoded by Schema (`SqlSchema.*` / `Model`) — no `as` casts on rows; `findOne` vs `findOneOption` chosen deliberately
- [ ] Repositories: `Context.Service` with a static `layer` wrapping `SqlClient`; `NoSuchElementError`/`SqlError` translated or turned into defects explicitly; no `withTransaction` inside
- [ ] Transaction boundaries declared in workflows; multi-write invariants covered by a transaction test (failure → nothing persisted)
- [ ] Hot lookups batched with `SqlResolver` where N+1 is possible
- [ ] Migrations numbered, forward-only, run by a `Migrator` layer in dev/test/prod alike
- [ ] Sensitive columns marked `Model.Sensitive`; case transforms configured once on the client
- [ ] Large result sets streamed (`.stream`), not loaded whole
