# Exhaustive Pattern Matching with `Match`

`if/else` chains and `switch` statements over domain variants are replaced by the `Match` module (or `$match` from `Data.taggedEnum`). The goal is always the same: **when a new variant is added, every place that inspects the type fails to compile until it handles the new case.** That compile error is the feature — never suppress it with a default branch.

## Matching tagged unions (the common case)

```ts
import { Match } from "effect"

// Build a matcher for a TYPE (reusable function)
const shippingCost = Match.type<OrderState>().pipe(
  Match.tag("Draft", () => Option.none()),
  Match.tag("Placed", "Paid", ({ items }) => Option.some(costFor(items))),  // multiple tags, one handler
  Match.tag("Shipped", "Cancelled", () => Option.none()),
  Match.exhaustive                       // ← compile error if any tag is unhandled
)

// Match a VALUE directly (one-off)
const label = Match.value(state).pipe(
  Match.tag("Cancelled", ({ reason }) => `cancelled: ${reason}`,),
  Match.orElse(() => "active")           // orElse only when a true catch-all is intended
)
```

For `Data.taggedEnum` types, prefer the generated `$match` (already exhaustive by construction); reach for `Match` when matching on other shapes, combining predicates, or building reusable matchers.

## Beyond tags

```ts
// Predicates and structural patterns
const describe = Match.type<number | string | { code: number }>().pipe(
  Match.when(Match.number, (n) => `number ${n}`),
  Match.when(Match.string, (s) => `string ${s}`),
  Match.when({ code: 404 }, () => "not found"),          // literal-field pattern
  Match.when({ code: Match.number }, ({ code }) => `code ${code}`),
  Match.exhaustive
)

// Guard-style refinement
Match.when((u: User) => isAdmin(u), handleAdmin)

// Different discriminant field than _tag
Match.discriminator("kind")("circle", (c) => ...)
```

## Closing a matcher — pick deliberately

| Closer | Returns | Use when |
|---|---|---|
| `Match.exhaustive` | `A` | Default. All cases must be handled — compiler-enforced |
| `Match.orElse(f)` | `A` | A genuine catch-all is part of the design (rare; justify in a comment) |
| `Match.option` | `Option<A>` | Partial match where "no match" is a normal outcome |
| `Match.either` | `Either<A, NoSuchElementException>` | Partial match where you'll handle the miss explicitly |

## Rules

- Domain variant inspection: `$match` / `Match.tag` + `Match.exhaustive`. Never `switch (x._tag)` with a `default`, never `if (x._tag === ...)` ladders.
- A matcher used in more than one place becomes a named `Match.type<T>()` function next to the type definition.
- `orElse` on a domain union is a smell — it silently absorbs future variants. If two cases share behavior, list both tags in one `Match.tag("A", "B", handler)` instead.
- Conditionals on plain booleans/numbers that pick between *behaviors* of a domain concept usually mean a missing union type — fix the model, not the branch.
