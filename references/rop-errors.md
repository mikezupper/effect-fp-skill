# Railway-Oriented Programming with Effect

The two-track model from [fsharpforfunandprofit.com/rop](https://fsharpforfunandprofit.com/rop/): every function is a switch — success continues down the track, failure diverts to the error track and bypasses subsequent steps. In Effect this is not a pattern you implement; it IS the `Effect<A, E, R>` type. `flatMap`/`gen` compose the success track; the error track short-circuits automatically and stays fully typed.

## Defining errors

One tagged error class per failure mode. The `_tag` is the discriminant that makes `catchTag` and exhaustive handling work. Use `Schema.TaggedError` by default: the fields are schemas, so the error is validated, serializable across process boundaries (HTTP responses, queues, workers), and usable directly in `HttpApi` endpoint definitions. Instances are yieldable — `return yield* new OrderNotFound({ orderId })` fails the effect.

```ts
import { Schema } from "effect"

export class OrderNotFound extends Schema.TaggedError<OrderNotFound>()("OrderNotFound", {
  orderId: OrderId,
}) {}

export class PaymentDeclined extends Schema.TaggedError<PaymentDeclined>()("PaymentDeclined", {
  reason: Schema.String,
  retriable: Schema.Boolean,
}) {}

export class InsufficientStock extends Schema.TaggedError<InsufficientStock>()("InsufficientStock", {
  sku: Sku,
  requested: Schema.Int,
  available: Schema.Int,
}) {}

// Wrapping an infra failure: keep the original as a Defect-typed cause (never an untyped string)
export class RepoError extends Schema.TaggedError<RepoError>()("RepoError", {
  cause: Schema.Defect(),
}) {}
```

Domain errors stay HTTP-free: the HTTP adapter attaches statuses in the API definition (`error: E.pipe(HttpApiSchema.status(409))` — see `app-shapes.md`). Reserve the class-level `{ httpApiStatus: 409 }` annotation for errors that are defined in the HTTP layer itself (e.g. `Unauthorized` from auth middleware).

`Data.TaggedError("X")<{...}>` still exists for purely in-process errors whose fields have no schema, but prefer `Schema.TaggedError` so every error can cross a boundary without rework.

Rules:
- Error names describe **what happened**, not who threw (`OrderNotFound`, not `DbError` leaking from a repository — translate infra errors into domain terms at the service boundary).
- Include the data a handler needs to react (ids, the invalid value, whether it's retriable). Never just a message string.
- Namespace tags in larger apps: `"Orders/PaymentDeclined"`.

## Errors with reasons

When one failure mode has several distinct causes a caller may react to differently, model it as a single error with a tagged `reason` union instead of widening `E` with many siblings. The outer tag keeps signatures small; the reason keeps handling precise:

```ts
import { Effect, Schema } from "effect"

export class CardExpired extends Schema.TaggedError<CardExpired>()("CardExpired", {}) {}
export class InsufficientFunds extends Schema.TaggedError<InsufficientFunds>()("InsufficientFunds", {
  shortBy: Schema.Int,
}) {}
export class FraudSuspected extends Schema.TaggedError<FraudSuspected>()("FraudSuspected", {
  score: Schema.Finite,
}) {}

export class ChargeFailed extends Schema.TaggedError<ChargeFailed>()("ChargeFailed", {
  reason: Schema.Union([CardExpired, InsufficientFunds, FraudSuspected]),
}) {}

// Handle one reason; the optional last handler covers the remaining reasons
export const outcome = chargeCard(payment).pipe(
  Effect.catchReason(
    "ChargeFailed",
    "InsufficientFunds",
    (r) => Effect.succeed(`short by ${r.shortBy}`),
    (r) => Effect.succeed(`charge failed: ${r._tag}`),
  )
)

// Or lift the reasons into the error channel and handle them like any other tags
export const lifted = chargeCard(payment).pipe(
  Effect.unwrapReason("ChargeFailed"),
  Effect.catchTag(["CardExpired", "InsufficientFunds"], () => Effect.succeed("ask for another card")),
) // FraudSuspected still in E
```

`Effect.catchReasons("ChargeFailed", { CardExpired: ..., FraudSuspected: ... })` handles several reasons at once.

## Expected errors vs defects — the taxonomy

This distinction keeps the error channel meaningful. Decide it per error, deliberately:

| | Expected error (error channel) | Defect (`Effect.die`) |
|---|---|---|
| What | Anticipated domain/infra outcome a caller might handle | Bug or broken invariant; unrecoverable |
| Examples | `UserNotFound`, `PaymentDeclined`, `RateLimited`, validation failure | Impossible state reached, config missing *after* startup validation, programmer error |
| Appears in `E`? | Yes, typed | No — invisible to the type, crashes the fiber |
| Handling | `catchTag`/`catchTags`/`catchReason`/`match` | Don't catch (except top-level logging); fix the bug |

```ts
import { Effect } from "effect"

// Convert an "impossible" expected error into a defect at the point you *know* it can't happen:
const currentUser = Effect.fn("currentUser")(function* (sessionUserId: UserId) {
  return yield* getUser(sessionUserId).pipe(Effect.orDie) // session guarantees existence
})
```

Never use `orDie`/`die` to avoid designing an error type — only to assert a locally-proven invariant.

## Handling errors — where and how

Handle errors **where you have the context to do something meaningful**, usually at the workflow edge (HTTP handler, CLI command, queue consumer). Mid-pipeline code should just let errors flow past.

```ts
import { Effect } from "effect"
import { HttpServerResponse } from "effect/http"

// Handle specific tracks; unhandled tags remain in the type — the compiler tracks what's left
const response = placeOrder(input).pipe(
  Effect.map((order) => HttpServerResponse.jsonUnsafe(order, { status: 201 })),
  Effect.catchTags({
    OrderNotFound: (e) => Effect.succeed(HttpServerResponse.jsonUnsafe({ orderId: e.orderId }, { status: 404 })),
    InsufficientStock: (e) => Effect.succeed(HttpServerResponse.jsonUnsafe(e, { status: 409 })),
    // PaymentDeclined intentionally not caught here → still in E, handled by caller
  })
)

// Same recovery for several tags: pass an array to catchTag
const lenient = loadCart(cartId).pipe(
  Effect.catchTag(["CartExpired", "CartNotFound"], () => Effect.succeed(emptyCart))
)

const describePricing = Effect.fn("describePricing")(function* (order: Order) {
  // Exhaustive fold when you must produce a value either way
  const summary = yield* priceOrder(order).pipe(
    Effect.match({
      onFailure: (e) => `pricing failed: ${e._tag}`,
      onSuccess: (p) => `total ${p.total}`,
    })
  )

  // Recover with a fallback — only when the fallback is genuinely correct, not to silence the type.
  // Tap before catching: once caught, the error is no longer on the track to observe.
  const config = yield* fetchRemoteConfig.pipe(
    Effect.tapErrorTag("ConfigServiceUnavailable", () => Effect.logWarning("using default config")),
    Effect.catchTag("ConfigServiceUnavailable", () => Effect.succeed(defaultConfig))
  )

  return { summary, config }
})
```

Forbidden: `Effect.catch(() => Effect.succeed(fallback))` — it swallows every current *and future* error silently. Catch tags you can name; let the rest propagate.

## Wrapping the outside world (interop edge)

Third-party promise/throwing APIs get wrapped exactly once, in the infrastructure layer, with a typed error:

```ts
import { Effect, Schema } from "effect"

export class StripeError extends Schema.TaggedError<StripeError>()("StripeError", {
  cause: Schema.Defect(),
  retriable: Schema.Boolean,
}) {}

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
import { Effect, Schema } from "effect"

// Schema: report every issue, not just the first
const decodeOrderInput = Schema.decodeUnknownEffect(OrderInput, { errors: "all" })

const runChecks = Effect.fn("runChecks")(function* (order: Order, items: ReadonlyArray<Item>) {
  // Independent checks (each: (order: Order) => Effect<void, OrderIssue>): collect all failures
  yield* Effect.validate(
    [checkInventory, checkAddress, checkFraud],
    (check) => check(order),
    { discard: true }
  ) // fails with NonEmptyArray<OrderIssue>

  // Partition successes/failures without failing at all (batch jobs). Order is [successes, failures].
  const [successes, failures] = yield* Effect.partition(items, processItem, { concurrency: 8 })
  yield* Effect.logInfo(`processed ${successes.length}, failed ${failures.length}`)
  return successes
})
```

Rule of thumb: accumulate at input boundaries and in batch processing; fail fast inside sequential business workflows.

## Retry belongs on the error track

Transient failures are handled by policy, not by hand-rolled loops — see `production.md` for full resilience patterns:

```ts
import { Effect, Schedule } from "effect"

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

- [ ] Every error is a `Schema.TaggedError` (or, in-process only, `Data.TaggedError`) with a descriptive tag and useful fields
- [ ] `E` in every public signature is a union of named errors — never `Error`, `unknown`, or `never` (unless truly infallible)
- [ ] One failure mode with several causes → one error with a tagged `reason`, handled with `catchReason`/`unwrapReason`
- [ ] Expected-vs-defect decided per error; `orDie` only on proven invariants
- [ ] Infra errors translated to domain errors at service boundaries
- [ ] Errors handled at workflow edges with `catchTag`/`catchTags`/`match`; no blanket `Effect.catch`
- [ ] Boundary validation accumulates errors; workflows fail fast
- [ ] All `tryPromise` wrappers thread the `AbortSignal`
