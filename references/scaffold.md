# Project Scaffold

Verified against Effect 3.22.0 (July 2026). **Target the Effect 3.x stable line.** Effect 4.0 is in beta and restructures packages (platform/cli merge into core) — do not use `4.0.0-beta.*` unless the user asks; re-verify this skill when 4.x goes stable.

## Option A: official generator

```bash
npx create-effect-app@latest            # interactive
npx create-effect-app -t basic my-app   # templates: basic | cli | monorepo
```

Templates ship pnpm, vitest, project-references tsconfig, `@effect/language-service`, changesets. Use `monorepo` for full-stack (shared `domain` package — see `app-shapes.md`).

## Option B: manual scaffold

### package.json (pin the family together)

Companion packages are 0.x and version-coupled to `effect` — **upgrade them together, never individually**.

```jsonc
{
  "type": "module",
  "packageManager": "pnpm@latest",
  "dependencies": {
    "effect": "^3.22.0",
    "@effect/platform": "^0.97.0",        // HTTP (note: HttpApi still flagged unstable)
    "@effect/platform-node": "^0.108.0",
    "@effect/opentelemetry": "^0.64.0"
    // per app shape: "@effect/cli": "^0.76.0", "@effect/sql": "^0.52.0", "@effect/sql-pg": "^0.53.0"
  },
  "devDependencies": {
    "@effect/vitest": "^0.30.0",
    "@effect/language-service": "^0.87.0",
    "@types/node": "^24.0.0",             // required: platform-node imports node:* modules
    "typescript": "^5.8.0",
    "vitest": "^3.0.0"
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

`@effect/language-service` adds Effect-aware diagnostics — **unexecuted floating Effects**, missing context, `any`/`unknown` leaking into error/requirement channels, outdated APIs — plus refactors (async fn → `Effect.gen`). In VS Code: "TypeScript: Select TypeScript Version" → **Use Workspace Version**, or the plugin silently doesn't run.

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
// src/config.ts
import { Config, LogLevel } from "effect"
export const AppConfig = {
  logLevel: Config.logLevel("LOG_LEVEL").pipe(Config.withDefault(LogLevel.Info)),
  environment: Config.literal("development", "staging", "production")("APP_ENV").pipe(
    Config.withDefault("development" as const)
  ),
}

// src/main.ts
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Layer } from "effect"

const AppLayer = Layer.mergeAll(/* live layers */)

NodeRuntime.runMain(program.pipe(Effect.provide(AppLayer)))
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
    "no-restricted-imports": ["error", { "paths": [
      { "name": "lodash", "message": "Effect's data modules only" }
    ]}],
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-non-null-assertion": "error"
  }
}
```

Scope exceptions narrowly with per-directory overrides (e.g. allow `Effect.tryPromise` wrappers in `src/services/`), never inline `eslint-disable` in domain code.

## Checklist

- [ ] Effect 3.x stable pinned; companion packages upgraded as a family
- [ ] tsconfig strict trio: `strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`
- [ ] `@effect/language-service` in plugins and workspace TS selected
- [ ] Directory skeleton matches services-layers.md; `main.ts` is the only `run*` site
- [ ] Lint rules enforcing the hard rules are active in CI
