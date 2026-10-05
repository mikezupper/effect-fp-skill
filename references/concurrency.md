# Concurrency Cookbook

Effect's structured concurrency: every fiber is owned by a scope; when the scope closes (success, failure, or interruption), its fibers are interrupted and their finalizers run. There are no orphaned promises and no leaked work — **if you follow the rules below**.

Verified against effect 4.0.1 (October 2026). Every primitive here (`Semaphore`, `Latch`, `Queue`, `PubSub`, `Deferred`, `SynchronizedRef`, `FiberSet`, `TxRef`) imports from core `"effect"`; only the rate limiter lives in `"effect/persistence"`.

## Core rules

1. **Bound everything.** `{ concurrency: n }` explicit on every fan-out. Unbounded fan-out against a DB/API is an outage.
2. **Own your fibers.** `Effect.forkScoped` (dies with the enclosing scope) or `Effect.forkChild` (dies with the parent fiber — v3's `Effect.fork`). `Effect.forkDetach` (v3's `forkDaemon`) almost never — it escapes supervision; if used, justify in a comment and manage shutdown manually. For a dynamic, growing set of fibers use `FiberSet`/`FiberMap`/`FiberHandle`, never a hand-kept array of `Fiber`s.
3. **Communicate by message, not shared mutation.** `Queue`/`PubSub`/`Deferred`/`Latch` between fibers; `Ref` for shared state; never module-level `let`.
4. **Interruption is normal control flow.** Losers of races and timed-out effects get interrupted; `acquireRelease` finalizers still run. Keep critical sections small with `Effect.uninterruptibleMask` and restore interruptibility inside.

## Recipes

### Bounded parallel map (the workhorse)
```ts
import { Effect } from "effect"

const results = yield* Effect.forEach(ids, fetchOne, { concurrency: 10 })
// batch job that must not abort on single failures — v4 order is [successes, failures]:
const [oks, failures] = yield* Effect.partition(ids, fetchOne, { concurrency: 10 })
```

### Race with cleanup / hedged requests
```ts
import { Effect } from "effect"

const fastest = yield* Effect.race(primaryLookup, replicaLookup)   // first SUCCESS wins; loser auto-interrupted
const first   = yield* Effect.raceAll([r1, r2, r3])
// first to COMPLETE (success or failure) wins: Effect.raceFirst
// hedge: fire backup only if primary is slow
const hedged = Effect.race(primary, backup.pipe(Effect.delay("200 millis")))
```

### Independent work in one request
```ts
import { Effect } from "effect"

const [profile, orders, prefs] = yield* Effect.all(
  [getProfile(id), getOrders(id), getPrefs(id)],
  { concurrency: "unbounded" }   // fine HERE: fixed small arity, not data-driven fan-out
)
```

### Producer/consumer with backpressure
```ts
import { Array, Effect, Queue } from "effect"

const queue = yield* Queue.bounded<Job>(64)          // bounded = producers suspend when full
yield* producer(queue).pipe(Effect.forkScoped)
yield* Effect.forEach(
  Array.range(1, 8),                                  // 8 workers
  () => worker(queue).pipe(Effect.forkScoped)         // worker: loop on Queue.take(queue)
)
// Dropping/sliding variants when overload should shed instead of block:
// Queue.dropping (reject newest), Queue.sliding (evict oldest)
```

### Dynamic set of fibers (per-connection / per-message handlers)
```ts
import { Effect, FiberSet } from "effect"

const handlers = yield* FiberSet.make<void, HandlerError>()   // scoped: scope closes → all interrupted
yield* FiberSet.run(handlers, handleConnection(conn))         // finished fibers are removed automatically
```

### One-shot signal / handoff between fibers
```ts
import { Deferred, Effect, Latch } from "effect"

const ready = yield* Deferred.make<ServerAddress, StartupError>()
yield* startServer(ready).pipe(Effect.forkScoped)     // server: Deferred.succeed(ready, addr)
const addr = yield* Deferred.await(ready)             // waiter: suspends until resolved

// gate with no payload, reusable (open/close): Latch (v3 Effect.makeLatch)
const gate = yield* Latch.make()                      // starts closed
yield* consume.pipe(gate.whenOpen, Effect.forkScoped) // runs once the gate opens
yield* gate.open
```

### Broadcast to many consumers
```ts
import { Effect, PubSub } from "effect"

const events = yield* PubSub.bounded<DomainEvent>(128)
// each subscriber gets EVERY event (vs Queue: each item to ONE consumer)
const sub = yield* PubSub.subscribe(events)           // scoped — auto-unsubscribes
const event = yield* PubSub.take(sub)
```

### Mutual exclusion / limiting access to a resource
```ts
import { Effect, Semaphore } from "effect"

const sem = yield* Semaphore.make(4)                  // at most 4 concurrent calls (v3: Effect.makeSemaphore)
const guarded = sem.withPermit(callLegacyApi(req))
// withPermit/withPermits(n) release on success, failure, AND interruption
// load shedding — don't queue, fail fast when saturated (Option.none() = no permit):
const shed = sem.withPermitsIfAvailable(1)(callLegacyApi(req))
```

### Rate limiting (calls per window, not just concurrency)
```ts
import { Effect, Layer } from "effect"
import { RateLimiter } from "effect/persistence"

const withLimit = yield* RateLimiter.makeWithRateLimiter      // needs the RateLimiter service
const limited = callThirdPartyApi(req).pipe(
  withLimit({ key: "third-party-api", limit: 100, window: "1 minute", onExceeded: "delay" })
)   // "delay" waits for the window; "fail" fails with RateLimiterError

// wiring: in-memory store for one instance; layerStoreRedis to share the limit across instances
const RateLimiterLive = RateLimiter.layer.pipe(Layer.provide(RateLimiter.layerStoreMemory))
```

### Shared state
```ts
import { Effect, HashMap, Ref, SynchronizedRef, TxRef } from "effect"

const counter = yield* Ref.make(0)
yield* Ref.update(counter, (n) => n + 1)              // atomic
// update that itself needs an effect (v3 Ref.Synchronized):
const cache = yield* SynchronizedRef.make(HashMap.empty<UserId, User>())
yield* SynchronizedRef.updateEffect(cache, refresh)
// multiple refs updated atomically together → TxRef + Effect.tx (v3 STM/TRef); rare, don't reach for it first
const from = yield* TxRef.make(100)
const to = yield* TxRef.make(0)
yield* Effect.tx(Effect.all([TxRef.update(from, (n) => n - 10), TxRef.update(to, (n) => n + 10)]))
```

### Background loop tied to app lifecycle
```ts
import { Effect, Layer, Schedule } from "effect"

const PollerLive = Layer.effectDiscard(
  pollOnce.pipe(
    Effect.repeat(Schedule.spaced("30 seconds")),
    Effect.forkScoped                                  // layer's scope → interrupted at shutdown
  )
)
```

### Streams instead of hand-rolled pipelines
When the shape is source → transform → sink over many/unbounded items, don't compose Queues and fibers manually — use `Stream`:
```ts
import { Effect, Stream } from "effect"

yield* Stream.fromIterable(files).pipe(
  Stream.mapEffect(parseFile, { concurrency: 4 }),
  Stream.grouped(100),                                 // batch for bulk insert
  Stream.mapEffect(insertBatch),
  Stream.runDrain
)
```
Constant memory, built-in backpressure, interruption-safe.

## Anti-patterns

- `Effect.runFork`/`runPromise` to "start background work" from inside app code → `forkScoped` in a `Layer.effect`/`Layer.effectDiscard`
- `{ concurrency: "unbounded" }` over a data-driven collection → pick a number
- Unbounded `Queue.unbounded` between fast producer and slow consumer → bounded (backpressure) or dropping/sliding (shed)
- Polling a `Ref` in a loop to wait for a value → `Deferred` (one value) or `Latch` (gate)
- `uninterruptible` around a whole workflow "to be safe" → smallest critical section only; it blocks shutdown
- Manual `Fiber.join` bookkeeping across many fibers → `Effect.forEach`/`Effect.all`/`Stream` express it structurally; dynamic sets → `FiberSet`
- `forkDetach` to "keep it running after the request" → `forkScoped` into a layer-owned scope, or `FiberSet.run` on a layer-owned `FiberSet`
- v3 leftovers that no longer compile: `Effect.fork`/`forkDaemon`, `Effect.makeSemaphore`/`makeLatch`, `Ref.Synchronized`, `STM`/`TRef`, `Queue.take` on a PubSub subscription

## Checklist

- [ ] Every data-driven `forEach`/`all`/`mapEffect` has a numeric `concurrency`
- [ ] Every fork is `forkScoped`/`forkChild`/`FiberSet.run`; any `forkDetach` carries a justification comment
- [ ] Every queue/pubsub is bounded, dropping, or sliding — never unbounded on a hot path
- [ ] Shared state is `Ref`/`SynchronizedRef`/`TxRef`; zero module-level `let`
- [ ] Semaphores/rate limiters guard every hot external dependency
- [ ] `uninterruptible` regions are minimal; interruption tested (shutdown drains cleanly)
