# Services, Layers, and Application Wiring

The "Recipe for a Functional App" realized: an onion. Pure domain at the center, workflows around it, infrastructure at the rim. Dependencies point inward only — the domain never imports infrastructure. Effect's `R` channel + `Layer` is the wiring mechanism.

## Project structure

```
src/
  domain/          # pure: schemas, branded types, unions, errors, pure functions
  workflows/       # Effect pipelines: use-cases composed from domain + service interfaces
  services/        # Context.Tag / Effect.Service definitions + live/test Layer implementations
  http/ | cli/     # thin adapters: decode request → run workflow → encode response
  config.ts        # all Config definitions in one place
  main.ts          # THE entry point: compose AppLayer, runMain. Nothing else runs effects.
```

## Defining services

`Effect.Service` for the common case (definition + default implementation together):

```ts
import { Effect, Layer } from "effect"

export class OrderRepo extends Effect.Service<OrderRepo>()("app/OrderRepo", {
  effect: Effect.gen(function* () {
    const db = yield* Db
    return {
      findById: (id: OrderId) =>
        db.queryOne(findOrderSql(id)).pipe(
          Effect.flatMap(Schema.decodeUnknown(OrderRow)),
          Effect.map(Option.map(rowToDomain)),
          Effect.mapError((e) => new RepoError({ cause: e })),  // translate to domain error
        ),
      save: (order: Order) => /* ... */,
    }
  }),
  dependencies: [Db.Default],
}) {}
```

Plain `Context.Tag` when you want the interface fully decoupled from any implementation (ports in the hexagonal sense, or library code):

```ts
export class PaymentGateway extends Context.Tag("app/PaymentGateway")<
  PaymentGateway,
  {
    readonly charge: (req: ChargeRequest) => Effect.Effect<Receipt, PaymentDeclined | GatewayError>
  }
>() {}

export const PaymentGatewayStripe = Layer.effect(PaymentGateway, Effect.gen(function* () { /* ... */ }))
export const PaymentGatewayFake   = Layer.succeed(PaymentGateway, { charge: () => Effect.succeed(fakeReceipt) })
```

Rules:
- **Everything nondeterministic or external is a service**: DB, HTTP clients, message queues, file system, clock, random, UUID generation, feature flags. If it touches the world or varies between runs, it goes behind a tag.
- Service methods return `Effect` with **domain-level** errors — infra errors are translated inside the service.
- Workflows depend on service *interfaces* only; the `R` channel documents exactly what each workflow needs.

## Layers: construction as a first-class value

Layers describe how to build services, including resources and dependencies. They are memoized — a layer used by many others is built once.

```ts
// Resource-owning layer: acquire/release tied to the app lifecycle
export const DbLive = Layer.scoped(
  Db,
  Effect.acquireRelease(
    Effect.gen(function* () {
      const url = yield* Config.redacted("DATABASE_URL")
      const pool = yield* Effect.tryPromise({ try: () => createPool(Redacted.value(url)), catch: (c) => new DbError({ cause: c }) })
      return makeDb(pool)
    }),
    (db) => Effect.promise(() => db.close())   // guaranteed on shutdown/interruption
  )
)

// Composition in main.ts — the ONLY place that knows concrete implementations
const AppLayer = Layer.mergeAll(
  OrderRepo.Default,
  PaymentGatewayStripe,
  NodeHttpClient.layer,
).pipe(
  Layer.provide(DbLive),
  Layer.provide(Logger.json),          // structured logs in prod
)
```

## Configuration

All config declared with `Config`, read at layer-construction time, validated at startup — the app fails fast with a precise message instead of failing at 3am on first use.

```ts
// config.ts — the single inventory of every knob the app has
export const AppConfig = {
  port: Config.integer("PORT").pipe(Config.withDefault(3000)),
  databaseUrl: Config.redacted("DATABASE_URL"),               // Redacted: never printed in logs/errors
  stripeKey: Config.redacted("STRIPE_API_KEY"),
  logLevel: Config.logLevel("LOG_LEVEL").pipe(Config.withDefault(LogLevel.Info)),
  environment: Config.literal("development", "staging", "production")("APP_ENV"),
}
```

Never read `process.env` directly. Secrets are always `Config.redacted` — `Redacted<string>` cannot be accidentally logged.

## The entry point

Exactly one per executable. `runMain` installs signal handlers, runs finalizers on SIGINT/SIGTERM (graceful shutdown), and reports errors/defects properly.

```ts
// main.ts
import { NodeRuntime } from "@effect/platform-node"

NodeRuntime.runMain(
  program.pipe(Effect.provide(AppLayer))
)
```

For environments that call *into* you (serverless handlers, test harnesses, frontend), build one `ManagedRuntime` at module scope and reuse it:

```ts
const runtime = ManagedRuntime.make(AppLayer)
export const handler = (event: unknown) => runtime.runPromise(handleEvent(event))
```

Anywhere else, `Effect.runPromise`/`runSync` in application code is a design error: it severs the dependency graph, loses interruption, spans, and config. Compose Effects instead.

## Test layers

The payoff of capability-based DI: swapping infrastructure is `Layer` substitution, not a mocking framework.

```ts
const TestLayer = Layer.mergeAll(
  OrderRepo.Default,
  PaymentGatewayFake,
).pipe(Layer.provide(DbInMemory))

it.effect("places an order", () =>
  Effect.gen(function* () {
    const result = yield* placeOrder(validInput)
    assert.strictEqual(result.state._tag, "Placed")
  }).pipe(Effect.provide(TestLayer))
)
```

## Checklist

- [ ] Dependency direction: domain ← workflows ← adapters; verified by imports (domain/ imports only `effect` and itself)
- [ ] Every external dependency behind a tag, including Clock/Random/UUID
- [ ] Service methods expose domain errors, not infra errors
- [ ] All resources built with `Layer.scoped`/`acquireRelease` — cleanup is guaranteed, never manual
- [ ] One `config.ts`; secrets `Redacted`; zero `process.env` reads elsewhere
- [ ] One entry point with `runMain` (or one module-scope `ManagedRuntime`); zero `run*` calls elsewhere
- [ ] Every service has (or can trivially have) a test/fake layer
