# Supplemental: LLM / AI Integration

Optional — read only when the app calls LLMs. The `@effect/ai` packages put model calls on the same railway as everything else: typed errors, retries by `Schedule`, streaming as `Stream`, providers behind Layers. **These packages evolve quickly — verify current module names against https://effect.website/docs/ai before use.**

## Principles (stable even as APIs move)

- **Provider as a service.** Program against the provider-agnostic interface from `@effect/ai`; supply a concrete provider layer (`@effect/ai-anthropic`, `@effect/ai-openai`) in `main.ts`. Swapping models/providers is a Layer change, not a refactor. Default to the latest Claude models for new apps.
- **LLM calls are fallible effects.** Network/rate-limit/content errors are tagged errors on the error track. Wrap every call with the standard external-call discipline from `production.md`: timeout, transient-only retry with backoff + jitter, span, metrics. Rate-limit with `RateLimiter`; bound concurrency with a `Semaphore`.
- **Structured output through Schema.** When you need data (not prose) from a model, request schema-shaped output and decode with the same domain schemas used everywhere else — "parse, don't validate" applies to model output more than anywhere, since it's untrusted by construction. On decode failure: retry with error feedback, bounded attempts, then a typed error.
- **Tools are schema-typed effects.** Tool-call parameters and results are Schemas; tool handlers are Effects with typed errors, running through your normal services — an LLM tool is just another adapter into the workflow layer.
- **Streaming is `Stream`.** Token/response streaming composes with the rest of the app (backpressure, interruption on client disconnect for free).
- **Test with fake layers.** A fake model layer returning canned/schema-generated responses makes agentic workflows unit-testable; keep a thin integration tier hitting a real model.
- **Cost & safety knobs are config.** Model id, max tokens, temperature via `Config`; API keys via `Config.redacted`. Track token usage as `Metric`s.

## Shape of the code

```ts
// Domain: what we want, typed
const Verdict = Schema.Struct({ sentiment: Schema.Literal("pos", "neg", "neutral"), confidence: Confidence })

// Workflow depends on the abstract model interface — no provider imports here
const classify = (text: ReviewText) =>
  generateObject({ schema: Verdict, prompt: classifyPrompt(text) }).pipe(
    Effect.retry({ schedule: backoffJittered, while: isTransient, times: 3 }),
    Effect.timeout("30 seconds"),
    Effect.withSpan("Ai.classify")
  )

// main.ts: pick the provider + model once
const AiLive = AnthropicProviderLayer({ apiKey: Config.redacted("ANTHROPIC_API_KEY"), model: "claude-opus-4-8" })
```

(Illustrative — resolve exact function/layer names from the current `@effect/ai` docs at implementation time.)
