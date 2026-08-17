## Python mechanics (`selfship-ai>=0.3.9,<0.4.0`)

SDK version floor: `selfship-ai>=0.3.9,<0.4.0` (0.3.9 guarantees a non-empty `status_message` on ERROR via `observe` / `langchain_callbacks`; 0.3.8 pins the trace I/O mirror to the workflow span and re-asserts at exit).

```python
import selfship

selfship.init(provider_mode="standalone")  # or shared with explicit tracer_provider

with selfship.workflow(
    name="handler",
    user_id=user_id or None,
    conversation_id=conversation_id or None,   # → trace session_id
    tags=[f"workflow:{workflow_key}"],          # trace-level tags, set directly
    metadata={"workflow_key": workflow_key},
) as handle:
    # Prefer langchain_callbacks() over raw Langfuse CallbackHandler construction.
    # Inside a running LangChain/LangGraph node, reuse parent_config["callbacks"]
    # instead of a fresh list — even with trace_context= (see hard rules 18–19).
    result = run_agent(...)
    if result.get("error"):
        # Soft failure that returns normally: auto set_error only fires on exceptions.
        handle.set_error(result["error"])
        return result
    handle.set_output(result)  # full user-visible deliverable — never a preview/receipt
    # Mid-run enrichment: handle.update(metadata=...)
```

- `workflow()` enriches the root trace in one call — session (`conversation_id`), `tags`, `metadata`, `user_id`. Do **not** call `trace_metadata()`/`build_trace_metadata()` expecting a trace to change (they only build a dict), and do **not** build a wrapper module.
- **Handle surface:** `set_output(output)`, `update(metadata=..., output=...)`, `set_error(error)`, `trace_id`, `observation_id`, `trace_context()`. Use `set_error` for soft failures (status-field errors that return normally) — auto-`set_error` only fires on exceptions that escape the `workflow()` body. Ties to hard rule 11 (record errors, don't leak spans). Across a thread/queue/worker boundary: `current_trace_context()` / `resume_trace(ctx)`. For a LangChain invoke with no LangChain parent: `langchain_callbacks(trace_context=wf.trace_context())`.
- Prefer `workflow()` (fail-open) over low-level `begin`/`end`
- Nested paths: SDK `observe` / Langfuse public observation APIs after SelfShip init
- **Context managers:** `workflow()` and `resume_trace()` — never `with selfship.observe|begin|track` (`observe` is `@selfship.observe` decorator only)
- Tools/model calls: prefer framework integrations after `init`; do not construct `Langfuse(...)` in app code
- Flat paths: `begin`/`end` or `track` only as compatibility wrappers

### Framework → integration mechanism

Pick the best mechanism per workflow's `framework`:

| Framework family | Preferred mechanism |
|---|---|
| LangChain / LangGraph | If LangSmith OTel (or other `gen_ai`/`llm` spans) already form a usable turn: preserve and route to SelfShip (`reference/frameworks.md`). Do not also attach `langchain_callbacks()`. Greenfield: `selfship.langchain_callbacks()` after `selfship.init()`, invoked **inside** `workflow()`. Nested invokes (judge, repair, retry) reuse the parent `CallbackManager` (`config["callbacks"]`) — never a fresh handler list. Cross-boundary chains with no LangChain parent: `langchain_callbacks(trace_context=wf.trace_context())`. |
| OpenAI | `langfuse.openai` wrapper |
| Unknown / raw HTTP | `workflow()` / `@observe()` after `init`. Do not call `update_current_trace()` (removed in Langfuse v4) or invent `client.trace()`. |

**OTel fallback (no SDK):**
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```
