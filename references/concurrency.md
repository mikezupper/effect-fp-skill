# Concurrency Cookbook

Effect's structured concurrency: every fiber is owned by a scope; when the scope closes (success, failure, or interruption), its fibers are interrupted and their finalizers run. There are no orphaned promises and no leaked work — **if you follow the rules below**.

## Core rules

1. **Bound everything.** `{ concurrency: n }` explicit on every fan-out. Unbounded fan-out against a DB/API is an outage.
2. **Own your fibers.** `Effect.forkScoped` (dies with the enclosing scope) or `Effect.fork` (dies with the parent fiber). `forkDaemon` almost never — it escapes supervision; if used, justify in a comment and manage shutdown manually.
3. **Communicate by message, not shared mutation.** `Queue`/`PubSub`/`Deferred` between fibers; `Ref` for shared state; never module-level `let`.
4. **Interruption is normal control flow.** Losers of races and timed-out effects get interrupted; `acquireRelease` finalizers still run. Keep critical sections small with `Effect.uninterruptibleMask` and restore interruptibility inside.

## Recipes

### Bounded parallel map (the workhorse)
```ts
const results = yield* Effect.forEach(ids, fetchOne, { concurrency: 10 })
// batch job that must not abort on single failures:
const [failures, oks] = yield* Effect.partition(ids, fetchOne, { concurrency: 10 })
```

### Race with cleanup / hedged requests
```ts
const fastest = yield* Effect.race(primaryLookup, replicaLookup)   // loser auto-interrupted
const first   = yield* Effect.raceAll([r1, r2, r3])
// hedge: fire backup only if primary is slow
const hedged = Effect.race(primary, backup.pipe(Effect.delay("200 millis")))
```

### Independent work in one request
```ts
const [profile, orders, prefs] = yield* Effect.all(
  [getProfile(id), getOrders(id), getPrefs(id)],
  { concurrency: "unbounded" }   // fine HERE: fixed small arity, not data-driven fan-out
)
```

### Producer/consumer with backpressure
```ts
const queue = yield* Queue.bounded<Job>(64)          // bounded = producers block when full
yield* producer(queue).pipe(Effect.forkScoped)
yield* Effect.forEach(
  Array.range(1, 8),                                  // 8 workers
  () => worker(queue).pipe(Effect.forkScoped)
)
// Dropping/sliding variants when overload should shed instead of block:
// Queue.dropping (reject newest), Queue.sliding (evict oldest)
```

### One-shot signal / handoff between fibers
```ts
const ready = yield* Deferred.make<ServerAddress, StartupError>()
yield* startServer(ready).pipe(Effect.forkScoped)     // server: Deferred.succeed(ready, addr)
const addr = yield* Deferred.await(ready)             // waiter: suspends until resolved
```

### Broadcast to many consumers
```ts
const events = yield* PubSub.bounded<DomainEvent>(128)
// each subscriber gets EVERY event (vs Queue: each item to ONE consumer)
const sub = yield* PubSub.subscribe(events)           // scoped — auto-unsubscribes
yield* Queue.take(sub)
```

### Mutual exclusion / limiting access to a resource
```ts
const sem = yield* Effect.makeSemaphore(4)            // at most 4 concurrent calls
const guarded = sem.withPermits(1)(callLegacyApi(req))
// withPermits releases on success, failure, AND interruption
```

### Rate limiting (calls per window, not just concurrency)
```ts
const limiter = yield* RateLimiter.make({ limit: 100, interval: "1 minute" })  // scoped
const limited = limiter(callThirdPartyApi(req))
```

### Shared state
```ts
const counter = yield* Ref.make(0)
yield* Ref.update(counter, (n) => n + 1)              // atomic
// update that itself needs an effect:
const cache = yield* Ref.Synchronized.make(Map.empty())
yield* Ref.Synchronized.updateEffect(cache, refresh)
// multiple refs updated atomically together → STM (TRef + STM.commit); rare, don't reach for it first
```

### Background loop tied to app lifecycle
```ts
const poller = pollOnce.pipe(
  Effect.repeat(Schedule.spaced("30 seconds")),
  Effect.forkScoped                                    // in a Layer.scoped → stops at shutdown
)
```

### Streams instead of hand-rolled pipelines
When the shape is source → transform → sink over many/unbounded items, don't compose Queues and fibers manually — use `Stream`:
```ts
yield* Stream.fromIterable(files).pipe(
  Stream.mapEffect(parseFile, { concurrency: 4 }),
  Stream.grouped(100),                                 // batch for bulk insert
  Stream.mapEffect(insertBatch),
  Stream.runDrain
)
```
Constant memory, built-in backpressure, interruption-safe.

## Anti-patterns

- `Effect.runFork`/`runPromise` to "start background work" from inside app code → `forkScoped` in a scoped layer
- `{ concurrency: "unbounded" }` over a data-driven collection → pick a number
- Unbounded `Queue.unbounded` between fast producer and slow consumer → bounded (backpressure) or dropping/sliding (shed)
- Polling a `Ref` in a loop to wait for a value → `Deferred`
- `uninterruptible` around a whole workflow "to be safe" → smallest critical section only; it blocks shutdown
- Manual `Fiber.join` bookkeeping across many fibers → `Effect.forEach`/`Effect.all`/`Stream` express it structurally
