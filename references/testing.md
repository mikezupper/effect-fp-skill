# Testing: Properties First, Layers over Mocks

From [The Property-Based Testing series](https://fsharpforfunandprofit.com/series/property-based-testing/): example tests show your code works for the cases you thought of; property tests attack the cases you didn't. The architecture (pure core, services behind tags) makes both cheap.

Stack: **vitest + @effect/vitest** (`it.effect`, `it.prop`, test services) and **FastCheck** (bundled with Effect — `import { FastCheck, Arbitrary } from "effect"`, no extra dependency).

## The test pyramid for this architecture

1. **Pure domain functions** — plain unit + property tests. No layers, no effects. Most tests live here; push logic into the pure core precisely so it can be tested this way.
2. **Workflows** — `it.effect` with test layers substituted for infrastructure. Deterministic: `TestClock` for time, seeded `Random`, in-memory fakes.
3. **Adapters/integration** — a few tests against real infra (testcontainers DB, real HTTP server on a random port) to verify the schemas/SQL/wiring match reality.

## it.effect and test services

```ts
import { assert, describe, it } from "@effect/vitest"
import { Effect, TestClock, Fiber, Duration } from "effect"

describe("session", () => {
  // it.effect provides TestContext: TestClock, deterministic environment
  it.effect("expires after 30 minutes", () =>
    Effect.gen(function* () {
      const session = yield* createSession(user)
      const fiber = yield* expireSessions.pipe(Effect.fork)

      yield* TestClock.adjust(Duration.minutes(29))
      assert.isTrue(yield* isActive(session))

      yield* TestClock.adjust(Duration.minutes(2))   // virtual time — test runs in µs
      assert.isFalse(yield* isActive(session))
      yield* Fiber.interrupt(fiber)
    }).pipe(Effect.provide(TestLayer))
  )

  // it.live when you genuinely need the real clock (rare)
})
```

Adjacent APIs worth knowing: `it.scoped` (test needs a `Scope`), `it.layer(SharedLayer)((it) => ...)` (build an expensive layer once, share across a suite — e.g. a testcontainer DB), `it.flakyTest` (retry genuinely nondeterministic integration tests — never use it to paper over a race in your own code).

Never `setTimeout`/`sleep` in tests; never test retry/timeout/scheduling logic against wall-clock time. `TestClock.adjust` makes time-dependent logic instant and deterministic.

## Property-based testing

**Schemas are your generators.** `Arbitrary.make(schema)` derives a FastCheck arbitrary that respects every constraint (brands, patterns, bounds) — the same source of truth drives validation, serialization, and test generation.

```ts
import { Arbitrary, FastCheck, Schema } from "effect"
import { it } from "@effect/vitest"

const orderArb = Arbitrary.make(Order)

// Pure property via it.prop
it.prop("total is invariant under line-item reordering", [orderArb], ([order]) => {
  const shuffled = { ...order, items: Array.reverse(order.items) }
  return Equal.equals(orderTotal(order), orderTotal(shuffled))
})

// Effectful property
it.effect.prop("saved orders round-trip", [orderArb], ([order]) =>
  Effect.gen(function* () {
    const repo = yield* OrderRepo
    yield* repo.save(order)
    const loaded = yield* repo.findById(order.id)
    assert.deepStrictEqual(loaded, Option.some(order))
  }).pipe(Effect.provide(TestLayer))
)
```

The property patterns to reach for (from Wlaschin's ["Choosing properties"](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)):

| Pattern | Example |
|---|---|
| Round-trip / "there and back again" | `decode(encode(x)) === x` — **write this for every schema**; DB row ⇄ domain; API DTO ⇄ domain |
| Invariants | total ≥ 0; output list same length; all outputs satisfy predicate |
| Idempotence | `normalize(normalize(x)) === normalize(x)`; applying a webhook twice = once |
| Commutativity / "different paths, same destination" | order of independent operations doesn't matter |
| Oracle / test against a simple model | optimized implementation === naive obvious implementation |
| Induction | property holds for empty + holds for `cons` ⇒ holds for all |
| "Hard to prove, easy to verify" | verify the output (sorted? parses back?) rather than recompute it |

Avoid "the code equals the code" properties that re-implement the function under test.

## What to property-test in every app (minimum bar)

- [ ] Round-trip for every boundary schema (`encode ∘ decode` and `decode ∘ encode` where applicable)
- [ ] Every state machine: valid transition sequences never reach illegal states; invalid transitions always produce typed errors
- [ ] Every money/quantity calculation: invariants (non-negative, sums preserved, no float drift)
- [ ] Every normalize/parse/format pure function: idempotence + round-trip

## Testing error tracks

The error track is API surface — test it like one:

```ts
it.effect("declined payment surfaces PaymentDeclined and does not persist the order", () =>
  Effect.gen(function* () {
    const result = yield* placeOrder(validInput).pipe(Effect.flip)  // flip: failure becomes success
    assert.strictEqual(result._tag, "PaymentDeclined")
    const saved = yield* (yield* OrderRepo).findById(orderIdOf(validInput))
    assert.isTrue(Option.isNone(saved))                              // failure left no partial state
  }).pipe(Effect.provide(TestLayerWithDecliningGateway))
)
```

Use `Effect.flip` or `Effect.exit` + `Exit.isFailure` — never try/catch in tests.

## Rules

- No mocking libraries — fakes are `Layer.succeed(Tag, fakeImpl)`; stateful fakes use `Ref` for tracked state
- Deterministic always: TestClock for time, fixed FastCheck seed in CI reproduction, no network in unit/workflow tiers
- Test names state behavior ("expires after 30 minutes"), not implementation ("calls repo.delete")
- A bug found in production or by FastCheck becomes a pinned regression test with the shrunk counterexample
