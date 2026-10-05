# Exhaustive Pattern Matching with `Match`

`if/else` chains and `switch` statements over domain variants are replaced by the `Match` module, by `$match` from `Data.taggedEnum`, or by `.match` from `Schema.TaggedUnion`. The goal is always the same: **when a new variant is added, every place that inspects the type fails to compile until it handles the new case.** That compile error is the feature. Never suppress it with a default branch.

Verified against effect 4.0.1 (October 2026).

## Matching tagged unions (the common case)

```ts
import { Match, Option } from "effect"

// Build a matcher for a TYPE (reusable function)
export const shippingCost = Match.type<OrderState>().pipe(
  Match.tag("Draft", () => Option.none()),
  Match.tag("Placed", "Paid", ({ items }) => Option.some(costFor(items))),  // multiple tags, one handler
  Match.tag("Shipped", "Cancelled", () => Option.none()),
  Match.exhaustive                       // ← compile error if any tag is unhandled
)

// Match a VALUE directly (one-off)
export const label = Match.value(state).pipe(
  Match.tag("Cancelled", ({ reason }) => `cancelled: ${reason}`),
  Match.orElse(() => "active")           // orElse only when a true catch-all is intended
)

// Object form: one handler per tag, exhaustive by construction
export const isOpen = Match.type<OrderState>().pipe(
  Match.tagsExhaustive({
    Draft: () => true,
    Placed: () => true,
    Paid: () => true,
    Shipped: () => false,
    Cancelled: () => false,
  })
)
```

For `Data.taggedEnum` types, prefer the generated `$match`, and for `Schema.TaggedUnion` schemas the generated `.match`. Both are exhaustive by construction. Reach for `Match` when you match on other shapes, combine predicates, or build reusable matchers.

## Beyond tags

```ts
import { Match } from "effect"

// Predicates and structural patterns
export const describe = Match.type<number | string | { code: number }>().pipe(
  Match.when(Match.number, (n) => `number ${n}`),
  Match.when(Match.string, (s) => `string ${s}`),
  Match.when({ code: 404 }, () => "not found"),          // literal-field pattern
  Match.when({ code: Match.number }, ({ code }) => `code ${code}`),
  Match.exhaustive
)

// Guard-style refinement (a plain predicate does not narrow, so close with orElse/option)
export const greet = Match.type<User>().pipe(
  Match.when((u: User) => isAdmin(u), (u) => `welcome back, admin ${u.name}`),
  Match.orElse((u) => `hi ${u.name}`)
)

// Different discriminant field than _tag
type Shape =
  | { readonly kind: "circle"; readonly radius: number }
  | { readonly kind: "square"; readonly side: number }

export const area = Match.type<Shape>().pipe(
  Match.discriminator("kind")("circle", (c) => Math.PI * c.radius ** 2),
  Match.discriminator("kind")("square", (s) => s.side ** 2),
  Match.exhaustive
)
```

Other members of the family: `Match.discriminatorsExhaustive("kind")({...})` is the object form for a custom discriminant, `Match.tagStartsWith` matches namespaced tags, `Match.not` adds negative patterns, and `Match.valueTags(value, {...})` is a one-shot exhaustive match on a value.

## Closing a matcher — pick deliberately

| Closer | Returns | Use when |
|---|---|---|
| `Match.exhaustive` | `A` | Default. All cases must be handled — compiler-enforced |
| `Match.orElse(f)` | `A` | A genuine catch-all is part of the design (rare; justify in a comment) |
| `Match.option` | `Option<A>` | Partial match where "no match" is a normal outcome |
| `Match.result` | `Result<A, Unmatched>` | Partial match where you'll handle the miss explicitly. The failure side carries the *unmatched input*, narrowed to the remaining cases (`Match.either` in Effect 3) |

## Rules

- Domain variant inspection goes through `$match`, `TaggedUnion.match`, or `Match.tag` + `Match.exhaustive`. Never `switch (x._tag)` with a `default`, never `if (x._tag === ...)` ladders.
- A matcher used in more than one place becomes a named `Match.type<T>()` function next to the type definition.
- `orElse` on a domain union is a smell, because it silently absorbs future variants. If two cases share behavior, list both tags in one `Match.tag("A", "B", handler)` instead.
- Conditionals on plain booleans or numbers that pick between *behaviors* of a domain concept usually mean a union type is missing. Fix the model, not the branch.

## Checklist

- [ ] Every inspection of a domain union is exhaustive (`$match` / `.match` / `Match.exhaustive` / `Match.tagsExhaustive`)
- [ ] No `switch`/`if` ladders on `_tag`; no `default` branches
- [ ] Every `Match.orElse` on a domain union has a comment justifying the catch-all
- [ ] Partial matches use `Match.option` / `Match.result`, not `orElse(() => undefined)`
- [ ] Reused matchers are named `Match.type<T>()` functions colocated with the type
