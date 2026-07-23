# effect-fp-skill — an Effect-First Functional TypeScript Skill

[![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-d97757?logo=anthropic&logoColor=white)](https://code.claude.com/docs/en/skills)
[![Effect](https://img.shields.io/badge/Effect-3.22%2B-black)](https://effect.website)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x_strict-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Paradigm](https://img.shields.io/badge/paradigm-functional-8A2BE2)](https://fsharpforfunandprofit.com/series/thinking-functionally/)
[![ROP](https://img.shields.io/badge/errors-railway--oriented-orange)](https://fsharpforfunandprofit.com/rop/)
[![Types](https://img.shields.io/badge/illegal_states-unrepresentable-success)](https://fsharpforfunandprofit.com/series/designing-with-types/)
[![Testing](https://img.shields.io/badge/testing-property--based-blueviolet)](https://fsharpforfunandprofit.com/series/property-based-testing/)
[![any](https://img.shields.io/badge/any-banned-red)](#hard-rules)
[![throw](https://img.shields.io/badge/throw-banned-red)](#hard-rules)
[![null](https://img.shields.io/badge/null-Option%3CA%3E-yellow)](#hard-rules)
[![References](https://img.shields.io/badge/references-12_files-informational)](#whats-inside)
[![Inspired by](https://img.shields.io/badge/inspired_by-F%23_for_Fun_and_Profit-306998)](https://fsharpforfunandprofit.com)
[![License](https://img.shields.io/badge/license-CC_BY_4.0-lightgrey)](LICENSE)

A [Claude Code skill](https://code.claude.com/docs/en/skills) that makes an AI coding agent build **every** TypeScript application — backend services, CLIs, full-stack apps, libraries — on top of [Effect](https://effect.website), applying functional programming end-to-end: railway-oriented error handling, domain modeling with types, capability-based dependency injection, property-based testing, and a production-hardening checklist that is treated as the definition of done.

> **Proof repo:** [effect-fp-skill-examples](https://github.com/mikezupper/effect-fp-skill-examples) — three working apps built strictly with this skill (a URL shortener, a full commerce API with atomic checkout, and a Lit SSR storefront), with CI-verified test suites. Bugs found building them were fed back into these references.

The design philosophy is Scott Wlaschin's ([F# for Fun and Profit](https://fsharpforfunandprofit.com)) — this skill is the opinionated mapping of that body of work onto its TypeScript-native realization, Effect.

---

## Table of contents

- [Motivation](#motivation)
- [Principles](#principles)
- [Hard rules](#hard-rules)
- [The Wlaschin → Effect mapping](#the-wlaschin--effect-mapping)
- [What's inside](#whats-inside)
- [Installation](#installation)
- [How the skill works](#how-the-skill-works)
- [How to best leverage it](#how-to-best-leverage-it)
- [Version policy](#version-policy)
- [Sources & credits](#sources--credits)

---

## Motivation

AI agents write plausible TypeScript by default: `async/await`, `try/catch`, `null` checks, `any` when the types get hard, `console.log` for observability, and a happy path that works in the demo and falls over in production. Every one of those defaults is a place where correctness leaks out of the type system and into runtime hope.

The functional-programming tradition — and Wlaschin's F# writing in particular — solved these problems decades ago:

- **Errors as values on a typed track** instead of invisible exceptions ([Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/))
- **Types that make illegal states unrepresentable** instead of validation scattered through the codebase ([Designing with Types](https://fsharpforfunandprofit.com/series/designing-with-types/))
- **Parse, don't validate** — decode untrusted data once at the boundary into rich domain types
- **A functional core with effects at the edges** ([A Recipe for a Functional App](https://fsharpforfunandprofit.com/series/a-recipe-for-a-functional-app/))
- **Properties, not examples** — test the invariants the code must hold for *all* inputs ([Property-Based Testing](https://fsharpforfunandprofit.com/series/property-based-testing/))

Effect brings all of this to TypeScript as one coherent system: `Effect<Success, Error, Requirements>` is the two-track railway with a third track for typed dependencies, `Schema` is the parse-don't-validate boundary, `Layer` is the functional app recipe's wiring, and the built-in `Schedule`/`Metric`/`Tracer`/`Stream`/`Scope` modules are what turn "correct" into "production-ready and built to scale."

An agent doesn't apply any of this unless you make it. This skill makes it — with hard rules, decision tables, anti-pattern lists, verified API examples, per-area checklists, and a mandatory self-review pass.

## Principles

1. **Railway-oriented programming.** Every operation that can fail returns `Effect<A, E, R>`. Errors are tagged values, never exceptions. Compose the happy path; handle failures explicitly where there's context to do so.
2. **Make illegal states unrepresentable.** Branded types, tagged unions, `Option`. If the compiler accepts it, it should be valid.
3. **Parse, don't validate.** `Schema.decodeUnknown` exactly once at each boundary (HTTP, DB, env, queue, file). The core only ever sees domain types.
4. **Functional core, managed shell.** Pure domain → Effect workflows → infrastructure behind services (`Context.Tag`) wired with `Layer`. Exactly one runtime entry point.
5. **Totality.** Functions handle every input in their type. No partial functions, no `throw`, no `any`, no silent `undefined`.

## Hard rules

The skill's non-negotiables, enforced three ways: stated in `SKILL.md`, lintable via the ESLint config in `references/scaffold.md`, and audited by the greps in `references/code-review.md`.

| Banned | Instead |
|---|---|
| `async/await`, `.then()` | `Effect.gen`, `pipe` |
| `throw`, `try/catch` | Tagged errors on the error channel; `Effect.try*` at the interop edge only |
| `null` / `undefined` in domain types | `Option<A>` |
| Boolean state flags | `Data.TaggedEnum` unions with per-state data |
| `as` casts on external data | `Schema.decodeUnknown` |
| `process.env` | `Config` (secrets as `Config.redacted`) |
| `Effect.runPromise` mid-codebase | One `runMain` / `ManagedRuntime` entry point |
| `Date.now()`, `Math.random()` in domain | `Clock` / `Random` / `DateTime` services (testable) |
| `console.log` | `Effect.log*` with annotations |
| External utility libraries (lodash, etc.) | Effect's `Array`, `Record`, `Option`, `String`, `Order`, … |
| Unbounded concurrency | Explicit `{ concurrency: n }`, bounded queues, `Stream` |
| Mocking libraries | Test `Layer`s (fakes as values) |

## The Wlaschin → Effect mapping

| F# for Fun and Profit concept | Effect realization |
|---|---|
| ROP / two-track `Result` | `Effect<A, E, R>` typed error channel; `Data.TaggedError`; `catchTag`/`catchTags` |
| Designing with types | `Schema.brand`, `Data.TaggedEnum`, `Option`, smart constructors via schemas |
| Parse, don't validate | `Schema` at every boundary; wire shape ≠ domain shape |
| Recipe for a functional app | Onion architecture; services via `Context.Tag`; wiring via `Layer`; one `runMain` |
| Commands in, events out | Workflows return `ReadonlyArray<DomainEvent>`; edges dispatch |
| Property-based testing | `@effect/vitest` + FastCheck; `Arbitrary.make(schema)` derives generators from the same schemas that validate |
| Thinking functionally | Immutability, totality, composition; `pipe` + Effect data modules |

One schema definition = the type + the validator + the JSON codec + the test generator. That single fact carries most of the quality story.

## What's inside

```
effect-fp-skill/
├── SKILL.md                        # entry point: philosophy, hard rules, decision table,
│                                   # anti-patterns, build workflow, reference index
└── references/
    ├── scaffold.md                 # create-effect-app, pinned versions, strict tsconfig,
    │                               # @effect/language-service, ESLint rule enforcement
    ├── domain-types.md             # brands, Schema.Class, tagged unions, Option,
    │                               # state transitions, commands-in/events-out
    ├── rop-errors.md               # tagged errors, expected-vs-defect taxonomy,
    │                               # accumulation vs fail-fast, interop wrapping
    ├── pattern-matching.md         # Match module, exhaustiveness discipline
    ├── services-layers.md          # Context.Tag / Effect.Service, Layer composition,
    │                               # Config & secrets, entry-point discipline, test layers
    ├── database.md                 # @effect/sql: repositories, fiber-propagated transactions,
    │                               # Model variants, SqlResolver batching, migrations, testcontainers
    ├── concurrency.md              # fibers, Queue/PubSub/Deferred/Semaphore/RateLimiter recipes
    ├── testing.md                  # @effect/vitest, TestClock, property patterns, error-track tests
    ├── production.md               # observability, resilience, resource safety, scale patterns,
    │                               # definition-of-done checklist
    ├── app-shapes.md               # HttpApi servers, @effect/cli, full-stack monorepo, libraries
    ├── code-review.md              # mandatory self-review: greps, audits, checklist sweep
    └── supplemental-ai.md          # optional: @effect/ai integration principles
```

Every reference ends in a checklist. All API examples were verified against the current stable releases (see [Version policy](#version-policy)).

## Installation

A skill is just a folder; installation is a copy.

**Global (all projects):**

```bash
git clone https://github.com/mikezupper/effect-fp-skill ~/.claude/skills/effect-fp-skill
rm -rf ~/.claude/skills/effect-fp-skill/.git   # keep the skill folder plain content
```

**Per-project:**

```bash
git clone https://github.com/mikezupper/effect-fp-skill <your-project>/.claude/skills/effect-fp-skill
rm -rf <your-project>/.claude/skills/effect-fp-skill/.git
```

Already have the repo checked out? A plain copy works the same: `cp -r effect-fp-skill ~/.claude/skills/effect-fp-skill` (minus the `.git`).

Then start a new Claude Code session. The skill's `description` frontmatter makes it trigger automatically whenever the agent creates or modifies a TypeScript app; you can also invoke it explicitly with `/effect-fp-skill`. To update later, re-clone (or `git pull` in your checkout and re-copy).

## How the skill works

The skill uses **progressive disclosure**, which is why it's a folder rather than one big file:

1. `SKILL.md` (~150 lines) is what loads when the skill triggers. It carries the philosophy, the hard rules, a decision table ("situation → tool"), the anti-pattern list, and a 9-step build workflow.
2. Each workflow step points at a **reference file** the agent reads only when working in that area — designing errors loads `rop-errors.md`, touching persistence loads `database.md`, and so on. Deep guidance without diluting attention.
3. The final step is **mandatory self-review** (`code-review.md`): mechanical greps for banned constructs, a dependency-direction audit, error-channel and type-design audits, and a sweep of every checklist touched during the task — before the agent may declare the work done.

## How to best leverage it

**Prompting**

- Just ask for the app — "build a URL-shortener API with Postgres" is enough; the skill supplies the architecture. You don't need to say "use Effect" or "use FP."
- Say **"production-ready"** or **"built to scale"** when you mean it — that pulls the full `production.md` checklist (spans, metrics, retries, timeouts, graceful shutdown, load-path review) into scope as acceptance criteria rather than suggestions.
- Name the app shape when it's ambiguous ("as a CLI", "as a library") so the right `app-shapes.md` section drives the scaffold.
- For persistence-heavy work, mention the database up front — transaction boundaries are an architectural decision the skill places at the workflow layer, and it's cheaper to get that right from the first file.

**Reviewing the output**

- The agent must run the `code-review.md` pass itself, but the same file is a great *human* review script — run the greps yourself in CI.
- Pair with `/code-review` (or `ultrareview` for big changes) for an independent adversarial pass; this skill biases construction, a reviewer biases destruction.
- Adopt the ESLint rules from `references/scaffold.md` in your repos so the hard rules hold even for code written without the skill.

**Customizing**

- The rules are opinions — edit them. Relax the strictness tier, swap Postgres guidance for your store, add house naming conventions. Keep the structure: hard rules + decision tables + anti-patterns + checklists is the format agents follow best.
- Add project-specific layers as a *project* skill that references this one, rather than forking it per repo.
- If you build LLM features, read `supplemental-ai.md` first — it's deliberately principle-level; have the agent verify current `@effect/ai` APIs at implementation time.

**Maintaining**

- The skill pins its knowledge to the Effect 3.x line. On major ecosystem moves (Effect 4.0 going stable), re-run an API-verification pass against the docs and update the reference files — the version notes at the top of each file mark what to check.

## Version policy

- **Target: Effect 3.x stable.** All API examples verified against `effect` 3.22.0 and the companion releases current at the time of writing (`@effect/platform` 0.97.x, `@effect/cli` 0.76.x, `@effect/sql` 0.52.x, `@effect/vitest` 0.30.x).
- Effect **4.0 is in beta** and restructures packages (platform/cli merge into core). The skill instructs agents not to use beta versions unless explicitly asked.
- Companion packages are 0.x and version-coupled to `effect`: pin them, upgrade them as a family.
- `HttpApi` and the other Http modules are officially flagged unstable; their canonical docs are the npm-published package READMEs.

## Sources & credits

- **Scott Wlaschin — [F# for Fun and Profit](https://fsharpforfunandprofit.com)**: [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) · [A Recipe for a Functional App](https://fsharpforfunandprofit.com/series/a-recipe-for-a-functional-app/) · [Designing with Types](https://fsharpforfunandprofit.com/series/designing-with-types/) · [Property-Based Testing](https://fsharpforfunandprofit.com/series/property-based-testing/) · [FP Patterns](https://fsharpforfunandprofit.com/fppatterns/) · [Thinking Functionally](https://fsharpforfunandprofit.com/series/thinking-functionally/)
- **[Effect](https://effect.website)** — docs, package READMEs, and published type declarations (the API-verification sources)
- **[Claude Code skills](https://code.claude.com/docs/en/skills)** — the skill format and progressive-disclosure model

*This skill encodes one team's opinionated synthesis; neither Scott Wlaschin nor the Effect team endorses it.*

## License

Text and markup licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) © 2026 Mike Zupper.
