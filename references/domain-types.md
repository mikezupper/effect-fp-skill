# Designing with Types

From [Designing with Types](https://fsharpforfunandprofit.com/series/designing-with-types/) and [Parse, don't validate]: the type system is the first line of defense. Time spent here eliminates whole test categories and whole classes of production bugs. **Always model types before writing logic.**

## Branded primitives — no naked strings/numbers in the domain

Every id, email, money amount, quantity, etc. gets a brand. Passing a `UserId` where an `OrderId` goes must not compile.

```ts
import { Schema } from "effect"

export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const Email = Schema.String.pipe(
  Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/),
  Schema.brand("Email")
)
export type Email = typeof Email.Type

// Money as integer minor units — never floats for currency
export const Cents = Schema.Int.pipe(Schema.nonNegative(), Schema.brand("Cents"))
export type Cents = typeof Cents.Type
```

The schema **is** the smart constructor — one definition gives you: the type, runtime validation, JSON codec, and (in tests) an arbitrary generator.

```ts
// Untrusted data → Effect with typed ParseError (boundary)
const email = yield* Schema.decodeUnknown(Email)(raw)

// Literal you can vouch for (tests, constants) → decodeSync is acceptable
const admin = Schema.decodeSync(Email)("admin@example.com")
```

## Structs: schemas define domain records

```ts
export class User extends Schema.Class<User>("User")({
  id: UserId,
  email: Email,
  name: Schema.NonEmptyTrimmedString,
  createdAt: Schema.DateTimeUtc,
  deactivatedAt: Schema.OptionFromNullOr(Schema.DateTimeUtc), // Option in the domain, null on the wire
}) {}
```

- `Schema.Class` gives an immutable, structurally-equal (`Equal.equals`) domain type + schema in one.
- Transform on the way in: `Schema.trimmed`, `Schema.lowercased`, `Schema.DateTimeUtc` (string ⇄ DateTime), `OptionFromNullOr` (null ⇄ Option; sibling `OptionFromNullishOr(value, onNoneEncoding)` takes a **second** argument choosing `null` vs `undefined` on encode). The wire shape and the domain shape are different types, converted by the schema — that's the point.
- Separate schemas per boundary when shapes differ: `UserRow` (DB), `UserResponse` (API), `User` (domain). Don't force one schema to serve all three.

## Make illegal states unrepresentable

Replace flag combinations and optional-field soup with tagged unions. Each state carries exactly the data valid in that state.

```ts
import { Data } from "effect"

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
const describe = OrderState.$match({
  Draft:     ({ items }) => `draft with ${items.length} items`,
  Placed:    ({ placedAt }) => `placed ${placedAt}`,
  Paid:      ({ receipt }) => `paid, receipt ${receipt}`,
  Shipped:   ({ trackingCode }) => `shipped: ${trackingCode}`,
  Cancelled: ({ reason }) => `cancelled: ${reason}`,
})
```

For unions that cross a boundary (persistence, API), use `Schema.TaggedStruct` + `Schema.Union` instead so the union is also serializable.

## Recursive schemas

Recursion needs `Schema.suspend` plus an explicit type annotation, and — learned the hard way — an **`identifier` annotation, or OpenAPI/JSON-Schema generation dies at runtime** (`Missing annotation ... requires an "identifier"`):

```ts
export interface CategoryTree {
  readonly id: string
  readonly name: string
  readonly children: ReadonlyArray<CategoryTree>
}
export const CategoryTree: Schema.Schema<CategoryTree> = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  children: Schema.Array(Schema.suspend(() => CategoryTree)),
}).annotations({ identifier: "CategoryTree" }) // REQUIRED for HttpApiSwagger/OpenApi
```

Tip: keep recursive *response* shapes as plain encoded types (`Type = Encoded`, no brands) — the recursive annotation then needs only one type parameter. Pure functions that traverse recursive structures must be **total**: handle self-reference and cycles (a `visited` set) rather than assuming well-formed input — and property-test that every input node appears in the output exactly once.

## State transitions as functions

Transitions are total functions between state types; invalid transitions are unrepresentable or return errors — never runtime surprises:

```ts
// Only a Placed order can be paid — enforced by the parameter type, not a runtime check
const markPaid = (
  order: Extract<OrderState, { _tag: "Placed" }>,
  receipt: ReceiptId,
  paidAt: DateTime.Utc
): Extract<OrderState, { _tag: "Paid" }> =>
  OrderState.Paid({ items: order.items, paidAt, receipt })
```

When the input state isn't statically known, return `Effect`/`Either` with a typed `InvalidTransition` error.

## Option, not null

```ts
import { Option } from "effect"

const findUser: (id: UserId) => Effect.Effect<Option.Option<User>, DbError, Db>

// Consuming
const greeting = Option.match(maybeUser, {
  onNone: () => "hello, guest",
  onSome: (u) => `hello, ${u.name}`,
})

// "Absent means error" conversions at the workflow level:
const user = yield* findUser(id).pipe(
  Effect.flatMap(Option.match({
    onNone: () => Effect.fail(new UserNotFound({ userId: id })),
    onSome: Effect.succeed,
  }))
)
```

Repositories return `Option` (absence is a normal outcome there); workflows decide whether absence is an error.

## Commands in, events out

Wlaschin's workflow pattern: a use-case takes a **command** and returns the **events** that happened — side effects are driven by the events at the edge, not buried inside the workflow. This keeps workflows pure-ish, testable ("given this command, exactly these events"), and makes adding subscribers (email, analytics, audit log) a change at the edge, not in the workflow.

```ts
export type OrderEvent = Data.TaggedEnum<{
  OrderPlaced:        { readonly order: Order }
  PaymentTaken:       { readonly receipt: ReceiptId; readonly amount: Cents }
  LowStockDetected:   { readonly sku: Sku; readonly remaining: number }
}>
export const OrderEvent = Data.taggedEnum<OrderEvent>()

// Workflow: command → Effect<events, errors, deps>. It DECIDES; it does not notify/email/log-to-audit.
const placeOrder: (cmd: PlaceOrderCommand) =>
  Effect.Effect<ReadonlyArray<OrderEvent>, PlaceOrderError, OrderRepo | PaymentGateway>

// Edge: publish/dispatch events — the only place that knows who cares about what
yield* Effect.forEach(events, publishDomainEvent, { concurrency: 1 })
```

Events are facts, named in past tense, carrying the data subscribers need. If events cross a process boundary (queue, outbox table), define them with `Schema.TaggedStruct` so they serialize.

## Constrain collections and numbers

- "At least one" → `NonEmptyReadonlyArray` / `Schema.NonEmptyArray`
- Bounded quantities → `Schema.Int.pipe(Schema.between(1, 100), Schema.brand("Quantity"))`
- Sets of unique things → `Schema.Set` / model uniqueness in the type, don't check it in five places

## Checklist

- [ ] Zero naked `string`/`number` in domain signatures — everything branded
- [ ] Every boundary has a schema; decoding happens exactly once per boundary
- [ ] Wire types (null, ISO strings) ≠ domain types (Option, DateTime) — schemas transform between them
- [ ] No boolean state flags; state machines are tagged unions with per-state data
- [ ] All matches exhaustive (`$match` / `Match.exhaustive`) — no `default` branches that hide new cases
- [ ] `Option` for absence everywhere; `null` only inside wire schemas
- [ ] Types written and reviewed before workflow logic
