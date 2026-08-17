## TypeScript mechanics (`@selfship-ai/sdk>=0.3.9 <0.4.0`)

SDK version floor: `@selfship-ai/sdk` `>=0.3.9 <0.4.0` (0.3.9 guarantees a non-empty `statusMessage` on ERROR via `observe` / `setError`; 0.3.5 fixed dropped `userId`/`conversationId`/`tags` on workflow traces; 0.3.6 adds the public trace-context API; 0.3.8 pins the I/O mirror to the workflow span and re-asserts at exit).

```typescript
import { init, workflow } from "@selfship-ai/sdk";

init({ providerMode: "standalone" });

const answer = await workflow(
  {
    name: "chat-turn",
    userId,
    conversationId,                       // → trace session
    tags: [`workflow:${workflowKey}`],    // trace-level tags, set directly
    metadata: { workflow_key: workflowKey },
  },
  async (handle) => {
    const result = await runAgent(...);
    if (result.error) {
      // Soft failure that returns normally: auto setError only fires on throws.
      handle.setError(result.error);
      return result;
    }
    handle.setOutput(result); // full user-visible deliverable — never a preview/receipt
    // Mid-run enrichment: handle.update({ metadata: ... })
    return result;
  },
);
```

- Prefer `workflow()` for fail-open request scoping
- **Handle surface:** `setOutput(output)`, `update({ metadata, output })`, `setError(error)`, `traceId`, `observationId`, `traceContext()`. Use `setError` for soft failures (status-field errors that return normally) — auto-`setError` only fires on thrown errors. Ties to hard rule 11 (record errors, don't leak spans). Across a thread/queue/worker boundary: `currentTraceContext()` / `resumeTrace(ctx, fn)`. There is no `langchainCallbacks` helper in the TypeScript SDK.
- Do not construct `Langfuse` / `LangfuseSpanProcessor` in customer code
- Model calls: framework wrappers that nest under the active SelfShip observation

### `observe` (secondary — not the root enrichment API)

TypeScript `observe` is **function-first** and returns a **wrapped function** (it does not run the body):

```typescript
import { observe } from "@selfship-ai/sdk";

// Correct
const run = observe(async () => anthropic.messages.create(params), {
  name: "anthropic.messages.create",
});
const response = await run();

// WRONG — options-first like workflow(); TypeScript types this as AsyncFn (Ronin bug)
// const response = await observe({ name: "…" }, async () => …);
```

Prefer `workflow({...}, async (handle) => …)` for request scoping. Never call `observe` with an options object as the first argument.

### Framework → integration mechanism

Pick the best mechanism per workflow's `framework`:

| Framework family | Preferred mechanism |
|---|---|
| LangChain (TypeScript) | If LangSmith OTel already forms a usable turn: `initializeOTEL` + invoke metadata (`reference/frameworks.md`). There is no `langchainCallbacks` helper — this is the supported JS LangChain path. Greenfield without LangSmith: nest `chain.invoke` inside `workflow()` / `resumeTrace()` so the OTel parent is ambient. Inside a running LangGraph node, reuse `config.callbacks` (the live `CallbackManager`) — a fresh `CallbackHandler()` leaves `parent_run_id` unset and Langfuse elevates that run's I/O onto the trace. Do not construct `CallbackHandler` with a hand-rolled `{ traceId, parentSpanId }` from `handle.traceContext()`; SelfShip spells the parent `observationId`, Langfuse expects `parentSpanId`. |
| OpenAI (TypeScript) | Nest `observeOpenAI` inside `workflow()` so it inherits the active observation. There is no `client.trace()` / `{ parent: trace }` factory. Pin OpenInference to the same `openai` major when that instrumentor is already present (`reference/frameworks.md`). |
| Vercel AI SDK | `experimental_telemetry: { isEnabled: true }` on `generateText` / `streamText` / `generateObject`. Do not invent OTLP for `OpenAIStream`. Then OTel env vars → `otel.selfship.ai`. |

**OTel fallback (no SDK):**
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```
