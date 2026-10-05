# App Shapes: HTTP APIs, CLIs, Full-Stack, Libraries

How the onion gets an outer shell for each kind of executable. The domain/workflow/service core is identical in all four — only the adapter layer changes. APIs below verified against effect 4.0.1 / `@effect/platform-node` 4.0.1 (October 2026). In v4 the HTTP, HttpApi and CLI modules live inside `effect` itself (`effect/http`, `effect/http-api`, `effect/cli`); `@effect/platform` and `@effect/cli` are v3-only — never install them alongside effect 4. Every one of these subpath modules is tagged `@stability unstable` ("subject to change"). The canonical docs ship in the package: `node_modules/effect/ai-docs/src/51_http-server`, `50_http-client`, `70_cli` (and `node_modules/effect/CLAUDE.md`); re-verify on version bumps.

## HTTP backend — `effect/http-api`

Define the API as a first-class, schema-driven value. Request/response/error schemas are the same domain schemas — one source of truth, and the contract is machine-readable (OpenAPI derivable, client derivable). Keep the API definition in its own module/package, separate from the server implementation, so clients can import it without pulling in server code.

```ts
import { HttpApi, HttpApiEndpoint, HttpApiGroup, HttpApiSchema } from "effect/http-api"
// InsufficientStock, OrderNotFound, ... are plain Schema.TaggedErrors from packages/domain — no HTTP knowledge

export class OrdersGroup extends HttpApiGroup.make("orders").add(
  HttpApiEndpoint.post("placeOrder", "/orders", {
    payload: PlaceOrderInput,                        // Schema — decoding is automatic; handler sees domain types
    success: OrderResponse.pipe(HttpApiSchema.status(201)),
    error: [                                         // domain errors stay HTTP-free; statuses attach here
      InsufficientStock.pipe(HttpApiSchema.status(409)),
      OutOfDeliveryArea.pipe(HttpApiSchema.status(422)),
    ]
  }),
  HttpApiEndpoint.get("getOrder", "/orders/:id", {
    params: { id: OrderId },                         // path params decode through the domain schema
    success: OrderResponse,
    error: OrderNotFound.pipe(HttpApiSchema.status(404))
  })
) {}

export class Api extends HttpApi.make("app").add(OrdersGroup) {}
```

Handlers (server side):

```ts
import { Context, Effect } from "effect"
import { HttpApiBuilder } from "effect/http-api"

// Handlers are thin: workflow call + error mapping. No logic in handlers.
export const OrdersHandlers = HttpApiBuilder.group(Api, "orders", Effect.fn(function* (handlers) {
  // Resolve services when the group is BUILT — see the gotcha below
  const services = yield* Effect.context<OrderRepo | Inventory>()
  return handlers
    .handle("placeOrder", ({ payload }) => placeOrder(payload).pipe(Effect.provideContext(services)))
    .handle("getOrder", ({ params }) => getOrder(params.id).pipe(Effect.provideContext(services)))
}))
```

- The compiler enforces that every endpoint has a handler (`"Endpoint not handled: getOrder"`) and that a handler's error channel contains only declared errors — anything else must be mapped or turned into a defect (`Effect.orDie` → 500). Use `handlers.handleAll({ ... })` to implement a whole group in one object.
- Endpoint constructors: `HttpApiEndpoint.get/post/put/patch/delete` (`del` is gone). Options: `params`, `query`, `headers` (a Schema or a plain record of field schemas, e.g. `query: { page: Schema.OptionFromOptionalKey(PageNo) }`), `payload` (request body for POST/PUT/PATCH — pass a real Schema such as a `Schema.Struct`/`Schema.Class`, not a bare fields object; for GET it is read from the query string), `success`, `error` (one schema or an array). Path params use `:name` segments. The handler receives `{ params, query, payload, headers, request }`. Group-level helpers: `.prefix("/v1")`, `.middleware(M)`, `.annotateMerge(OpenApi.annotations({ title }))`.
- Error → status: attach it in the API definition with `E.pipe(HttpApiSchema.status(409))` (preferred — domain errors stay HTTP-free, per the onion rule; the same error can map to different statuses on different endpoints). `{ httpApiStatus: 401 }` as the third arg of `Schema.TaggedError` is acceptable only for errors that are defined in the HTTP/API module itself (e.g. `Unauthorized` below). An error with no status defaults to 500. `HttpApiSchema.asNoContent({ decode })` sends an error/success with an empty body; built-ins live in `HttpApiError` (`NotFound`, `Unauthorized`, ...). Request-decoding failures become a 400 automatically — with an **empty body** in v4 (no JSON `HttpApiDecodeError` payload); use `HttpApiMiddleware.layerSchemaErrorTransform` if clients need details.
- **Gotcha — handler requirements are per-request.** Services a handler effect *requires* become `HttpRouter` per-request requirements; `Layer.provide(AppLayer)` on the API layer does **not** satisfy them, and the program fails to type-check at `Layer.launch(...).pipe(NodeRuntime.runMain)` with `"OrderRepo" is not assignable to "never"`. Resolve services inside the group's build effect (as above: `yield* OrderRepo`, or capture `Effect.context<...>()` and `Effect.provideContext` in each handler) and provide the app layer to the *group* layer. Don't use `HttpRouter.provideRequest` for stateful layers — they are rebuilt per request.

Serving (canonical wiring — routes are layers merged and served by `HttpRouter.serve`):

```ts
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { Layer } from "effect"
import { HttpRouter } from "effect/http"
import { HttpApiBuilder, HttpApiSwagger } from "effect/http-api"
import { createServer } from "node:http"

const ApiRoutes = HttpApiBuilder.layer(Api, { openapiPath: "/openapi.json" }).pipe(
  Layer.provide(OrdersHandlers.pipe(Layer.provide(AppLayer)))   // your services/infra
)
const DocsRoute = HttpApiSwagger.layer(Api, { path: "/docs" })  // or HttpApiScalar.layer

const ServerLive = HttpRouter.serve(Layer.mergeAll(ApiRoutes, DocsRoute)).pipe(
  Layer.provide(NodeHttpServer.layer(createServer, { port: 3000 }))
)

Layer.launch(ServerLive).pipe(NodeRuntime.runMain)
```

Serverless / edge: `HttpRouter.toWebHandler(routes.pipe(Layer.provide(HttpServer.layerServices)))` returns `{ handler, dispose }` with `handler: (Request) => Promise<Response>`.

Auth middleware — a service that *provides* `CurrentUser`, so protected handlers can't be wired without it:

```ts
import { Context, Effect, Layer, Redacted, Schema } from "effect"
import { HttpApiMiddleware, HttpApiSecurity } from "effect/http-api"

export class CurrentUser extends Context.Service<CurrentUser, User>()("app/http/CurrentUser") {}

export class Unauthorized extends Schema.TaggedError<Unauthorized>()(
  "Unauthorized", { message: Schema.String }, { httpApiStatus: 401 }
) {}

// Definition — lives with the API (shared with clients); Unauthorized is an http/ error, so httpApiStatus on the class is fine
export class Authorization extends HttpApiMiddleware.Service<Authorization, {
  provides: CurrentUser
  requires: never
}>()("app/http/Authorization", {
  security: { bearer: HttpApiSecurity.bearer },
  error: Unauthorized
}) {}
// apply with .middleware(Authorization) on an endpoint or a whole group

// Implementation — a Layer on the server side only
export const AuthorizationLive = Layer.effect(Authorization, Effect.gen(function* () {
  const sessions = yield* SessionStore
  return Authorization.of({
    bearer: Effect.fn(function* (httpEffect, { credential }) {
      const user = yield* sessions.verify(Redacted.value(credential)).pipe(
        Effect.mapError(() => new Unauthorized({ message: "invalid or expired token" }))
      )
      return yield* Effect.provideService(httpEffect, CurrentUser, user)
    })
  })
}))
```

Handlers read the user with `yield* CurrentUser`. Provide `AuthorizationLive` to `HttpApiBuilder.layer(Api)` itself (the router resolves middleware when routes are built) — providing it only to the group layer is not enough. Add `requiredForClient: true` to the middleware options when the derived client must attach credentials; clients then provide `HttpApiMiddleware.layerClient(Authorization, ({ next, request }) => next(HttpClientRequest.bearerToken(request, token)))`.

OpenAPI comes free: `OpenApi.fromApi(Api)` for the spec, `openapiPath` on `HttpApiBuilder.layer` to serve it, `HttpApiSwagger.layer` / `HttpApiScalar.layer` for the UI.

Test handlers without a server — `HttpApiTest` builds a typed client wired straight to the handlers (same encoding, routing and decoding as production):

```ts
import { assert, it } from "@effect/vitest"
import { Effect, Layer } from "effect"
import { HttpServer } from "effect/http"
import { HttpApiTest } from "effect/http-api"

const TestLayer = Layer.mergeAll(
  OrdersHandlers.pipe(Layer.provide(InMemoryAppLayer)),
  HttpServer.layerServices
)

it.effect("placeOrder → 409 InsufficientStock", () =>
  Effect.gen(function* () {
    const client = yield* HttpApiTest.groups(Api, ["orders"])
    const error = yield* client.orders.placeOrder({ payload: oversizedOrder }).pipe(Effect.flip)
    assert.strictEqual(error._tag, "InsufficientStock")
  }).pipe(Effect.provide(TestLayer)))
```

## CLI — `effect/cli`

```ts
import { NodeRuntime, NodeServices } from "@effect/platform-node"
import { Effect } from "effect"
import { Argument, Command, Flag } from "effect/cli"

const verbose = Flag.Boolean("verbose").pipe(Flag.withAlias("v"), Flag.withDefault(false))
const input = Argument.File("input", { mustExist: true })

const processCmd = Command.make("process", { verbose, input }, ({ verbose, input }) =>
  processFile(input).pipe(Effect.tap((summary) => verbose ? logDetails(summary) : Effect.void))
)

Command.make("mytool").pipe(
  Command.withSubcommands([processCmd]),
  Command.run({ version: "1.0.0" }),
  Effect.provide(NodeServices.layer),   // FileSystem, Path, Stdio, Terminal, ChildProcessSpawner
  NodeRuntime.runMain
)
```

- Flags/Arguments are typed and validated: `Flag.String/Boolean/Int/Literals/File/Redacted/...`, `Argument.String/File/Directory/...`, refined with `Flag.withSchema(Email)` / `Argument.withSchema(...)`; `Flag.optional` yields an `Option`. `--help`, `--version`, `--completions`, parse errors and `--wizard` come free. `Command.run` reads argv from the `Stdio` service — no `process.argv` plumbing.
- Shared parent flags: `Command.withSharedFlags({ ... })` on the root; subcommand handlers read them with `const root = yield* rootCommand`.
- Same rules as services: workflows do the work; commands are thin adapters. File system via `FileSystem` (core `effect` module in v4 — a service, testable in memory), child processes via `effect/process` `ChildProcess`; never `node:fs` / `node:child_process` directly.
- Exit codes come from the error track: `runMain` exits nonzero on failure automatically.

## Full-stack

The prime directive: **one `packages/domain` shared by client and server** containing schemas, branded types, unions, error types — and the `HttpApi` definition itself. The wire contract is those schemas; client and server cannot drift.

```
packages/
  domain/     # schemas, types, pure functions, HttpApi definition — imports only `effect` / `effect/http-api`
  server/     # handlers + workflows + infra layers
  web/        # frontend
```

- Derive the client from the API definition — fully typed calls including typed error channels, no hand-written fetch code:
  ```ts
  import { Effect } from "effect"
  import { FetchHttpClient } from "effect/http"
  import { HttpApiClient } from "effect/http-api"

  const program = Effect.gen(function* () {
    const client = yield* HttpApiClient.make(Api, { baseUrl: "https://api.example.com" })
    const order = yield* client.orders.placeOrder({ payload })   // error channel: InsufficientStock | OutOfDeliveryArea | transport errors
    return order
  }).pipe(Effect.provide(FetchHttpClient.layer))
  ```
  Wrap it in a `Context.Service` with `HttpApiClient.ForApi<typeof Api>` as its shape; use `transformClient` for base URL, retries (`HttpClient.retryTransient`) and auth.
- Frontend state: keep the functional core. Effect runs in the browser; build one `ManagedRuntime` with browser layers (`@effect/platform-browser`, HttpClient, storage services) at app root. For React integration use `@effect/atom-react` (the v4 successor to `@effect-atom/atom-react`) or run workflows via the runtime from event handlers — components stay thin views; logic stays in Effect workflows, testable without a DOM.
- Validation is shared: the same `Email` schema powers the form field hint and the server rejection.

## Libraries / publishable packages

- Public API returns `Effect`/`Option`/`Result` with **exported, documented tagged errors** — the error union is part of semver.
- Depend on service tags, never concrete platforms: accept `HttpClient` from `effect/http` rather than bundling fetch/axios; consumers provide the platform layer (`FetchHttpClient.layer`, `NodeHttpClient`, ...). Zero Node-only imports unless the package is explicitly Node-only.
- Provide a `layer` (and `layerConfig`) as static members of the `Context.Service` class; expose the service so consumers can fake it in *their* tests.
- Peer-depend on `effect` (`^4`); never bundle a second copy (breaks service identity and instanceof checks). Companion packages (`@effect/platform-*`, `@effect/sql-*`, `@effect/ai-*`) are versioned in lockstep with `effect` — keep them on the same version.
- Ship tree-shakeable ESM, dual-package if needed; `sideEffects: false`.
- Property-test the public surface with arbitraries derived from your own schemas; the round-trip tests double as living documentation.

## Checklist

- [ ] HTTP/CLI modules imported from `effect/http`, `effect/http-api`, `effect/cli` — no `@effect/platform` / `@effect/cli` in an effect 4 project
- [ ] API definition (groups, endpoints, middleware tags; error statuses attached in the API definition via `HttpApiSchema.status`) in a shareable module with no server imports; domain errors carry no HTTP status
- [ ] Handlers only call workflows and map errors; services resolved in the group's build effect, not left as per-request requirements
- [ ] Auth via `HttpApiMiddleware.Service` that `provides` the current user; implementation is a server-only Layer
- [ ] Handlers tested through `HttpApiTest` with in-memory layers; OpenAPI served from the same definition
- [ ] CLIs: typed `Flag`/`Argument`, `Command.run` + `NodeServices.layer` + `NodeRuntime.runMain`; file/process access through `FileSystem` / `ChildProcess` services
- [ ] Clients derived with `HttpApiClient.make`; libraries depend on `HttpClient`, peer-depend on `effect`
