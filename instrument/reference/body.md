# Instrument — SelfShip Integration

Single skill for **all** instrumentation work: initial, sync (add/remove), and verify/fix mode.

Load **`reference/langfuse_best_practices.md`** for contextual tracing config (sessions, identity, tags, metadata, observation types, token/cost, feedback, releases).

Load **`reference/frameworks.md`** when the repo already emits AI spans, uses LangSmith, LiveKit, Vercel AI, or OpenInference — OTel-first routing, LangSmith OTel recipe, and framework gotchas.

Credentials: `SELFSHIP_ORG_ID` + `SELFSHIP_REPO_SECRET` (per-repo secret). Optional `SELFSHIP_REPO=owner/repo` for `metadata.repo` enrichment.

---

## Modes

| Mode | Job |
|---|---|
| **initial** | Inspect existing telemetry first. Preserve a usable OTel/LangSmith path; otherwise global `init()` + instrument selected workflows |
| **sync** | Add selected-but-missing; **remove** deselected-but-instrumented (active/started only); **verify** already-instrumented selected workflows |
| **fix / verify** | Audit `to_verify` workflows for gaps; apply minimal fixes |

---

## Global tracing config (app-wide, once)

- Call `init()` at the real process entry point per the SDK contract **unless** this process only redirects an existing OTel/LangSmith exporter (then skip SDK `init()`)
- `.env.example`: `SELFSHIP_ORG_ID`, `SELFSHIP_REPO_SECRET`, optional `SELFSHIP_REPO`
- `SELFSHIP_ENV=production` (commented; SDK defaults to `development`)
- `LANGFUSE_RELEASE=` (commented; git SHA in CI)
- Do **not** hardcode environment in code

---

## Quality pre-edit (mandatory)

Before editing, inspect neighboring tests, dependency manifests, package scripts / `pyproject.toml`,
formatter and linter config, and repository guidance. Preserve logging and serialization behavior.
Add tests in the established location and style for every non-trivial tracing helper. Report commands
actually run and unavailable commands separately; never install packages to run quality checks.

---

## Inspect existing telemetry first (mandatory)

Before adding SelfShip SDK `init()` / `workflow()` / `langchain_callbacks()`, inspect the real AI entrypoint:

- OpenTelemetry provider/exporter, `OTEL_EXPORTER_OTLP_*`, `@vercel/otel`, OpenInference instrumentors
- `LANGSMITH_OTEL_ENABLED`, `LANGSMITH_OTEL_ONLY`, `LANGSMITH_TRACING`, `langsmith[otel]`, `initializeOTEL`
- Vercel AI `experimental_telemetry`, LiveKit `set_tracer_provider` / `telemetry.setTracerProvider`
- Representative spans from one real interaction (`gen_ai.*`, `ai.*`, `llm.*`, `lk.*`, LangSmith, MCP)

**Preserve the existing path** when those spans already form a usable conversation (user, session, turn I/O, tool children). Route the exporter to SelfShip — do not install a second mechanism (hard rule 10). See `reference/frameworks.md`.

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

The gateway accepts HTTP/protobuf only. There is no gRPC listener.

If OTel exists but the real AI path emits no supported spans, add the framework's official instrumentation or the smallest missing attributes (`reference/frameworks.md`). Only then fall back to SelfShip SDK `init()` + `workflow()`.

If you only redirect an existing exporter, do **not** also call `init()`. `provider_mode` isolated/shared (below) applies only when you add the SelfShip SDK.

Treat automatic OTel detection as a candidate, not proof. Validate one real turn (below) before declaring the route usable.

---

## Coexisting with existing telemetry (`depth_detail.observability_stack`)

Complex apps often already run OpenTelemetry (Sentry, Datadog, Logfire, `opentelemetry-sdk`, OTel auto-instrumentation). OTel uses **one global `TracerProvider` per process**. Our SDK floor is `langfuse>=4,<5` (Python v4 / JS v5), which has **no default span filter** — whatever provider our processor sees, it exports. Two failure modes:

- **Another tool owns the global provider** (e.g. `sentry_sdk.init`, Datadog). A naive `init()` attaches nothing → **zero LLM traces**, silently.
- **We attach to an auto-instrumented global provider** → HTTP/DB/framework spans get exported as **billable units** and clutter every trace.

Use `depth_detail.observability_stack.recommended_mode`:

| Mode | When | What to do |
|---|---|---|
| `clean` | No existing telemetry | Default `init()` is fine |
| `isolated` | Another tool owns the global provider, **or** auto-instrumentation would flood non-LLM spans | Use standalone provider mode; do not replace the app global provider |
| `shared_filtered` | We can add to the existing provider and want one distributed trace | Use shared provider mode with the explicit app-owned provider and an export filter |

Never do a bare global `init()` when `recommended_mode != "clean"`. Use the public `workflow()` API in the same context as the LLM call to avoid orphaned spans.

---

## Streaming responses (`depth_detail.response_streaming`)

When a path streams its LLM output (SSE, `StreamingResponse`, async generator, `stream=True`), the **generation must end after the stream is fully drained**, not when the handler returns. Ending early loses `output` and token usage (looks like a dark generation). Consume the stream **inside** the traced scope, or end the observation in the generator's `finally` / on completion. Native framework wrappers handle this only if the stream is consumed within the traced scope.

---

## Process lifetime & flushing (`depth_detail.process_lifetime`)

- `long_lived` / `request` (daemons, web servers): the SDK flushes periodically — no explicit flush needed.
- `serverless` (Lambda/Vercel/Cloudflare): call **`flush()` / `forceFlush()` per invocation** (or `waitUntil` / Vercel `after`). Do **not** call `shutdown()` — it tears down the client so warm invocations stop tracing, and `atexit` does not fire in Lambda.
- `batch` / CLI / one-shot: call `shutdown()` (flush + terminate) before exit.

SelfShip treats a trace as finished when the **root observation has a non-null `endTime`**. `flush()` / `shutdown()` deliver buffered spans; they do not close an open root. End the root span (`workflow()` context / `end()`) before flushing.

---

## Workflow inventory fields

Workflows may include an `architecture` field from depth exploration — a tech narrative of end-to-end wiring and path coverage. Use it when present to drive instrumentation coverage. `description` and `product_objective` remain product-level context.

## Contextual config (per workflow)

Per `reference/langfuse_best_practices.md`:

- **Trace tag:** `workflow:<workflow_key>` on every trace for this workflow
- **Metadata:** `workflow_key: "<key>"` on root trace
- Identity: real `user_id` / `conversation_id` → `session_id`; never fabricate
- Tags: 1–3 low-cardinality feature tags
- Metadata: model, prompt_version, request_id, tenant, etc. (discovery checklist)
- Observation types: `tool` for tools, `generation` for model calls, `span` for orchestration
- Usage pattern: request / threaded / scheduled — shape traces accordingly

---

## Sync / removal

When removing instrumentation for a deselected workflow:
- Locate via `workflow:<workflow_key>` tag / `metadata.workflow_key` markers
- Remove only that workflow's contextual instrumentation
- **Preserve global config** (`init()`, `.env.example`) if any workflow remains selected
- Do not change behavior of wrapped code

---

## Verify / fix mode (`to_verify`)

For each workflow in `to_verify` (selected + `started`/`active`):

1. Locate existing instrumentation via `workflow:<key>` tags / `metadata.workflow_key`
2. Audit against this skill's checklist (global config, tags, mechanism, path coverage, contextual config)
3. If gaps found (missing tags/metadata, wrong mechanism, dark segments, or missing `init()` **on an SDK route**):
   - Apply the **minimal** fix
   - Re-stamp `workflow:<key>` tags
   - Do not treat missing `init()` as a gap when the chosen route is preserved OTel/LangSmith
4. If already correct: leave unchanged

When only `to_verify` work remains and no code changes are needed, return `"confidence": "none"`.

Trace-eval findings (`tool_not_scoped`, missing tags/metadata, dark segments) may also drive fixes when present in the prompt context.

### Real-turn validation (after edits)

Inventory-tag presence is not enough. When you can run the app and MCP is configured (customer/local skill):

1. Restart the process so new env / provider wiring loads.
2. Drive **one real** chat or tool call (not a mocked unit test).
3. Confirm the turn with `traces.list` then `traces.get`. Pass `environment` matching the app (`SELFSHIP_ENV` / `LANGFUSE_TRACING_ENVIRONMENT`). MCP defaults to **production**; the SDK defaults to **development** when `SELFSHIP_ENV` is unset. An empty production list is not proof the route failed — query the env you actually emit to before changing code.
4. `sessions.get` requires `user_key` (and `session_id` or `trace_ids`). Do not call it bare.

Check:

- `user_id` is stable across two conversations from the same user
- `conversation_id` / session is stable within one conversation and different across two
- tools appear as children of the turn, not sibling roots
- the generation carries token/usage when the provider reports it
- nested LangChain/LangGraph runs do **not** stamp `is_langchain_root` or overwrite trace I/O (hard rule 19)

If you cannot run a real turn (hosted first-PR sandbox, no MCP, no credentials): do **not** invent a live call and do **not** add a second mechanism. Leave real-turn as a followup; code audit + tags remain the verify path.

If the preserved OTel path fails this checklist, add the missing attributes on that path first. Do not stack `CallbackHandler` on top.

---

## Hard rules (non-negotiable)

1. **Server-side only:** instrument server/worker/CLI runtimes only. Never edit browser/client bundles, `"use client"` modules, or paths in `depth_detail.client_side_paths`. Never use `NEXT_PUBLIC_*` for SelfShip credentials (`SELFSHIP_ORG_ID` / `SELFSHIP_REPO_SECRET` on the server only). If this partition's only remaining surfaces are client-side, return `capability_gap` with an empty diff.
2. **Identity:** pass real nullable `user_id` / `conversation_id` directly to `workflow()` / `begin()` — SDK **0.3.3+** normalizes UUIDs internally. Never coerce absent identity to the string `"None"`. Do **not** invent `session_id=` on SelfShip APIs (use `conversation_id`). Direct `propagate_attributes` / framework callback identity must be string-safe or use the SDK entrypoints. Keep callback metadata and SDK calls consistent.
3. **Shared helpers (SelfShip forbid):** do **not** attach a Langfuse `CallbackHandler` or unconditional `@observe` on shared LLM builders / factories used by callers outside the sync set. Gate on active workflow context, or instrument only selected call sites. List unselected callers in `followups` — never silently expand instrumentation scope.
4. **Multi-caller cores:** set `workflow_key` / `workflow:<key>` at each **call site**, not as a single constant inside a shared function reused by multiple workflows.
5. **Threads:** Python contextvars and OTel async-local storage do **not** cross `ThreadPoolExecutor`, queue hand-offs, or worker processes. Prefer opening the root observation in the same execution context as the LLM work. When the boundary is unavoidable, carry context with the SDK's first-class API — `selfship.current_trace_context()` on the producing side and `selfship.resume_trace(ctx)` (TS: `currentTraceContext()` / `resumeTrace(ctx, fn)`) on the far side — rather than hand-rolling `contextvars.copy_context()` or `ThreadingInstrumentor`. If the far side is a LangChain invoke that has **no LangChain parent**, pass the context straight to the callbacks: `selfship.langchain_callbacks(trace_context=ctx)` (Python; see rule 18). If the far side is already inside a LangChain/LangGraph node, reuse the parent `CallbackManager` instead of a fresh handler list (see rule 19). Never assume nesting across threads: lost context leaves the root trace with **zero observations**, which reads as an empty trace even though the work ran.
6. **Init:** `init()` is **idempotent and thread-safe** — the SDK locks internally and auto-registers shutdown on the enabled path, so long-lived workers need no manual `shutdown()`. Call it directly at each real entry point; do **NOT** add your own init flag/lock or a custom tracing wrapper module. Put SDK imports at module top level (hard dependency). Confirm SDK calls no-op when credentials are absent (not only on `ImportError`).
7. **Existing telemetry:** when `observability_stack.recommended_mode != "clean"`, never do a bare global `init()`. Use public `provider_mode="standalone"` or `provider_mode="shared"` with the explicit app provider and filter; never wire raw Langfuse processors in customer code.
8. **Streaming:** for `response_streaming` paths, end the generation **after** the stream drains (consume inside the traced scope / end in `finally`), never when the handler returns.
9. **Flush vs shutdown:** `serverless` → per-invocation `flush()`/`forceFlush()` (never `shutdown()`); `batch`/CLI → `shutdown()` before exit; long-lived servers → rely on periodic flush. `flush()` and `shutdown()` are **fail-open** (never raise in non-strict mode) — call them bare in `finally`/teardown with a one-line comment (`# fail-open: never raises` or `// fail-open: never raises`); never wrap in try/except.
10. **No double instrumentation:** never stack more than one integration on the same call — it double-counts generations and cost. One mechanism per call site.
11. **Span lifecycle:** open observations with context managers / `try…finally` so they close on exceptions; record errors (`level=ERROR` + a **non-empty** status message) rather than leaking open spans. Never set `level=ERROR` without a reason. Soft failures (timeout, empty completion, retry) must call `handle.set_error(...)` / `setError(...)` with a concrete reason **or** go through SDK `observe` / `langchain_callbacks` (0.3.9+) so the failed attempt keeps a message. Do not delete or downgrade the failed span after a retry — Structure stays strict; the message is what makes the gap useful. Do not leave generations open on cancellation/timeout. **Python:** `workflow()` and `resume_trace()` are context managers — never `with selfship.observe|begin|track` (`observe` is a decorator).
12. **Async background tasks:** `asyncio.create_task`, `gather`, and framework background tasks (e.g. FastAPI `BackgroundTasks`) capture context at **creation** time — if the root ends first, children orphan. Open the root in the same task, or carry context explicitly with `current_trace_context()` / `resume_trace()` (TS: `currentTraceContext()` / `resumeTrace()`).
13. **Lint hygiene:** no unused imports; match repo import style. Customer CI lint must pass on changed files.
14. **Test authenticity:** never mock the whole SDK. Add at least one test that imports the **real** SDK and asserts the workflow body still runs/returns in disabled mode (no `SELFSHIP_ORG_ID`/`SELFSHIP_REPO_SECRET`). Mocking the SDK hides fail-open and signature regressions. Do **not** rely on AST-only wiring tests — the changed `workflow()`/`begin()` call site must be **invoked**.
15. **`get_client()` / `getClient()`:** prefer `workflow()` for identity/tags/metadata/`release` enrichment. Use `get_client()` only for scores, flush helpers, and advanced Langfuse client ops after a successful `init()`. Never invent `release`/`session_id`/`version` kwargs on SelfShip `workflow`/`init`. `update_current_trace()` was removed in Langfuse Python v4 — use `workflow(metadata=...)` / `propagate_attributes(...)` / `LANGFUSE_RELEASE`. TypeScript `getClient()` is the SelfShip runtime, not a trace factory (`client.trace()` does not exist).
16. **Output completeness:** root `set_output` / `end(output=...)` must carry the full, verbatim user-visible deliverable — the same value (post-sanitization, post-fallback) that is sent/persisted for the user. Never a slice, preview, length, or status receipt; status fields go in metadata. Compute the deliverable once and pass the same variable to the send path and `set_output`; scope the root span to cover output finalization — duplicated answer-selection logic in the trace path is a smell. On streamed paths, set the root output to the fully drained answer. The metadata/tags "≤ 200 chars / truncate" guidance never applies to root output.
17. **Never read SDK internals:** to get the current trace or span id, use `handle.trace_id` / `handle.observation_id` / `handle.trace_context()` or the module-level `current_trace_id()` / `current_observation_id()` / `current_trace_context()` (TS: `handle.traceId` / `currentTraceId()` / `currentTraceContext()` / `resumeTrace()`). Never touch `handle._span` or any other underscore-prefixed attribute — those are private, unsupported, and change between releases.
18. **LangChain across a boundary:** bare `langchain_callbacks()` (Python) attaches to whatever span is active when the chain runs, so a chain invoked from a thread pool, an `asyncio` hand-off, or a worker finds no parent, becomes its own LangChain root, and **overwrites the trace's input/output**. Capture the context on the producing side and pass it in: `selfship.langchain_callbacks(trace_context=wf.trace_context())`. TypeScript has no `langchainCallbacks` helper — nest `chain.invoke` inside `workflow()` / `resumeTrace()`, and do not construct a fresh `CallbackHandler` on the far side of a hop. Do not hand-roll Langfuse's `{"trace_id", "parent_span_id"}` dict or construct `CallbackHandler` directly in Python — the SDK translates SelfShip's `observation_id` to Langfuse's `parent_span_id`.
19. **Never replace callbacks inside a running LangChain/LangGraph node.** `trace_context=` parents the OTel span; LangChain's `parent_run_id` lives on the live `CallbackManager`. Rebuilding `RunnableConfig` with a fresh handler list — even `langchain_callbacks(trace_context=ctx)` or a new `CallbackHandler()` — leaves `parent_run_id=None`. Langfuse then stamps `is_langchain_root` on the nested observation and **elevates that run's I/O onto the trace**, so the dashboard shows a judge prompt where the user query was. Inside a graph node, reuse `parent_config["callbacks"]` (the manager, not a new list). Use `langchain_callbacks(trace_context=...)` only when starting a chain that has no LangChain parent. Thread the same `config=` through structured-output repair / retry invokes so those do not become roots either.

## Checklist

- [ ] Global config: `init()`, `.env.example` with creds + `SELFSHIP_ENV`
- [ ] Server-side only (no `"use client"` / browser bundles / `NEXT_PUBLIC_*` SelfShip creds)
- [ ] Per-workflow: `workflow:<key>` tag + `metadata.workflow_key`
- [ ] Framework-appropriate integration mechanism
- [ ] Full path coverage (no dark segments)
- [ ] Contextual config per `langfuse_best_practices.md`
- [ ] Removal preserves global config when other workflows selected
- [ ] `to_verify` workflows audited; gaps fixed or confirmed OK
- [ ] Sync-set equality (no extras); shared helpers gated; thread boundaries respected
- [ ] Existing telemetry respected: isolated/filtered provider when `recommended_mode != "clean"` (no bare global attach)
- [ ] Existing AI spans inspected before `init()`; usable OTel/LangSmith path preserved (no second mechanism)
- [ ] One real turn validated when runnable (`traces.get` with matching `environment`, or followup if hosted/no MCP): stable user/session, tool children, tokens, no `is_langchain_root` I/O overwrite
- [ ] Framework gotchas applied when relevant (LiveKit URL, Vercel `experimental_telemetry`, OpenInference pin, flush vs shutdown, HTTP/protobuf only) — `reference/frameworks.md`
- [ ] Streaming paths end the generation after stream drain; serverless uses `flush` (not `shutdown`)
- [ ] One integration per call site (no double instrumentation); spans close on error via `try/finally`
- [ ] No custom tracing wrapper module; `init()`/`workflow()` called directly (init is thread-safe)
- [ ] Trace enriched via `workflow()` args (`conversation_id`/`tags`/`metadata`); no metadata-builder misuse as trace setter
- [ ] Tests import the real SDK and assert disabled-mode fail-open (no whole-SDK mocking)
- [ ] SDK import at module top level; fail-open flush/shutdown calls carry inline comment
- [ ] Scheduled/batch roots carry batch tagging (literal `scheduled` **or** dynamic `channel` in tags plus `metadata["channel"]=channel` with default `channel="scheduled"`) and `workflow:<key>` tags plus a real stable run identifier in metadata (`job_id`/`run_id` or a domain-native `schedule_id`, `task_id`, `execution_id`, `event_slug`, `sce_id`)
- [ ] No absent-identity coercion to `"None"`; pass nullable identity to SDK 0.3.3+ (no coercion wrappers)
- [ ] No reads of private SDK internals (`handle._span`, any `_`-prefixed attribute); trace/span ids come from `trace_id`/`observation_id`/`current_trace_*` (TS: `traceId` / `currentTraceId()` / `resumeTrace()`)
- [ ] Every root observation actually receives children: work crossing a thread, queue, or worker boundary carries context via `resume_trace()`/`resumeTrace()`, or `langchain_callbacks(trace_context=...)` for Python LangChain invokes that have no LangChain parent (a root with zero observations is an empty trace)
- [ ] Nested LangChain/LangGraph invokes reuse the parent `CallbackManager` (`config["callbacks"]`); they do not rebuild a fresh handler list. Repair/retry LLM calls receive the same `config=`
- [ ] Root `set_output` / `end(output=...)` carries the full user-visible deliverable (same value as send/persist path); never a preview/slice/length/status receipt; status fields in metadata
