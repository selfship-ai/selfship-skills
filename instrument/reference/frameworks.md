# Framework routes and gotchas

Load this file when the repo already emits AI spans, uses LangSmith, LiveKit, Vercel AI, or OpenInference, or when `init()` + `workflow()` would be a second mechanism. Pair with `reference/selfship_integrations.md` for the attribute contract.

The SelfShip OTLP gateway accepts **HTTP/protobuf only**. There is no gRPC listener.

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Use `bps_` ingest secrets here — never `ssa_` Agent secrets. Do not invent extra header names.

---

## Preserve existing OTel (do this first)

If a real user turn already produces `gen_ai.*`, `ai.*`, `llm.*`, `lk.*`, LangSmith, or MCP spans that form a conversation (user, session, turn I/O, tool children), **point the existing exporter at SelfShip and stop**.

Do not also add `selfship.langchain_callbacks()`, `CallbackHandler`, or a second `workflow()` wrapper around the same call (hard rule 10).

Add missing identity on the existing path when needed:

- stable `user_id` / `gen_ai.user.id` across conversations
- stable `conversation_id` / `session.id` / `langfuse.session.id` within one conversation
- `workflow:<key>` via existing tags/metadata if the framework already supports them

If OTel is configured but the real AI path emits no supported spans, add the framework's official instrumentation (below) or the smallest missing attributes. Only then fall back to SelfShip SDK `init()` + `workflow()`.

---

## LangChain / LangGraph — LangSmith OTel mode

Use this **only** when the repo already has LangSmith tracing (or the owner asked to keep it). Do not add LangSmith to a clean app just because TypeScript has no `langchainCallbacks` helper — that helper gap is hard rule 18, and the greenfield TS path is still `workflow()` / `resumeTrace()`.

This is **not** a second mechanism on top of `CallbackHandler`. If this path is live, do not also attach SelfShip callbacks.

### Environment (both languages)

```bash
LANGSMITH_TRACING=true
LANGSMITH_OTEL_ENABLED=true
LANGSMITH_OTEL_ONLY=true
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

`LANGSMITH_OTEL_ONLY=true` keeps spans on the OTel path only (no dual-write to LangSmith cloud). Omit it only when the customer explicitly wants both.

### Python

```bash
pip install "langsmith[otel]"
```

Set the env vars **before** the first LangChain import. LangSmith then uses the process `TracerProvider`. If the app has no provider yet, install one that exports HTTP/protobuf to `otel.selfship.ai` with the SelfShip headers above.

Pass identity on invoke — do not invent a second wrapper:

```python
result = graph.invoke(
    {"messages": messages},
    config={
        "metadata": {
            "session_id": conversation_id,
            "user_id": user_id,
        },
        "tags": [f"workflow:{workflow_key}"],
    },
)
```

If a bare `llm.invoke(...)` emits no span, wrap the entrypoint with `@traceable` from `langsmith` (or `traceable()`). Do not add `CallbackHandler`.

CLI / one-shot scripts: `force_flush()` the provider before exit. Do not call `shutdown()` on a long-lived server.

### TypeScript

```bash
npm install @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-proto
```

```ts
import { initializeOTEL } from "langsmith/experimental/otel/setup";

initializeOTEL({
  exporterConfig: {
    url: "https://otel.selfship.ai/v1/traces",
    headers: {
      "X-SelfShip-Org-ID": process.env.SELFSHIP_ORG_ID!,
      "X-SelfShip-Repo-Secret": process.env.SELFSHIP_REPO_SECRET!,
    },
  },
});
```

Also set `LANGSMITH_TRACING=true`, `LANGSMITH_OTEL_ENABLED=true`, and `LANGSMITH_TRACING_MODE=otel` (or `LANGSMITH_OTEL_ONLY=true` if the installed langsmith version documents that name).

```ts
await graph.invoke(input, {
  metadata: {
    session_id: conversationId,
    user_id: userId,
  },
  tags: [`workflow:${workflowKey}`],
});
```

If a bare runnable emits no span, wrap with `traceable()` from `langsmith`. There is no `langchainCallbacks` helper — do not invent one.

CLI / serverless: `forceFlush()` before the process exits. On warm Lambdas, flush — never `shutdown()`.

### When LangSmith OTel is the wrong tool

- Greenfield Python LangChain with no existing tracer → `selfship.init()` + `langchain_callbacks()` inside `workflow()` (mechanics table).
- Greenfield TypeScript LangChain with no LangSmith → nest `chain.invoke` inside `workflow()` / `resumeTrace()` (mechanics table). Do not add `langsmith` to get OTel.
- The real turn still has no generation after LangSmith OTel + `@traceable` → fall back to SDK `workflow()`, still without stacking both.

---

## Gotcha cards

### LiveKit Agents

Never set `LIVEKIT_OBSERVABILITY_URL` (or `observability_url=`) to SelfShip. That URL is LiveKit Cloud's own intake, not an OTLP sink. Pointing it at `otel.selfship.ai` silently drops traces.

Use `set_tracer_provider` (Python) or `telemetry.setTracerProvider` (JS) with an OTLP HTTP/protobuf exporter to `https://otel.selfship.ai` and the SelfShip headers. Keep LiveKit Cloud observability on its own URL if the customer uses it.

### Vercel AI SDK

`generateText` / `streamText` / `generateObject` honor `experimental_telemetry: { isEnabled: true }`. That is the supported emit path.

Do not invent an OTLP exporter for `OpenAIStream` / the legacy Pages-router streaming helper — it has no telemetry hook. Prefer the AI SDK functions above, or fall back to SelfShip SDK `observeOpenAI` / `workflow()` on that route only.

### OpenInference + OpenAI TypeScript

Pin OpenInference to the **same major** as the `openai` package. A v4 instrumentor on openai v5 (or the reverse) emits nothing. After the pin, route the existing provider to SelfShip — do not add a second OpenAI wrapper.

### Process lifetime

| Runtime | Flush | Shutdown |
|---|---|---|
| CLI / one-shot script | `force_flush()` / `forceFlush()` before exit | OK after flush |
| Long-lived server | periodic or request-end flush if the SDK requires it | never on the hot path |
| Warm Lambda / serverless | flush at the end of the invocation | **never** — it kills the reused provider |

### Transport

HTTP/protobuf only. `grpc`, `http/json`, and custom collector URLs are unsupported. Endpoint host is `otel.selfship.ai` — not `mcp.selfship.ai` and not a Langfuse cloud host.
