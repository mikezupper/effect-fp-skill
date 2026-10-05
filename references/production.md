# Production Readiness & Scale

"Works on the happy path" is the starting line. Everything in this file is **required** for any deployable service — Effect makes each item a few lines, so there is no excuse to skip them. The checklist at the bottom is the definition of done.

Verified against effect 4.0.1 (October 2026). Logging, tracing, metrics, OTLP export, caching, semaphores and rate limiting all ship in `effect` itself (`"effect"`, `"effect/observability"`, `"effect/persistence"`); the only extra packages are the platform runtime (`@effect/platform-node`) and, optionally, `@effect/opentelemetry`.

## Observability

### Structured logging
```ts
import { Config, Effect, Layer, Logger, References } from "effect"

// Effect.log* — never console.log. Logs carry fiber id, span, timestamps, annotations.
yield* Effect.logInfo("order placed").pipe(
  Effect.annotateLogs({ orderId: order.id, userId: user.id })
)

// Annotate whole scopes so every log inside carries context:
handleRequest.pipe(Effect.annotateLogs({ requestId }))

// Production layer: JSON logs + level from config
const LoggingLive = Layer.mergeAll(
  Logger.layer([Logger.consoleJson]),                               // v3: Logger.json
  Layer.effect(References.MinimumLogLevel,                          // v3: Logger.minimumLogLevel
    Config.LogLevel("LOG_LEVEL").pipe(Config.withDefault("Info"))   // a Config is an Effect
  )
)
```

### Tracing
Every workflow and every service method that does I/O gets a span. This is the rule that makes incidents debuggable. `Effect.fn("Name")` is the default way to write them: it opens a span named after the function and improves stack traces.

```ts
import { Effect } from "effect"

const placeOrder = Effect.fn("Orders.placeOrder")(function* (input: PlaceOrderInput) {
  yield* Effect.annotateCurrentSpan({ "order.channel": input.channel })
  /* ... */
})

// non-function effects: Effect.withSpan
const warmup = loadCatalog.pipe(Effect.withSpan("Catalog.warmup"))
```

Wire export once, at the edge. For new projects use the lightweight OTLP exporters in `effect/observability` (traces, logs and Effect metrics in one layer); use `@effect/opentelemetry`'s `NodeSdk.layer` only when you must integrate with an existing OpenTelemetry SDK setup. Spans nest automatically across services, and errors are recorded on spans without extra code.

```ts
import { Config, Effect, Layer } from "effect"
import { FetchHttpClient } from "effect/http"
import { Otlp } from "effect/observability"

const TelemetryLive = Layer.unwrap(
  Effect.gen(function* () {
    const baseUrl = yield* Config.String("OTEL_EXPORTER_OTLP_ENDPOINT")   // e.g. http://collector:4318
    return Otlp.layerJson({ baseUrl, resource: { serviceName: "orders-api", serviceVersion: "1.4.0" } })
  })
).pipe(Layer.provide(FetchHttpClient.layer))
```

### Metrics
```ts
import { Effect, Metric } from "effect"

const ordersPlaced   = Metric.counter("orders_placed_total", { incremental: true })
const orderValue     = Metric.histogram("order_value_cents", {
  boundaries: Metric.exponentialBoundaries({ start: 100, factor: 2, count: 16 })
})
const paymentLatency = Metric.timer("payment_latency")
const paymentErrors  = Metric.counter("payment_errors_total", { incremental: true })

yield* chargePayment(order).pipe(
  Effect.trackDuration(paymentLatency),                       // v3: Metric.trackDuration
  Effect.trackErrors(paymentErrors, () => 1),
  Effect.tap(() => Metric.update(ordersPlaced, 1)),           // v3: Metric.increment
  Effect.tap(() => Metric.update(orderValue, order.totalCents))
)
```

Minimum set per service: request count/latency/error-rate per endpoint, queue depths, external-call latency + failure count, plus your key business counters.

## Resilience

```ts
import { Effect, Schedule } from "effect"

// The standard external-call wrapper — apply to EVERY network call:
const callGateway = Effect.fn("PaymentGateway.charge")((req: ChargeRequest) =>
  gateway.charge(req).pipe(
    Effect.timeout("5 seconds"),                                   // nothing waits forever (adds Cause.TimeoutError to E)
    Effect.retry({
      schedule: Schedule.exponential("200 millis").pipe(Schedule.jittered),
      times: 3,
      while: (e) => e._tag === "GatewayError" && e.retriable,      // NEVER retry business rejections
    })
  )
)
```

- **Timeouts on every external call** — DB, HTTP, queue. Choose deliberately per call; no default infinities. Translate the resulting `TimeoutError` into your domain error inside the service (`Effect.catchTag("TimeoutError", ...)`).
- **Retries only on transient errors**, always with exponential backoff + jitter. Retrying `PaymentDeclined` double-charges customers. For reusable policies, build the schedule once: `Schedule.exponential(...).pipe(Schedule.jittered, Schedule.setInputType<E>(), Schedule.while(({ input }) => input.retriable))`, and cap delays with `Schedule.min([Schedule.exponential(...), Schedule.spaced("10 seconds")])`.
- **Idempotency**: any retried or queue-driven operation must be idempotent (idempotency keys on writes; dedupe on consume).
- **Circuit breaking / load shedding** on hot external dependencies: bound concurrent calls with a `Semaphore` (`Semaphore.make(n)` + `sem.withPermit(effect)`) so a slow dependency can't absorb every fiber; fail fast when saturated with `sem.withPermitsIfAvailable(1)(effect)` (returns `Option.none()` instead of queueing). Per-window quotas: `RateLimiter` from `effect/persistence`.
- **Backpressure over buffering**: bounded `Queue`s (`Queue.bounded`) and `Stream` — unbounded buffers turn overload into OOM.

## Resource safety & graceful shutdown

- Every resource (pool, socket, file, consumer) is acquired with `Effect.acquireRelease` inside a `Layer.effect` (v4 layers own a scope; `Layer.scoped` is gone) — release runs on success, failure, defect, *and* interruption. There is no code path that leaks.
- `NodeRuntime.runMain` (from `@effect/platform-node`) handles SIGINT/SIGTERM by interrupting the main fiber → finalizers run in reverse order: stop accepting requests, drain in-flight work, close pools, flush telemetry (the OTLP layers flush on release; `shutdownTimeout` bounds it).
- In-flight work you must not lose on shutdown: `Effect.uninterruptibleMask` around the small critical section only (not whole workflows).

## Concurrency rules

```ts
import { Effect } from "effect"

// ✅ bounded, interruption-safe, fails fast
yield* Effect.forEach(userIds, notifyUser, { concurrency: 10 })

// ✅ independent branches: race/zip with automatic cleanup of the loser
const [profile, orders] = yield* Effect.all([getProfile(id), getOrders(id)], { concurrency: 2 })

// ✅ background work tied to a scope — dies with its scope, never orphaned
yield* Effect.forkScoped(pollLoop)
```

- **Always set `concurrency` explicitly**; unbounded fan-out to a DB/API is an outage. Default is sequential — fine, but decide.
- Structured concurrency: fibers are owned by scopes. No fire-and-forget `Effect.runFork` in app code; use `forkScoped`/`forkChild`/`FiberSet` by default and `forkDetach` (v3 `forkDaemon`) only deliberately, with a comment.
- Shared mutable state: `Ref` (atomic), `SynchronizedRef` (effectful updates), `TxRef` + `Effect.tx` (multi-ref transactions), never module-level `let`.

See `concurrency.md` for the full cookbook.

## Scale patterns

- **Streams for unbounded data**: file processing, DB pagination, event feeds → `Stream` with `Stream.buffer({ capacity: n })`, `Stream.grouped(n)`, `Stream.mapEffect(..., { concurrency })`. Constant memory regardless of input size.
- **Batching & dedup**: N+1 external calls → `Request.Class` + `RequestResolver.make` + `Effect.request` (batches concurrent requests, dedupes identical ones automatically; `RequestResolver.setDelay` widens the batch window).
- **Caching**: `Effect.cachedWithTTL(effect, "30 seconds")` for single values; `Cache.make({ capacity, timeToLive, lookup })` for keyed lookups; `RcMap`/`ScopedCache` for keyed *resources* that need release. Cache at the service layer, keyed by branded types.
- **Statelessness**: services hold no per-request state outside the request scope → horizontal scaling is free. Session/shared state lives in external stores behind `Context.Service` tags.
- **Long-running jobs**: chunked via `Stream`, checkpointed, resumable, idempotent per chunk.

## Security baseline

- Secrets only via `Config.Redacted("NAME")`; `Redacted` values never interpolated into logs/errors/URLs (`Redacted.value` only at the call that needs the raw secret).
- All input decoded by `Schema` (this is also your injection/overflow guard); output encoded by `Schema` (no accidental field leaks — response schemas whitelist fields).
- Authn/authz as middleware services in the `R` channel (e.g. a `CurrentUser` `Context.Service` provided by an `HttpApiMiddleware` from `effect/http-api`) — handlers requiring `CurrentUser` cannot be wired without the auth middleware, enforced at compile time.

## Definition of done — production checklist

- [ ] Every workflow + I/O service method has a span (`Effect.fn("Name")` or `withSpan`); OTLP exporter wired
- [ ] JSON structured logs (`Logger.consoleJson`) with request/correlation ids; zero `console.log`
- [ ] Metrics: RED per endpoint + external-call health + key business counters
- [ ] Every external call: timeout + transient-only retry with backoff/jitter
- [ ] Retried/queued writes are idempotent
- [ ] All resources scoped (`acquireRelease` in `Layer.effect`); graceful shutdown verified (SIGTERM drains cleanly)
- [ ] All fan-out bounded; queues bounded; no unbounded buffering
- [ ] Config validated at startup; secrets `Config.Redacted`
- [ ] Health/readiness endpoints; readiness reflects dependency status
- [ ] Load-tested the hot path; memory flat under sustained load (streams, no leaks)
