# Self-Review Pass — run before declaring any work done

After implementing, review your own output as a hostile reviewer would. Do this **every time**, before reporting completion. Fix everything found, then re-run the pass.

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

```bash
# domain/ may import ONLY from "effect" and "./": any platform/infra import here is a violation
grep -rn "^import" src/domain/ --include="*.ts" | grep -v "from \"effect\"\|from \"\./\|from \"\.\./"
# workflows/ may not import concrete infrastructure (services/ implementations, sql, platform-node)
grep -rn "@effect/platform-node\|@effect/sql" src/workflows/ --include="*.ts"
```

## 3. Error-channel audit

Read every exported function signature and check:
- [ ] `E` is a union of named tagged errors — no `Error`, `unknown`, `ParseError` leaking past the boundary, no accidental `never`
- [ ] No `catchAll` that swallows; every `orDie` has a comment proving the invariant
- [ ] Infra errors are translated to domain errors inside services (no `SqlError` visible in a workflow signature)
- [ ] Errors carry the data a handler needs (ids, retriability), not just messages

## 4. Type-design audit

- [ ] No naked `string`/`number` crossing function boundaries in the domain — branded types
- [ ] No boolean flag pairs encoding states — tagged unions
- [ ] Every `$match`/`Match` pipeline ends in `exhaustive` (no `orElse` hiding unhandled cases without justification)
- [ ] Every boundary decodes with Schema exactly once; zero `as` casts on external data

## 5. Runtime & resource audit

- [ ] Exactly one `runMain`/`ManagedRuntime`; everything else composes
- [ ] Every resource acquired via `acquireRelease`/`Layer.scoped`
- [ ] Every `Effect.forEach`/`Effect.all` fan-out has explicit `concurrency`
- [ ] Every external call has a timeout; retries are transient-only with backoff+jitter
- [ ] Workflows and I/O service methods have `withSpan`; logs are annotated, structured

## 6. Test audit

- [ ] Pure domain logic has direct unit + property tests (round-trip on every schema at minimum)
- [ ] Error tracks are tested (`Effect.flip`/`Exit`), including "failure leaves no partial state"
- [ ] Time-dependent logic uses `TestClock`; zero real sleeps in tests
- [ ] Fakes are Layers, not mocking libraries
- [ ] `tsc --noEmit`, lint, and the full test suite pass — actually run them and report the output; never claim green without running

## 7. Checklist sweep

Open each reference file used during the task and walk its end-of-file checklist against the diff. Report the result honestly: if an item is unmet, either fix it or state explicitly why it does not apply.
