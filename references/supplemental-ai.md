# Supplemental: LLM / AI Integration

Optional — read only when the app calls LLMs. Effect's AI modules put model calls on the same railway as everything else: typed errors, retries by `Schedule`, streaming as `Stream`, providers behind Layers. In v4 the provider-agnostic core lives in `effect/ai` (`LanguageModel`, `Tool`, `Toolkit`, `Chat`, `Prompt`, `AiError`, `EmbeddingModel`, `McpServer`, ...); providers are separate packages versioned with `effect` (`@effect/ai-anthropic`, `@effect/ai-openai`). `@effect/ai` is v3-only. Verified against effect 4.0.1 / `@effect/ai-anthropic` 4.0.1 (October 2026); the modules are tagged `@stability unstable` and move quickly — re-check names against `node_modules/effect/ai-docs/src/71_ai` before use.

## Principles (stable even as APIs move)

- **Provider as a service.** Program against `LanguageModel.LanguageModel` from `effect/ai`; supply a concrete provider layer (`AnthropicLanguageModel.layer({ model })` + `AnthropicClient.layerConfig(...)` from `@effect/ai-anthropic`, or the `@effect/ai-openai` equivalents) in `main.ts`. Swapping models/providers is a Layer change, not a refactor; `ExecutionPlan` expresses provider fallback chains declaratively. Default to the latest Claude models for new apps.
- **LLM calls are fallible effects.** Network/rate-limit/content errors arrive as `AiError` on the error track, with a classified `reason` (`RateLimitError`, `NetworkError`, `ContentPolicyError`, `StructuredOutputError`, ...) and an `isRetryable` flag. Wrap every call with the standard external-call discipline from `production.md`: timeout, transient-only retry (`while: (e) => e.isRetryable`) with backoff + jitter, span, metrics. Rate-limit with `RateLimiter`; bound concurrency with a `Semaphore`. Map `AiError` into your own tagged error at the service boundary (its `reason` is a Schema, so it can be embedded).
- **Structured output through Schema.** When you need data (not prose) from a model, use `LanguageModel.generateObject({ schema, prompt })` with the same domain schemas used everywhere else — "parse, don't validate" applies to model output more than anywhere, since it's untrusted by construction. On decode failure: retry with error feedback, bounded attempts, then a typed error.
- **Tools are schema-typed effects.** `Tool.make(name, { parameters, success, failure })` — parameters and results are Schemas; group them with `Toolkit.make(...)` and implement handlers with `toolkit.toLayer(...)` as Effects with typed errors, running through your normal services — an LLM tool is just another adapter into the workflow layer. The same toolkit can be exposed over MCP via `McpServer`.
- **Streaming is `Stream`.** `LanguageModel.streamText(...)` returns a `Stream` of response parts; it composes with the rest of the app (backpressure, interruption on client disconnect for free). Multi-turn state lives in `Chat`.
- **Test with fake layers.** A fake `LanguageModel` layer returning canned/schema-generated responses makes agentic workflows unit-testable; keep a thin integration tier hitting a real model.
- **Cost & safety knobs are config.** Model id, max tokens, temperature via `Config`; API keys via `Config.Redacted`. Track token usage (`response.usage`) as `Metric`s.

## Shape of the code

```ts
import { AnthropicClient, AnthropicLanguageModel } from "@effect/ai-anthropic"
import { Config, Effect, Layer, Schedule, Schema } from "effect"
import { LanguageModel } from "effect/ai"
import { FetchHttpClient } from "effect/http"

// Domain: what we want, typed
const Verdict = Schema.Struct({ sentiment: Schema.Literals(["pos", "neg", "neutral"]), confidence: Confidence })

// Workflow depends on the abstract LanguageModel service — no provider imports here
export const classify = Effect.fn("Ai.classify")(
  function* (text: ReviewText) {
    const response = yield* LanguageModel.generateObject({
      objectName: "verdict",
      schema: Verdict,                      // output decoded through the domain schema
      prompt: classifyPrompt(text),
    })
    return response.value
  },
  Effect.retry({ schedule: backoffJittered, while: (error) => error.isRetryable, times: 3 }),
  Effect.timeout("30 seconds")
)

// main.ts: pick the provider + model once
const AnthropicLive = AnthropicClient.layerConfig({
  apiKey: Config.Redacted("ANTHROPIC_API_KEY"),
}).pipe(Layer.provide(FetchHttpClient.layer))

export const AiLive = Layer.unwrap(Effect.gen(function* () {
  const model = yield* Config.String("AI_MODEL").pipe(Config.withDefault("claude-opus-5-5"))
  return AnthropicLanguageModel.layer({ model })
})).pipe(Layer.provide(AnthropicLive))
```

(Imports only `effect/ai` in workflows; provider packages appear only in the composition root. In a real service, wrap `classify` in a `Context.Service` and map `AiError` to a domain error before it leaves the AI adapter.)
