# Project Scaffold

Verified against effect 4.0.1 (October 2026). **Target the Effect 4.x line.** v4 folded `@effect/platform`, `@effect/cli`, `@effect/sql` and `@effect/ai` into `effect` itself (`effect/http`, `effect/http-api`, `effect/cli`, `effect/sql`, `effect/ai`, …) — never install those four packages in a v4 project. Never mix v3 and v4 packages in one dependency tree.

## Option A: manual scaffold (default)

`create-effect-app` (0.0.6) still generates **Effect 3.x** templates as of October 2026. Prefer the manual scaffold below; if you do use the generator (`npx create-effect-app -t basic|cli|monorepo`), immediately bump its `package.json` to the versions below and delete any `@effect/platform`/`@effect/cli`/`@effect/sql` dependency.

### package.json (pin the family together)

The `effect` family is released in lockstep: `effect` and every `@effect/*` runtime/driver package (`platform-node`, `platform-bun`, `sql-pg`, `sql-sqlite-node`, `vitest`, `opentelemetry`, `ai-anthropic`, …) share one version number. **Pin them to the same exact version and upgrade them together, never individually** — a mismatch loads two copies of `effect` and breaks service identity.

```jsonc
{
  "type": "module",
  "packageManager": "pnpm@latest",
  "dependencies": {
    "effect": "4.0.1",                    // includes effect/http, http-api, cli, sql, ai
    "@effect/platform-node": "4.0.1"      // NodeRuntime, NodeHttpServer, NodeServices
    // per app shape: "@effect/sql-pg": "4.0.1" | "@effect/sql-sqlite-node": "4.0.1"
    // existing OTel setup only: "@effect/opentelemetry": "4.0.1" (new apps: effect/observability Otlp)
  },
  "devDependencies": {
    "@effect/vitest": "4.0.1",            // peer: vitest >=5 <6
    "@effect/language-service": "^0.87.3",
    "@types/node": "^24.0.0",             // required: platform-node imports node:* modules
    "tsx": "^4.23.0",
    "typescript": "~6.0.3",               // see "TypeScript 6 vs 7" below
    "vitest": "^5.0.3"
  },
  "scripts": {
    "dev": "tsx watch src/main.ts",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "build": "tsc -b"
  }
}
```

### tsconfig.json — strictness is part of the skill

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "plugins": [{ "name": "@effect/language-service" }]
  },
  "include": ["src"]
}
```

`@effect/language-service` adds Effect-aware diagnostics — **unexecuted floating Effects**, missing context, `any`/`unknown` leaking into error/requirement channels, `outdatedApi` (v3 APIs removed or renamed in v4) — plus refactors (async fn → `Effect.gen`). In VS Code: "TypeScript: Select TypeScript Version" → **Use Workspace Version**, or the plugin silently doesn't run.

In 4.0.1 every subpath module (`effect/http`, `effect/http-api`, `effect/sql`, `effect/cli`, `effect/ai`, `effect/schema`) is tagged `@stability unstable` — APIs may change in minor releases, so re-verify those call sites on every upgrade. On TS 7, `@effect/tsgo`'s `unstableApiUsage` diagnostic flags them; acknowledge the modules the app deliberately depends on with `"allowedUnstableApis": ["effect/http", "effect/http-api", "effect/sql"]` rather than disabling the rule.

### TypeScript 6 vs 7

| | TS 6.0 (default) | TS 7.0 (native `tsgo`) |
|---|---|---|
| Effect diagnostics | `@effect/language-service` plugin | `@effect/tsgo` (`npx @effect/tsgo setup`) — use it *instead of* plain `tsgo` |
| ESLint / typescript-eslint | supported | **not yet** — typescript-eslint 8.x peers `typescript <6.1` |
| Speed | baseline | much faster typecheck (native compiler) |

Default to TS 6.0 because the lint enforcement layer below requires typescript-eslint. Choose TS 7 + `@effect/tsgo` only if the team accepts moving the hard-rule lint checks elsewhere (e.g. tsgo's Oxlint integration); the application code is identical under both.

### vitest.config.ts

```ts
import { defineConfig } from "vitest/config"
export default defineConfig({
  test: { include: ["test/**/*.test.ts"], globals: false },
})
```

### Directory skeleton + starter files

```
src/
  domain/        # schemas, brands, unions, errors, pure functions — imports "effect" only
  workflows/     # use-cases over domain + service interfaces
  services/      # tags + live layers (infra)
  http/          # or cli/ — thin adapters
  config.ts      # every Config declaration
  main.ts        # the only entry point
test/
```

```ts
// src/config.ts — v4 Config constructors are Schema-backed and PascalCase
import { Config } from "effect"
export const AppConfig = {
  logLevel: Config.LogLevel("LOG_LEVEL").pipe(Config.withDefault("Info" as const)),
  environment: Config.Literals(["development", "staging", "production"], "APP_ENV").pipe(
    Config.withDefault("development" as const)
  ),
}

// src/main.ts
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Layer, References } from "effect"
import { AppConfig } from "./config.js"

const AppLayer = Layer.mergeAll(OrderRepo.layer, PaymentGateway.layer) // every live layer

const main = Effect.gen(function* () {
  const level = yield* AppConfig.logLevel
  return yield* program.pipe(Effect.provideService(References.MinimumLogLevel, level))
})

NodeRuntime.runMain(main.pipe(Effect.provide(AppLayer)))
// HTTP apps: Layer.launch(ServerLive).pipe(NodeRuntime.runMain) — see app-shapes.md
```

### Lint enforcement of the skill's hard rules

Make the rules mechanical, not aspirational — ESLint with `no-restricted-syntax` so violations fail CI:

```jsonc
// eslint flat config — the enforcement layer for SKILL.md hard rules
{
  "rules": {
    "no-restricted-syntax": ["error",
      { "selector": "ThrowStatement", "message": "Return a tagged error via the Effect error channel" },
      { "selector": "TryStatement", "message": "Use Effect.try / Effect.tryPromise at the interop edge" },
      { "selector": "FunctionDeclaration[async=true], ArrowFunctionExpression[async=true]",
        "message": "Use Effect.gen instead of async functions" },
      { "selector": "CallExpression[callee.object.name='console']", "message": "Use Effect.log*" },
      { "selector": "MemberExpression[object.object.name='process'][object.property.name='env']",
        "message": "Use Config in config.ts" }
    ],
    "no-restricted-imports": ["error", {
      "paths": [
        { "name": "lodash", "message": "Effect's data modules only" },
        { "name": "@effect/platform", "message": "Effect v4: import from effect/http, effect/http-api, or effect" },
        { "name": "@effect/sql", "message": "Effect v4: import from effect/sql" },
        { "name": "@effect/cli", "message": "Effect v4: import from effect/cli" },
        { "name": "@effect/schema", "message": "Schema lives in effect" }
      ]
    }],
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-non-null-assertion": "error"
  }
}
```

Scope exceptions narrowly with per-directory overrides (e.g. allow `Effect.tryPromise` wrappers in `src/services/`), never inline `eslint-disable` in domain code.

## Checklist

- [ ] `effect` 4.x and every `@effect/*` package pinned to the same exact version; upgraded as a family
- [ ] No `@effect/platform`, `@effect/cli`, `@effect/sql`, `@effect/ai`, `@effect/schema` dependencies (v3-era packages)
- [ ] vitest 5 with `@effect/vitest` 4.x
- [ ] tsconfig strict trio: `strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`
- [ ] `@effect/language-service` in plugins and workspace TS selected (or `@effect/tsgo` on TS 7); deliberate unstable modules listed in `allowedUnstableApis`
- [ ] Directory skeleton matches services-layers.md; `main.ts` is the only `run*` site
- [ ] Lint rules enforcing the hard rules are active in CI
