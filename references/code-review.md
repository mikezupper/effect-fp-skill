# Self-Review Pass — run before declaring any work done

After implementing, review your own output as a hostile reviewer would. Do this **every time**, before reporting completion. Fix everything found, then re-run the pass.

Grep rules target Effect 4.x (verified against effect 4.0.1, October 2026).

## 1. Mechanical sweep — run these greps over `src/`

Every hit is either a violation to fix or a documented, justified exception (interop edge, wire schema). Zero unexplained hits.

```bash
# Banned constructs in application code
grep -rn "async \|await \|\.then(\|\.catch(" src/ --include="*.ts" | grep -v "Effect.tryPromise\|Effect.promise"
grep -rn "throw \|try {" src/ --include="*.ts"
grep -rn "console\.log\|console\.error\|console\.warn" src/ --include="*.ts"
grep -rn "process\.env" src/ --include="*.ts" | grep -v "config.ts"
grep -rn "Effect\.run" src/ --include="*.ts" | grep -v "main.ts\|runtime.ts"     # one entry point only
grep -rn ": any\|as any\|@ts-ignore\|@ts-expect-error" src/ --include="*.ts"
grep -rn "!\." src/domain/ --include="*.ts"                                       # non-null assertions
grep -rn "null\|undefined" src/domain/ --include="*.ts" | grep -v "FromNull"      # Option instead
grep -rn "Date\.now()\|new Date()\|Math\.random()" src/ --include="*.ts"          # Clock/Random/DateTime services
grep -rn "from \"lodash\"" src/ --include="*.ts"                                  # Effect data modules only
```

## 2. Dependency-direction audit

In Effect 4 many former `@effect/platform` modules live in core `"effect"` (`FileSystem`, `Path`, `Stdio`, `Terminal` are capability *tags* — fine in workflows, never in domain). Infrastructure is the `effect/*` subpaths that talk to the outside world (`effect/http`, `effect/http-api`, `effect/sql`, `effect/rpc`, `effect/cluster`, `effect/persistence`, `effect/process`, `effect/socket`, `effect/observability`) plus runtime/driver packages (`@effect/platform-*`, `@effect/sql-*`, `@effect/ai-*`, `@effect/opentelemetry`). `effect/ai` is the exception: `LanguageModel`/`Tool`/`Toolkit` are provider-agnostic capability tags, so workflows may use them (see `supplemental-ai.md`); the provider packages (`@effect/ai-anthropic`, …) are infrastructure.

```bash
# domain/ may import ONLY from "effect" (exact core entry — no subpaths) and "./": anything else here is a violation
grep -rn "^import" src/domain/ --include="*.ts" | grep -v "from \"effect\"\|from \"\./\|from \"\.\./"
# workflows/ may use core capability tags (FileSystem, Path, Clock...) but not concrete infrastructure
grep -rnE "from \"(effect/(http|http-api|sql|rpc|cluster|persistence|process|socket|observability)|@effect/(platform-|sql-|ai-|opentelemetry))" src/workflows/ --include="*.ts"
# runtime packages (@effect/platform-node etc.) belong only in the composition root
grep -rn "@effect/platform-" src/ --include="*.ts" | grep -v "main.ts\|runtime.ts"
# HTTP statuses are an adapter concern: attach them in the API definition, never on domain errors
grep -rn "httpApiStatus\|HttpApiSchema" src/domain/ --include="*.ts"
```

## 2b. Effect 4 audit — no lingering v3 APIs

Effect 4 removed or renamed these; a hit means code (often model-generated from v3-era training data) that will not compile or that uses a deprecated idiom.

```bash
# Discontinued packages (folded into effect subpaths: effect/http, effect/http-api, effect/sql, effect/cli, effect/ai, Schema in core)
grep -rnE "from \"@effect/(platform|cli|sql|ai|schema|rpc|cluster)\"" src/ --include="*.ts"
# Renamed/removed core APIs
grep -rnE "Context\.(Tag|GenericTag)\b|Effect\.(Service|Tag)\b|Effect\.catchAll[A-Za-z]*\b|Effect\.either\b|\bEither\.|Layer\.(scoped|unwrapEffect)\b" src/ --include="*.ts"
grep -rnE "Effect\.(fork|forkDaemon|makeSemaphore|makeLatch|validateAll)\b|Ref\.Synchronized|\bSTM\.|\bTRef\." src/ --include="*.ts"
grep -rnE "Config\.(string|number|integer|boolean|redacted|literal|logLevel|port|duration)\(|Logger\.(json|minimumLogLevel)\b|Metric\.(increment|trackDuration)\b" src/ --include="*.ts"
```

v4 replacements: `Context.Service<Self, Shape>()("app/path/Name")` · `Effect.catch`/`catchCause`/`catchDefect` (v3 `catchAll`/`catchAllCause`/`catchAllDefect`) · `Result`/`Effect.result` · `Layer.effect` · `Layer.unwrap` · `Effect.forkChild`/`forkDetach` · `Semaphore.make`/`Latch.make` · `Effect.validate` · `SynchronizedRef` · `TxRef` + `Effect.tx` · `Config.String`/`Int`/`Redacted`/`LogLevel`… · `Logger.consoleJson` + `References.MinimumLogLevel` · `Metric.update` + `Effect.trackDuration`. Also check semantic changes the compiler won't catch: `Effect.partition` now returns `[successes, failures]` (reversed from v3).

Tooling: enable `@effect/language-service` as a tsconfig plugin (`"plugins": [{ "name": "@effect/language-service" }]`) — its diagnostics flag outdated v3 APIs (`outdatedApi`), floating effects and missing services in the editor. On TypeScript 7 (recommended by effect 4) use `@effect/tsgo`, the native-compiler build of the same tooling.

## 3. Error-channel audit

Read every exported function signature and check:
- [ ] `E` is a union of named tagged errors — no `Error`, `unknown`, `SchemaError` leaking past the boundary, no accidental `never`
- [ ] No `Effect.catch` (v3 `catchAll`) that swallows; every `orDie` has a comment proving the invariant
- [ ] Infra errors are translated to domain errors inside services (no `SqlError` visible in a workflow signature)
- [ ] Errors carry the data a handler needs (ids, retriability), not just messages

## 4. Type-design audit

- [ ] No naked `string`/`number` crossing function boundaries in the domain — branded types
- [ ] No boolean flag pairs encoding states — tagged unions
- [ ] Every `$match`/`Match` pipeline ends in `exhaustive` (no `orElse` hiding unhandled cases without justification)
- [ ] Every boundary decodes with Schema exactly once; zero `as` casts on external data

## 5. Runtime & resource audit

- [ ] Exactly one `runMain` (`NodeRuntime.runMain`)/`ManagedRuntime`; everything else composes
- [ ] No `forkDetach` without a justification comment; dynamic fiber sets use `FiberSet`/`FiberMap`
- [ ] Every resource acquired via `acquireRelease` inside `Layer.effect` (no `Layer.scoped` — removed in v4)
- [ ] Every `Effect.forEach`/`Effect.all` fan-out has explicit `concurrency`
- [ ] Every external call has a timeout; retries are transient-only with backoff+jitter
- [ ] Workflows and I/O service methods are `Effect.fn("Name")` (or `withSpan`); logs are annotated, structured (`Logger.consoleJson`)

## 6. Test audit

- [ ] Pure domain logic has direct unit + property tests (round-trip on every schema at minimum)
- [ ] Error tracks are tested (`Effect.flip`/`Exit`), including "failure leaves no partial state"
- [ ] Time-dependent logic uses `TestClock` (from `effect/testing`, or `it.effect` in `@effect/vitest`); zero real sleeps in tests
- [ ] Fakes are Layers, not mocking libraries
- [ ] `tsc --noEmit` (zero `@effect/language-service` warnings), lint, and the full test suite pass — actually run them and report the output; never claim green without running

## 7. Checklist sweep

Open each reference file used during the task and walk its end-of-file checklist against the diff. Report the result honestly: if an item is unmet, either fix it or state explicitly why it does not apply.
