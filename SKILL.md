---
name: effect-fp-skill
description: Build production-grade TypeScript applications using Effect (effect.website) and functional programming throughout. Use whenever creating or modifying a TypeScript app, service, CLI, library, or full-stack project. Enforces railway-oriented programming, domain modeling with types, capability-based dependency injection, and property-based testing.
---

# Effect-First Functional TypeScript

You build every TypeScript application with [Effect](https://effect.website) as the foundation, applying functional programming end-to-end. The design philosophy comes from Scott Wlaschin's F# work (railway-oriented programming, "Designing with Types", "Parse, don't validate", functional core / imperative shell) — Effect is its TypeScript-native realization.

**Target the Effect 4.x line** (APIs herein verified against effect 4.0.1, October 2026). The effect-family packages (`effect`, `@effect/platform-node`, `@effect/sql-pg`, `@effect/vitest`, `@effect/opentelemetry`, `@effect/ai-*`, …) share one version number: pin them exactly and upgrade them together. `@effect/platform`, `@effect/cli`, `@effect/sql` and `@effect/ai` are discontinued — their modules now live in `effect` subpaths (`effect/http`, `effect/http-api`, `effect/cli`, `effect/sql`, `effect/ai`, `effect/testing`, …). A 3.x codebase should be migrated as a whole, never mixed with 4.x.

## Philosophy

1. **Railway-oriented programming.** Every operation that can fail returns `Effect<Success, Error, Requirements>`. Errors are values on a typed track, never exceptions. Compose the happy path; handle failures explicitly where you have context to do so.
2. **Make illegal states unrepresentable.** Model the domain with branded types, tagged unions, and `Option`. If the compiler accepts it, it should be valid.
3. **Parse, don't validate.** Decode untrusted data with `Schema` exactly once at each boundary (HTTP, DB, env, queue, file). The core only ever sees rich domain types.
4. **Functional core, managed shell.** The domain is pure functions over immutable data. Workflows are `Effect` pipelines. Infrastructure lives behind services (`Context.Service`) wired with `Layer`. Exactly one runtime entry point.
5. **Totality.** Functions handle every input in their type. No partial functions, no `throw`, no `any`, no silent `undefined`.

## Hard rules (non-negotiable)

- **Effect only** — do NOT use lodash or any other external FP/utility library. Effect's built-in modules (`Array`, `Record`, `Option`, `Result`, `String`, `Number`, `Order`, `Predicate`, `pipe`, `Struct`) are the standard library.
- **No `async/await`, no `try/catch`, no `throw`** in application code. Wrap third-party async/throwing APIs once at the edge with `Effect.tryPromise`/`Effect.try` and give the failure a tagged error type.
- **No `null`/`undefined` in domain types** — use `Option<A>`. No boolean flags encoding state machines — use tagged unions.
- **Every fallible operation carries a typed error.** Define errors with `Schema.TaggedError`. Never `Effect<A, Error>` or `Effect<A, unknown>`.
- **Decode at every boundary** with `Schema`; never cast (`as`) external data into domain types.
- **One runtime entry point** (`runMain` / a single `ManagedRuntime`). `Effect.runPromise` scattered through the codebase is the #1 smell.
- **All dependencies are services** — DB, HTTP clients, clock, random, UUIDs, file system, loggers. Defined with `Context.Service` (or `Context.Reference` when there is a sane default), provided via `Layer`. Domain code never imports infrastructure.
- **Strict TypeScript**: `"strict": true`, `"exactOptionalPropertyTypes": true`, `"noUncheckedIndexedAccess": true`. No `any`, no non-null assertions (`!`).
- **Immutability everywhere**: `readonly` fields, `ReadonlyArray`, no in-place mutation outside a contained local scope.

## Decision table

| Situation | Reach for |
|---|---|
| Fallible/effectful operation | `Effect<A, E, R>` with `Effect.gen` or `pipe` |
| Reusable effectful function / workflow | `Effect.fn("name")(function* (...) {...})` — traced; `Effect.fnUntraced` for hot paths |
| Absence of a value | `Option<A>` |
| Pure validation result you need as data | `Result<A, E>` |
| Domain error | `class X extends Schema.TaggedError<X>()("X", {...})` |
| Handling specific errors | `Effect.catchTag` (one tag or an array) / `Effect.catchTags`; nested reasons: `Effect.catchReason` |
| Boundary data (HTTP body, env, DB row, JSON) | `Schema.decodeUnknownEffect` |
| ID / constrained primitive | `Schema.brand` branded type |
| State machine / variants | `Data.TaggedEnum` + `$match` |
| Dependency (DB, clock, API client…) | `Context.Service` + `static readonly layer` |
| Configuration & secrets | `Config` / `Config.Redacted` |
| Retry / repeat / backoff | `Schedule` |
| Resource needing cleanup | `Effect.acquireRelease` + `Scope` |
| Concurrency over a collection | `Effect.forEach` with `{ concurrency: n }` — always bounded |
| Unbounded / large data | `Stream` |
| Current time / randomness | `Clock` / `Random` services (testable), never `Date.now()`/`Math.random()` in domain code |
| Caching / memoization | `Effect.cachedWithTTL`, `Cache` |
| Batching & deduping requests | `Effect.request` + `RequestResolver` |

## Anti-patterns — never do these

```ts
// ❌ throw in domain code            → return a tagged error in the Effect error channel
// ❌ try/catch                       → Effect.try / Effect.tryPromise at the interop edge only
// ❌ async function doThing()        → const doThing = Effect.fn("doThing")(function* () { ... })
// ❌ user: User | null               → Option<User>
// ❌ isPaid: boolean, isShipped: boolean  → Data.TaggedEnum<{ Draft:{}; Paid:{...}; Shipped:{...} }>
// ❌ JSON.parse(body) as OrderDto    → Schema.decodeUnknownEffect(OrderDto)(body)
// ❌ process.env.DATABASE_URL!       → Config.Redacted("DATABASE_URL")
// ❌ Effect.runPromise inside a service/handler  → compose Effects; run once at the entry point
// ❌ Effect.catch(() => Effect.succeed(default))  → swallowing errors; catch specific tags
// ❌ new Date() / Math.random() in domain  → DateTime.now / Clock / Random (injectable, testable)
// ❌ _.chain(xs).map(...).filter(...)      → pipe(xs, Array.map(...), Array.filter(...)) from "effect"
// ❌ let total = 0; for (...) total += x   → Array.reduce / pure expression
```

## Workflow for building an app

1. **Scaffold** (`references/scaffold.md`): strict tsconfig, pinned versions, language-service plugin, lint enforcement.
2. **Model the domain first** (`references/domain-types.md`): types, brands, unions, schemas. Write the types before any logic — wrong states should fail to compile.
3. **Define the error taxonomy** (`references/rop-errors.md`): one tagged error per failure mode; decide expected error vs defect.
4. **Define service interfaces** the workflows need (`references/services-layers.md`) — interfaces first, implementations later.
5. **Write workflows** as `Effect.fn("name")` pipelines over domain types and service interfaces. Pure decisions in plain functions; effects only for I/O. Prefer the commands-in/events-out shape (`references/domain-types.md`).
6. **Implement infrastructure** as Layers (`references/database.md` for persistence); wire the app in one place; run with `runMain`.
7. **Test** (`references/testing.md`): pure functions directly, workflows with test layers, invariants with property-based tests derived from schemas.
8. **Production-harden** (`references/production.md`, `references/concurrency.md`): spans, metrics, retries, timeouts, resource safety, graceful shutdown. Non-optional — see the checklist there.
9. **Self-review** (`references/code-review.md`): run the full review pass before declaring the work done. Mandatory.

## Reference files — read before working in each area

| File | Read when |
|---|---|
| `references/scaffold.md` | Starting a project: package versions, tsconfig, lint rules, language-service |
| `references/rop-errors.md` | Designing error types, error handling, retries vs failures, accumulating validations |
| `references/domain-types.md` | Modeling the domain: brands, schemas, tagged unions, smart constructors, domain events |
| `references/pattern-matching.md` | Any branching over variants: the Match module, exhaustiveness |
| `references/services-layers.md` | Dependency injection, Layer wiring, config, app entry points |
| `references/database.md` | Persistence: `effect/sql`, repositories, transactions, migrations, batching |
| `references/concurrency.md` | Fibers, queues, pub/sub, semaphores, rate limiting, background work |
| `references/testing.md` | Any test writing: @effect/vitest, TestClock, property-based testing |
| `references/production.md` | Observability, resilience, resource safety, deployment checklist |
| `references/app-shapes.md` | Starting an HTTP API, CLI, full-stack app, or publishable library |
| `references/code-review.md` | ALWAYS, at the end of every task — the mandatory self-review pass |
| `references/supplemental-ai.md` | Supplemental — only when the app integrates LLMs (`effect/ai`) |

## Canonical style

```ts
import { Array, Effect, Order, pipe } from "effect"

// Workflows: Effect.fn("name") — generator body, stack traces, and a tracing span for free
const placeOrder = Effect.fn("placeOrder")(function* (input: unknown) {
  const order = yield* decodeOrder(input)          // parse, don't validate
  const priced = yield* priceOrder(order)          // may fail: PricingUnavailable
  yield* chargePayment(priced)                     // may fail: PaymentDeclined
  yield* Effect.log("order placed", { orderId: priced.id })
  return priced
})

// Data transformation: pipe + Effect's data modules
const activeNames = (users: ReadonlyArray<User>) =>
  pipe(
    users,
    Array.filter((u) => u.status._tag === "Active"),
    Array.map((u) => u.name),
    Array.sort(Order.String)
  )
```
