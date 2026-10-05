# Testing: Properties First, Layers over Mocks

From [The Property-Based Testing series](https://fsharpforfunandprofit.com/series/property-based-testing/): example tests show your code works for the cases you thought of; property tests attack the cases you didn't. The architecture (a pure core, with services behind `Context.Service` keys) makes both cheap.

Stack: **vitest 5 + @effect/vitest 4.0.1** (`it.effect`, `it.prop`, `it.effect.prop`, `layer`). Property testing is **built into `effect`**. The `Arbitrary` module derives generators from Schemas, and `it.prop` takes Schemas directly. FastCheck is no longer bundled or needed; don't add it. Test services live in `effect/testing` (`TestClock`, `TestConsole`, `TestSchema`).

Verified against effect 4.0.1 (October 2026).

## The test pyramid for this architecture

1. **Pure domain functions.** Plain unit and property tests, with no layers and no effects. Most tests live here. That's why logic gets pushed into the pure core: so it can be tested this way.
2. **Workflows.** `it.effect` with test layers substituted for infrastructure. Keep them deterministic: `TestClock` for time, seeded generators, in-memory fakes.
3. **Adapters/integration.** A few tests against real infra (testcontainers DB, a real HTTP server on a random port) to verify that the schemas, SQL and wiring match reality.

## it.effect and test services

```ts
import { assert, describe, it } from "@effect/vitest"
import { Duration, Effect, Fiber } from "effect"
import { TestClock } from "effect/testing"

describe("session", () => {
  // it.effect provides TestClock (virtual time) and a Scope
  it.effect("expires after 30 minutes", () =>
    Effect.gen(function* () {
      const session = yield* createSession(user)
      const fiber = yield* Effect.forkChild(expireSessions)

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

Adjacent APIs worth knowing:

- `it.effect` already provides a `Scope`, so v4 has no `it.scoped`.
- `layer(SharedLayer)("suite", (it) => ...)` from `@effect/vitest` (or `it.layer(...)` nested) builds an expensive layer once and shares it across a suite, e.g. a testcontainer DB. State is shared between the tests inside it.
- `it.effect.each([...])("name %#", (case) => ...)` runs parameterized cases.
- `it.flakyTest(effect, timeout)` retries genuinely nondeterministic integration tests. Never use it to paper over a race in your own code.

Effect 3's `Effect.fork` is `Effect.forkChild` in v4. `TestClock.adjust` accepts any `Duration.Input` (`Duration.minutes(2)`, `"2 minutes"`, millis).

Never use `setTimeout`/`sleep` in tests, and never test retry/timeout/scheduling logic against wall-clock time. `TestClock.adjust` makes time-dependent logic instant and deterministic.

## Property-based testing

**Schemas are your generators.** Pass a Schema straight to `it.prop` / `it.effect.prop`, or derive an explicit generator with `Arbitrary.schema(MySchema)`. Generated values satisfy every check (brands, patterns, bounds, non-empty), so the same source of truth drives validation, serialization and test generation.

```ts
import { assert, it } from "@effect/vitest"
import { Array, Effect, Option } from "effect"

// Pure property via it.prop — pass the Schema; the generator is derived from it
it.prop("total is invariant under line-item reordering", [Order], ([order]) => {
  const reversed = { ...order, items: Array.reverse(order.items) }
  return orderTotal(order) === orderTotal(reversed)
})

// Effectful property
it.effect.prop("saved orders round-trip", [Order], ([order]) =>
  Effect.gen(function* () {
    const repo = yield* OrderRepo
    yield* repo.save(order)
    const loaded = yield* repo.findById(order.id)
    assert.deepStrictEqual(loaded, Option.some(order))
  }).pipe(Effect.provide(OrderRepo.layerTest))
)
```

A property fails if it returns `false`, throws, fails an `assert`, or (for `it.effect.prop`) fails its Effect. Failing inputs are shrunk automatically.

```ts
import { it } from "@effect/vitest"
import { Arbitrary, Schema } from "effect"

const Percent = Schema.Int.check(Schema.isBetween({ minimum: 0, maximum: 100 }))
const applyDiscount = (total: number, percent: number) => Math.round(total * (100 - percent) / 100)

// Explicit Arbitrary when you need to compose: map / filter / flatMap / all / array
const payableOrders = Arbitrary.schema(Order).pipe(
  Arbitrary.filter((order) => orderTotal(order) > 0)   // filters must reject RARELY
)

it.prop(
  "a discount never increases the total",
  { order: payableOrders, percent: Percent },          // Schemas and Arbitraries mix freely
  ({ order, percent }) => applyDiscount(orderTotal(order), percent) <= orderTotal(order),
  { arbitrary: { seed: 42, runs: 200 } }               // pinned seed: reproduce a CI failure
)
```

Generator gotchas, both seen as `Property exhausted after 0 run(s) and N discard(s)`:

- An `Arbitrary.filter` (or a `Schema.refine`/`makeFilter` predicate the generator can't see into) that rejects most values exhausts the generator. Narrow the *schema* with built-in checks such as `isBetween`, `isMinLength`, `isPattern`, or build the value with `Arbitrary.map`/`flatMap`.
- `isPattern` regexes with flags (`/.../i`) aren't honored by generation. Spell the character classes out instead (`[A-Za-z]`).

### Round-trip every boundary schema

`TestSchema.Asserts` from `effect/testing` turns "decode ∘ encode = id" into one line:

```ts
import { it } from "@effect/vitest"
import { Schema } from "effect"
import { TestSchema } from "effect/testing"

// decode(encode(x)) == x for generated x — one line per boundary schema
it.effect("Order round-trips through its encoded form", () =>
  new TestSchema.Asserts(Order).verifyRoundTripEffect({ runs: 200 }))

// ...and through the JSON codec HttpApi / RPC actually use
it.effect("Order round-trips through JSON", () =>
  new TestSchema.Asserts(Schema.toCodecJson(Order)).verifyRoundTripEffect())
```

The same class also offers `.decoding().succeed(input, expected)` / `.fail(input, message)`, `.encoding()`, and `.arbitrary().verifyGeneration()`. Use them to pin specific wire examples and error messages.

The property patterns to reach for (from Wlaschin's ["Choosing properties"](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)):

| Pattern | Example |
|---|---|
| Round-trip / "there and back again" | `decode(encode(x)) === x`. **Write this for every schema** (`TestSchema.Asserts(...).verifyRoundTripEffect()`): DB row ⇄ domain, API DTO ⇄ domain |
| Invariants | total ≥ 0; output list same length; all outputs satisfy predicate |
| Idempotence | `normalize(normalize(x)) === normalize(x)`; applying a webhook twice = once |
| Commutativity / "different paths, same destination" | order of independent operations doesn't matter |
| Oracle / test against a simple model | optimized implementation === naive obvious implementation |
| Induction | property holds for empty + holds for `cons` ⇒ holds for all |
| "Hard to prove, easy to verify" | verify the output (sorted? parses back?) rather than recompute it |

Avoid "the code equals the code" properties that re-implement the function under test.

## What to property-test in every app (minimum bar)

- [ ] Round-trip for every boundary schema (`encode ∘ decode` and `decode ∘ encode` where applicable), including through `Schema.toCodecJson` for JSON boundaries
- [ ] Every state machine: valid transition sequences never reach illegal states; invalid transitions always produce typed errors
- [ ] Every money/quantity calculation: invariants (non-negative, sums preserved, no float drift)
- [ ] Every normalize/parse/format pure function: idempotence + round-trip

## Testing error tracks

The error track is API surface. Test it like one:

```ts
it.effect("declined payment surfaces PaymentDeclined and does not persist the order", () =>
  Effect.gen(function* () {
    const error = yield* placeOrder(validInput).pipe(Effect.flip)  // flip: failure becomes success
    assert.strictEqual(error._tag, "PaymentDeclined")
    const saved = yield* (yield* OrderRepo).findById(orderIdOf(validInput))
    assert.isTrue(Option.isNone(saved))                             // failure left no partial state
  }).pipe(Effect.provide(TestLayerWithDecliningGateway))
)
```

Use `Effect.flip`, `Effect.exit` + `Exit.isFailure`, or `Effect.result` + `Result.isFailure`. Never use try/catch in tests.

## Rules

- No mocking libraries. Fakes are `Layer.succeed(Service, Service.of(fakeImpl))` or a `static readonly layerTest` on the service class. Stateful fakes keep their tracked state in a `Ref`. Use `Layer.provideMerge` when the test needs to reach the fake's state too.
- Always deterministic: TestClock for time, pinned `{ arbitrary: { seed } }` (or the failure's `replay` token) to reproduce a property failure, no network in the unit and workflow tiers.
- Test names state behavior ("expires after 30 minutes"), not implementation ("calls repo.delete").
- A bug found in production or by a property test becomes a pinned regression test with the shrunk counterexample.
