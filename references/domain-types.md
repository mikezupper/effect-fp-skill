# Designing with Types

From [Designing with Types](https://fsharpforfunandprofit.com/series/designing-with-types/) and [Parse, don't validate]: the type system is the first line of defense. Time spent here eliminates whole test categories and whole classes of production bugs. **Always model types before writing logic.**

Verified against effect 4.0.1 (October 2026).

## Branded primitives — no naked strings/numbers in the domain

Every id, email, money amount, quantity, etc. gets a brand. Passing a `UserId` where an `OrderId` goes must not compile.

```ts
import { Schema } from "effect"

export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const Email = Schema.String.pipe(
  Schema.check(Schema.isPattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  Schema.brand("Email")
)
export type Email = typeof Email.Type

// Money as integer minor units — never floats for currency
export const Cents = Schema.Int.pipe(
  Schema.check(Schema.isGreaterThanOrEqualTo(0)),
  Schema.brand("Cents")
)
export type Cents = typeof Cents.Type
```

In Effect 4, validation rules are **checks**: `Schema.check(Schema.isPattern(...), ...)` in a pipe, or the `.check(...)` method on any schema. They are all named `is*` (`isPattern`, `isGreaterThanOrEqualTo`, `isBetween`, `isMinLength`, `isNonEmpty`, `isTrimmed`, `isLowercased`, `isUUID`, ...). `Schema.brand` **only** changes the TypeScript type. It adds no runtime check, so the checks that justify the brand have to come before it. Ready-made refinements are also available: `Schema.Int`, `Schema.Natural` (non-negative int), `Schema.NonEmptyString`, `Schema.Trimmed`.

Gotcha: write `isPattern` regexes **without flags**. Use `/^[A-Za-z]+$/`, not `/^[a-z]+$/i`. Validation still works with flags, but the Arbitrary derived from the schema ignores them. Property tests then fail with `Property exhausted after 0 run(s) and 1001 discard(s)`.

The schema **is** the smart constructor. One definition gives you the type, runtime validation, the codec, and (in tests) a property-test generator.

```ts
import { Effect, Schema } from "effect"

// Untrusted data → Effect failing with a typed SchemaError (boundary).
// Build the decoder once, reuse it at the edge.
export const decodeEmail = Schema.decodeUnknownEffect(Email)

export const program = Effect.gen(function* () {
  const email = yield* decodeEmail(raw)
  return email
})

// Literal you can vouch for (tests, constants) → make / decodeSync (throw on invalid)
const admin = Email.make("admin@example.com")
const support = Schema.decodeSync(Email)("support@example.com")
```

Decoder family: `decodeUnknownEffect` at boundaries inside Effect code. `decodeUnknownResult`, `decodeUnknownOption` and `decodeUnknownExit` are for pure code that has to branch. `decodeUnknownSync`, `decodeSync` and `schema.make(...)` throw, so use them only on values you vouch for. `schema.makeOption` / `schema.makeEffect` are non-throwing constructors.

## Structs: schemas define domain records

```ts
import { Schema, SchemaTransformation } from "effect"

export class User extends Schema.Class<User>("User")({
  id: UserId,
  email: Email,
  name: Schema.Trim.check(Schema.isNonEmpty()),                         // trims on decode, rejects blank
  createdAt: Schema.DateTimeUtcFromString,                              // ISO string ⇄ DateTime.Utc
  deactivatedAt: Schema.OptionFromNullOr(Schema.DateTimeUtcFromString) // Option in the domain, null on the wire
}) {}

// Normalizing transformation: lowercase on the way in
export const Username = Schema.String.pipe(
  Schema.decode(SchemaTransformation.toLowerCase()),
  Schema.brand("Username")
)
```

- `Schema.Class` gives you an immutable, structurally equal (`Equal.equals`) domain type and its schema in one. `User.make({...})` validates the fields; `new User({...})` does too.
- Transform on the way in. `Schema.Trim` (decode trims; the encoded side is any string) differs from `Schema.Trimmed`, which only *checks*. `Schema.decode(SchemaTransformation.toLowerCase())` lowercases; `isLowercased()` only checks. `OptionFromNullOr` maps null ⇄ Option. Its sibling `OptionFromNullishOr(schema, { onNoneEncoding: null })` takes an **options object** that chooses `null` or `undefined` on encode (the default is `undefined`). The wire shape and the domain shape are different types, and the schema converts between them. That's the point.
- **`Schema.DateTimeUtc` is not a string codec in v4.** It is the *type* schema, and its encoded side is also `DateTime.Utc`. Use `Schema.DateTimeUtcFromString` when the wire carries ISO strings (likewise `DateFromString`, `NumberFromString`, `BigIntFromString`). The alternative is to decode through `Schema.toCodecJson(User)`, which derives the JSON codec for type-level schemas. HttpApi and RPC apply that JSON codec for you. Your own `JSON.parse` boundaries don't.
- Separate schemas per boundary when shapes differ: `UserRow` (DB), `UserResponse` (API), `User` (domain). Don't force one schema to serve all three.

## Make illegal states unrepresentable

Replace flag combinations and optional-field soup with tagged unions. Each state carries exactly the data that is valid in that state.

```ts
import { Data, DateTime } from "effect"
import type { NonEmptyReadonlyArray } from "effect/Array"

// ❌ WRONG: 2^3 flag combinations, most meaningless; paidAt "sometimes set"
// interface Order { isPaid: boolean; isShipped: boolean; isCancelled: boolean; paidAt?: Date }

// ✅ Each state is its own shape
export type OrderState = Data.TaggedEnum<{
  Draft:     { readonly items: ReadonlyArray<LineItem> }
  Placed:    { readonly items: NonEmptyReadonlyArray<LineItem>; readonly placedAt: DateTime.Utc }
  Paid:      { readonly items: NonEmptyReadonlyArray<LineItem>; readonly paidAt: DateTime.Utc; readonly receipt: ReceiptId }
  Shipped:   { readonly trackingCode: TrackingCode; readonly shippedAt: DateTime.Utc }
  Cancelled: { readonly reason: CancellationReason; readonly cancelledAt: DateTime.Utc }
}>
export const OrderState = Data.taggedEnum<OrderState>()

// Exhaustive matching — adding a state breaks every match until handled (this is a feature)
export const describe = OrderState.$match({
  Draft:     ({ items }) => `draft with ${items.length} items`,
  Placed:    ({ placedAt }) => `placed ${DateTime.formatIso(placedAt)}`,
  Paid:      ({ receipt }) => `paid, receipt ${receipt}`,
  Shipped:   ({ trackingCode }) => `shipped: ${trackingCode}`,
  Cancelled: ({ reason }) => `cancelled: ${reason}`,
})
```

For unions that cross a boundary (persistence, API, queue), use `Schema.TaggedUnion` so the union is also decodable, encodable and generatable. It comes with an exhaustive `match`, `guards` and `cases`:

```ts
export const PaymentMethod = Schema.TaggedUnion({
  Card:   { last4: Schema.String.check(Schema.isPattern(/^\d{4}$/)) },
  Wallet: { provider: Schema.Literals(["apple", "google"]) },
  Invoice:{ dueDays: Schema.Int.check(Schema.isBetween({ minimum: 1, maximum: 90 })) },
})
export type PaymentMethod = typeof PaymentMethod.Type

export const label = (pm: PaymentMethod) =>
  PaymentMethod.match(pm, {
    Card:    ({ last4 }) => `card •••• ${last4}`,
    Wallet:  ({ provider }) => `${provider} pay`,
    Invoice: ({ dueDays }) => `invoice, net ${dueDays}`,
  })
```

(The equivalent long form is `Schema.Union([Schema.TaggedStruct("Card", {...}), ...]).pipe(Schema.toTaggedUnion("_tag"))`. Note that `Schema.Union` takes an **array** in v4.)

## Recursive schemas

Recursion needs `Schema.suspend` plus explicit type annotations, both on the schema constant and on the suspend thunk's return type. Also add an **`identifier` annotation**. Effect 3 crashed OpenAPI/JSON-Schema generation without one (`Missing annotation ... requires an "identifier"`). Effect 4 no longer throws, but it invents a component name such as `Objects_`, and that name leaks into your public OpenAPI document and generated clients:

```ts
export interface CategoryTree {
  readonly id: string
  readonly name: string
  readonly children: ReadonlyArray<CategoryTree>
}
export const CategoryTree: Schema.Codec<CategoryTree> = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  children: Schema.Array(Schema.suspend((): Schema.Codec<CategoryTree> => CategoryTree)),
}).annotate({ identifier: "CategoryTree" }) // stable OpenAPI/JSON-Schema component name
```

Tip: keep recursive *response* shapes as plain encoded types (`Type = Encoded`, no brands). The annotation then needs only one type parameter (`Schema.Codec<T>`, which is `Codec<T, T>`). When the shapes differ, annotate both: `Schema.Codec<CategoryTree, CategoryTreeEncoded>`. Pure functions that traverse recursive structures must be **total**. Handle self-reference and cycles with a `visited` set instead of assuming well-formed input, and property-test that every input node appears in the output exactly once.

## State transitions as functions

Transitions are total functions between state types. Invalid transitions are either unrepresentable or return errors, never runtime surprises:

```ts
// Only a Placed order can be paid — enforced by the parameter type, not a runtime check
export const markPaid = (
  order: Extract<OrderState, { _tag: "Placed" }>,
  receipt: ReceiptId,
  paidAt: DateTime.Utc
): Extract<OrderState, { _tag: "Paid" }> =>
  OrderState.Paid({ items: order.items, paidAt, receipt })
```

When the input state isn't statically known, return `Effect` or `Result` (the v4 name for `Either`) with a typed `InvalidTransition` error.

## Option, not null

```ts
import { Effect, Option, Schema } from "effect"

export class UserNotFound extends Schema.TaggedError<UserNotFound>()("UserNotFound", {
  userId: Schema.String,
}) {}

declare const findUser: (id: UserId) => Effect.Effect<Option.Option<User>, DbError, Db>

// Consuming
export const greeting = Option.match(maybeUser, {
  onNone: () => "hello, guest",
  onSome: (u) => `hello, ${u.name}`,
})

// "Absent means error" conversions at the workflow level:
export const program = Effect.gen(function* () {
  const user = yield* findUser(id).pipe(
    Effect.flatMap(Option.match({
      onNone: () => Effect.fail(new UserNotFound({ userId: id })),
      onSome: Effect.succeed,
    }))
  )
  return user
})
```

Repositories return `Option`, because absence is a normal outcome there. Workflows decide whether absence is an error.

## Commands in, events out

This is Wlaschin's workflow pattern. A use-case takes a **command** and returns the **events** that happened. Side effects are driven by those events at the edge instead of being buried inside the workflow. Workflows stay pure-ish and testable ("given this command, exactly these events"), and adding a subscriber (email, analytics, audit log) becomes a change at the edge, not in the workflow.

```ts
export type OrderEvent = Data.TaggedEnum<{
  OrderPlaced:        { readonly order: Order }
  PaymentTaken:       { readonly receipt: ReceiptId; readonly amount: Cents }
  LowStockDetected:   { readonly sku: Sku; readonly remaining: number }
}>
export const OrderEvent = Data.taggedEnum<OrderEvent>()

// Workflow: command → Effect<events, errors, deps>. It DECIDES; it does not notify/email/log-to-audit.
declare const placeOrder: (cmd: PlaceOrderCommand) =>
  Effect.Effect<ReadonlyArray<OrderEvent>, PlaceOrderError, OrderRepo | PaymentGateway>

// Edge: publish/dispatch events — the only place that knows who cares about what
export const edge = Effect.gen(function* () {
  yield* Effect.forEach(events, publishDomainEvent, { concurrency: 1 })
})
```

Events are facts. Name them in the past tense and give them the data subscribers need. If events cross a process boundary (queue, outbox table), define them with `Schema.TaggedUnion` so they serialize.

## Constrain collections and numbers

- "At least one" → `NonEmptyReadonlyArray` (type, from `effect/Array`) / `Schema.NonEmptyArray(Item)` (schema)
- Bounded quantities → `Schema.Int.pipe(Schema.check(Schema.isBetween({ minimum: 1, maximum: 100 })), Schema.brand("Quantity"))`
- Counts/indexes → `Schema.Natural`
- Sets of unique things → `Schema.ReadonlySet(Item)` or `Schema.UniqueArray(Item)`. Model uniqueness in the type; don't check it in five places.

## Checklist

- [ ] Zero naked `string`/`number` in domain signatures — everything branded
- [ ] Every brand is preceded by the checks that justify it (`brand` alone validates nothing)
- [ ] Every boundary has a schema; decoding happens exactly once per boundary
- [ ] Wire types (null, ISO strings) ≠ domain types (Option, DateTime) — schemas transform between them (`*FromString`, `OptionFromNullOr`, or `Schema.toCodecJson`)
- [ ] No boolean state flags; state machines are tagged unions with per-state data
- [ ] All matches exhaustive (`$match` / `TaggedUnion.match` / `Match.exhaustive`) — no `default` branches that hide new cases
- [ ] Recursive schemas carry an `identifier` annotation
- [ ] `Option` for absence everywhere; `null` only inside wire schemas
- [ ] Types written and reviewed before workflow logic
