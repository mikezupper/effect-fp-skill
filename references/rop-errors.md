# Railway-Oriented Programming with Effect

The two-track model from [fsharpforfunandprofit.com/rop](https://fsharpforfunandprofit.com/rop/): every function is a switch — success continues down the track, failure diverts to the error track and bypasses subsequent steps. In Effect this is not a pattern you implement; it IS the `Effect<A, E, R>` type. `flatMap`/`gen` compose the success track; the error track short-circuits automatically and stays fully typed.

## Defining errors

One tagged error class per failure mode. The `_tag` is the discriminant that makes `catchTag` and exhaustive handling work.

```ts
import { Data, Schema } from "effect"

// Standard domain error
export class OrderNotFound extends Data.TaggedError("OrderNotFound")<{
  readonly orderId: OrderId
}> {}

export class PaymentDeclined extends Data.TaggedError("PaymentDeclined")<{
  readonly reason: string
  readonly retriable: boolean
}> {}

// Error that must cross a process boundary (HTTP response, queue) → Schema.TaggedError
// (serializable, and usable directly in HttpApi endpoint definitions)
export class InsufficientStock extends Schema.TaggedError<InsufficientStock>()(
  "InsufficientStock",
  { sku: Sku, requested: Schema.Int, available: Schema.Int }
) {}
```

Rules:
- Error names describe **what happened**, not who threw (`OrderNotFound`, not `DbError` leaking from a repository — translate infra errors into domain terms at the service boundary).
- Include the data a handler needs to react (ids, the invalid value, whether it's retriable). Never just a message string.
- Namespace tags in larger apps: `"Orders/PaymentDeclined"`.

## Expected errors vs defects — the taxonomy

This distinction keeps the error channel meaningful. Decide it per error, deliberately:

| | Expected error (error channel) | Defect (`Effect.die`) |
|---|---|---|
| What | Anticipated domain/infra outcome a caller might handle | Bug or broken invariant; unrecoverable |
| Examples | `UserNotFound`, `PaymentDeclined`, `RateLimited`, validation failure | Impossible state reached, config missing *after* startup validation, programmer error |
| Appears in `E`? | Yes, typed | No — invisible to the type, crashes the fiber |
| Handling | `catchTag`/`catchTags`/`match` | Don't catch (except top-level logging); fix the bug |

```ts
// Convert an "impossible" expected error into a defect at the point you *know* it can't happen:
const user = yield* getUser(sessionUserId).pipe(Effect.orDie) // session guarantees existence
```

Never use `orDie`/`die` to avoid designing an error type — only to assert a locally-proven invariant.

## Handling errors — where and how

Handle errors **where you have the context to do something meaningful**, usually at the workflow edge (HTTP handler, CLI command, queue consumer). Mid-pipeline code should just let errors flow past.

```ts
import { Effect } from "effect"

// Handle specific tracks; unhandled tags remain in the type — the compiler tracks what's left
const result = placeOrder(input).pipe(
  Effect.catchTags({
    OrderNotFound: (e) => HttpResponse.notFound(e.orderId),
    InsufficientStock: (e) => HttpResponse.conflict(e),
    // PaymentDeclined intentionally not caught here → still in E, handled by caller
  })
)

// Exhaustive fold when you must produce a value either way
const summary = yield* priceOrder(order).pipe(
  Effect.match({
    onFailure: (e) => `pricing failed: ${e._tag}`,
    onSuccess: (p) => `total ${p.total}`,
  })
)

// Recover with a fallback — only when the fallback is genuinely correct, not to silence the type
const config = yield* fetchRemoteConfig.pipe(
  Effect.catchTag("ConfigServiceUnavailable", () => Effect.succeed(defaultConfig)),
  Effect.tapErrorTag("ConfigServiceUnavailable", () => Effect.logWarning("using default config"))
)
```

Forbidden: `Effect.catchAll(() => Effect.succeed(fallback))` — it swallows every current *and future* error silently. Catch tags you can name; let the rest propagate.

## Wrapping the outside world (interop edge)

Third-party promise/throwing APIs get wrapped exactly once, in the infrastructure layer, with a typed error:

```ts
export class StripeError extends Data.TaggedError("StripeError")<{
  readonly cause: unknown
  readonly retriable: boolean
}> {}

const charge = (req: ChargeRequest) =>
  Effect.tryPromise({
    try: (signal) => stripe.charges.create(toStripeShape(req), { signal }),
    catch: (cause) => new StripeError({ cause, retriable: isNetworkish(cause) }),
  })
```

`try` receives an `AbortSignal` — pass it through so Effect interruption/timeouts actually cancel the underlying request.

## Fail-fast vs error accumulation

ROP short-circuits by default (first failure wins). Validation should usually **accumulate** so users see all problems at once:

```ts
// Schema: report every issue, not just the first
const decode = Schema.decodeUnknown(OrderInput, { errors: "all" })

// Independent checks: collect all failures
const checked = yield* Effect.validateAll(
  [checkInventory(order), checkAddress(order), checkFraud(order)],
  (eff) => eff
) // fails with NonEmptyArray of all errors

// Partition successes/failures without failing at all (batch jobs)
const [failures, successes] = yield* Effect.partition(items, processItem)
yield* Effect.logInfo(`processed ${successes.length}, failed ${failures.length}`)
```

Rule of thumb: accumulate at input boundaries and in batch processing; fail fast inside sequential business workflows.

## Retry belongs on the error track

Transient failures are handled by policy, not by hand-rolled loops — see `production.md` for full resilience patterns:

```ts
import { Schedule } from "effect"

const resilientCharge = charge(req).pipe(
  Effect.retry({
    schedule: Schedule.exponential("100 millis").pipe(Schedule.jittered),
    times: 5,
    while: (e) => e.retriable,     // only retry what's actually transient
  }),
  Effect.timeout("10 seconds")
)
```

## Checklist

- [ ] Every error is a `Data.TaggedError`/`Schema.TaggedError` with a descriptive tag and useful fields
- [ ] `E` in every public signature is a union of named errors — never `Error`, `unknown`, or `never` (unless truly infallible)
- [ ] Expected-vs-defect decided per error; `orDie` only on proven invariants
- [ ] Infra errors translated to domain errors at service boundaries
- [ ] Errors handled at workflow edges with `catchTags`/`match`; no blanket `catchAll`
- [ ] Boundary validation accumulates errors; workflows fail fast
- [ ] All `tryPromise` wrappers thread the `AbortSignal`
