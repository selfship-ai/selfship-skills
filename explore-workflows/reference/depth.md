---
name: explore-workflow-depth
description: >-
  Deep-dive a single agentic workflow: every file, tool, code path, upstream and
  downstream contract, cost signals, user/context mapping, feedback loops, and a
  tech-heavy architecture narrative. Produces a rich inventory + mermaid diagram.
  Use for the exploration_depth job (read-only, no PR).
disable-model-invocation: true
---

# Explore Workflow Depth

Read-only analysis of **one** workflow. **Do not modify any files.**

You receive a shallow seed inventory from the breadth pass. Your job is to deepen it until the inventory is trustworthy for instrumentation and product understanding.

## Must cover (be exhaustive — never stop at the first few)

1. **Files** — every source file the workflow touches (entry points, tools, prompts, schemas, callers, callees). Chase imports and use grep to find all references.
2. **Tools** — complete list with where defined, when/why invoked, and arguments.
3. **Paths** — all distinct branches: happy path plus every sad path (errors, retries, fallbacks, timeouts, rate limits, empty/invalid input, human review, early returns).
4. **All upstreams** — every trigger (HTTP routes, queues, cron, CLI, UI, direct user/agent invocation, other workflows). Do not stop at one. The entrypoint that triggers the workflow always counts, so `upstreams` is never empty.
5. **All downstreams** — every effect (the response returned to the caller/user, DB writes, external APIs, emails, files, events, other agents/workflows). The produced result always counts, so `downstreams` is never empty.
6. **Cost** — models, `max_tokens`/token limits, pricing constants, and per-call/per-run estimates where derivable.
7. **User + context** — session/org/user identity, memory, RAG/context sources, conversation state, and how each is threaded in.
8. **Feedback loops** — evals, scores, retries, human review, self-critique, guardrails.
9. **Mermaid** — detailed flowchart connecting ALL upstreams → loop → tools → ALL downstreams, including branch/error paths. Must render as `flowchart` or `graph`.
10. **Architecture** — required top-level tech-heavy narrative covering components, all paths (happy + sad), tools, upstream/downstream wiring, and key contracts. Distinct from `description` (what) and `product_objective` (why).
11. **Entry callers** — shared entry symbols and every caller with its `workflow_key`.
12. **Shared helpers** — multi-caller LLM builders / callback factories with an `instrumentation_rule`.
13. **Concurrency** — `async` | `sync` | `worker_thread` | `mixed`; `thread_boundaries` required for worker/mixed.
14. **Observability stack** — does the app already run OpenTelemetry or another tracer (Sentry, Datadog, Logfire, `opentelemetry-sdk`, OTel auto-instrumentation)? Grep for `set_tracer_provider`, `NodeSDK`, `register(`, `sentry_sdk.init`, `TracerProvider`. Record `existing_otel`, `tools`, `global_provider_owner`, and `recommended_mode` (`isolated` when another tool owns the global provider or auto-instrumentation would flood non-LLM spans; `shared_filtered` when we can add a filtered processor; `clean` when no telemetry exists).
15. **Process lifetime** — `request` | `long_lived` | `serverless` | `batch`. Serverless/batch drive flush-before-exit (not `shutdown`).
16. **Response streaming** — `true` when the path streams LLM output (SSE, `StreamingResponse`, async generator, `stream=True`). The generation must end **after** the stream drains.
17. **Client-side paths** — from the files you already explored (`files`, `tools[].defined_in`, `entry_callers[].file`, `shared_helpers[].file`), list every path that runs in a browser/client bundle (React/Next `"use client"` components, client-only providers, anything shipped to the browser). Do not invent new paths. Use `[]` when none are client-side.

## Quality gates (checker will reject otherwise)

- Mermaid must compile/render (`flowchart` or `graph`).
- Every path in `files` must exist in the checked-out repo.
- `architecture` must be present and substantive for a complete depth result.
- `entry_callers`, `shared_helpers` arrays present (may be `[]`); `concurrency` present.
- `observability_stack` (with `existing_otel` bool + `recommended_mode`), `process_lifetime`, and `response_streaming` (bool) present.
- `client_side_paths` present in new depth output (normally `[]` for pure-backend workflows); every entry must be a path already listed in this workflow's explored surfaces and running in a browser/client bundle.

## Verify & assert upstreams and downstreams — contract AND logic

For **every** boundary (upstream, downstream, tool):

- **Contract**: capture the exact inputs/outputs, shared types, and payload shapes.
- **Logic**: confirm the caller actually supplies what the workflow requires, and that each consumer handles *every* output (including error/empty results).
- Set `verified: true` only when confirmed in code; otherwise `verified: false` with a reason.
- Record any mismatch, unhandled case, missing validation, or likely bug in `depth_detail.contract_findings`. **Report only — never fix (read-only).**

## Verification rules

- Prefer reading code over guessing. Do not invent tools, paths, or edges you did not find.
- Stay scoped to the single seed `workflow_key`. If it no longer exists in code, return `"confidence": "none"`.
- Never omit required arrays. `upstreams` and `downstreams` are never empty; use `[]` only for `tools`, `cost_estimates`, `feedback_loops`, `contract_findings`, `entry_callers`, or `shared_helpers`, and only after verifying the category is absent. `concurrency` is always required.

## Output

Return JSON per `task-explore-depth.md`. Keep `workflow_key` identical to the seed.

## Checklist

- [ ] All seed files read; transitive files chased via imports/grep.
- [ ] Tools exhaustive for this workflow.
- [ ] Paths cover happy + all sad paths.
- [ ] ALL upstreams and ALL downstreams listed (not just the obvious ones).
- [ ] Each boundary verified for contract AND logic; mismatches recorded in `contract_findings`.
- [ ] Cost estimates / user-context / feedback loops captured when present.
- [ ] Rich `flow_mermaid` including branch/error paths; mermaid renders; listed files exist.
- [ ] `architecture` present, tech-heavy, covers all paths/tools/wiring/contracts.
- [ ] `entry_callers` / `shared_helpers` / `concurrency` filled (empty arrays / simple model OK).
- [ ] `observability_stack` / `process_lifetime` / `response_streaming` recorded (grep for existing tracer setup, not guessed).
- [ ] `client_side_paths` recorded from already-explored surfaces (`[]` for pure-backend workflows).
- [ ] No file modifications. Scoped to the single workflow_key.
