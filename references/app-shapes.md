# App Shapes: HTTP APIs, CLIs, Full-Stack, Libraries

How the onion gets an outer shell for each kind of executable. The domain/workflow/service core is identical in all four — only the adapter layer changes. APIs below verified against `@effect/platform` 0.97.0 / `@effect/cli` 0.76.0 (July 2026). Note: the Http modules are officially flagged **unstable** ("subject to change") — the canonical docs are the npm-published package READMEs; re-verify on version bumps.

## HTTP backend — `@effect/platform` HttpApi

Define the API as a first-class, schema-driven value. Request/response/error schemas are the same domain schemas — one source of truth, and the contract is machine-readable (OpenAPI derivable, client derivable).

```ts
import { HttpApi, HttpApiBuilder, HttpApiEndpoint, HttpApiGroup } from "@effect/platform"

const OrdersGroup = HttpApiGroup.make("orders").add(
  HttpApiEndpoint.post("placeOrder", "/orders")
    .setPayload(PlaceOrderInput)          // Schema — decoding is automatic; handler sees domain types
    .addSuccess(OrderResponse)
    .addError(InsufficientStock, { status: 409 })   // Schema.TaggedError → typed, documented error responses
)

export const Api = HttpApi.make("app").add(OrdersGroup)

// Handlers are thin: workflow call + error mapping. No logic in handlers.
const OrdersLive = HttpApiBuilder.group(Api, "orders", (handlers) =>
  handlers.handle("placeOrder", ({ payload }) => placeOrder(payload))
)
```

- The compiler enforces that every endpoint has a handler and every declared error is either returned or mapped.
- Path params via template-literal endpoints: `HttpApiEndpoint.get("getOrder")\`/orders/${HttpApiSchema.param("id", OrderId)}\``.
- Error → status: either per-use (`.addError(E, { status: 409 })`) or baked into the error schema with `HttpApiSchema.annotations({ status: 409 })` as the third arg to `Schema.TaggedError`.

Serving (canonical wiring — note the intermediate `HttpApiBuilder.api` layer and `Layer.launch`):

```ts
import { createServer } from "node:http"
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { HttpApiSwagger } from "@effect/platform"

const ApiLive = HttpApiBuilder.api(Api).pipe(Layer.provide(OrdersLive))

const ServerLive = HttpApiBuilder.serve().pipe(
  Layer.provide(HttpApiSwagger.layer()),      // serves Swagger UI from the api definition
  Layer.provide(ApiLive),
  Layer.provide(AppLayer),                    // your services/infra
  Layer.provide(NodeHttpServer.layer(createServer, { port: 3000 }))
)

Layer.launch(ServerLive).pipe(NodeRuntime.runMain)
```

Auth middleware — a tag that *provides* `CurrentUser`, so protected handlers can't be wired without it:

```ts
class Authorization extends HttpApiMiddleware.Tag<Authorization>()("Authorization", {
  failure: Unauthorized,
  provides: CurrentUser,
  security: { bearer: HttpApiSecurity.bearer },
}) {}
// apply with .middleware(Authorization) at endpoint, group, or API level; implement as a Layer
```

OpenAPI comes free: `OpenApi.fromApi(Api)` for the spec, `HttpApiSwagger.layer()` for the UI.

## CLI — `@effect/cli`

```ts
import { Command, Options, Args } from "@effect/cli"

const verbose = Options.boolean("verbose").pipe(Options.withAlias("v"))
const file = Args.file({ name: "input" })

const process = Command.make("process", { verbose, file }, ({ verbose, file }) =>
  processFile(file).pipe(verbose ? Effect.tap(logDetails) : identity)
)

const cli = Command.run(Command.make("mytool").pipe(Command.withSubcommands([process])),
  { name: "mytool", version: "1.0.0" })

cli(process.argv).pipe(Effect.provide(NodeContext.layer), NodeRuntime.runMain)
```

- Options/Args are typed and validated (schema-backed); `--help`, errors, and shell completions come free.
- Same rules as services: workflows do the work; commands are thin adapters. File system via `FileSystem` from `@effect/platform` (a service — testable in memory), never `node:fs` directly.
- Exit codes come from the error track: `runMain` exits nonzero on failure automatically.

## Full-stack

The prime directive: **one `packages/domain` shared by client and server** containing schemas, branded types, unions, and error types. The wire contract is those schemas; client and server cannot drift.

```
packages/
  domain/     # schemas, types, pure functions — zero platform imports
  server/     # HttpApi + workflows + infra layers
  web/        # frontend
```

- Derive the client from the API definition — fully typed calls including typed error channels, no hand-written fetch code:
  ```ts
  const client = yield* HttpApiClient.make(Api, { baseUrl: "https://api.example.com" })
  const order = yield* client.orders.placeOrder({ payload })   // needs an HttpClient, e.g. FetchHttpClient.layer
  ```
- Frontend state: keep the functional core. Effect runs in the browser; build one `ManagedRuntime` with browser layers (HttpClient, storage services) at app root. For React integration use an Effect-native state library (e.g. `@effect-atom/atom-react`) or run workflows via the runtime from event handlers — components stay thin views; logic stays in Effect workflows, testable without a DOM.
- Validation is shared: the same `Email` schema powers the form field hint and the server rejection.

## Libraries / publishable packages

- Public API returns `Effect`/`Option`/`Either` with **exported, documented tagged errors** — the error union is part of semver.
- Depend on service tags, never concrete platforms: accept `HttpClient` from `@effect/platform` rather than bundling fetch/axios; consumers provide the platform layer. Zero Node-only imports unless the package is explicitly Node-only.
- Provide a `layer` (and `layerConfig`) as the entry point; expose the tag so consumers can fake it in *their* tests.
- Peer-depend on `effect`; never bundle a second copy (breaks tag identity and instanceof checks).
- Ship tree-shakeable ESM, dual-package if needed; `sideEffects: false`.
- Property-test the public surface with arbitraries derived from your own schemas; the round-trip tests double as living documentation.
