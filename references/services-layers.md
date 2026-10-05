# Services, Layers, and Application Wiring

The "Recipe for a Functional App" realized: an onion. Pure domain at the center, workflows around it, infrastructure at the rim. Dependencies point inward only — the domain never imports infrastructure. Effect's `R` channel + `Layer` is the wiring mechanism.

## Project structure

```
src/
  domain/          # pure: schemas, branded types, unions, errors, pure functions
  workflows/       # Effect.fn pipelines: use-cases composed from domain + service interfaces
  services/        # Context.Service definitions + live/test Layer implementations
  http/ | cli/     # thin adapters: decode request → run workflow → encode response
  config.ts        # all Config definitions in one place
  main.ts          # THE entry point: compose AppLayer, runMain. Nothing else runs effects.
```

## Defining services

`Context.Service` is the one way to define a service: the interface is a type parameter, the identifier is a string namespaced by package and path, and the default implementation hangs off the class as a `static readonly layer`. Build the implementation with `Self.of({...})` and define each method with `Effect.fn("Service.method")` so every call gets a span:

```ts
import { Context, Effect, Layer, Option, Schema } from "effect"

export class OrderRepo extends Context.Service<OrderRepo, {
  readonly findById: (id: OrderId) => Effect.Effect<Option.Option<Order>, RepoError>
  readonly save: (order: Order) => Effect.Effect<void, RepoError>
}>()("app/services/OrderRepo") {
  static readonly layer = Layer.effect(
    OrderRepo,
    Effect.gen(function* () {
      const db = yield* Db

      const findById = Effect.fn("OrderRepo.findById")(
        function* (id: OrderId) {
          const row = yield* db.queryOne(findOrderSql(id))                 // unknown | null
          return yield* Schema.decodeUnknownEffect(Schema.OptionFromNullOr(OrderRow))(row)
        },
        Effect.map(Option.map(rowToDomain)),
        Effect.mapError((cause) => new RepoError({ cause })),  // translate to domain error
      )

      const save = Effect.fn("OrderRepo.save")(
        (order: Order) => db.execute(saveOrderSql(order)),
        Effect.mapError((cause) => new RepoError({ cause })),
      )

      return OrderRepo.of({ findById, save })
    })
  )
}
```

The layer's requirements (`Db` here) stay in its `R` type — they are wired in `main.ts`, not baked in. If a service should ship pre-wired, expose both: `layerNoDeps` (requirements open, for tests) and `layer = layerNoDeps.pipe(Layer.provide(Db.layer))`.

For a port with several interchangeable implementations (hexagonal adapters, library code), declare the service with no static layer and give each implementation its own `Layer`:

```ts
import { Context, Effect, Layer } from "effect"

export class PaymentGateway extends Context.Service<PaymentGateway, {
  readonly charge: (req: ChargeRequest) => Effect.Effect<Receipt, PaymentDeclined | GatewayError>
}>()("app/services/PaymentGateway") {}

export const PaymentGatewayStripe = Layer.effect(PaymentGateway, Effect.gen(function* () {
  const stripe = yield* StripeClient
  return PaymentGateway.of({ charge: Effect.fn("PaymentGateway.charge")((req: ChargeRequest) => stripe.charge(req)) })
}))
export const PaymentGatewayFake = Layer.succeed(PaymentGateway, PaymentGateway.of({ charge: () => Effect.succeed(fakeReceipt) }))
```

A capability with a sensible default (feature flags, tunables, per-request settings) is a `Context.Reference` — it needs no layer, and tests or callers override it with `Effect.provideService`:

```ts
import { Context } from "effect"

export const MaxLineItems = Context.Reference<number>("app/config/MaxLineItems", {
  defaultValue: () => 100
})
```

Rules:
- **Everything nondeterministic or external is a service**: DB, HTTP clients, message queues, file system, clock, random, UUID generation, feature flags. If it touches the world or varies between runs, it goes behind a `Context.Service`.
- Service methods return `Effect` with **domain-level** errors — infra errors are translated inside the service.
- Workflows depend on service *interfaces* only; the `R` channel documents exactly what each workflow needs. Access a service with `yield* OrderRepo` (or `OrderRepo.use((repo) => ...)` for one-liners).
- Need the shape as a type? `OrderRepo["Service"]`.
- **HttpApi gotcha:** inside handler bodies, service requirements become per-request requirements — providing the repo layer to the API layer does not satisfy them. Resolve services (`const repo = yield* OrderRepo`) while building the handler group, then close over them in the handlers (details in `app-shapes.md`).

## Layers: construction as a first-class value

Layers describe how to build services, including resources and dependencies. They are memoized — a layer used by many others is built once.

```ts
import { NodeHttpClient } from "@effect/platform-node"
import { Config, Effect, Layer, Logger, Redacted } from "effect"

// Resource-owning layer: acquire/release tied to the app lifecycle.
// Layer.effect scopes the resource to the layer — no separate "scoped" constructor.
export const DbLive = Layer.effect(
  Db,
  Effect.gen(function* () {
    const url = yield* Config.Redacted("DATABASE_URL")
    const pool = yield* Effect.acquireRelease(
      Effect.tryPromise({ try: () => createPool(Redacted.value(url)), catch: (cause) => new DbError({ cause }) }),
      (pool) => Effect.promise(() => pool.close())   // guaranteed on shutdown/interruption
    )
    return makeDb(pool)
  })
)

// Composition in main.ts — the ONLY place that knows concrete implementations
const AppLayer = Layer.mergeAll(
  OrderRepo.layer,
  PaymentGatewayStripe,
).pipe(
  Layer.provide(DbLive),
  Layer.provide(StripeClient.layer),
  Layer.provide(NodeHttpClient.layerUndici),
  Layer.provide(Logger.layer([Logger.consoleJson])),   // structured logs in prod
)
```

`Layer.provide` satisfies requirements and hides the provider; `Layer.provideMerge` satisfies them *and* re-exports the provider (useful when tests also need direct access to it). To pick an implementation from config at startup, return a layer from an effect with `Layer.unwrap`.

## Configuration

All config declared with `Config`, read at layer-construction time, validated at startup — the app fails fast with a precise message instead of failing at 3am on first use.

```ts
// config.ts — the single inventory of every knob the app has
import { Config } from "effect"

export const AppConfig = {
  port: Config.Port("PORT").pipe(Config.withDefault(3000)),
  databaseUrl: Config.Redacted("DATABASE_URL"),               // Redacted: never printed in logs/errors
  stripeKey: Config.Redacted("STRIPE_API_KEY"),
  logLevel: Config.LogLevel("LOG_LEVEL").pipe(Config.withDefault("Info" as const)),
  environment: Config.Literals(["development", "staging", "production"], "APP_ENV"),
}
```

Never read `process.env` directly. Secrets are always `Config.Redacted` — `Redacted<string>` cannot be accidentally logged. To apply the configured log level, provide `Layer.succeed(References.MinimumLogLevel, level)` from a `Layer.unwrap`.

## The entry point

Exactly one per executable. `runMain` installs signal handlers, runs finalizers on SIGINT/SIGTERM (graceful shutdown), and reports errors/defects properly.

```ts
// main.ts
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Layer } from "effect"

NodeRuntime.runMain(
  program.pipe(Effect.provide(AppLayer))
)

// Or, when the whole app is layers (HTTP server + background workers):
// NodeRuntime.runMain(Layer.launch(ServerLayer))
```

For environments that call *into* you (serverless handlers, test harnesses, frontend), build one `ManagedRuntime` at module scope and reuse it:

```ts
import { ManagedRuntime } from "effect"

const runtime = ManagedRuntime.make(AppLayer)
export const handler = (event: unknown) => runtime.runPromise(handleEvent(event))
```

Anywhere else, `Effect.runPromise`/`runSync` in application code is a design error: it severs the dependency graph, loses interruption, spans, and config. Compose Effects instead.

## Test layers

The payoff of capability-based DI: swapping infrastructure is `Layer` substitution, not a mocking framework.

```ts
import { assert, it } from "@effect/vitest"
import { Effect, Layer } from "effect"

const TestLayer = Layer.mergeAll(
  OrderRepo.layer,
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
- [ ] Every external dependency behind a `Context.Service`, including Clock/Random/UUID; defaults via `Context.Reference`
- [ ] Service implementations built with `Self.of({...})`; methods defined with `Effect.fn("Service.method")`
- [ ] Service methods expose domain errors, not infra errors
- [ ] All resources acquired with `acquireRelease` inside `Layer.effect` — cleanup is guaranteed, never manual
- [ ] One `config.ts`; secrets `Config.Redacted`; zero `process.env` reads elsewhere
- [ ] One entry point with `runMain` (or one module-scope `ManagedRuntime`); zero `run*` calls elsewhere
- [ ] Every service has (or can trivially have) a test/fake layer
