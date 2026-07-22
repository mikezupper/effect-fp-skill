# Production Readiness & Scale

"Works on the happy path" is the starting line. Everything in this file is **required** for any deployable service — Effect makes each item a few lines, so there is no excuse to skip them. The checklist at the bottom is the definition of done.

## Observability

### Structured logging
```ts
// Effect.log* — never console.log. Logs carry fiber id, span, timestamps, annotations.
yield* Effect.logInfo("order placed").pipe(
  Effect.annotateLogs({ orderId: order.id, userId: user.id })
)

// Annotate whole scopes so every log inside carries context:
handleRequest.pipe(Effect.annotateLogs({ requestId }))

// Production layer: JSON logs + level from config
Logger.json, Logger.minimumLogLevel(cfg.logLevel)
```

### Tracing
Every workflow and every service method that does I/O gets a span. This is the rule that makes incidents debuggable.

```ts
const placeOrder = (input: PlaceOrderInput) =>
  Effect.gen(function* () { /* ... */ }).pipe(
    Effect.withSpan("Orders.placeOrder", { attributes: { "order.channel": input.channel } })
  )
```

Wire OpenTelemetry once with `@effect/opentelemetry` (`NodeSdk.layer` exporting OTLP); spans nest automatically across services, and errors are recorded on spans without extra code.

### Metrics
```ts
const ordersPlaced = Metric.counter("orders_placed_total")
const orderValue   = Metric.histogram("order_value_cents", MetricBoundaries.exponential({ start: 100, factor: 2, count: 16 }))

yield* chargePayment(order).pipe(
  Metric.trackDuration(paymentLatency),
  Effect.tap(() => Metric.increment(ordersPlaced))
)
```

Minimum set per service: request count/latency/error-rate per endpoint, queue depths, external-call latency + failure count, plus your key business counters.

## Resilience

```ts
// The standard external-call wrapper — apply to EVERY network call:
const callGateway = (req: ChargeRequest) =>
  gateway.charge(req).pipe(
    Effect.timeout("5 seconds"),                                   // nothing waits forever
    Effect.retry({
      schedule: Schedule.exponential("200 millis").pipe(Schedule.jittered),
      times: 3,
      while: (e) => e._tag === "GatewayError" && e.retriable,      // NEVER retry business rejections
    }),
    Effect.withSpan("PaymentGateway.charge")
  )
```

- **Timeouts on every external call** — DB, HTTP, queue. Choose deliberately per call; no default infinities.
- **Retries only on transient errors**, always with exponential backoff + jitter. Retrying `PaymentDeclined` double-charges customers.
- **Idempotency**: any retried or queue-driven operation must be idempotent (idempotency keys on writes; dedupe on consume).
- **Circuit breaking / load shedding** on hot external dependencies: bound concurrent calls with a `Semaphore` (`Effect.makeSemaphore(n)` + `sem.withPermits(1)`) so a slow dependency can't absorb every fiber; fail fast when saturated.
- **Backpressure over buffering**: bounded `Queue`s (`Queue.bounded`) and `Stream` — unbounded buffers turn overload into OOM.

## Resource safety & graceful shutdown

- Every resource (pool, socket, file, consumer) is acquired with `Effect.acquireRelease` inside a `Layer.scoped` — release runs on success, failure, defect, *and* interruption. There is no code path that leaks.
- `runMain` handles SIGINT/SIGTERM by interrupting the main fiber → finalizers run in reverse order: stop accepting requests, drain in-flight work, close pools, flush telemetry.
- In-flight work you must not lose on shutdown: `Effect.uninterruptibleMask` around the small critical section only (not whole workflows).

## Concurrency rules

```ts
// ✅ bounded, interruption-safe, fails fast
yield* Effect.forEach(userIds, notifyUser, { concurrency: 10 })

// ✅ independent branches: race/zip with automatic cleanup of the loser
const [profile, orders] = yield* Effect.all([getProfile(id), getOrders(id)], { concurrency: 2 })

// ✅ background work tied to a scope — dies with its parent, never orphaned
yield* Effect.forkScoped(pollLoop)
```

- **Always set `concurrency` explicitly**; unbounded fan-out to a DB/API is an outage. Default is sequential — fine, but decide.
- Structured concurrency: fibers are owned by scopes. No fire-and-forget `Effect.runFork` in app code; use `forkScoped`/`Daemon` deliberately.
- Shared mutable state: `Ref` (atomic), `Ref.Synchronized` (effectful updates), never module-level `let`.

## Scale patterns

- **Streams for unbounded data**: file processing, DB pagination, event feeds → `Stream` with `Stream.buffer({ capacity: n })`, `Stream.grouped`, `Stream.mapEffect(..., { concurrency })`. Constant memory regardless of input size.
- **Batching & dedup**: N+1 external calls → `Effect.request` + `RequestResolver` (batches concurrent requests, dedupes identical ones automatically).
- **Caching**: `Effect.cachedWithTTL(effect, "30 seconds")` for single values; `Cache.make({ capacity, timeToLive, lookup })` for keyed lookups. Cache at the service layer, keyed by branded types.
- **Statelessness**: services hold no per-request state outside the request scope → horizontal scaling is free. Session/shared state lives in external stores behind service tags.
- **Long-running jobs**: chunked via `Stream`, checkpointed, resumable, idempotent per chunk.

## Security baseline

- Secrets only via `Config.redacted`; `Redacted` values never interpolated into logs/errors/URLs.
- All input decoded by `Schema` (this is also your injection/overflow guard); output encoded by `Schema` (no accidental field leaks — response schemas whitelist fields).
- Authn/authz as middleware services in the `R` channel (e.g. a `CurrentUser` tag) — handlers requiring `CurrentUser` cannot be wired without the auth middleware, enforced at compile time.

## Definition of done — production checklist

- [ ] Every workflow + I/O service method has a span; OTel exporter wired
- [ ] JSON structured logs with request/correlation ids; zero `console.log`
- [ ] Metrics: RED per endpoint + external-call health + key business counters
- [ ] Every external call: timeout + transient-only retry with backoff/jitter
- [ ] Retried/queued writes are idempotent
- [ ] All resources scoped (`acquireRelease`); graceful shutdown verified (SIGTERM drains cleanly)
- [ ] All fan-out bounded; queues bounded; no unbounded buffering
- [ ] Config validated at startup; secrets `Redacted`
- [ ] Health/readiness endpoints; readiness reflects dependency status
- [ ] Load-tested the hot path; memory flat under sustained load (streams, no leaks)
